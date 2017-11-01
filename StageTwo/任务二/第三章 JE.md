# 第三章 JE crash处理流程

&emsp;&emsp;Java层Crash, 往往是抛出了一个未捕获的异常uncaughtException而导致的崩溃。那是不是把所有的异常都catch住系统就没有问题呢, 这个是要分情况的,有时候的异常强制崩溃可能会留下更大的问题, 有些异常的抛出是需要深入分析Root Cause,从根源来解决问题, 而非简单粗糙的捕获住所有的异常。
<br>&emsp;&emsp;上层应用都是由Zygote fork而来，分为system server系统进程和各种应用进程，这些进程在创建时，会设置未捕获异常的处理器，当系统抛出未捕获的异常时，最终都交给异常处理器。
>1. system_server进程启动过程中由RuntimeInit.java的commonInit方法设置UncaughtHandler来处理未捕获异常；
2. 普通应用进程，在创建过程中，同样会调用RuntimeInit.java的commonInit方法设置UncaughtHandler。

## 3.1 java crash处理流程

那么接下来以commonInit()方法为起点来展开说明。
```java
public class RuntimeInit {
    ...
    private static final void commonInit() {
        //设置默认的未捕获异常处理器，UncaughtHandler实例化过程【】
        Thread.setDefaultUncaughtExceptionHandler(new UncaughtHandler());
        ...
    }
}
```
接下来看看UncaughtHandler对象实例化过程：
```java
private static class UncaughtHandler implements Thread.UncaughtExceptionHandler {
    //覆写接口方法
    public void uncaughtException(Thread t, Throwable e) {
        try {
            //保证crash处理过程不会重入
            if (mCrashing) return;
            mCrashing = true;

            if (mApplicationObject == null) {
                //system_server进程
                Clog_e(TAG, "*** FATAL EXCEPTION IN SYSTEM PROCESS: " + t.getName(), e);
            } else {
                //普通应用进程
                StringBuilder message = new StringBuilder();
                message.append("FATAL EXCEPTION: ").append(t.getName()).append("\n");
                final String processName = ActivityThread.currentProcessName();
                if (processName != null) {
                    message.append("Process: ").append(processName).append(", ");
                }
                message.append("PID: ").append(Process.myPid());
                Clog_e(TAG, message.toString(), e);
            }

            //启动crash对话框，等待处理完成 【见】
            ActivityManagerNative.getDefault().handleApplicationCrash(
                    mApplicationObject, new ApplicationErrorReport.CrashInfo(e));
        } catch (Throwable t2) {
            ...
        } finally {
            //确保当前进程彻底杀掉【见小节11】
            Process.killProcess(Process.myPid());
            System.exit(10);
        }
    }
}
```
1. 当system进程crash的信息
<br>&emsp;&emsp;（1）开头`*** FATAL EXCEPTION IN SYSTEM PROCESS [线程名]`；
<br>&emsp;&emsp;（2）接着输出发生crash时的调用栈信息到logcat；

2. 当app进程crash时的信息：
<br>&emsp;&emsp;（1）开头`FATAL EXCEPTION: [线程名]`；
<br>&emsp;&emsp;（2）紧接着 Process: [进程名], PID: [进程id]；
<br>&emsp;&emsp;（3）最后输出发生crash时的调用栈信息到logcat。

&emsp;&emsp;要从log中搜索crash信息，只需要搜索关键词`FATAL EXCEPTION`；如果需要进一步筛选只搜索系统crash信息，则可以搜索的关键词可以有多样，比如`*** FATAL EXCEPTION`。

<br>&emsp;&emsp;注意: mApplicationObject等于null,一定不是普通的app进程. 但是除了system进程, 也有可能是shell进程, 即通过app_process + 命令参数 的方式创建的进程.

当UncaughtHandler()输出完crash信息到logcat里面，这只是crash流程的刚开始阶段。接下来
```
AMP.handleApplicationCrash
    AMS.handleApplicationCrash
        AMS.findAppProcess
        AMS.handleApplicationCrashInner
            AMS.addErrorToDropBox
            AMS.crashApplication
                AMS.makeAppCrashingLocked
                    AMS.startAppProblemLocked
                    ProcessRecord.stopFreezingAllLocked
                        ActivityRecord.stopFreezingScreenLocked
                            WMS.stopFreezingScreenLocked
                                WMS.stopFreezingDisplayLocked
                    AMS.handleAppCrashLocked
                mUiHandler.sendMessage(SHOW_ERROR_MSG)

Process.killProcess(Process.myPid());
System.exit(10);
```

1. handleApplicationCrashInner()：将Crash信息写入到Event log,将错误信息添加到DropBox,并调用AMS.crashApplication（）

2. AMS.crashApplication（）：通过AMS.makeAppCrashingLocked（）继续处理crash。

3. AMS.makeAppCrashingLocked（）：封装crash信息到crashingReport对象，并调用AMS.startAppProblemLocked（）`获取当前用户下的crash应用的error receiver并忽略当前app的广播接收`、ProcessRecord.stopFreezingAllLocked（）`停止屏幕冻结`、AMS.handleAppCrashLocked（）
（）

