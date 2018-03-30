Java层通过ServiceManager复用了servicemanager，因此ServiceManager相当于native层servicemanager在Java层的代理。菜外Java层通过JNI将BpBinder和BBinder封装为Java对象。

### Java系统服务的创建过程
> 以power系统服务为例。该服务由PowerManagerService实现，启动过程位于ServerThread中。
> PowerManagerService继承自SystemService，在ZygoteInit.startCoreService中通过onStart方法启动。而PMS的onStart方法会将BinderService注册到ServiceManager中。
> BinderService继承自Binder，构造函数中会调用Binder.attachInterface，通过getNativeBBinderHolder方法来跟native层的BBinder建立联系(Binder的构造函数中将mObject设置为BBinder地址)

### Java系统服务的注册过程
> ServiceManager.addService->getIServiceManager().addService
>
> getIServiceManager方法有两个作用：
> 1. d调用BinderInternal.getContextObj返回一个IBinder类型的对象
> 2. 以上一步的返回值作为参数调用ServiceManagerNative.asInterface，返回IServiceManager类型的sServiceManager单例

##### 调用BinderInternal.getContextObject方法
> JNI实现在android_util_Binder.cpp, 创建了一个BpBinder并返回

##### 执行javaObjectForIBinder方法
> javaObjectForIBinder将native层的BpBinder与Java层的BinderProxy对象做了关联，而关联信息由gBinderProxyOffsets结构体提供

##### 调用ServiceManagerNative.asInterface方法
> BinderProxy.queryLocalInterface == null->返回ServiceManagerProxy

##### 调用ServiceManagerProxy.addService方法注册
> 1. Parcel.writeStrongBinder用于向Parcel中写入Service对象，实现在android_os_Parcel.cpp中