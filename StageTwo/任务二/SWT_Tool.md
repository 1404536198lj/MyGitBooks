#第一章 Blocked in binder
## 1.1 条件
当发生SWT时，层层查看thread的堆栈，直到发现有一个（1）“Native”状态的thread，堆栈如下：
```
"NetworkPolicy" prio=5 tid=48 Native
| group="main" sCount=1 dsCount=0 obj=0x13335c10 self=0x7a72f20e00
| sysTid=1329 nice=0 cgrp=default sched=0/0 handle=0x7a701e1450
| state=S schedstat=( 1797045521 883116316 11041 ) utm=126 stm=52 core=2 HZ=100
| stack=0x7a700df000-0x7a700e1000 stackSize=1037KB
| held mutexes=
kernel: (couldn't read /proc/self/task/1329/stack)
native: #00 pc 000000000006c93c /system/lib64/libc.so (__ioctl+4)
native: #01 pc 000000000001fba0 /system/lib64/libc.so (ioctl+140)
native: #02 pc 0000000000055578 /system/lib64/libbinder.so (_ZN7android14IPCThreadState14talkWithDriverEb+264)
native: #03 pc 000000000005638c /system/lib64/libbinder.so (_ZN7android14IPCThreadState15waitForResponseEPNS_6ParcelEPi+368)
native: #04 pc 000000000004b1f4 /system/lib64/libbinder.so (_ZN7android8BpBinder8transactEjRKNS_6ParcelEPS1_j+72)
native: #05 pc 00000000000fe8d0 /system/lib64/libandroid_runtime.so (???)
native: #06 pc 0000000001157478 /data/dalvik-cache/arm64/system@framework@boot.oat (Java_android_os_BinderProxy_transactNative__ILandroid_os_Parcel_2Landroid_os_Parcel_2I+196)
at android.os.BinderProxy.transactNative(Native method)
at android.os.BinderProxy.transact(Binder.java:626)
at com.android.internal.telephony.ISub$Stub$Proxy.getSimStateForSlotIdx(ISub.java:1079)
at android.telephony.SubscriptionManager.getSimStateForSlotIdx(SubscriptionManager.java:1471)
at android.telephony.TelephonyManager.getSimState(TelephonyManager.java:1998)
at com.android.server.net.NetworkPolicyManagerService.setNetworkTemplateEnabled(NetworkPolicyManagerService.java:1364)
at com.android.server.net.NetworkPolicyManagerService.updateNetworkEnabledLocked(NetworkPolicyManagerService.java:1332)
at com.android.server.net.NetworkPolicyManagerService.setNetworkPolicies(NetworkPolicyManagerService.java:2091)
- locked <0x0b275b9f> (a java.lang.Object)
at com.android.server.net.NetworkPolicyManagerService.addNetworkPolicyLocked(NetworkPolicyManagerService.java:2112)
at com.android.server.net.NetworkPolicyManagerService.ensureActiveMobilePolicyLocked(NetworkPolicyManagerService.java:1617)
at com.android.server.net.NetworkPolicyManagerService.ensureActiveMobilePolicyLocked(NetworkPolicyManagerService.java:1579)
at com.android.server.net.NetworkPolicyManagerService.-wrap8(NetworkPolicyManagerService.java:-1)
at com.android.server.net.NetworkPolicyManagerService$14.onReceive(NetworkPolicyManagerService.java:1282)
- locked <0x0b275b9f> (a java.lang.Object)
at android.app.LoadedApk$ReceiverDispatcher$Args.run(LoadedApk.java:1214)
at android.os.Handler.handleCallback(Handler.java:836)
at android.os.Handler.dispatchMessage(Handler.java:103)
at android.os.Looper.loop(Looper.java:203)
at android.os.HandlerThread.run(HandlerThread.java:61)
```
（2）其中包含三行以上的
```
native: #00 pc 000000000006c93c /system/lib64/libc.so (__ioctl+4)
native: #01 pc 000000000001fba0 /system/lib64/libc.so (ioctl+140)
native: #02 pc 0000000000055578 /system/lib64/libbinder.so (_ZN7android14IPCThreadState14talkWithDriverEb+264)
native: #03 pc 000000000005638c /system/lib64/libbinder.so (_ZN7android14IPCThreadState15waitForResponseEPNS_6ParcelEPi+368)
native: #04 pc 000000000004b1f4 /system/lib64/libbinder.so (_ZN7android8BpBinder8transactEjRKNS_6ParcelEPS1_j+72)
native: #05 pc 00000000000fe8d0 /system/lib64/libandroid_runtime.so (???)
native: #06 pc 0000000001157478 /data/dalvik-cache/arm64/system@framework@boot.oat
```
（3）且第三行的native：#行中包含关键字：talkWithDriverEb。
<br>通过以上三个条件可判定，SWT发生的原因为：Blocked in binder。

## 1.2 定位
<br>**接下来的寻找问题根源的步骤为：**

1. 从SYS_PROCESS_AND_THREAD获得当前进程的process name 、thread name对应的from_pid,from_tid。
>详细：查找SYS_PROCESS_AND_THREAD文件中，PID为system_server pid的行，当前thread如果是“main”则直接返回该行;否则通过当前thread的名字查找对应的行，并返回。该步骤的作用主要是通过Process和thread的name在SYS_PROCESS_AND_THREAD中获取对应的PID值（from_pid,from_tid）。如果当前thread是"main"，则from_tid=from_pid。

2. 从SYS_BINDER_INFO获得与当前进程进行binder通信的process和thread的to_pid,to_tid。
>详细：通过上个步骤获取的from_pid,from_tid到SYS_BINDER_INFO中查找以“outgoing transaction”或者“incoming transaction”开头，并且包含from_pid：from_tid的行，通过该行可获取正在与当前thread进行通信的to_pid,to_tid

3. 从SYS_PROCESS_AND_THREAD获得to_pid,to_tid对应的name。
>详细：查找SYS_PROCESS_AND_THREAD文件中，PID为to_tid，PPID为to_pid的行，以及PID为to_pid的行

4. 记录。
>详细：根据（from_pid,from_tid）以及name、（to_pid,to_tid）以及name来记录“Blocked in binder ---> from ”..."to"。

5. 从整个system_server的traces中根据to_pid ，to_pid的name找到多个traces。

6. 从上个步骤中找到的多个traces中，找到离from_pid时间最近的一个。

<br>继续在binder通信的process B中查找问题根源：以to_tid为开始从traces中找到一个线程运行状态不是Blocked，并且该线程没有dead lock、binder full、Blocked in binder。该线程中则存在问题根源。

## 1.3 总结
Blocked in binder情况下的SWT，其流程为：从当前进程A的Blocked线程a1查找到binder通信的native状态的线程an，并从SYS_BINDER_INFO中找到与当前进程进行通信的进程B和进程中的通信线程b1,在B中通过b1找到最终"Native"状态，不通信的线程bn。极端情况下，B还可能卡在进程C的binder中。

>补充：
（1）当前线程为"main"时，其tid为所属进程的pid。
（2）SYS_BINDER_INFO中的通信对象，当tid为0意味着thread已经被释放了。

# 第二章 binder full
## 2.1 条件






