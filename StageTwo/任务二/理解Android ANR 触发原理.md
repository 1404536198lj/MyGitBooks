# 第一章 理解Android ANR 触发原理
## 1.1 概述
&emsp;&emsp;ANR（Application not responding）是指程序未响应，Android系统中的一些事件需要在`规定时间内`完成，如果超过预定时间未能得到`有效响应`或者`响应时间过长`，都会造成ANR。发生ANR时，往往会弹出一个提示狂，告知用户当前程序未响应，用户可以选择继续等待或者强制关闭。
<br>&emsp;&emsp;以下场景会造成ANR。（需要扩充）
1. Service Timeout：前台服务在20秒内未完成。
2. BroadcastQueue Timeout：前台广播在10秒内未完成。
3. ContentProvider Timeout：内容提供者在publish超时10秒。
4. InputDispatching Timeout：输入事件分发超时5秒，包括按键和触摸事件。

&emsp;&emsp;触发ANR的过程可分为三个步骤：埋炸弹、拆炸弹、引炸弹。

## 1.2 Service
&emsp;&emsp;Service Timeout 是位于“ActivityManager”线程中的AMS.MainHandler收到`SERVICE_TIMEOUT_MSG`消息时触发。
<br>&emsp;&emsp;Service超时有两种情况：
1. 前台服务，超时阈值SERVICE_TIMEOUT为20秒；
2. 后台服务，超时阈值SERVICE_BACKGROUND_TIMEOUT为200秒；
由变量ProcessRecord.execServicesFg来决定是否前台启动。

### 1.2.1 埋炸弹
&emsp;&emsp;以下是Service的启动流程：
1. Process A进程：是指调用startService命令所在的进程，也就是启动服务的发起端进程，比如点击桌面App图标，此处Process A便是Launcher所在进程。
2. system_server进程：系统进程，是java framework框架的核心载体，里面运行了大量的系统服务，比如这里提供ApplicationThreadProxy（简称ATP），ActivityManagerService（简称AMS），这个两个服务都运行在system_server进程的不同线程中，由于ATP和AMS都是基于IBinder接口，都是binder线程，binder线程的创建与销毁都是由binder驱动来决定的，每个进程binder线程个数的上限为16。
3. Zygote进程：是由init进程孵化而来的，用于创建Java层进程的母体，所有的Java层进程都是由Zygote进程孵化而来；
4. Remote Service进程：远程服务所在进程，是由Zygote进程孵化而来的用于运行Remote服务的进程。主线程主要负责Activity/Service等组件的生命周期以及UI相关操作都运行在这个线程； 另外，每个App进程中至少会有两个binder线程 ApplicationThread(简称AT)和ActivityManagerProxy（简称AMP），当然还有其他线程，这里不是重点就不提了。

![](../imgs/ANR与JE.jpg)

&emsp;&emsp;图中涉及3种IPC通信方式：Binder、Socket以及Handler，在图中分别用3种不同的颜色来代表这3种通信方式。一般来说，同一进程内的线程间通信采用的是 Handler消息队列机制，不同进程间的通信采用的是binder机制，另外与Zygote进程通信采用的Socket。

<br>&emsp;&emsp;启动流程：
1. Process A进程采用Binder IPC向system_server进程发起startService请求；
2. system_server进程接收到请求后，向zygote进程发送创建进程的请求；
3. zygote进程fork出新的子进程Remote Service进程；
4. Remote Service进程，通过Binder IPC向sytem_server进程发起attachApplication请求；
5. system_server进程在收到请求后，进行一系列准备工作后，再通过binder IPC向remote Service进程发送scheduleCreateService请求；
6. Remote Service进程的binder线程在收到请求后，通过handler向主线程发送CREATE_SERVICE消息；
7. 主线程在收到Message后，通过发射机制创建目标Service，并回调Service.onCreate()方法。

&emsp;&emsp;到此，服务便正式启动完成。当创建的是本地服务或者服务所属进程已创建时，则无需经过上述步骤2、3，直接创建服务即可。

