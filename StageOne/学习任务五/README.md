# Introduction
本章的学习任务是：熟悉android的IPC通信机制，会使用binder进行编程。
<br>
Android中的进程间通信方法很多，例如：

1. Bundle。当我们在一个进程中启动另外一个进程的Activity、Service、Receiver时，我们就可以在Bundle中附加我们所需要传输给远程进程的信息并通过Intent发送出去。需要注意的是，传递的数据需要能够被序列化，这种方式的进程间通信只能是单向传输，因此具有局限性。
2. 文件共享。两个进程通过读写同一个文件来达到数据交换。这种方法适合同步要求不高的进程间通信，因为并发读写会导致严重的问题。
3. Messenger。Messenger是一种轻量级的IPC方案，类似binder。
4. Content Provider。
5. 广播。当程序向系统发送广播时，其他进程只能被动地接收广播数据，而不能主动与广播发送者沟通。
6. socket。

&emsp;&emsp;由于时间有限，本阶段学习主要对IPC方式中较为常用的binder与socket进行熟悉。
