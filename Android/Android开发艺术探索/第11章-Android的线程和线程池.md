除了Thread本身, 还有AsyncTask, IntentService和HandlerThread.  
AsyncTask封装了Handler和线程池, 它主要是为了方便更新UI.  
HandlerThread是一个具有消息循环的线程, 在它的内部可以使用Handler.  
IntentService是一个服务, 内部采用HandlerThread来执行任务, 执行完毕后IntentService会自动退出.

# 主线程和子线程
# Android中的线程形态
## AsyncTask
1. AsyncTask必须在主线程中加载.
2. AsyncTask对象必须在主线程中创建.
3. execute方法必须在主线程调用.
4. 不要在程序中直接调用onPreExecute(), onPostExecute(), doInBackground和onProgressUpdate方法.
5. 一个AsyncTask对象只能执行一次.
6. 在1.6之前, AsyncTask是串行, 1.6开始并行,3.0又改为串行.

## AsyncTask的工作原理
AsyncTask有两个线程池( SerialExecutor和THREAD_POOL_EXECUTOR), 一个Handler(InternalHandler). 其中SerialExecutor用于任务的排队, THREAD_POOL_EXECUTOR用于执行任务. InternalHandler用于将执行任务环境从线程池切换到主线程.

## HandlerThread
run()方法中通过Looper.prepare()来创建消息对立, 并通过Looper.loop()开启消息循环, 这样在实际使用中就允许在HandlerThread中创建Handler了.  
HandlerThread的一个具体使用场景是IntentService的实现.

## IntentService

# Android中的线程池
## ThreadPoolExecutor
ThreadPoolExecutor执行任务时大致遵循如下规则:
1. 如果线程池中的线程数量未达到核心线程的数量, 那么会直接使用核心线程执行.
2. 如果线程池中线程数量已经达到或超过核心线程数量, 那么任务会被插入到任务队列中排队等待执行.
3. 如果2中无法插入成功, 往往是因为任务队列已满. 这时若线程数量未达到线程池规定的最大值, 那么会立刻启动一个非核心线程来执行.
4. 如果3中已达到最大数量, 那么拒绝执行此任务, 调用RejectedExecutionHandler的rejectExecition来通知调用者.

## 线程池的分类
### FixedThreadPool
线程数量固定. 当线程处于空闲状态时, 不会被回收, 除非线程池被关闭了. 当所有线程都处于活动状态时, 新任务会处于等待状态, 知道空闲线程出来. **只有核心线程,任务队列没有大小限制**.

### CachedThreadPool
**只有非核心线程, 线程数限制为Integer.MAX_VALUE.**适合执行大量的耗时较少的任务. 当整个线程池都处于闲置状态时, 线程池中的线程都会超时而停止, 这个时候CachedThreadPool之中没有任何线程, 几乎不占任务系统资源. 

### ScheduledThreadPool
**核心线程数量固定, 非核心线程数量无限制, 并且当非核心线程闲置时会被立即回收.**主要用于执行定时任务和具有固定周期的重复任务.

### SingleThreadExecutor
**它确保所有任务都在同一个线程池中顺序执行.**