<br>&emsp;&emsp;Service在启动过程中，在attach到system_server进程的过程中会调用`realStartServiceLocked()`方法来埋下炸弹。
[ActiveServices.java：：realStartServiceLocked]
```java
private final void realStartServiceLocked(ServiceRecord r,
        ProcessRecord app, boolean execInFg) throws RemoteException {
    ...
    //发送delay消息(SERVICE_TIMEOUT_MSG)，【见下】
    bumpServiceExecutingLocked(r, execInFg, "create");
    try {
        ...
        //最终执行服务的onCreate()方法
        app.thread.scheduleCreateService(r, r.serviceInfo,
                mAm.compatibilityInfoForPackageLocked(r.serviceInfo.applicationInfo),
                app.repProcState);
    } catch (DeadObjectException e) {
        mAm.appDiedLocked(app);
        throw e;
    } finally {
        ...
    }
}
```
[AS.bumpServiceExecutingLocked]
&emsp;&emsp;该方法的主要工作是发送delay消息`SERVICE_TIMEOUT_MSG`。炸弹已埋下，我们并不希望引爆炸弹，那么，就需要在引爆之前拆炸弹。
```java
private final void bumpServiceExecutingLocked(ServiceRecord r, boolean fg, String why) {
    ... 
    scheduleServiceTimeoutLocked(r.app);
}

void scheduleServiceTimeoutLocked(ProcessRecord proc) {
    if (proc.executingServices.size() == 0 || proc.thread == null) {
        return;
    }
    long now = SystemClock.uptimeMillis();
    Message msg = mAm.mHandler.obtainMessage(
            ActivityManagerService.SERVICE_TIMEOUT_MSG);
    msg.obj = proc;
    
    //当超时后仍没有remove该SERVICE_TIMEOUT_MSG消息，则执行service Timeout流程【见2.3.1】
    mAm.mHandler.sendMessageAtTime(msg,
        proc.execServicesFg ? (now+SERVICE_TIMEOUT) : (now+ SERVICE_BACKGROUND_TIMEOUT));
}
```
### 1.2.2 拆炸弹
&emsp;&emsp;在system_server进程AS.realStartServiceLocked()调用的过程会埋下一颗炸弹, 超时没有启动完成则会爆炸. 那么什么时候会拆除这颗炸弹的引线呢? 经过Binder等层层调用进入目标进程的主线程handleCreateService()的过程。
[ActivityThread.java：：handleCreateService（）]
```java
    private void handleCreateService(CreateServiceData data) {
        ...
        java.lang.ClassLoader cl = packageInfo.getClassLoader();
        Service service = (Service) cl.loadClass(data.info.name).newInstance();
        ...

        try {
            //创建ContextImpl对象
            ContextImpl context = ContextImpl.createAppContext(this, packageInfo);
            context.setOuterContext(service);
            //创建Application对象
            Application app = packageInfo.makeApplication(false, mInstrumentation);
            service.attach(context, this, data.info.name, data.token, app,
                    ActivityManagerNative.getDefault());
            //调用服务onCreate()方法 
            service.onCreate();
            
            //拆除炸弹引线[见下]
            ActivityManagerNative.getDefault().serviceDoneExecuting(
                    data.token, SERVICE_DONE_EXECUTING_ANON, 0, 0);
        } catch (Exception e) {
            ...
        }
    }
```
&emsp;&emsp;这个过程会创建目标service对象，并回调onCreate()方法，紧接再次经过多次调用回到system_server来执行serviceDoneExecuting。
<br>[AS.serviceDoneExecutingLocked]
```java
private void serviceDoneExecutingLocked(ServiceRecord r, boolean inDestroying,
            boolean finishing) {
    ...
    if (r.executeNesting <= 0) {
        if (r.app != null) {
            r.app.execServicesFg = false;
            r.app.executingServices.remove(r);
            if (r.app.executingServices.size() == 0) {
                //当前服务所在进程中没有正在执行的service
                mAm.mHandler.removeMessages(ActivityManagerService.SERVICE_TIMEOUT_MSG, r.app);
        ...
    }
    ...
}
```
该方法的主要工作是，当Service启动完成，则移除服务超时消息SERVICE_TIMEOUT_MSG。

