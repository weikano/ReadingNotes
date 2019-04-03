### 第06章-进程间通信-Binder

Binder驱动→路由器

ServiceManager→DNS

Binder client→客户端

Binder server→服务器

#### 6.1 智能指针

//TODO

#### 6.2 进程间数据传递载体-Parcel

Parcel.java中的mNativePtr指向Parcel.cpp中的Parcel对象

Binder在构造函数中通过getNativeBBinderHolder方法讲mObject指向native层的JavaBBinderHolder对象（android_util_Binder.cpp中）

##### 6.1 Parcel设置先关

- dataSize：当前已经存储的数据大小
- setDataCapacity：设置Parcel的空间大小
- setDataPosition：改变Parcel的读写位置
- dataAvail：当前Parcel中 的可读数据大小
- dataCapacity：当前Parcel的存储能力
- dataPosition：当前的位置
- dataSize：Parcel包含的数据大小

##### 6.2 ActiveObjects

- Binder：利用Parcel将Binder写入，读取时就能取得原始的Binder对象或者他的特殊代理实现。先通过Parcel.writeInterfaceToken(String)写入Binder的DESCRIPTOR，然后调用writeStrongBinder写入Binder对象。
- FileDescriptor：Linux中的文件描述符。

#### 6.3 Binder驱动与协议

Binder驱动初始化在aosp/kernel/goldfish/drivers/android/binder.c中

```c
device_init(binder_init);
static int __init binder_init(void) {
  init_binder_device(name);
}
static int __init init_binder_device(const char* name) {
  binder_device->miscdev.fosp = &binder_fops;
  //...
  ret = misc_register(&binder_device->miscdev);
}
static const struct file_operations binder_fops = {
  .owner = THIS_MODULE,
  .poll = binder_poll,
  .unlocked_ioctl = binder_ioctl,
  .compat_ioctl = binder_ioctl,
  .mmap = binder_mmap,
  .open = binder_open,
  .flush = binder_flush,
  .release = binder_release
};
```

代码在native/cmds/servicemanager/binder.c中

##### 6.3.1 打开Binder驱动-binder_open、binder_mmap

```c++
struct binder_state *binder_open(const char* driver, size_t mapsize) {
  struct binder_state* bs;
  bs = malloc(sizeof(*bs));
  //打开驱动文件
  bs->fd = open(driver,0,O_RDWR|O_CLOEXEC);
  bs->mapsize = mapsize;
  //内存映射
  bs->mapped = mmap(NULL, mapsize, PROT_READ, MAP_PRIVATE, bs->fd, 0);
  return bs;
}
```

#### 6.4 DNS服务-servicemanager（Binder server）

##### 6.4.1 ServiceManager的启动

init.rc中的init中会start servicemanager，代码在frameworks/native/cmds/servicemanager中

##### 6.4.2 ServiceManager的构建

```c++
//frameworks/native/cmds/servicemanager/service_manager.c
int main(int argc, char** argv) {
  char* driver = "/dev/binder";
  struct binder_state* bs = binder_open(driver,128 *1024);
  binder_become_context_manager(bs);
  //进入循环，等待客户的请求
  //交给Binder_parse方法调用svcmgr_handler方法指针来处理
  binder_loop(bs, svcmgr_handler);
}


int binder_become_context_manager(struct binder_state* bs) {
  return ioctl(bs->fd, BINDER_SET_CONTEXT_MGR, 0);
}
//framworks/native/cmds/servicemanager/binder.c
struct binder_state* binder_open(const char* driver, size_t mapsize) {
  struct binder_state* bs;
  bs = malloc(sizeof(binder_state*));
  bs->fd = open(driver, O_RDWR | O_CLOEXEC);
  bs->mapsize = mapsize;
  bs->mapped = mmap(NULL, mapsize, PROT_READ, MAP_PRIVATE, bs->fd, 0);
  return bs;
}

int binder_parse(struct binder_state* bs, struct binder_io* bio, uintptr_t ptr, size_t size, binder_handler func) {
  //忽略其他，只考虑cmd = BR_TRANSACTION
  struct binder_transaction_data* txn = (struct binder_transaction_data*) ptr;
  if(func) {
    struct binder_io msg;
    struct binder_io reply;
    unsigned rdata[256/4];
    int res;
    bio_init(&reply, rdata, sizeof(rdata),4);
    bio_init_from_txn(&msg, txn);
    res = func(bs,txn, &msg, txn);
  }
}
```

