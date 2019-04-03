### 第08章-管理Activity和组件运行状态的系统进程-AMS

#### 8.1 AMS功能概述

```java
//SystemServer.java
private void startBootstrapServices() {
  //startService会通过反射调用构造方法，然后再调用onStart。即构造一个AMS，然后调用AMS.start方法
  mActivityManagerService = mSystemServiceManager.startService(ActivityManagerService.Lifecycle.class).getService();
  //忽略其他
  mActivityManagerService.setSystemProcess();
}
//启动其他服务
public void setSystemProcess() {
  //忽略其他
  try {
    ServiceManager.addService(Context.ACTIVITY_SERVICE, this, /* allowIsolated= */ true,
                              DUMP_FLAG_PRIORITY_CRITICAL | DUMP_FLAG_PRIORITY_NORMAL | DUMP_FLAG_PROTO);
    ServiceManager.addService(ProcessStats.SERVICE_NAME, mProcessStats);
    ServiceManager.addService("meminfo", new MemBinder(this), /* allowIsolated= */ false,
                              DUMP_FLAG_PRIORITY_HIGH);
    ServiceManager.addService("gfxinfo", new GraphicsBinder(this));
    ServiceManager.addService("dbinfo", new DbBinder(this));
    if (MONITOR_CPU_USAGE) {
      ServiceManager.addService("cpuinfo", new CpuBinder(this),
                                /* allowIsolated= */ false, DUMP_FLAG_PRIORITY_CRITICAL);
    }
    ServiceManager.addService("permission", new PermissionController(this));
    ServiceManager.addService("processinfo", new ProcessInfoService(this));

  } catch (PackageManager.NameNotFoundException e) {
    throw new RuntimeException(
      "Unable to find android system package", e);
  }
	
}

```

#### 8.2 管理当前系统中Activity的状态-ActivityStack

- ActivityStackSupervisor：mStackSupervisor。用来管理ActivityStack
- ActivityStack：mHomeStack、mFocusedStack
- ActivityStartController：mActivityStartController
- ClientLifecycleManager：mLifecycleManager。用来控制Activity的状态，主要用来执行ClientTransactionItem的子类（Launch/Resume/Pause/DestroyActivityItem），通过Item的execute方法，调用到ClientTransactionHandler的子类ActivityThread中的各种方法
- RecentTasks：mRecentTasks
- ActiveServices：mServices
- BroadcastQueue：包括mFgBroadcastQueue、mBgBroadcastQueue以及mBroadcastQueues
- ActivityRecord：mLastResumedActivity
- ProcessRecord：mHomeProcess, mPreviousProcess
- ProcessList：mProcessList
- ProcessMap<ProcecssRecord>：mProcessNames
- SparseArray<ProcessRecord>：mIsolatedProcesses
- ArrayList<ProcessRecord>：mProcessOnHold、mPersistentStartingProcesses、mRemovedProcesses、mLruProcesses、mProcessesToGc、mPendingPssProcesses
- HashMap<IBinder, ReceiverList>：mRegisterReceivers
- ProviderMap：mProviderMap
- ArrayList<ContentProviderRecord>：mLaunchingProviders

##### 8.2.1 ActivityStack分析：

AMS是通过ActivityStack来记录、管理系统中的Activity状态

- ActivityState：描述Activity的状态迁移图
- mLRUActivities：ArrayList<ActivityRecord>记录运行中的Activity，按照最后使用时间排序，第一个是最近使用的
- mTaskHistory：ArrayList<TaskRecord>任务栈历史，可能包括正在运行的
- mPausingActivity：在启动下一个Activity之前，保存当前被pause掉的Activity
- mLastPausedActivity：上一个被暂停的Activity
- mLastNoHistoryActivity：保存NoHistory标志的Activity。当设备sleep时，记录下它，唤醒后关闭它
- mResumedActivity当前被resume的Activity，可以为null

##### 8.2.2 ActivityStackSuperVisor分析

Activity应用界面的管理分为5个层次：