### 1.2.3 引爆炸弹
&emsp;&emsp;上述拆除炸弹是在未超时的情况下，在超时的时候需要去引爆炸弹。在system_server中有一个线程是ActivityManager，当发生了超时就会向该线程发送SERVICE_TIMEOUT_MSG消息。
<br>[ActivityManagerService.java ::MainHandler]
```java
final class MainHandler extends Handler {
    public void handleMessage(Message msg) {
        switch (msg.what) {
            case SERVICE_TIMEOUT_MSG: {
                ...
                //【见下】
                mServices.serviceTimeout((ProcessRecord)msg.obj);
            } break;
            ...
        }
        ...
    }
}
```
[AMS.serviceTimeout]
```java
void serviceTimeout(ProcessRecord proc) {
    String anrMessage = null;

    synchronized(mAm) {
        if (proc.executingServices.size() == 0 || proc.thread == null) {
            return;
        }
        final long now = SystemClock.uptimeMillis();
        final long maxTime =  now -
                (proc.execServicesFg ? SERVICE_TIMEOUT : SERVICE_BACKGROUND_TIMEOUT);
        ServiceRecord timeout = null;
        long nextTime = 0;
        for (int i=proc.executingServices.size()-1; i>=0; i--) {
            ServiceRecord sr = proc.executingServices.valueAt(i);
            if (sr.executingStart < maxTime) {
                timeout = sr;
                break;
            }
            if (sr.executingStart > nextTime) {
                nextTime = sr.executingStart;
            }
        }
        if (timeout != null && mAm.mLruProcesses.contains(proc)) {
            Slog.w(TAG, "Timeout executing service: " + timeout);
            StringWriter sw = new StringWriter();
            PrintWriter pw = new FastPrintWriter(sw, false, 1024);
            pw.println(timeout);
            timeout.dump(pw, "    ");
            pw.close();
            mLastAnrDump = sw.toString();
            mAm.mHandler.removeCallbacks(mLastAnrDumpClearer);
            mAm.mHandler.postDelayed(mLastAnrDumpClearer, LAST_ANR_LIFETIME_DURATION_MSECS);
            anrMessage = "executing service " + timeout.shortName;
        }
    }

    if (anrMessage != null) {
        //当存在timeout的service，则执行appNotResponding
        mAm.appNotResponding(proc, null, null, false, anrMessage);
    }
}
```
其中anrMessage的内容为”executing service [发送超时serviceRecord信息]”;
### 1.2.3 小结
&emsp;&emsp;在service的启动过程中，目标进程attach到system_server时，system_server中会调用AMS.realStartServicesLocked()方法给AMS.MainHandler发送delay消息SERVICES_TIMEOUT_MSG，当service在规定时间内完成启动，会回到system_server中调用AMS.serviceDoneExecutingLocked()来移除SERVICES_TIMEOUT_MSG消息。如果倒计时结束前未能及时移除SERVICES_TIMEOUT_MSG，AMS会收到SERVICES_TIMEOUT_MSG消息，会调用AMS.serviceTimeout（）方法报告ANR（首先判断是否超时）。

## 1.3 BroadcastReceiver 
&emsp;&emsp;BroadcastReceiver Timeout是位于”ActivityManager”线程中的`BroadcastQueue.BoradcastHandler`收到`BROADCAST_TIMEOUT_MSG`时消息时触发。
<br>广播超时有两种情况：
1. 前台广播，超时阈值BROADCAST_FG_TIMEOUT为10s。
2. 后台广播，超时阈值BROADCAST_BG_TIMEOUT为60s。

BroadcastReceiver根据注册方式可分为两类：
1. 静态广播接收者：通过AndroidManifest.xml的标签来申明的BroadcastReceiver;
2. 动态广播接收者：通过AMS.registerReceiver()方式注册的BroadcastReceiver, 不需要时记得调用unregisterReceiver();

BroadcastReceiver根据广播发送方式可分为三类：
1. 普通广播（并行）:sendBroadcast()
2. 有序广播（串行）:sendOrderedBroadcast()
3. Sticky广播:sendStickyBroadcast()

