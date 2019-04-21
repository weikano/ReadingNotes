### HermesEventBus 跨进程EventBus

#### 一、使用方法

```kotlin
//在Application中
override fun onCreate() {
  super.onCreate()
  //初始化
  HermesEventBus.getDefault().init(this)
}
//在感兴趣的地方，比如Activity中
override fun onCreate(savedInstanceState:Bundle?) {
  super.onCreate(savedInstanceState)
  HermesEventBus.getDefault().register(this)
}
override fun onDestroy() {
  HermesEventBus.getDefault().unregister(this)
}
@Subscribe(threadMode = ThreadMode.MAIN)
fun showText(text:String) {
  Log.i("HermesEventBus", "received $text")
}
```

#### 二、Hermes的原理

##### 1. 初始化

```kotlin
//HermesEventBus
//getDefault获取单例，不同进程的sInstancec不是同一个
fun init(context:Context) {
  if(mMainProcess) {
    //将Hermes.sContext 设置为applicationContext
    Hermes.init(context)
    //TypeCenter.getInstance().register(MainService::class.java)：validate(class), registerClass, registerMethod
    //根据MainService是否有被ClassId注解过，分别放进mRawClasses（未用ClassId注解）和mAnnotatedClasses中
    //将MainService中未被MethodId注解的public方法放入mRawMethods，即IMainService接口中定义的方法，注解过的方法放入mAnnotatedMethods中
    Hermes.register(MainService::class.java)
    mMainApis = MainService.getInstance()
  }else {
    mState = STATE_CONNECTING
    //给CHANNEL设置Listener，在HermesServiceConnection中会触发listener的方法
    Hermes.setHermesListener(HermesListener())
    //最终会调用CHANNEL.bind方法，Service继承自HermesService，是一个空的继承，没有override任何东西，可以看成是一个HermesService, CHANNEL.bind最终生成一个HermesServiceConnection，然后调用context.bindService(intent, connection, Context.BIND_AUTO_CREATE)
    //val connection = HermesServiceConnection(clazz)
    Hermes.connect(context, Service::class.java)
    //ISubService只有post和cancelEventDelivery两个方法
    Hermes.register(SubService::class:java)
  }
}
//bindService方法的关键代码在于它的ServiceConnection，接着分析HermesServiceConnection
//Channel.java
//mListener为HermesListener，在ServiceConnection中毁掉connect和disconnect
//mUiHandler为主线程handler
//下面的变量都是用来缓存
//mBounds用来保存bindService状态
//mHermesServices用来保存HermesService的Class对象和IHermesService之间的映射
//mHermesServiceConnections用来保存HermesService和HermesServiceConnection之间的映射
//mBindings用来保存HermesService和bindService当前是否正在进行中，进行中为true
override fun onServiceConnected(className:ComponentName, service:IBinder) {
  //忽略其他
	val hermesService = IHermesService.Stub.asInterface(service)
  hermesService.register(mHermesServiceCallback, Process.myPid())
  mListener?.onHermesConnected(mClass)
}
override fun onServiceDisconnected(className:ComponentName) {
  mHermesServices.remove(mClass)
  mListener?.onHermesDisconnected(mClass)
}
//HermesListener中的回调
override fun onServiceConnected()
```

#### 2. HermesService

```kotlin
//HermesService中的关键为mBinder
//mCallbacks为pid和IHermesCallback的映射
//OBJECT_CENTER
override fun onBind(intent:Intent):IBinder = mBinder
private val mBinder = IHermesService.Stub() {
  override fun send(mail:Mail):Reply {
    val receiver = ReceiverDesignator.getReceiver(mail.object)
    val pid = mail.pid
    val callback = mCallbacks.get(pid)
    receiver?.setHermesServiceCallback(callback)
    return receiver.action(mail.timeStamp, mail.method, mail.parameters)
  }
  override fun register(callback:IHermesServiceCallback, pid:Int) {
    mCallbacks.put(pid, callback)
  }
  override fun gc(times:List<Long>) {
    OBJECT_CENTER.deleteObjects(times)
  }
}
```

#### 3. 非Main进程中的init的HermesListener