1. 多个Display对应多个屏幕，一般第一个为内置屏幕，第二个为hdmi，第三个为虚拟屏幕
2. 每个DIsplay上包括多个类型的ActivityStack：
3. 每个ActivityStack上包含多个TaskRecord，通过mTaskHistory维护
4. 每个TaskRecord上有多个ActivityRecord，通过mActivities维护
5. 每个Activity在WMS中代表一个AppWindowToken，用allAppWindows维护子window信息，每个AppWindowTOken上面可能有一些子window，用WindowToken表示。每个Window用WindowState记录状态，AMS只能看到AppWindowToken信息，看不到子Window

field介绍：

- mHomeStack、mFocusedStack、mLastFocusedStack：launcher相关的stack；当前获取焦点的或者正在启动的下一个Activity所属的stack；在下一个stack被resume之前，一直是mFocusedStack。
- mRecentTasks：历史task的列表，包括inactive状态的tasks
- mRunningTasks：用来获取当前运行中的task的抽象逻辑类
- mActivitiesWaitingForVisible：等待新的Activity变成visible的Activity的列表
- mStoppingActivities：等待进入stopped状态的Activity集合，但是在等下一个Activity准备好
- mFinishingActivities：等待finish的Activity集合，只要前面的Activity准备好
- mGoingToSleepActivities：将要进入睡眠状态的进程中的Activity集合
- //todo

#### 8.3 startActivity流程

略。参考vsdx文件

#### 8.4 Activity Task

- taskAffinity：Activity希望归属的task，默认情况下，同一个APP的所有Activity有共同的taskAffinity。何时生效：
  1. 当intent中带有FLAG_ACTIVITY_NEW_TASK：如果当前有一个task，它的affinity与Activity是一样的，那么会直接用此task来完成操作；否则系统需要重新创建一个task
  2. 当allowTaskReparenting为true时。以联系人APP发送短信为例子：A打开联系人详情（Activity1），再打开短信编辑（ActivityB，属于短信App），这时候在联系人详情所属的taskA中有Activity1和ActiivtyB两个。如果设置了allowTaskReparenting，那么当后期短信App转为前台时，ActivityB会挪到短信的task中
- android:launchMode：singleTask的Activity永远在task的根位置。singleInstance永远处在一个独立的task中
- android:allowTaskReparenting：参考前面
- android:clearTaskOnLaunch：清除task中所有除了根元素之外的元素
- android:alwaysRetainTaskState：如果用户在一定时间内不再访问task，那么系统可能会清除task中除了根之外的其他元素。设置为true可以避免
- android:finishOnTaskLaunch：当task被再次启动时，Activity是否需要被销毁。
- FLAG_ACTIVITY_NEW_TASK：与singleTask相同
- FLAG_ACTIVITY_SINGLE_TOP：与singleTop相同
- FLAG_ACTIVITY_CLEAR_TOP：如果要启动的Activity已经在task中了，那么在它之上的Activity都会被销毁掉
- FLAG_ACTIVITY_NO_HISTORY：不会保留在historystack中
- FLAG_ACTIVITY_MULTIPLE_TASK：需要与NEW_TASK一同使用。系统总是启动一个新的task来容纳要启动的Activity
- FLAG_ACTIVITY_EXCLUDE_FROM_RECENTS：不会在最近启动中出现
- FLAG_ACTIVITY_BROUGHT_TO_FRONT：singleTask会自动加上这个标志
- FLAG_ACTIVITY_RESET_TASK_IF_NEEDED：不管是在新的task还是老的task中，Activity都会处于最上端
- FLAG_ACTIVITY_LAUNCH_FROM_HISTORY：自动设置，说明是从最近任务中启动的
- FLAG_ACTIVITY_CLEAR_WHEN_TASK_RESET：FLAG_ACTIVITY_NEW_DOCUMENT代替
- FLAG_ACTIVITY_NO_USER_ACTION：Activity中的onUserLeaveHint不会被执行
- FLAG_ACTIVITY_REORDER_TO_FRONT：如果要启动的Activity在history stack中运行，那么调整顺序放到最前段
- FLAG_ACTIVITY_NO_ANIMATION
- FLAG_ACTIVITY_CLEAR_TASK：只能与FLAG_ACTIVITY_NEW_TASK一起使用，会清除task中的其他元素
- FLAG_ACTIVITY_TASK_ON_HOME：被启动的Activity将被放在task中Launcher的上面，这样返回时就会是LAUNCHER了，只能与FLAG_ACTIVITY_NEW_TASK一起使用

