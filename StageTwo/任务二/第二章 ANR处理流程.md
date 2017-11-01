# 第二章 ANR处理流程
&emsp;&emsp;当四大组件或进程发生ANR时，都会调用AMS.appNotResponding来想用户提供反馈，并收集信息。以下场景会调用到AMS.appNotResponding：

1. service Timeout：前台服务超过20s未完成，后台服务超过200s未完成。
2. BroadCastReceiver Timeout：前台广播在10s内未完成，后台广播60s未完成。
3. InputDispatching Timeout：输入分发5s内未完成。

&emsp;&emsp;contentProvider在10s内未成功publish会造成ANR，但是不会调用AMS.appNotResponding，而是会杀进程并清理相关信息。

## 2.1  appNotResponding处理流程
```java
final void appNotResponding(ProcessRecord app, ActivityRecord activity,
        ActivityRecord parent, boolean aboveSystem, final String annotation) {
    ...
    updateCpuStatsNow(); //第一次 更新cpu统计信息
    synchronized (this) {
      //PowerManager.reboot() 会阻塞很长时间，因此忽略关机时的ANR
      if (mShuttingDown) {
          return;
      } else if (app.notResponding) {
          return;
      } else if (app.crashing) {
          return;
      }
      //记录ANR到EventLog
      EventLog.writeEvent(EventLogTags.AM_ANR, app.userId, app.pid,
              app.processName, app.info.flags, annotation);
              
      // 将当前进程添加到firstPids
      firstPids.add(app.pid);
      int parentPid = app.pid;
      
      //将system_server进程添加到firstPids
      if (MY_PID != app.pid && MY_PID != parentPid) firstPids.add(MY_PID);
      
      for (int i = mLruProcesses.size() - 1; i >= 0; i--) {
          ProcessRecord r = mLruProcesses.get(i);
          if (r != null && r.thread != null) {
              int pid = r.pid;
              if (pid > 0 && pid != app.pid && pid != parentPid && pid != MY_PID) {
                  if (r.persistent) {
                      firstPids.add(pid); //将persistent进程添加到firstPids
                  } else {
                      lastPids.put(pid, Boolean.TRUE); //其他进程添加到lastPids
                  }
              }
          }
      }
    }
    
    // 记录ANR输出到main log
    StringBuilder info = new StringBuilder();
    info.setLength(0);
    info.append("ANR in ").append(app.processName);
    if (activity != null && activity.shortComponentName != null) {
        info.append(" (").append(activity.shortComponentName).append(")");
    }
    info.append("\n");
    info.append("PID: ").append(app.pid).append("\n");
    if (annotation != null) {
        info.append("Reason: ").append(annotation).append("\n");
    }
    if (parent != null && parent != activity) {
        info.append("Parent: ").append(parent.shortComponentName).append("\n");
    }
    
    //创建CPU tracker对象
    final ProcessCpuTracker processCpuTracker = new ProcessCpuTracker(true);
    //输出traces信息【】
    File tracesFile = dumpStackTraces(true, firstPids, processCpuTracker, 
            lastPids, NATIVE_STACKS_OF_INTEREST);
            
    updateCpuStatsNow(); //第二次更新cpu统计信息
    //记录当前各个进程的CPU使用情况
    synchronized (mProcessCpuTracker) {
        cpuInfo = mProcessCpuTracker.printCurrentState(anrTime);
    }
    //记录当前CPU负载情况
    info.append(processCpuTracker.printCurrentLoad());
    info.append(cpuInfo);
    //记录从anr时间开始的Cpu使用情况
    info.append(processCpuTracker.printCurrentState(anrTime));
    //输出当前ANR的reason，以及CPU使用率、负载信息
    Slog.e(TAG, info.toString()); 
    
    //将traces文件 和 CPU使用率信息保存到dropbox，即data/system/dropbox目录
    addErrorToDropBox("anr", app, app.processName, activity, parent, annotation,
            cpuInfo, tracesFile, null);

    synchronized (this) {
        ...
        //后台ANR的情况, 则直接杀掉
        if (!showBackground && !app.isInterestingToUserLocked() && app.pid != MY_PID) {
            app.kill("bg anr", true);
            return;
        }

        //设置app的ANR状态，病查询错误报告receiver
        makeAppNotRespondingLocked(app,
                activity != null ? activity.shortComponentName : null,
                annotation != null ? "ANR " + annotation : "ANR",
                info.toString());

        //重命名trace文件
        String tracesPath = SystemProperties.get("dalvik.vm.stack-trace-file", null);
        if (tracesPath != null && tracesPath.length() != 0) {
            //traceRenameFile = "/data/anr/traces.txt"
            File traceRenameFile = new File(tracesPath);
            String newTracesPath;
            int lpos = tracesPath.lastIndexOf (".");
            if (-1 != lpos)
                // 新的traces文件= /data/anr/traces_进程名_当前日期.txt
                newTracesPath = tracesPath.substring (0, lpos) + "_" + app.processName + "_" + mTraceDateFormat.format(new Date()) + tracesPath.substring (lpos);
            else
                newTracesPath = tracesPath + "_" + app.processName;

            traceRenameFile.renameTo(new File(newTracesPath));
        }
                
        //弹出ANR对话框
        Message msg = Message.obtain();
        HashMap<String, Object> map = new HashMap<String, Object>();
        msg.what = SHOW_NOT_RESPONDING_MSG;
        msg.obj = map;
        msg.arg1 = aboveSystem ? 1 : 0;
        map.put("app", app);
        if (activity != null) {
            map.put("activity", activity);
        }
        
        //向ui线程发送，内容为SHOW_NOT_RESPONDING_MSG的消息
        mUiHandler.sendMessage(msg);
    }
    
}
```
当发生ANR时, 会按顺序依次执行:
1. 记录ANR到EventLog。也就是说ANR触发的时间点最接近的就是EventLog中输出的am_anr信息（am_anr）;
2. dumpStackTraces（）输出重要进程的traces信息（该方法比较耗时）（traces）。
3. 记录当前各个进程的CPU使用情况以及CPU负载情况、ANR reason并输出到main log（ANR in）。
4. 将traces文件、anr reason和 CPU使用情况信息保存到dropbox，即data/system/dropbox目录
5. 根据进程类型,来决定直接后台杀掉,还是弹框告知用户。