##### 6.4.3 获取ServiceManager服务-设计思考

如果要访问SM（BinderServer）的服务，流程是怎样的呢？

- 打开Binder设备
- 执行mmap
- 通过Binder驱动想SM（BinderServer）发送请求（SM的handle值为0）
- 获得结果

核心工作如上，但下面是一些细节要考虑的：

- 向SM发送请求的Binder Client可能是apk应用程序，所以SM必须提供Java层接口
- 如果每个Binder Client都要自己执行以上步骤来获取SM服务，那么会浪费不少时间。Android提供了封装。
- 如果应用程序代码中每次使用SM服务或者其他BinderServer服务，都需要打开一次Binder驱动执行mmap，那么很快就会把系统资源小号殆尽。所以是每个进程只允许打开一次Binder驱动且只做一次mmap。

那么如何设计这样一个Binder Client：

###### 1. ProcessState和IPCThreadState

ProcessState：管理每个应用进程中的Binder操作。与Binder驱动进行实际命令通信的是IPCThreadState

###### 2. Proxy

Android将与Binder驱动通信的功能交由ServiceManagerProxy来实现

##### 6.4.4 ServiceManagerProxy

ServiceManager是在ServiceManagerProxy中又做了一层封装

```java
public static IBinder getService(String name) {
  try {
    IBinder service = sCache.get(name);
    if(service != null) {
      return service;
    }
    return getIServiceManager().getService(name);
  }catch(RemoteException e) {
    
  }
  return null;
}

private static IServiceManager getIServiceManager() {
  return ServiceManagerNative.asInterface(BinderInternal.getContextObject());
}
```

BInderInternal.getContextObject方法会触发native层的方法

```c++
//android_util_Binder.cpp
static jobject android_os_BinderInternal_getContextObject(JNIEnv* env, jobject clazz) {
  //javaObjectForIBinder最终会生成一个BinderProxy对象，它的mNativeData指向native层的BinderProxyNativeData对象
  
  return javaObjectForIBinder(env, ProcessState::self()->getContextObject(NULL));
}
```

ServiceManagerNative.asInterface方法中的asInterface

```java
static public IServiceManager asInterface(IBinder obj) {
  IServiceManager in = obj.queryLocalInterface(descriptor);
  if(in != null) {
    return in;
  }
  return new ServiceManagerProxy(obj);
}
//Binder.java
public IInterface queryLocalInterface(String descriptor) {
  //mOwner对象是通过attachInterface设置的，attachInterface是在ServiceManagerNative的构造函数中调用
  if(TextUtils.equals(mDescriptor, descriptor)) {
    return mOwner;
  }
  return null;
}
```

作为SM的代理，ServiceManagerProxy必定是要与Binder驱动通信的，因为它的构造函数中传入了IBInder。分析getService方法，实际工作只有

```java
mRemote.transact(GET_SERVICE_TRANSACTION, data, reply, 0);
//GET_SERVICE_TRANSACTION对应的是native/cmds/servicemanager/binder.h中的SVC_MGR_GET_SERVICE = 1这个枚举值
//mRemote其实是BinderProxy, transact方法交由transactNative，对应的jni方法是android_util_Binder.cpp中的android_os_BinderProxy_transact方法
```

##### 6.4.5 IBinder和Binder

IBinder对象是通过BinderInternal.getContextObject方法获取，参考上述内容，最后返回的是BinderProxy（Java），BinderProxyNativeData(android_util_Binder.cpp)。

由mRemote.transact解析可知，Java层的IBinder.transact最终交由android_os_BinderProxy_transact方法处理

