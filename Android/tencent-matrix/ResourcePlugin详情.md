### ResourcePlugin详情

用于监听activity泄露，通过ActivityRefWatcher实现

#### 1. ActivityRefWatcher原理

> 1. 通过ActivityLifecycleCallbacks监听Activity的create和destroy
> 2. 通过mScanDestroyedActivitiesTask来检查是否有leak存在
> 3. 调用mHeapDump.dumpHeap，分析dumpFile然后上报issue

#### 2. mScanDestroyedActivitiesTask解析

```java
//ActivityRefWatcher$mScanDestroyedActivitiesTask
public Status execute() {
  //mDestroyedActivityInfos会在ActivityLifecycleCallbacks的onActivityDestroyed方法中加入数据然后notifyAll
  while(mDestroyedActivityInfos.isEmpty()) {
    synchronized(mDestroyedActivityInfos) {
      try {
        mDestroyedActivityInfos.wait();
      }catch {
        
      }
    }
  }
  //跳过debugger状态检测
  //sentinelRef扮演一个哨兵的角色，如果sentinelRef.get不为NULL，那么表示gc没有生效，需要充实
  final WeakReference<Object> sentinelRef = new WeakReference<>(new Object());
  triggerGc();
  if(sentinelRef.get()!=null) {
    return Status.RETRY;
  }
  //遍历mDestroyedActivityInfos
  final Iterator<DestroyedActivityInfo> infoIt = mDestroyedActivityInfos.iterator();
	while (infoIt.hasNext()) {
    final DestroyedActivityInfo destroyedActivityInfo = infoIt.next();
    //已经是发布过的issue，跳过
    if (isPublished(destroyedActivityInfo.mActivityName)) {
      infoIt.remove();
      continue;
    }
    //已经被回收了，跳过
    if (destroyedActivityInfo.mActivityRef.get() == null) {
      // The activity was recycled by a gc triggered outside.
      infoIt.remove();
      continue;
    }
    ++destroyedActivityInfo.mDetectedCount;
    long createdActivityCountFromDestroy = mCurrentCreatedActivityCount.get() - destroyedActivityInfo.mLastCreatedActivityCount;
    if (destroyedActivityInfo.mDetectedCount < mMaxRedetectTimes
        || (createdActivityCountFromDestroy < CREATED_ACTIVITY_COUNT_THRESHOLD && !mResourcePlugin.getConfig().getDetectDebugger())) {
      // Although the sentinel tell us the activity should have been recycled,
      // system may still ignore it, so try again until we reach max retry times.
      continue;
    }
    //接下来就是导出heap，省略
    return Status.RETRY;
  }
}
```

