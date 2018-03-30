### ActivityManager提供以下功能
> 1. 启动或杀死应用进程
> 2. 启动并调度Activity生命周期
> 3. 启动并调度Service生命周期
> 4. 注册BroadcastReceiver，并接收和分发Broadcast
> 5. 启动并发布ContentProvider
> 6. 调度task
> 7. 检查、授予、收回访问URI的权限
> 8. 处理应用程序crash
> 9. 调整进程调度优先级及策略(OOM adj)
> 10. 查询当前系统运行状态(Memory, Graphics, CPU, Database等)

### ActivityManager的组成主要由以下六部分
> 1. Binder接口：由IBinder和Binder提供进程间通信的接口
> 2. 服务接口：由IInterface和IActivityManager提供系统服务接口，此处IActivityManager并不是由IActivityManager.aidl转换而来的。
> 3. 服务中枢：ActivityManagerNative继承自Binder并实现了IActivityManager，它提供了服务接口和Binder接口的转化功能，并在内部存储服务代理对象并提供了getDefault放回服务代理。
> 4. 服务代理：ActivityManagerProxy，用于与server端提供的系统服务进行进程间通信
> 5. client：由ActivityManager封装了一部分服务接口供client调用。ActivityManager内部通过调用ActivityManagerNative.getDefault方法得到一个ActivityManagerProxy对象
> 6. server：由ActivityManagerService实现

### ActivityManagerService在系统启动阶段的主要工作
> ActivityManagerService在SystemServer.startBootstrapServices中通过ActivityManagerService.LifeCycle初始化，在SystemServer.startOtherServie中调用installSystemPRoviders, 在用setWindowManager关联WindowManager。WindowManagerService也在SystemServer.startOtherServices中调用main初始化启动。AMS.systemReady回调中会启动很多其他的服务

### 启动AMS
> AMS.Lifecycle.onStart会调用AMS的构造函数和start方法

### 系统Context的创建和初始化
> 1. ActivityThread.attach方法会创建系统Context
> 2. ContextImpl.getSystemContext创建系统context
> 3. LoadedApk创建后调用init方法执行系统Context的第二次初始化

### 创建Application对象
> ActivityThread.attach中通过Instrumentation创建Application, 然后app.attach将系统Context关联

### 创建ActivityStack类
> 用于管理Activity栈并维护其状态，由ActivityStackSupervisor管理。