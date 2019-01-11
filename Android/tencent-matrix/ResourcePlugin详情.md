### ResourcePlugin详情

用于监听activity泄露，通过ActivityRefWatcher实现

#### ActivityRefWatcher原理

> 1. 通过ActivityLifecycleCallbacks监听Activity的create和destroy
> 2. 通过mScanDestroyedActivitiesTask来检查是否由leak存在
> 3. 调用mHeapDump.dumpHeap，分析dumpFile然后上报issue