```c++
static jboolean android_os_BinderProxy_transact(env, obj, jint code, jobject dataObj, jobject replyObj, jint flags) {
  Parcel* data = parcelForJavaObject(env, dataObj);
  Parcel* reply = parcelForJavaObject(env, replyObj);
  //mObject指向BpBinder
  IBinder* target = getBPNativeData(env, obj)->mObject.get();
  status_t err = target->transact(code, *data, reply, flags);
}
//native/libs/binder/BpBinder.cpp
status_t BpBinder::transact(uint32_t code, const Parcel& data, Parcel* reply, uint32_t flags) {
  IPCThreadState::self()->transact(mHandle, code, data, reply, flags);
}
```

##### 6.4.6 ProcessState和IPCThreadState

```c++
//native/libs/binder/ProcessState.cpp
sp<ProcessState> ProcessState::self() {
  Mutex::Autolock _l(gProcessMutex);
  if(gProcess != NULL) {
    return gProcess;
  }
  gProcess = new ProcessState(kDefaultDriver);
  return gProcess;
}
ProcessState::ProcessState(const char* driver)
  :mDriverName(String8(driver))
  ,mDrivderFD(open_driver(mDriverName))//打开驱动并设置最大线程数
  //..忽略
  {
    mVMStart = mmap(nullptr, BINDER_VM_SIZE, PROT_READ, MAP_PRIVATE|MAP_NORESERVE, mDriverFD, 0);
  }    
```

接着查看ProcessState的getContextObject方法，会调用getStrongProxyForHandle

```c++
sp<IBInder> getStrongProxyForHandle(int32_t handl
e) {
	//从vector表mHandleToObject中查找已经建立的先关的Binder信息，如果找不到会自动添加一个，所以一般情况下e都不会为nullptr
  handle_entry* e = lookupHandleLocked(handle);
 	IBinder* b = e->binder;
  if(b == nullptr || !e->refs->attempIncWeak(this)) {
    if(handle == 0) {
      Parcel data;
      IPCThreadState::self()->transact(0, IBinder::PING_TRANSACTION, data, nullptr, 0);
    }
    b = BpBinder::create(handle);
    e->binder = b;
    result = b;
  }
  return result;
}
```

BpBInder是native层的Binder代理，最后会转化为Java层的BInderProxy

```c++
BpBinder::create(int32_t handle)
  :mHandle(handle)//如果是SM，那么handle为0
  ,isAlive(1),
	//忽略
{
  extendObjectLifetime(OBJECT_LIFETIME_WEAK);
  IPCThreadState::self()->incWeakHandle(handle);
}
```

IPCThreadState解析

```c++
//最开始gHaveTLS为false，然后调用pthread_key_create方法后跳转至restart，
//然后new IPCThreadState
IPCThreadState* IPCThreadState::self() {
  if(gHaveTLS) {//最开始为false
restart:
    const pthread_key_t k = gTLS;
    IPCThreadState* st = (IPCThreadState*)pthread_getspecific(k);
    if(st) return st;
    return new IPCThreadState;    
  }
  if(!gHaveTLS) {
    int key_create_value = pthread_key_create(&gTLS, threadDestructor);
    gHaveTLS = true;
  }
	goto restart;
}

IPCThreadState IPCThreadState::IPCThreadState()
  :mProcess(ProcessState::self()),
	 //忽略
{
  pthread_setspecific(gTLS, this);
  clearCaller();
  //mIn, mOut都是Parcel
  mIn.setDataCapacity(256);
  mOut.setDataCapacity(256);
  //该方法与IPCThreadState::self逻辑一样，最后也是返回一个new IPCThreadStateBase
  mIPCThreadStateBase = IPCThreadStateBase::self();
}

IPCThreadStateBase IPCThreadStateBase::IPCThreadStateBase() {
  pthread_setspecific(gTLS, this);
}
    
```

上述以ServiceManager.getService方法来追踪代码，调用流程如下：

SerivceManagerProxy.getService->BInderProxy.transact->BpBinder.transact->IPCThreadState.transact。

针对getService、transact函数中各参数分别为：

- handle为0
- code为GET_SERVICE_TRANSACTION
- data：data.writeInterface(IServiceManager.descritpro); data.writeString(name);//这里的name是要查询的服务，比如window表示WindowManagerService
- flags为0

