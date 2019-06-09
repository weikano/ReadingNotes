### TracePlugin原理

> TracePlugin通过AnrTracer、EvilMethodTracer、StartUpTracer、FrameTracer来监控系统performance

```java
//TracePlugin.java
public void init() {
  //调用AnrTracer、FrameTracer、EvilMethodTracer和StartupTracer的构造方法来初始化对应的field
}
public void start() {
  Runnable runnable = ()->{
    UIThreadMonitor.getMonitor().init(traceConfig);
    AppMethodBeat.getInstance().onStart();
    UIThreadMonitor.getMonitor().onStart();
    anrTracer.onStartTrace();
    frameTracer.onStartTrace();
    evilMethodTracer.onStartTrace();
    startupTracer.onStartTrace();
  };
  //主线程执行上面的runnable
}
```

#### 实现细节

##### 1. 编译期

通过代理编译期间的任务 transformClassesWithDexTask，将全局 class 文件作为输入，利用 **ASM** 工具，高效地对所有 class 文件进行扫描及插桩。

插桩过程有几个关键点：

1. 选择在该编译任务执行时插桩，是因为 proguard 操作是在该任务之前就完成的，意味着插桩时的 class 文件已经被混淆过的。而选择 proguard 之后去插桩，是因为如果提前插桩会造成部分方法不符合内联规则，没法在 proguard 时进行优化，最终导致程序方法数无法减少，从而引发方法数过大问题。

2. 为了减少插桩量及性能损耗，通过遍历 class 方法指令集，判断扫描的函数是否只含有 PUT/READ FIELD 等简单的指令，来过滤一些默认或匿名构造函数，以及 get/set 等简单不耗时函数。

3. 针对界面启动耗时，因为要统计从 Activity#onCreate 到 Activity#onWindowFocusChange 间的耗时，所以在插桩过程中需要收集应用内所有 Activity 的实现类，并覆盖 onWindowFocusChange 函数进行打点。

4. 为了方便及高效记录函数执行过程，我们为每个插桩的函数分配一个独立 ID，在插桩过程中，记录插桩的函数签名及分配的 ID，在插桩完成后输出一份 mapping，作为数据上报后的解析支持。

##### 2. 运行期

编译期已经对全局的函数进行插桩，在运行期间每个函数的执行前后都会调用 MethodBeat.i/o 的方法，如果是在主线程中执行，则在函数的执行前后获取当前距离 MethodBeat 模块初始化的时间 offset（为了压缩数据，存进一个long类型变量中），并将当前执行的是 MethodBeat i或者o、mehtod id 及时间 offset，存放到一个 long 类型变量中，记录到一个预先初始化好的数组 long[] 中 index 的位置（预先分配记录数据的 buffer 长度为 100w，内存占用约 7.6M）

通过向 Choreographer 注册监听，在每一帧 doframe 回调时判断距离上一帧的时间差是否超出阈值（卡顿），如果超出阈值，则获取数组 index 前的所有数据（即两帧之间的所有函数执行信息）进行分析上报。 同时，我们在每一帧 doFrame 到来时，重置一个定时器，如果 5s 内没有 cancel，则认为 ANR 发生，这时会主动取出当前记录的 buffer 数据进行独立分析上报，对这种 ANR 事件进行单独监控及定位。

另外，考虑到每个方法执行前后都获取系统时间（System.nanoTime）会对性能影响比较大，而实际上，单个函数执行耗时小于 5ms 的情况，对卡顿来说不是主要原因，可以忽略不计，如果是多次调用的情况，则在它的父级方法中可以反映出来，所以为了减少对性能的影响，通过另一条更新时间的线程每 5ms 去更新一个时间变量，而每个方法执行前后只读取该变量来减少性能损耗。

**堆栈聚类问题**： 如果将收集的原始数据进行上报，数据量很大而且后台很难聚类有问题的堆栈，所以在上报之前需要对采集的数据进行简单的整合及裁剪，并分析出一个能代表卡顿堆栈的 key，方便后台聚合。

通过遍历采集的 buffer ，相邻 i 与 o 为一次完整函数执行，计算出一个调用树及每个函数执行耗时，并对每一级中的一些相同执行函数做聚合，最后通过一个简单策略，分析出主要耗时的那一级函数，作为代表卡顿堆栈的key。 