&emsp;&emsp;`只有串行广播才需要考虑超时`，因为接收者是有序处理的，前一个receiver处理慢，会影响后一个receiver；并行广播通过一个循环一次性向所有的receiver分发广播事件，所以不存在彼此影响的问题，则没有广播超时；
>&emsp;&emsp;Tip: 静态注册的receivers始终采用串行方式来处理（processNextBroadcast）； 动态注册的registeredReceivers处理广播的方式是串行还是并行方式, 取决于广播的发送方式(processNextBroadcast)是串行还是并行。

<br>&emsp;&emsp;`有序广播超时情况1`：某个广播总处理时间 > 2* receiver总个数 * mTimeoutPeriod, 其中mTimeoutPeriod，前台队列默认为10s，后台队列默认为60s;
<br>&emsp;&emsp;`有序广播超时情况2`：某个receiver的执行时间超过mTimeoutPeriod；

<br>&emsp;&emsp;`sendBroadcast()`这个方法的广播是能够发送给所有广播接收者，按照注册的先后顺序，如果你这个时候设置了广播接收者的优先级，优先级如果恰好与注册顺序相同，则不会有任何问题，如果顺序不一样，会出leaked IntentReceiver 这样的异常，并且在前面的广播接收者不能调用abortBroadcast()方法将其终止，如果调用会出BroadcastReceiver trying to return result during a non-ordered broadcast的异常，当然，先接收到广播的receiver可以修改广播数据。

<br>&emsp;&emsp;`sendOrderedBroadcast()`方法顾名思义就是priority的属性能起作用，并且在队列前面的receiver可以随时终止广播的发送。还有这个api能指定final的receiver，这个receiver是最后一个接收广播时间的receiver，并且一定会接收到广播事件，是不能被前面的receiver拦截的。实际做实验的情况是这样的，假设我有3个receiver依序排列，并且sendOrderedBroadcast()方法指定了一个finalReceiver，那么intent传递给这4个Receiver的顺序为Receiver1-->finalReceiver-->Receiver2-->finalReceiver-->Receiver3-->finalReceiver。这个特性可以用来统计系统中能监听某种广播的Receiver的数目。

<br>&emsp;&emsp;`sendStickyBroadcast()`字面意思是发送粘性的广播，使用这个api需要权限android.Manifest.permission.BROADCAST_STICKY,粘性广播的特点是Intent会一直保留到广播事件结束，而这种广播也没有所谓的10秒限制，10秒限制是指普通的广播如果onReceive方法执行时间太长，超过10秒的时候系统会将这个广播置为可以干掉的candidate，一旦系统资源不够的时候，就会干掉这个广播而让它不执行。

