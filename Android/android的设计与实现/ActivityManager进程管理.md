### LRU weight机制
> AMS中的mLruProcesses以LRU顺序存储了当前进行的应用程序进程信息，mLruProcesses中第一个元素便是最近最少使用的进程对应的ProcessRecord，用于管理应用程序进程。
>
>以下三种情况会更新mLruProcesses
> 1. 应用程序异常退出时：调用handleAppDiedLocked更新mLruProcesses
> 2. 显示杀死指定进程：调用AMS显式杀死进程时更新mLruProcesses
> 3. 启动和调度四大组件：调用updateLruProcessLocked更新mLruProcesses, updateLruProcessLocked->updateLruProcessInternalLocked
>
>internal方法主要工作如下：
> 1. 为指定的进程计算LRU序列号和LRU weight
> 2. 根据lruweight值将指定的进程信息插入mLruProcesses表示的进程LRU列表中，lruweight越大在列表中越靠后
> 3. 如果进程使用了ContentProvider或者Service，还需要更新ContentProvider或者Service所在进程的lruweight及其在LRU列表中的位置
> 4. 根据oomAdj值决定是否同时调整OOM adj的值

### OOM adj机制
> OOM adj是内存不足状态的调整级别，系统根据运行时占有内存和CPU等情况为每个进程计算出一个adj值，该值的范围为-10001到+1001, adj值越大越容易被杀死, 可以通过cat /proc/</pid/>/oom_adj命令查看指定进程的OOM adj值
>
> android.server.am.ProcessList中定义了调整级别

##### 更新OOM adj的值
> updateOomAdjLocked() -> computeOomAdjLocked(app, ProcessList.UNKNOWN_ADJ, TOP_APP, true, now)。
> 
> 其中TOP_APP为TOP_ACT.app(ProcessRecord)，TOP_ACT=resumedAppLocked()->ActivityStackSupervisor.getResumedActivityLocked()
>
>ActivityStackSupervisor.getResumedActivityLocked首先获取当前的mFocusStack(获取到输入焦点或者正在启动下一个activity的ActivityStack), 然后获取ActivityStack.mResumedActivity。如果mResumedActivity娶不到或者mResumedActivity.app为null，那么就取ActviityStack.mPausingActivity。如果mPausingActivity。如果取不到或者mPausingActivity.app为null。那么调用ActivityStack.topRunningActivityLocked获取。topRunningActivityLocked会从mTaskHistory(所有之前和可能运行中的activity的后退历史)中获取最后一个元素，即TaskRecord.topRunningActivityLocked()
>
>接下来进入AMS.computeOomLocked方法
> 1. 如果待修改的app是TOP_APP，那么adj= FOREGROUND_APP_ADJ, schedGroup = SCHED_GROUP_TOP_APP
> 2. 如果app.instr!=null，即有运行中的instrumentation，那么adj = FOREGROUP_APP_ADJ, schedGroup = SCHED_GROUP_DEFAULT
> 3. 如果isReceivingBroadcastLocked()，即正在接收广播，那么adj = FOREGROUND_APP_ADJ, schedGroup根据广播是否包含mFgBroadcastQueue(registerReceiver时根据Intent.FLAG_RECEIVER_FOREGROUND标记判断是放入mFg还是mBg)。如果app的广播包括mFg，那么schedGroup = SCHED_GROUP_DEFAULT，否则是SCHED_GROUP_BACKGROUND
> 4. 如果app.executingServices.size>0，那么adj= FOREGROUND_APP_ADJ, schedGroup = app.execServicesFg ? SCHED_GROUP_DEFAULT : SCHED_GROUP_BACKGROUND
> 5. 其他情况下先把schedGroup设置为SCHED_GROUP_BACKGROUND, adj设置为updateOomAdjLocked中传入的UNKOWN_ADJ
> 6. 如果app != TOP_APP并且有运行中的Activity，那么遍历运行中的activity。如果visible，那么如果adj比VISIBLE_APP_ADJ大，那么修改为VISIBLE_APP_ADJ；如果是PAUSING或者PAUSED状态，修改adj为PERCEPTIBLE_APP_ADJ；如果是STOPPING状态，adj设置为perceptible_app_adj. 然后如果adj=VISIBLE_APP_ADJ,设置adj+=minLayer(VISIBLE_APP_LAYER_MAX)
> 7. 如果adj> PERCECPTIBLE_APP_ADJ或者procState>AM.PROCESS_STATE_FOREGROUND_SERVICE。如果APP正在执行foreground service，那么修改procState为AM.PROCESS_STATE_FOREGROUND_SERVICE; 如果app.hasOverlayUi，那么procState为AM.PROCESS_STATE_IMPORTANT_FOREGROUND
> 8. 如果adj>PERCEPTIBLE_APP_ADJ或者procState>AM.PROCESS_STATE_TRANSIENT_BACKGROUND。如果app.forcingToImportant, procState为PROCESS_STATE_TRANSIENT_BACKGROUND
> 9. 如果app==mHeavyWeightProcess。如果adj>HEAVY_WEIGHT_APP_ADJ, 那么adj=HEAVY_WEIGHT_APP_ADJ, schedGroup = SCHED_GROUP_BACKGROUND。如果procState>AM.PROCESS_STATE_HEAVY_WEIGHT, 那么procState = AM.PROCESS_STATE_HEAVY_WEIGHT
> 10. app==mHomeProcess, 设置adj为(adminj, HOME_APP_ADJ), schedGroup = SCHED_GROUP_BACKGROUND, procState=min(procState, AM.PROCESS_STATE_HOME)
> 11. app==mPreviousProcess并且有运行中的activity(上一个向用户显示的进程). adj=min(adj, PREVIOUS_APP_ADJ), schedGroup = SCHED_GROUP_BACKGROUND, procState=min(procState, AM.PROCESS_STATE_LAST_ACTIVITY
> 12. 如果app正在进行备份，那么adj=min(adj, BACKUP_APP_ADJ), procState=min(procState, AM.PROCESS_STATE_BACKUP)
> 13. 接下来根据app的services(运行中的服务)来做处理。ServiceRecord.startRequested(调用了start方法)、有活动的绑定的service(会根据ConnectionRecord.binding.client调用computeOomAdjLocked方法来计算clientadj, 进行复杂处理……)
> 14. 然后处理ContentProviders，computeOomAdjLocked方法处理ContentProviderConnection.client对象的adj值，复杂的处理逻辑……
>
>接下来进入applyOomAdjLocked方法:
> 1. 调用ProcessList.setOomAdj方法修改进程oomadj值
> 2. 调用ProcessRecord.kill方法杀死等待被杀死的进程
> 3. setProcessGroup修改进程的调度组
> 4. setProcessState
>
>回到updateOomAdjLocked中，根据app.curProcState以及某些条件来确定是否需要kill掉当前app
>
>接下来回到updateOomAdjLocked中，调用ActivityThread.scheduleTrimMemory通知应用程序释放内存，调用ActivityStackSupervisor.scheduleDestroyAllActivities

### LowMemoryKiller机制
> 源码位于kernel/drivers/staging/android/lowmsemorykiller.c中，其中OOM adj等级由/sys/module/lowmemorykiller/parameters/adj文件指定，最小内存阈值由/sys/module/lowmemorykiller/parameters/minfree文件指定
>
>ProcessList构造函数中会调用updateOomLevels去更新对应的值