```c++
//IPCThreadState.cpp
status_t IPCThreadState::transact(int32_t handle, uint32_t code, const Parcel& data, Parcel* reply, uint32_t flags) {
  flags |= TF_ACCEPT_FDS;
  //打包成Binder协议binder_transaction_data，然后将其写入mOut中
  err = writeTransactionData(BC_TRANSACTION, flags, handle, code, data, nullptr);
  //TF_ONE_WAY表示当前业务是异步的，不需要等待
  if((flags & TF_ONE_WAY) == 0) {//非异步
    //waitForResponse
    //1. talkWithDriver，讲数据包装后发送给Binder驱动
    if(reply) {
      err = waitForResponse(reply);
    }else {
      Parcel fakeReply;
      err = waitForResponse(fakeReply);
    }    
  }
}

status_t IPCThreadState::waitForResponse(Parcecl* reply, status_t * acquireResult) {
  uint32_t cmd;
  int32_t err;
  while(1) {//循环等待结果
    //处理与Binder之间的交互命令
    if((err=talkWithDriver())<NO_ERROR) break;
    //cmd包含BR_NOOP和BR_TRANSACTION_COMPLETE
    switch(cmd) {
      case BR_TRANSACTION_COMPLETE:
        goto finish;
        break;
      default:
        //实际啥都没做
        err = executeCommand(cmd);
        goto finish;
        break;
    }
  }
}

statu_t IPCThreadState::talkWithDriver(bool doReceive) {
  binder_write_read bwr;
  const bool needRead = mIn.dataPosition()>=mIn.dataSize();
  //当mIn需要去读取，同时调用者又希望读取时(doReceive=true)，我们就不能写mOut
  const size_t outAvail = (!doReceive || needRead) ? mOut.dataSize():0;
  bwr.write_size = outAvail;
  bwr.write_buffer = (uintprt_t)mOut.data();
  do {
    //与Binder驱动通信
    if(ioctl(mProcess->mDriverFD,BINDER_WRITE_READ, &bwr) >= 0) {
      err = NO_ERROR;
    }    
    if(mProcess->mDriverFD <=0 ) {
      errr = -EBADF;
    }
  }while(err == -EINTR);
  if(err>=NO_ERROR) {
    //Binder驱动消耗了mOut中数据，因此要移除掉这部分数据
    if(bwr.write_consumed > 0) {
      if(bwr.write_consumed < mOut.dataSize()) {
        mOut.remove(0, bwr.write_consumed);
      }else {
        mOut.setDataSize(0);
        processForWriteDerefs();
      }
    }
    if(bwr.read_consumed>0) {
      mIn.setDataSize(bwr.read_consumed);
      mIn.setDataPosition(0);
    }
  }
}
```

接下来分析下与Binder驱动通信的ioctl函数