### 1.3.1 埋炸弹
&emsp;&emsp;广播启动流程中通过调用 processNextBroadcast来处理广播。其流程为先处理并行广播,再处理当前有序广播,最后获取并处理下条有序广播。
<br>[BroadcastQueue.java::processNextBroadcast()]
```java
    final void processNextBroadcast(boolean fromMsg) {
    synchronized(mService) {
        //part1: 处理并行广播
        while (mParallelBroadcasts.size() > 0) {
            r = mParallelBroadcasts.remove(0);
            r.dispatchTime = SystemClock.uptimeMillis();
            r.dispatchClockTime = System.currentTimeMillis();
            final int N = r.receivers.size();
            for (int i=0; i<N; i++) {
                Object target = r.receivers.get(i);
                //分发广播给已注册的receiver 【】
                deliverToRegisteredReceiverLocked(r, (BroadcastFilter)target, false);
            }
            addBroadcastToHistoryLocked(r);//将广播添加历史统计
        }

        //part2: 处理当前有序广播
        do {
            if (mOrderedBroadcasts.size() == 0) {
                mService.scheduleAppGcsLocked(); //没有更多的广播等待处理
                if (looped) {
                    mService.updateOomAdjLocked();
                }
                return;
            }
            r = mOrderedBroadcasts.get(0); //获取串行广播的第一个广播
            boolean forceReceive = false;
            int numReceivers = (r.receivers != null) ? r.receivers.size() : 0;
            if (mService.mProcessesReady && r.dispatchTime > 0) {
                long now = SystemClock.uptimeMillis();
                if ((numReceivers > 0) && (now > r.dispatchTime + (2*mTimeoutPeriod*numReceivers))) {
                    broadcastTimeoutLocked(false); //当广播处理时间超时，则强制结束这条广播
                }
            }
            ...
            if (r.receivers == null || r.nextReceiver >= numReceivers
                    || r.resultAbort || forceReceive) {
                if (r.resultTo != null) {
                    //处理广播消息消息，调用到onReceive()
                    performReceiveLocked(r.callerApp, r.resultTo,
                        new Intent(r.intent), r.resultCode,
                        r.resultData, r.resultExtras, false, false, r.userId);
                }

                cancelBroadcastTimeoutLocked(); //取消BROADCAST_TIMEOUT_MSG消息
                addBroadcastToHistoryLocked(r);
                mOrderedBroadcasts.remove(0);
                continue;
            }
        } while (r == null);

        //part3: 获取下一个receiver
        r.receiverTime = SystemClock.uptimeMillis();
        if (recIdx == 0) {
            r.dispatchTime = r.receiverTime;
            r.dispatchClockTime = System.currentTimeMillis();
        }
        if (!mPendingBroadcastTimeoutMessage) {
            long timeoutTime = r.receiverTime + mTimeoutPeriod;
            setBroadcastTimeoutLocked(timeoutTime); //设置广播超时延时消息
        }

        //part4: 处理下条有序广播
        ProcessRecord app = mService.getProcessRecordLocked(targetProcess,
                info.activityInfo.applicationInfo.uid, false);
        if (app != null && app.thread != null) {
            app.addPackage(info.activityInfo.packageName,
                    info.activityInfo.applicationInfo.versionCode, mService.mProcessStats);
            processCurBroadcastLocked(r, app); //[处理串行广播]
            return;
            ...
        }

        //该receiver所对应的进程尚未启动，则创建该进程
        if ((r.curApp=mService.startProcessLocked(targetProcess,
                info.activityInfo.applicationInfo, true,
                r.intent.getFlags() | Intent.FLAG_FROM_BACKGROUND,
                "broadcast", r.curComponent,
                (r.intent.getFlags()&Intent.FLAG_RECEIVER_BOOT_UPGRADE) != 0, false, false))
                        == null) {
            ...
            return;
        }
    }
}
```
对于广播超时处理时机：
1. 首先在part3的过程中setBroadcastTimeoutLocked(timeoutTime) 设置超时广播消息；
2. 然后在part2根据广播处理情况来处理：
>当广播接收者等待时间过长，则调用broadcastTimeoutLocked(false);
<br>当执行完广播,则调用cancelBroadcastTimeoutLocked;

[setBroadcastTimeoutLocked]
```java
final void setBroadcastTimeoutLocked(long timeoutTime) {
    if (! mPendingBroadcastTimeoutMessage) {
        Message msg = mHandler.obtainMessage(BROADCAST_TIMEOUT_MSG, this);
        mHandler.sendMessageAtTime(msg, timeoutTime);
        mPendingBroadcastTimeoutMessage = true;
    }
}
```
&emsp;&emsp;设置delay消息BROADCAST_TIMEOUT_MSG，当前广播还没处理完毕，则BroadcastQueue.BrocastHandler将收到超时消息。

### 1.3.2 拆炸弹
在processNextBroadcast()过程, 执行完performReceiveLocked,便会通过cancelBroadCastLocked（）来拆炸弹.
```java
final void cancelBroadcastTimeoutLocked() {
    if (mPendingBroadcastTimeoutMessage) {
        mHandler.removeMessages(BROADCAST_TIMEOUT_MSG, this);
        mPendingBroadcastTimeoutMessage = false;
    }
}
```
移除广播接收者超时消息BROADCAST_TIMEOUT_MSG。