```kotlin
//HermesEventBus$HermesListener
override fun onHermesConnected(service:Class<HermesService>) {
  //动态代理，mainService中的proxy实际为主进程中的MainService.getInstance()
  val mainService = Hermes.getInstanceInService(service, IMainService::class.java)
  mainService.register(Process.myPid(), SubService.getInstance())
  //ObjectCanary2.action方法在object==null时会阻塞，调用set方法后会唤醒执行action.call()方法，详情看post解析
  mRemoteApis.set(mainService)
  mState = STATE_CONNECTED
  hermesListener?.onHermesConnected(service)
}
override fun onHermesDisconnected(service:Class<HermesService>){
  mState = STATE_DISCONNECTED
  mRemoteApis.action(object:Action<IMainService>(){
    override fun call(o:IMainService) {
      o.unregister(Process.myPid())
    }
  })
  hermesListener?.onHermesDisconnected(service)
}
//Hermes.java
componion object {
  fun getInstanceService(service:Class<HermesService>, clazz:Class<T>, ps:Array<Any>) : T {
    return getInstanceWithMethodNameInService(service, clazz, "", parameters)
  }
  //methodName 为 ""
  //paramters长度为0
  fun getInstanceWithMethodNameInService(service, clazz, methodName, parameters) {
    val object = ObjectWrapper(clazz, ObjectWrapper.TYPE_OBJECT_TO_GET)
    //sender = InstanceGettingSender(service, object)
    val sender = SenderDesignator.getPostOffice(service, SenderDesignator.TYPE_GET_INSTANCE, object)
    val tmp = Array<Any>(ps.length+1)//paramters[0]为methodName
    tmp[0] = methodName
    //tmp后面为parameters
    val reply = sender.send(null, tmp)
    //设置TYPE_OBJECT就能使用OBJECT_CENTER中生成的MainService实例了
    object.setType(ObjectWrapper.TYPE_OBJECT)
    //通过HermesInvocationHandler动态代理，其中的mSender为ObjectSender，对应的Receiver
    return getProxy(service, object)
  }
}
//Sender.java
//method 为 null, parameters为[""]
fun send(method:Method, parameters:Array<Any>):Reply {
  //pw = [ParameterWrapper(isName = true, mClass = String::class.java, mData = "")]
  val pw = getParameterWrappers(method, parameters)
  //mMethod = MethodWrapper(isName=true, methodName = "", parameterTypes=String::class.java, mReturnType = null)
  mMethod = getMethodWrapper(method, pw)
  registerClass(method)
  setParameterWrappers(pw)
  //mObject = object，为getInstanceWIthMethodName中的object(clazz, ObjectWrapper.TYPE_GET_INSTANCE)
  val mail = Mail(time, mObjecct, mMethod, mParameters)
  //接下来交给HermesService中的mBind来调用send方法，参考2
  return CHANNEL.send(mService, mail)
}

//HermesService$mBinder
override fun send(mail:Mail) : Reply {
  //receiver = InstanceGettingReceiver(mail.object)
  val receiver = ReceiverDesignator.getReceiver(mail.object)
  val pid = mail.pid
  val callback = mCallbackcs.get(pid)
  receiver.setHermesServiceCallback(callback)
  return receiver.action(mail.timestamp, mail.method, mail.parameters)
}
//Receiver.java
fun action(time, method, params): Reply {
  //mMethod为MainService.getInstance方法
  setMethod()
  setParameters()
  //invokeMethod中会调用getINstance方法并放到OBJECT_CENTER中
  val result = invokeMethod()
  return result?:Reply(ParameterWrapper(result))
} 

```

#### 4. register

```kotlin
//不区分进程，分别都通过各自进程中的EventBus.getDefault().register方法注册，同理unregister
fun register(subscriber:Any) {
  mEventBus.register(subscriber)
}
```

#### 5. post

```kotlin
//HermesEventBus.java
fun post(event:Any) {
  actionInternal(object:Action<IMainService>(){
    override fun call(o:IMainService) {
      o.post(event)
    }
  })
}
private fun actionInternal(action:Action<IMainService>) {
  if(mMainProcess) {
    //mMainApis在init方法中生成，为MainService单例
    //即mMainApis.post(event)
    action.call(mMainApis)
  }else {
    if(mState == STATE_DISCONNECTED) {
      Log.w(TAG, DISCONNECTED_HERMES)
    }else {
      //mRemoteApis在构造函数中生成，是一个ObjectCanary2对象
      //最终也是调用MainService.post方法
      mRemoteApis.action(object:Action<IMainService>(){
        override fun call(o:IMainService) {
          action.call(o)
        }
      })
    }
  }
}
//MainService.java
override fun post(event:Any) {
  mEventBus.post(event)
  //mSubServices是在子进程中通过HermesListener.onHermesConnected中的回调中通过MainService.register方法注册进来，参考3
  mSubServices.values().forEach {
    it.post(event)
  }
}
//SubService.java
override fun post(event:Any) {
  mEventBus.post(event)
}
```