4. ProcessRecord.stopFreezingAllLocked（）
```
    处理屏幕旋转相关逻辑；
    移除冻屏的超时消息；
    屏幕旋转动画的相关操作;
    使能输入事件分发功能；
    display冻结时，执行gc操作；
    更新当前的屏幕方向；
    向mH发送configuraion改变的消息。
```

5. AMS.handleAppCrashLocked

<br>（1）当同一进程在1分钟之内连续两次crash，则执行的情况下：
>对于非persistent进程：
>>直接结束该应用所有activity；
<br>杀死该进程以及同一个进程组下的所有进程
<br>恢复栈顶第一个非finishing状态的activity。

>对于persistent进程:
>> 恢复栈顶第一个非finishing状态的activity。

<br>（2）否则
> 结束栈顶正在运行activity。

6.mUiHandler.sendMessage(SHOW_ERROR_MSG)发送消息SHOW_ERROR_MSG，系统会弹出提示crash的对话框，并阻塞等待用户选择是“退出”或 “退出并报告”，当用户不做任何选择时5min超时后，默认选择“退出”，当手机休眠时也默认选择“退出”。

<br>7.通过finnally语句块保证能执行并彻底杀掉Crash进程Process.killProcess(Process.myPid())

<br>8.当Crash进程被杀后，并没有完全结束，还有Binder死亡通知的流程还没有处理完成。

## 3.2 Binder死亡通知
&emsp;&emsp;在应用进程创建的过程中，`attachApplicationLocked`方法的过程中便会创建死亡通知。
<br>[ActivityManagerService.java::attachApplicationLocked()]
```java
private final boolean attachApplicationLocked(IApplicationThread thread,
            int pid) {
    try {
        //创建binder死亡通知
        AppDeathRecipient adr = new AppDeathRecipient(
                app, pid, thread);
        thread.asBinder().linkToDeath(adr, 0);
        app.deathRecipient = adr;
    } catch (RemoteException e) {
        app.resetPackageList(mProcessStats);
        startProcessLocked(app, "link fail", processName);
        return false;
    }
    ...
}
```
&emsp;&emsp;当Binder的服务端挂了之后，会通过binder的AppDeathRecipient来`通知AMS对客户端进行清理工作`。当crash的进程被kill后，会回调binderDied()方法。
```java
private final class AppDeathRecipient implements IBinder.DeathRecipient {
    public void binderDied() {
        synchronized(ActivityManagerService.this) {
            appDiedLocked(mApp, mPid, mAppThread, true);//【见】
        }
    }
}
```
&emsp;&emsp;`binderDied（）`方法调用了appDiedLocked（），如果app没有被kill，则执行kill。接着，如果应用进程没有重启过，那么通过handleAppDiedLocked（）开始执行清理：
1. 清理应用程序service, BroadcastReceiver, ContentProvider，process相关信息；
2. 清理activity相关信息；
3. 恢复栈顶第一个非finish的activity。

清理应用程序service, BroadcastReceiver, ContentProvider，process相关信息包括：

1. 清理service
<br>&emsp;&emsp;如果存在，则清除crash/anr/wait对话框，解除app的死亡通告，将app移除前台进程，清理service信息。

2. 清理ContentProvider
<br>&emsp;&emsp;获取该进程已publish的ContentProvider,ContentProvider服务端被杀，则client端进程也会被杀;处理正在启动并且是有client端正在等待的ContentProvider;取消已连接的ContentProvider的注册;

3. 清理BroadcastReceiver
<br>&emsp;&emsp;取消注册的广播接收者

4. 清理Process
<br>&emsp;&emsp;`mLaunchingProviders`:记录着存在client端等待的ContentProvider list。当进程包含这样的ContentProvider，则需要重启进程。当ContentProvider一旦发布则将该ContentProvider将从该list去除。
<br>&emsp;&emsp;`mPersistentStartingProcesses`：记录着试图在系统ready之前就启动的进程。在那时并不启动这些进程，先记录下来,等系统启动完成则启动这些进程。当进程属于这种类型也需要重启。

## 3.3 小结
当应用发生crash后，处理流程如下：
1. 进程创建之初就准备好的UncaughtHandler对象处理未捕获的异常，输出crash 信息以及调用栈到logcat。
2. 发生crash的进程通过binder远程调用system _server的AMP.handleApplicationCrash（）：输出crash 信息到eventlog，输出进程crash信息到dropbox；（2）忽略接收到的广播、停止冻屏；（3）根据是否频繁crash以及是否是persistent进程来进行activity的相关操作；（4）弹出crash对话框。
3. system_server进程执行完成，此时需要执行kill发生crash的进程；
4. 当crash进程被kill，通过binder死亡通知，来告诉AMS来执行appDiedLocked()；
5. 最后，执行清理应用相关的activity/service/ContentProvider/receiver组件信息。