### 1.3.3 引爆炸弹
[BroadcastQueue.java ::BroadcastHandlerhandleMessage]
```java
private final class BroadcastHandler extends Handler {
    public void handleMessage(Message msg) {
        switch (msg.what) {
            case BROADCAST_TIMEOUT_MSG: {
                synchronized (mService) {
                    //【见小节】
                    broadcastTimeoutLocked(true);
                }
            } break;
            ...
        }
        ...
    }
}
```
[BroadcastRecord.java：：broadcastTimeoutLocked（）]
```java
//fromMsg = true
final void broadcastTimeoutLocked(boolean fromMsg) {
    if (fromMsg) {
        mPendingBroadcastTimeoutMessage = false;
    }

    if (mOrderedBroadcasts.size() == 0) {
        return;
    }

    long now = SystemClock.uptimeMillis();
    BroadcastRecord r = mOrderedBroadcasts.get(0);
    if (fromMsg) {
        if (mService.mDidDexOpt) {
            mService.mDidDexOpt = false;
            long timeoutTime = SystemClock.uptimeMillis() + mTimeoutPeriod;
            setBroadcastTimeoutLocked(timeoutTime);
            return;
        }
        
        if (!mService.mProcessesReady) {
            return; //当系统还没有准备就绪时，广播处理流程中不存在广播超时
        }

        long timeoutTime = r.receiverTime + mTimeoutPeriod;
        if (timeoutTime > now) {
            //如果当前正在执行的receiver没有超时，则重新设置广播超时
            setBroadcastTimeoutLocked(timeoutTime);
            return;
        }
    }

    BroadcastRecord br = mOrderedBroadcasts.get(0);
    if (br.state == BroadcastRecord.WAITING_SERVICES) {
        //广播已经处理完成，但需要等待已启动service执行完成。当等待足够时间，则处理下一条广播。
        br.curComponent = null;
        br.state = BroadcastRecord.IDLE;
        processNextBroadcast(false);
        return;
    }

    r.receiverTime = now;
    //当前BroadcastRecord的anr次数执行加1操作
    r.anrCount++;

    if (r.nextReceiver <= 0) {
        return;
    }
    ...
    
    Object curReceiver = r.receivers.get(r.nextReceiver-1);
    //查询App进程
    if (curReceiver instanceof BroadcastFilter) {
        BroadcastFilter bf = (BroadcastFilter)curReceiver;
        if (bf.receiverList.pid != 0
                && bf.receiverList.pid != ActivityManagerService.MY_PID) {
            synchronized (mService.mPidsSelfLocked) {
                app = mService.mPidsSelfLocked.get(
                        bf.receiverList.pid);
            }
        }
    } else {
        app = r.curApp;
    }

    if (app != null) {
        anrMessage = "Broadcast of " + r.intent.toString();
    }

    if (mPendingBroadcast == r) {
        mPendingBroadcast = null;
    }

    //继续移动到下一个广播接收者
    finishReceiverLocked(r, r.resultCode, r.resultData,
            r.resultExtras, r.resultAbort, false);
    scheduleBroadcastsLocked();

    if (anrMessage != null) {
        // 进入ANR处理流程
        mHandler.post(new AppNotResponding(app, anrMessage));
    }
}
```
1. mOrderedBroadcasts已处理完成，则不会anr;
2. 正在执行dexopt，则不会anr;
3. 系统还没有进入ready状态(mProcessesReady=false)，则不会anr;
4. 如果当前正在执行的receiver没有超时，则重新设置广播超时，不会anr;

### 1.3.4 小结

串行广播超时有两种情况：
1. 某个广播总处理时间 > 2* receiver总个数 * mTimeoutPeriod, 其中mTimeoutPeriod，前台队列默认为10s，后台队列默认为60s;
2. 某个receiver的执行时间超过mTimeoutPeriod；

&emsp;&emsp;BroadcastReceiver的埋炸弹、拆炸弹、引爆炸弹都在processNextBroadcast（）中完成，超时只监测串行处理的广播。埋炸弹通过setBroadcastTimeoutLocked（）方法将一个delay消息发送给BroadcastQueue.BrocastHandler。当串行广播未超时（两种情况都未超时），则移除delay消息，否则调用broadcastTimeoutLocked（）来处理超时。