```c++
//aosp/kernel/goldfish/drivers/android/binder.c，从fops中可以得知ioctl最终调用了binder_ioctl方法，只看其中的BINDER_WRITE_READ实现
static long binder_ioctl(struct file* filp, unsigned int cmd, unsigned long arg) {
  struct binder_proc* proc = filp->private_data;
  struct binder_thread* thread;
  int ret;
  unsigned int size = IOC_SIZE(cmd);//命令的大小
  void __user* ubuf = (void __user*)arg;
  ret = wait_event_interruptible(binder_user_error_wait, binder_stop_on_user_errror<2);
  //向proc的threads链表查询是否已经添加了当前线程的节点，如果没有则插入一个表示当前线程的新节点。threads链表是按pid大小排序的
  thread = binder_get_thread(proc);
  switch(cmd) {
    case BINDER_WRITE_READ:
      ret = binder_ioctl_write_read(filp, cmd, arg, thread);
      break;
  }
  //忽略……
  return ret;
}

static int binder_ioctl_write_read(struct file* flip, unsigned int cmd, unsigned long arg, struct binder_thread* thread) {
  int ret = 0;
  struct binder_proc* proc = flip->private_data;
  unsigned int size = IOC_SIZE(cmd);
  void __user* ubuf = (void __user*)arg;
  struct binder_write_read bwr;
  //从用户空间复制数据
  if(copy_from_user(&bwr, ubuf, sizeof(bwr))) {
    ret = -EFAULT;
    goto out;
  }
  if(bwr.write_size>0) {//有数据需要写
    ret = binder_thread_write(proc, thread, bwr.write_buffer, bwr.write_size, &bwr.write_consumed);
    if(ret < 0) {//成功执行返回0， 附属表示有错误
      bwr.read_consumed = 0;
      if(copy_to_user(ubuf, &bwr, sizeof(bwr))) {
        ret = -EFAULT;
      }
      goto out;
    }
  }
  //这时候ServiceManager已经被唤醒了
  if(bwr.read_size>0) {//有数据需要读
    //走到BINDER_WORK_TRANSACTION_COMPLETE,执行完成后回到IPCThreadState.talkWithDriver方法，最后回到waitForResponse
    //这里读取了两个命令，BR_NOOP和BR_TRANSACTION_COMPLETE
    ret = binder_thread_read(proc, thread, bwr.read_buffer, bwr.read_size, &bwr.read_consumed, flip->f_flags&0_NONBLOCK);
    if(!binder_worklist_empty_ilocked(&proc->todo)) {
      binder_wakeup_proc_ilocked(proc);
    }
  }
  if(copy_to_user(ubuf, &bwr, sizeof(bwr))) {
    ret = -EFAULT;
    goto out;
  }
out:
  return ret;
}
//接下来分析binder_thread_write
//在getService这个场景下:
//proc为调用者进程
//thread为调用者线程
//buffer为bwr.write_buffer。在talkWithDriver中，被设置为bwr.write_buffer = (long unsigned int )mOut.data();因而就是writeTransaction中对mOut写入的数据，主要有两部分：
//mOut.writeInt32(BC_TRANSACTION);mOut.write(&tr, sizeof(tr));tr为binder_transaction_data；
//size为bwr.write_size
//consumed为bwr.write_consimed;
static int binder_thread_write(struct binder_proc* proc, struct binder_thread* thread, binder_uintptr_t binder_buffer, size_t size, binder_size_t *consumed) {
  uint32_t cmd;
	struct binder_context *context = proc->context;
	void __user *buffer = (void __user *)(uintptr_t)binder_buffer;
  //ptr、end分别为要处理的数据起点和终点
	void __user *ptr = buffer + *consumed;//忽略已经处理过的部分
	void __user *end = buffer + size;//buffer尾部
  //因为buffer可能包含多个cmd，所以每个cmd都要处理
  while(ptr < end && thread->return_error.cmd == BR_OK) {
    if(get_user(cmd, (uint32_t __user*)ptr)) {//获取一个命令cmd
      return -EFAULT;
    }
    ptr+=sizeof(uint32_t);//跳过cmd所占的空间大小
    //统计数据
    if (_IOC_NR(cmd) < ARRAY_SIZE(binder_stats.bc)) {
			atomic_inc(&binder_stats.bc[_IOC_NR(cmd)]);
			atomic_inc(&proc->stats.bc[_IOC_NR(cmd)]);
			atomic_inc(&thread->stats.bc[_IOC_NR(cmd)]);
		}
    switch(cmd) {
        //忽略其他……
      case BC_TRANSACTION:
      case BC_REPLY_SG: {
        struct binder_transaction_data_sg tr;
        //从用户空间拷贝数据到tr中
        if(copy_from_user(&tr, ptr, sizeof(tr))) {
          return -EFAULT;
        }
        //跳过tr占的空间继续执行while
        ptr+= sizeof(tr);
        binder_transaction(proc, thread, &tr.transaction_data, cmd == BC_REPLY_SG, tr.buffers_size)
       	break; 
      }        
    }
  }
}

//reply为false
static void binder_transaction(struct binder_proc* proc, struct binder_thread* thread, struct binder_transaction_data *tr, int reply, binder_size_t extra_buffer_size) {
  //target_proc目标对象所在的进程。在getService这个场景中，即ServiceManager。获取target_proc是该方法的重点之一
  //target_thread目标对象所在的线程
  //t 一个transaction操作
  //tcomplete 一个未完成的操作。因为一个transaction通常设计到两个进程A和B。当A向B发送请求后，B需要一段时间来执行；此时对A来说就是未完成的操作，直到B返回结果后，Binder驱动才会启动继续执行
  //这里跳过reply为true的情况
  //第一步，获取target_node，binder_get_node_refs_for_txn获取target_proc=target_node->proc和target_node
  if(tr->target.handle) {//handle不为0
    struct binder_ref *ref;
    ref = binder_get_ref_olocked(proc, tr->target.handle, true);
    target_node = binder_get_node_refs_for_txn(ref->node, &target_proc, &return_error);
  }else {//代表是ServiceManager
    target_node = context->binder_context_mgr_node;
  }
  //生成一个binder_transaction
  t = kzalloc(sizeof(*t), GFP_KERNEL);
  //生成一个未完成binder_work变量，用来说明一个未完成的transaction，最后会被添加到本线程的TODO队列
  tcomplete = kzalloc(sizeof(*tcomplete), GFP_KERNEL);
  //填写t的信息
  f (!reply && !(tr->flags & TF_ONE_WAY))
		t->from = thread;
	else
		t->from = NULL;
	t->sender_euid = task_euid(proc->tsk);
	t->to_proc = target_proc;//ServiceManager对应的进程和线程
	t->to_thread = target_thread;
	t->code = tr->code;//GET_SERVICE_TRANSACTION
	t->flags = tr->flags;
	if (!(t->flags & TF_ONE_WAY) &&
	    binder_supported_policy(current->policy)) {
		/* Inherit supported policies for synchronous transactions */
		t->priority.sched_policy = current->policy;
		t->priority.prio = current->normal_prio;
	} else {
		/* Otherwise, fall back to the default priority */
		t->priority = target_proc->default_priority;
	}
  //申请的内存你区域，也就是Binder驱动mmap所管理的内存区域
	t->buffer = binder_alloc_new_buf(&target_proc->alloc, tr->data_size,
		tr->offsets_size, extra_buffers_size,
		!reply && (t->flags & TF_ONE_WAY));
  off_start = (binder_size_t *)(t->buffer->data +
				      ALIGN(tr->data_size, sizeof(void *)));
	offp = off_start;
  //t->buffer所指向的内存空间和目标对象是共享的，所以只需要复制一次就把数据从Binder Client复制到了Binder Server中了
  if (copy_from_user(t->buffer->data, (const void __user *)(uintptr_t)
			   tr->data.ptr.buffer, tr->data_size)){}
  if (copy_from_user(offp, (const void __user *)(uintptr_t)
			   tr->data.ptr.offsets, tr->offsets_size)){}
  //接下来的for循环用来处理数据中的binder_object对象。getSrevice是向ServiceManager查询一个对象，可以跳过
  //reply为false，没有TF_ONE_WAY
  t->need_reply = 1;
		t->from_parent = thread->transaction_stack;
		thread->transaction_stack = t;
  //接着加入todo队列和唤醒proc，即ServiceManager
  //binder_wakeup_thread_ilocked->wake_up_interruptible(target_thread->wait)
  binder_proc_transaction(t,target_proc, target_thread)
}

```

ServiceManager被唤醒后，分两条路进行

1. Binder Client端的处理：调用wait_event_freezable进入等待
2. ServiceManager被唤醒后的操作：此时SM在wait_event_freezable_exclusive进入睡眠等待中。

```c++
//native/cmds/servicemanager/binder.c
res = ioctl(bs->fd, BINDER_WRITE_READ, &buf);
//func为svcmgr_handler,调用do_fund_services返回ptr，最后要转换成IBinder的那个指针，接着把ptr写入回复bio_put_ref(reply, ptr);
res = binder_parse(bs,0,bwr.read_consumed, func);
//ptr是什么值呢，分析addService方法，最终也是调用了svcmgr_handle中的SVC_MGR_ADD_SERVICE
//handle即是ptr
//bio_get_ref产生的对象的最终类型是BINDER_TYPE_HANDLE, ptr是Binder的句柄
handle = bio_get_ref(msg);
do_add_service(bs, s, len, handle, txn->sender_euid, allow_isolated, dumpsys_priority, txn->sender_pid);
```

TODO ，太复杂了

#### 6.5 Binder客户端-Binder Client



