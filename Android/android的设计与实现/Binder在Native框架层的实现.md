### Binder与CS体系结构概述
> 1. Client(用户端)：使用Server端提供的service的一方
> 2. Server(服务端)：为client端提供service的一方
> 3. Proxy(服代理)：位于client端，提供访问服务的接口。主要作用为屏蔽用户端与server端通信的细节，如请求数据的序列化和对响应数据的反序列化等等。
> 4. Stub(服务存根)：位于server端，屏蔽Proxy与Server端通信的细节。如对client端proxy请求数据的反序列化和对server端响应数据的序列化、通信协议的处理、匹配客户端调用service的具体方法。因此，stub相当于service的代理。
> 5. Service(服务)：服务端运行，用于处理client的请求
> 6. 通信协议：主要使用Binder作为通信协议。
>
> 为了简化client使用service的流程，特意增加了一个额外的组件servicemanager。该组件提供了service注册和service检索功能。servicemanager是由init启动的进程，本身也是一个server。

### servicemanager进程的启动过程
> servicemanager是由init.rc中启动的service，对应的代码在framework/base/cmds/servicemanager/service_manager.c
>
> main函数中主要任务就是
> 1. binder_open(打开Binder设备，映射共享内存用于接收IPC数据，/dev/binder)
> 2. 将servicemanager注册为context manager, binder_become_context
> 3. 无限循环等待binder的IPC数据, binder_loop

##### 1. 初始化Binder通信环境
> binder_open函数初始化Binder的通信环境，调用mmap对内存映射。主要工作分为三部分：
> 1. 创建binder_state结构体并分配内存
> 2. open调用以读写方式打开设备文档/dev/binder
> 3. mmap讲设备文件映射到当前进程的虚拟内存地址

##### 2. 注册上下文管理者
> ioctl -> kernal/drivers/staging/android/binder.c的binder_ioctl
>
> 结果是增加一个全局的Binder node节点，由参数NULL制定该节点的索引值为0.最终0节点存入全局变量binder_context_mgr_node中，以此保证系统唯一

##### 3. 等待接收并处理IPC通信请求
> binder_loop方法主要用binder_write和一个无限for循环组成。
> 1. binder_write位于framework/base/cmds/servicemanager/binder.c中, 回济南贵servicemanager的main线程对应的looper设置为BINDER_LOOPER_STATE_ENTERED
> 2. Binder Looper状态。每发送一次binder ioctl指令最终都会调用binder驱动的binder_thread_read函数，从binder驱动中读取IPC数据，然后通过binder_parse方法解析。其中BR_TRANSACTION指令用于注册和检索service(service_manager.c中的do_find_service和do_add_service)

### Server的启动和Service的注册过程
> 以AudioFlinger服务为例。mediaserver的main函数位于/frameworks/av/media/mediaserver/main_mediaserver.cpp
> 1. 创建ProcessState对象
> 2. 获取servicemanager对象
> 3. 注册Service
> 4. Server进程开启线程池

##### 创建ProcessState对象
> frameworks/native/libs/binder/ProcessState.cpp
>
> ProcessState通过self函数返回一个单例的对象，并将该对象保存至Static.h的gProcess中。ProcessState的初始化包括调用open_driver初始化mDriveFD成员变量，调用mmap讲Binder设备映射到进程的地址空间。

##### 获取servicemanager的代理对象
> servicemanager的类层次结构主要由四大部分组成
> 1. Binder通信接口：主要用IBinder，BBinder, BpBinder组成。IBinder定义了Binder通信的接口，BBinder是Service对应的Binder对象，BpBinder是client端访问BBinder的代理对象，负责打开binder设备与服务端通信
> 2. Binder服务接口：定义了client端可以访问server端提供的哪些服务，由IServiceManager提供。
> 3. Proxy: 主要由BpInterface和BpServiceManager实现。BpServiceManager实现了接口声明中的方法，BpInterface继承自BpRefBase, 其成员变量mRemote中存储client创建的BpBinder对象。
> 4. Stub: 主要由BpInterface和BnServiceManager实现。系统中并未真正的使用BnServiceManager，而是直接以servicemanager进程作为server端。
>
> defaultManagerService用于获取servicemanager的proxy对象，位于framworks/native/libs/binder/IServiceManager.cpp中
> 1. ProcessState::getContextObject方法，参数为null。这部分返回上下文管理者对应的BpBinder，该对象是一个服务代理对象。
> 2. 将BpBinder封装为client端易于使用的BpServiceManager对象(interface_cast，位于/framworks/native/include/binder/IInterface.h中，是一个泛型方法，可以看成inline sp<IServiceManager> interface_cast(const sp<IBinder>& obj) { return IServiceManager::asInterface(obj);})，IServiceManager中的asInterface方法是通过IInterface.h的DECLARE_META_INTERFACE宏定义，通过IMPLEMENT_META_INTERFACE宏实现的。其实现调用了BpBinder.queryLocalInterface返回null，所以通过BpServiceManager构造方法(IServiceManager.cpp)，将BpBinder传给了BpInterface(IInterface.h)，接着到BpRefBase(Binder.h)。即IServiceManager::asInterface最终将BpBinder存入了BpServiceManager的父类BpRefBase的私有变量mRemote中。

##### 注册Service
> addService其实就是把Binder通信请求写入Parcel类型的data中，然后调用之前的BpBinder对象的transact方法，最后在Parcel类型的reply中获得返回结果。
>
> BpBinder.transact最终调用了IPCThreadState.transact完成addService请求。最终会通过binder_open函数唤醒servicemanager，处理addService。

##### Server进程开启线程池
> ProcessState.startThreadPool->spawnPooledThread->IPCThreadState.joinThreadPool

### client端使用服务代理对象
> frameworks/av/media/libmedia/AudioSystem.cpp中定义了一个get_audio_flinger方法使用AudioFlinger服务，调用了BpServiceManager.getService方法返回服务，最终通过service_manager的do_find_service方法，通过Binder ioctl命令返回一个BpAudioFlinger, 服务的proxy对象

### 服务代理与服务通信
> 通过IPCThreadState.executeCommand->BBinder.transact->AudioFlinger.onTransact->BnAudioFlinger.onTransact(IAudioFlinger.cpp中)