## 1.4 ContentProvider
&emsp;&emsp;ContentProvider Timeout是位于”ActivityManager”线程中的AMS.MainHandler收到CONTENT_PROVIDER_PUBLISH_TIMEOUT_MSG消息时触发。
<br>&emsp;&emsp;ContentProvider 超时为CONTENT_PROVIDER_PUBLISH_TIMEOUT = 10s. 这个跟前面的Service和BroadcastQueue完全不同, 由Provider进程启动过程相关。

### 1.4.1 埋炸弹
&emsp;&emsp;埋炸弹的过程其实是在进程创建后会调用attachApplicationLocked()进入system_server进程。
<br>[AMS.attachApplicationLocked]
```java
private final boolean attachApplicationLocked(IApplicationThread thread,
        int pid) {
    ProcessRecord app;
    if (pid != MY_PID && pid >= 0) {
        synchronized (mPidsSelfLocked) {
            app = mPidsSelfLocked.get(pid); // 根据pid获取ProcessRecord
        }
    } 
    ...
    
    //系统处于ready状态或者该app为FLAG_PERSISTENT进程则为true
    boolean normalMode = mProcessesReady || isAllowedWhileBooting(app.info);
    List<ProviderInfo> providers = normalMode ? generateApplicationProvidersLocked(app) : null;

    //app进程存在正在启动中的provider,则超时10s后发送CONTENT_PROVIDER_PUBLISH_TIMEOUT_MSG消息
    if (providers != null && checkAppInLaunchingProvidersLocked(app)) {
        Message msg = mHandler.obtainMessage(CONTENT_PROVIDER_PUBLISH_TIMEOUT_MSG);
        msg.obj = app;
        mHandler.sendMessageDelayed(msg, CONTENT_PROVIDER_PUBLISH_TIMEOUT);
    }
    
    thread.bindApplication(...);
    ...
}
```
&emsp;&emsp;AMS的attachApplicationLocked（）方法向AMS.mainHandler发送一个delay消息CONTENT_PROVIDER_PUBLISH_TIMEOUT。

### 1.4.2 拆炸弹
&emsp;&emsp;当provider成功publish之后,便会拆除该炸弹。
<br>[AMS.publishContentProviders]
```java
public final void publishContentProviders(IApplicationThread caller,
       List<ContentProviderHolder> providers) {
   ...
   
   synchronized (this) {
       final ProcessRecord r = getRecordForAppLocked(caller);
       
       final int N = providers.size();
       for (int i = 0; i < N; i++) {
           ContentProviderHolder src = providers.get(i);
           ...
           ContentProviderRecord dst = r.pubProviders.get(src.info.name);
           if (dst != null) {
               ComponentName comp = new ComponentName(dst.info.packageName, dst.info.name);
               
               mProviderMap.putProviderByClass(comp, dst); //将该provider添加到mProviderMap
               String names[] = dst.info.authority.split(";");
               for (int j = 0; j < names.length; j++) {
                   mProviderMap.putProviderByName(names[j], dst);
               }

               int launchingCount = mLaunchingProviders.size();
               int j;
               boolean wasInLaunchingProviders = false;
               for (j = 0; j < launchingCount; j++) {
                   if (mLaunchingProviders.get(j) == dst) {
                       //将该provider移除mLaunchingProviders队列
                       mLaunchingProviders.remove(j);
                       wasInLaunchingProviders = true;
                       j--;
                       launchingCount--;
                   }
               }
               //成功pubish则移除该消息
               if (wasInLaunchingProviders) {
                   mHandler.removeMessages(CONTENT_PROVIDER_PUBLISH_TIMEOUT_MSG, r);
               }
               synchronized (dst) {
                   dst.provider = src.provider;
                   dst.proc = r;
                   //唤醒客户端的wait等待方法
                   dst.notifyAll();
               }
               ...
           }
       }
   }    
}
```
### 1.4.3 引爆炸弹
&emsp;&emsp;当AMS.PublishContentProvider（）在delay消息CONTENT_PROVIDER_TIMEOUT_MSG发送前未成功移除该消息，则”ActivityManager”线程的AMS.mainHandler会收到。AMS的 handleMessage会调用相应的AMS.processContentProviderPublishTimedOutLocked方法去处理超时。
```java
private final void processContentProviderPublishTimedOutLocked(ProcessRecord app) {
    cleanupAppInLaunchingProvidersLocked(app, true); //[见]
    //[见]
    removeProcessLocked(app, false, true, "timeout publishing content providers");
}
```
&emsp;&emsp;cleanupAppInLaunchingProvidersLocked中调用了removeDyingProviderLocked来移除死亡的provider：
1. 对于stable类型的provider(即conn.stableCount > 0),则会杀掉所有跟该provider建立stable连接的非persistent进程。
2. 对于unstable类的provider(即conn.unstableCount > 0),并不会导致client进程被级联所杀。