dumpStackTraces（）输出重要进程的traces信息，这些进程包含:
1. firstPids队列：第一个是ANR进程，第二个是system_server，剩余是所有persistent进程；
2. Native队列：是指/system/bin/目录的mediaserver,sdcard 以及surfaceflinger进程；
3. lastPids队列: 是指mLruProcesses中的不属于firstPids的所有进程。

dumpStackTraces（）会保证data/anr/traces.txt文件内容是全新的方式，而非追加。主要功能，依次输出：
<br>1.收集firstPids进程的stacks；
>&emsp;&emsp;第一个是发生ANR进程；
<br>&emsp;&emsp;第二个是system_server；
<br>&emsp;&emsp;mLruProcesses中所有的persistent进程；

<br>2.收集Native进程的stacks；(dumpNativeBacktraceToFile) 
>依次是mediaserver,sdcard,surfaceflinger进程；收集该信息时，会向debuggerd守护进程发送命令DEBUGGER_ACTION_DUMP_BACKTRACE， debuggerd收到该命令，在子进程中调用 dump_backtrace()来输出backtrace。

<br>3.收集lastPids进程的stacks;
>依次输出CPU使用率top 5的进程；

firstPids列表中的进程, 两个进程之间会休眠200ms, 可见persistent进程越多,则时间越长。top 5进程的traces过程中, 同样是间隔200ms, 另外进程使用情况的收集也是比较耗时。


## 2.2 总结
触发ANR时系统会输出关键信息：(这个较耗时,可能会有10s)：
1. 将am_anr信息,输出到EventLog.(ANR开始起点看EventLog)
2. 获取重要进程trace信息，保存到/data/anr/traces.txt(会先删除老的文件)， 包括：Java进程的traces、Native进程的traces;。
3. ANR reason以及CPU使用情况信息，输出到main log;
4. 再将CPU使用情况和进程trace文件信息，再保存到/data/system/dropbox；

ANR traces还可通过命令的方式获取：
1. kill -3 [pid]
2. debuggerd -b [pid]

&emsp;&emsp;kill -3命令需要虚拟机的支持，所以无法输出Native进程traces.而debuggerd -b [pid]也可用于Java进程，但信息量远没有kill -3多。 总之，ANR信息最为重要的是dropbox信息，比如system_server_anr。