&emsp;&emsp;使用removeProcessLocked来移除进程。

### 1.4.4 小结
&emsp;&emsp;contentprovider在进程启动后，调用AMS.attachApplicationLocked()方法向AMS.mainHandler发送一个delay消息CONTENT_PROVIDER_PUBLISH_TIMEOUT。当AMS.publishContentProviders（）方法成功执行（contentProvider成功publish）后将会移除CONTENT_PROVIDER_PUBLISH_TIMEOUT。如果在AMS.mainHandler收到超时消息前，contentProvider未成功publish，那么AMS将会调用processContentProviderPublishTimedOutLocked（）方法来处理超时。



##　1.5 总结
### 1.5.1 超时阈值
Service超时有两种情况：
1. 前台服务，超时阈值SERVICE_TIMEOUT为20秒；
2. 后台服务，超时阈值SERVICE_BACKGROUND_TIMEOUT为200秒；

<br>广播超时有两种情况：
1. 前台广播，超时阈值BROADCAST_FG_TIMEOUT为10s。
2. 后台广播，超时阈值BROADCAST_BG_TIMEOUT为60s。

ContentProvider超时：
1. publish超时阈值CONTENT_PROVIDER_PUBLISH_TIMEOUT 为10s

InputDispatching Timeout：
1. 输入事件分发超时5秒，包括按键和触摸事件。

### 1.5.2 超时监测
Service超时检测机制
1. 超过一定时间没有执行完相应操作来触发移除延时消息，则会触发anr;

BroadcastReceiver超时检测机制
1. 有序广播的总执行时间超过 2* receiver个数 * timeout时长，则会触发anr;
2. 有序广播的某一个receiver执行过程超过 timeout时长，则会触发anr;

ContentProvider超时监测机制
1. publish contentProvider超时执行，未来得及撤销delay msg，触发ANR。

&emsp;&emsp;对于Service, Broadcast, Input发生ANR之后,最终都会调用AMS.appNotResponding;对于provider,在其进程启动时publish过程可能会出现ANR, 则会直接杀进程以及清理相应信息,而不会弹出ANR的对话框。
1.5 总结
1.5.1 超时阈值
Service超时有两种情况：
1. 前台服务，超时阈值SERVICE_TIMEOUT为20秒；
2. 后台服务，超时阈值SERVICE_BACKGROUND_TIMEOUT为200秒；

广播超时有两种情况：
1. 前台广播，超时阈值BROADCAST_FG_TIMEOUT为10s。
2. 后台广播，超时阈值BROADCAST_BG_TIMEOUT为60s。
ContentProvider超时：
1. publish超时阈值CONTENT_PROVIDER_PUBLISH_TIMEOUT 为10s
InputDispatching Timeout：
1. 输入事件分发超时5秒，包括按键和触摸事件。
1.5.2 超时监测
Service超时检测机制
1. 超过一定时间没有执行完相应操作来触发移除延时消息，则会触发anr;
BroadcastReceiver超时检测机制
1. 有序广播的总执行时间超过 2 receiver个数 timeout时长，则会触发anr;
2. 有序广播的某一个receiver执行过程超过 timeout时长，则会触发anr;
ContentProvider超时监测机制
1. publish contentProvider超时执行，未来得及撤销delay msg，触发ANR。
  对于Service, Broadcast, Input发生ANR之后,最终都会调用AMS.appNotResponding;对于provider,在其进程启动时publish过程可能会出现ANR, 则会直接杀进程以及清理相应信息,而不会弹出ANR的对话框。

















 
 
 




