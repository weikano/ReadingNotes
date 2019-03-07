### 第14章-android应用程序的键盘消息处理机制

键盘事件由系统统一监控。键盘消息与普通消息的区别在于前者是由硬件中断触发，后者由APP自己产生。

android的键盘消息统一由WindowManagerService管理。WindowServiceManager在启动时会创建一个InputManager来监控键盘事件。APP窗口和InputManager之间的连接是通过一对InputChannel描述的。

#### 14.1 键盘消息处理模型

- Java层中WindowManagerService中InputManageService的创建，其中native方法是在frameworks/base/services/core/jni/onload.cpp中的JNI_OnLoad中通过frameworks/base/services/core/jni/com_android_server_input_InputManagerService.cpp中的register_android_server_InputManager方法关联

```kotlin
//SystemServer.java
private fun startOtherService() {
	val inputManager = InputManagerService(context)
	val wm = WindowManagerService.main(context,inputManager,xxx,xxx,xxx);  
}
private val mPtr:Long = 0L //通过nativeInit指向C++层的NativeInputManager
```

- 在C++层中，有一个NativeInputManager对象，它的mInputManager指向C++层中的InputManager。C++层中的InputManager的mReader指向InputReaderInterface，mDispatcher指向InputDispatcherInterface。二者运行在独立的线程中。

  > 1. InputDispacherInterface实现类是InputDispatcher(frameworks/native/services/inputflinger/InputDispatcher.h(cpp))，用来分发键盘事件给系统当前激活的APP应用窗口。InputDispatcher包含一个Looper和mFocusedWindow。looper用来和InputReader通信，mFocusedWindow描述系统当前激活的APP窗口。
  > 2. InputReaderInterface实现类是InputReader，用来监控系统的键盘事件。（frameworks/native/services/inputflinger/InputReader.h(cpp)）

当APP窗口被激活时，WMS会调用InputManagerService#setInputWindows方法将它设置到mFocusedWindow，然后通过nativeSetInputWindows调用InputDispatcher::setInputWindows保存至mFocusedWindow中，然后调用InputReader::requestRefreshConfiguration方法，触发EventHub::wake方法。

```c++
//frameworks/native/services/inputflinger/EventHub.cpp
//EventHub是真正用来监控系统键盘事件的
EventHub::EventHub(void) {
  mEpollFd = epoll_create(EPOLL_SIZE_HINT);
  int wakeFds[2];
  pipe(wakeFds);
  //epoll+pipe
  //read用于EventHub::getEvents中接收事件
  mWakeReadPipeFd = wakeFds[0];  
  mWakeWritePipeFd = wakeFds[1];
}

void EventHub::wake() {
  do {
    nWrite = write(mWakeWritePipeFd, "W", 1);
  }while(nWrite != -1 && errorno == EINTR);
}
```

SystemServer在WMS在创建完成后，会在wms构造函数中触发InputManagerService.monitorInput方法，然后调用nativeRegisterinputChannel将InptuChannel关联起来。接着SystemServer会调用InputManagerService.start方法，触发InputManagerService.nativeStart。

```c++
//frameworks/base/services/core/jni/com_android_server_input_InputManagerService.cpp
static void nativeStart() {
  im->getInputManager()->start();
}
//frameworks/native/services/inputflinger/InputManager.cpp
//frameworks/native/services/inputflinger/InputReader.cpp
//frameworks/native/services/inputflinger/InputDispatcher.cpp
status_t InputManager::start(){
  //启动InputDispatcher和InputReader需要的两个线程
  //触发InputDispatcher->dispatchOnce();
  mDispatcherThread->run("InputDispatcher", PRIORITY_URGENT_DISPLAY);
  //触发InputReader->loopOnce();
  mReaderThread->run("InputReader", PRIORITY_URGENT_DISPLAY);
}
```

InputDispatcher启动之后会不断调用dispatchOnce来检查InputReader是否给它分发了键盘事件。如果没有，调用mLooper.pollOnce进入睡眠等待，直到被InputReader唤醒位置。

InputReader会通过loopOnce调用EventHub->getEvents方法监控系统是否有键盘事件发生。有键盘事件发生时，会调用InputDispatcher.notifyKey来唤醒InputDispatcher。

APP窗口通过注册两个InputChannel来与InputManager建立连接，一个server一个client。server要注册到InputManager中，client要注册到APP主线中。

```kotlin
//WindowManagerService.java
//PointerEventDispatcher构造函数中会调用InputEventReceiver.nativeInit，返回一个NativeInputEventReceiver地址给mReceiverPtr，NativeInputEventReceiver包含一个C++层InputChannel对象（保存在InputConsumer）
class WindowManagerService() {
  init {
    val inputChannel = mInputManager.monitorInput(TAG_WH)
    mPointerEventDispatcher = if(inputChannel != null) PointerEventDispatcher(inputChannel) else null
  }
}
//InputManagerService.java
fun monitorInput(name:String):InputChannel {
  //调用InputChannel.nativeOpenInputChannelPair(name);
  //frameworks/base/core/jni/android_view_InputChannel.cpp#android_view_InputChannel_nativeOpenInputChannels()，创建两个C++层的InputChannel对象;
  val inputChannels = InputChannel.openInputChannels(name);
  nativeRegsiterinputChannel(mPtr, inputChannels[0], null, true);
  inputChannels[0].dispose();
  return inputChannels[1];
}
```

C++中InputChannel包含三个文件描述符mAshmemFd, mReceivPipeFd和mSendPipeFd。mAshmemFd用来在APP窗口与inputManager中传递数据的; 其余两个用来描述两个不同的管道的读写端，用来在APP窗口和InputManager中建立交叉连接。

C++中的InputWindow用来描述一个APP窗口，含有一个InputManager变量。

```c++
//frameworks/base/services/core/jni/com_android_server_input_InputManagerService.cpp
//frameworks/native/include/input/InputTransport.h，frameworks/native/input/InputTransport.cpp中包含InputChannel
```

#### 14.2 InputManager的启动过程

##### 14.2.1 创建InputManager

SystemServer.startOtherServices中通过构造函数创建一个InputManagerService，mPtr=nativeInit，指向NativeInputManager。NativeInputManager的构造函数会生成一个C++层的InputManager和EventHub。InputManager构造函数会生成InputDispatcher和InputReader，并调用initialize方法生成inputReaderThread和InputDispatchThread对象。

> frameworks/base/services/core/jni/com_android_server_input_InputManagerService.cpp
>
> frameworks/native/services/inputflinger/InputManager.cpp

##### 14.2.2 启动InputManager

SystemServer.startOtherService中通过InputManager.start()方法触发nativeStart，再到com_android_server_input_InputManager.cpp的nativeStart，再到C++层中的InputManager的start，再到inputReaderThread和InputDispatchThread运行。最终会调用InputDispatcher->dispatchOnce和InputReader->loopOnce

##### 14.2.3 启动InputDispatcher

InputDispatcher.dispatchOnce。调用dispatchOnceInnerlLocked用来检查系统当前是否有新的键盘事件需要分发。如果没有，那么它需要隔一段时间后再来检查是否有新的键盘事件需要分发，这段事件内InputDispatcher会进入睡眠等待。

调用runCommandsLockedInterruptible来检查InputDispatcher中是否有一些未执行的缓存命令。

假设系统当前没有键盘事件发生，那么InputDispatcher在启动之后会马上进入睡眠等待，知道被InputReader唤醒位置。

##### 14.2.4 启动InputReader

InputReader的启动是通过loopOnce方法

```c++
//frameworks/native/services/inputflinger/InputReader.cpp
private:
RawEvent mEventBuffer[EVENT_BUFFER_SIZE];
void InputReader::loopOnce() {
  size_t count = mEventHub->getEvents(timeoutMillis, mEventBuffer, EVENT_BUFFER_SIZE);
  processEventsLocked(mEventBuffer, count);
}

//frameworks/native/services/inputflinger/EventHub.cpp
EventHub::getEvents(int timeout, RawEvent* buffer, size_t bufferSize) {
  for(;;){
    //mNeedToReopenDevices = false
    //mClosingDevices = false
    //mNeedToScanDevices = true
    //scanDevices最终会调用openDeviceLocked，打开/dev/input，然后registerDeviceForEpollLocked和addDeviceLocked
    scanDevicesLocked();
  }
}
//略……
```

#### 14.3 InputChannel的注册过程

一个APP窗口是由一个activity组件描述的。每个activity都有一个关联的ViewRootImpl对象。每个ViewRootImpl对象除了负责绘制activity组件描述的APP窗口外，还负责接收InputDispatcher分发过来的键盘事件。

每个Activity组件含有一个PhoneWindow对象，PhoneWindow中有DecorView。每个DecorView又会被设置到ViewRootImpl中的mView中（setView）。

一个activity组件想要接收到系统的键盘事件，需要主动与InputManager建立一个连接。连接是通过一个server端的InputChannel和一个client端的InputChannel描述的。

##### 14.3.1 创建InputChannel

SystemServer中调用WindowManagerService.main()方法通过WMS的构造函数时，会调用InputManager.monitorInput方法创建InputChannel。即：

InputManager.monitorInput->InputChannel.openInputChannelPari->InputChannel.nativeOpenInputChannelPair->InputManagerService.nativeRegisterInputChannel

ViewRootImpl.setView会调用mWindowSession.addToDisplay方法。mWindowSession是WindowManagerGlobal.getWindowSession()获取到，由WindowManagerService.openSession()返回的Session对象。所以mWindowSession.addToDisplay方法最后调用的还是WMS.addWindow方法。

```c++
//frameworks/base/core/jni/android_view_InputChannel.cpp
static jobjectArray android_view_InputChannel_nativeOpenInputChannelPair() {
  sp<InputChannel> serverChannel;
  sp<InputChannel> clientChannel;
  //frameworks/native/libs/input/InputTransport.cpp
  status_t result = InputChannel::openInputChannelPair(name, serverChannel, clientChannel);
  //创建一个Java层的InputChannel并关联NativeInputChannel
  jobject serverChannelObj = android_view_InputChannel_createInputChannel(env,std::make_unique<NativeInputChannel>(serverChannel));
  jobject clientChannelObj = android_view_InputChannel_createInputChannel(env,std::make_unique<NativeInputChannel>(clientChannel));
  //然后创建一个jobjectArray并把client和server赋值进去，返回jobjectArray
  jobjectArray channelPair = env->NewObjectArray(2, gInputChannelClassInfo.clazz, NULL);
  env->SetObjectArrayElement(channelPair,0, serverChannelObj);
  env->SetObjectArrayElement(channelPair,1, clientChannelObj);
  return channelPair;
}

static jobject android_view_InputChannel_createInputChannel(nativeInputChannel) {
  //创建一个java层的InputChannel对象
  jobject inputChannelObj = env->NewObject(gInputChannelClassInfo.clazz, gInputChannelClassInfo.ctor);
  //setNative方法是把NativeInputChannel对象的地址赋值给java层的InputChannel.mPtr
  android_view_InputChanenl_setNativeInputChannel(env, inputChannelObj, nativeInputChannel.release())
  return inputChannelObj;
}

//frameworks/native/libs/input/InputTransport.cpp
status_t InputChannel::openInputChannelPair(string name, sp<InputChannel> outServer, sp<InputChannel> outClient) {
  int sockets[2];//用来存储server和client的socket的fd
  setsockopt(sockets[0],SOL_SOCKET,SO_SNDBUF, &bufferSize, sizeof(bufferSize));
  setsockopt(sockets[0],SOL_SOCKET,SO_RCVBUF, &bufferSize, sizeof(bufferSize));
  setsockopt(sockets[1],SOL_SOCKET,SO_SNDBUF, &bufferSize, sizeof(bufferSize));
  setsockopt(sockets[1],SOL_SOCKET,SO_RCVBUF, &bufferSize, sizeof(bufferSize));
  //通过构造函数把fd设置给对应的outServer和outClient
  outServer = new InputChannel(serverChannelName, sockets[0]);
}
```

##### 14.3.2 注册server端InputChannel

```c++
//frameworks/base/services/core/jni/com_android_server_InputManagerService.cpp
//ptr为NativeInputManager的地址, inputChannelObj为Java层InputChannel, inputWindow null, monitor true
static void nativeRegisterInputChannel(env,jclass, jlong ptr, jobject inputChannelObj, jobject inputWindowHandleObj, jboolean monitor) {
  //获取NativeInputManager对象
  NativeInputManager* im = reinterpret_cast<NativeInputManager*>(ptr);
  //获取Native层的InputChannel对象，从Java层的InputManager.mPtr中获取
  sp<InputChannel> inputChannel = android_view_InputChannel_getInputChannel(env, inputChannelObj);
  im->registerInputChannel(env, inputChannel,xxx);
}
//frameworks/native/services/inputflinger/InputDispatcher.cpp
status_t InputDispatcher;:registerInputChannel(&inputChannel, inputWindowHandle, bool monitor) {
  //忽略前面的检测是否已注册
  //封装成connection
  sp<Connection> connection = new Connection(inputChannel, inputWindowHandle, monitor);
  //将conenction跟inputChannel的fd关联起来放到mConnectionsByFd
  mConnectionsByFd.add(inputChannel->getFd(), connection);
  //当ALOOPER_EVENT_INPUT发生时，触发handleReceiveCallback方法
  mLooper->addFd(fd,0,ALOOPER_EVENT_INPUT, handleReceiveCallback,this);
  mLooper->wake();
}
```

##### 14.3.3 注册系统当前激活的应用程序窗口

InputManagerService.setInputWindows<-InputMonitor.UpdateInputForAllWindowsConsumer.updateInputWindos<-InputMonitor.updateInputWindowsLw()<-WMS.addWindow。

InputManagerService.setInputWindows会触发nativeSetInputWindows

```c++
//frameworks/base/services/core/jni/com_android_server_InputManagerService.cpp
static void nativeSetInputWindows(env, clazz, ptr, windowHandlObjArray) {
  NativeInputManager* im = reinterpret_cast<NativeInputManager*>(ptr);
  //保存至NativeInputManager的windowHandles中
  //c++层InputDispatcher.setInputWindows(windowHandles);
  im->setInputWindows(env, windowHandleObjArray);
}
//frameworks/native/services/inputflinger/InputDispatcher.cpp
void InputDispatcher::setInputWindows(Vector<sp<InputWindowHandle>>& handles) {
  //注册到native层的InputDispatcher中
}
```

##### 14.3.4 注册client端InputChannel

WMS的构造函数中，通过将InputChannel包装成PointerEventDispatcher来注册。PointerEventDispather继承自InputEventReceiver，其构造函数中会调用nativeInit方法（frameworks/base/core/jni/android_view_InputEventReceiver.cpp）将ptr指向NativeInputEventReceiver。

```c++
//frameworks/base/core/jni/android_view_InputEventRecevier.cpp
//receiverWeak是InputEventReceiver
//inputChannelObj为Java层的InputChannel的client端,
//messageQueueObj是Java层的MessageQueue对象
static jlong nativeInit(env, clazz, receiverWeak, inputChannelObj, messageQueueObj) {
  sp<NativeInputEventRecevier> receiver = new NativeInputEventRecever(env,receiverWeak, inputChannelObj, messageQueue);
  //触发setFdEvents();
  receiver.initialize();
}

void NativeInputEventReceiver::setFdEvents(int events) {
  int fd = mInputConsumer.getChannel()->getFd();
  //会回调handleEvent方法
  mMessageQueue->getLooper()->addFd(fd, 0, events, this, NULL);
}
```

#### 14.4 键盘消息的分发过程

当系统没有键盘事件发生时，InputReader和InputDispatcher会处于睡眠状态。如果当前激活的APP窗口所运行的APP进程的主线程无事可做，就会在消息循环中等待InputDispatcher分发键盘事件。

键盘事件发生时，InputReader会被唤醒，接着InputReader再将InputDispatcher唤醒，以便将键盘事件分发给系统当前激活的APP窗口处理。这时InputDispatcher又会再次进入睡眠等待，直到键盘事件被系统当前激活的应用程序窗口处理完成为止。

系统当前激活的APP窗口并不是直接收到InputDispatcher分发的键盘事件。首先当前激活的APP窗口锁运行的APP进程的主线程接收到InputDispatcher分发过来的键盘事件，然后这个线程再将接收到的键盘事件封装成一个键盘消息发送到自己的消息队列中，最后由APP窗口处理。

APP窗口处理完成后，会唤醒InputDispatcher，以便开始新一轮的键盘事件分发工作。

##### 14.4.1 InputReader获得键盘事件

InputReader启动后会调用loopOnce等待键盘事件，**最后调用mQueueListener->flush()**，其中会经历InputReader::loopOnce->InputReader::processEventsLocked->InputReader::processEventsForDeviceLocked->InputDevice::process->InputMapper::process。即键盘事件发生后，会交给InputMapper处理，而InputMapper有很多子类：SwitchInputMapper、VirbratorInputMapper、KeyboardInputMapper、CursorInputMapper、RotaryEncoderInputMapper、(Multi/Single)TouchInputMapper、ExternalStylusInputMapper、JoystickInputMapper。由于是键盘事件，只关注KeyboardInputMapper::process事件

```c++
//frameworks/native/services/inputflinger/InputReader.cpp
void KeyboardInputMapper::process(const RawEvent* rawEvent) {
  switch(rawEvent->type) {
    case EV_KEY: {
      int32_t scanCode = rawEvent->code;
      int32_t usageCode = mCurrentHidUsage;
      mCurrentHidUsage = 0;
      //是否是一个合法的键盘按键
      if(isKeyboardOrGamepadKey(scanCode)) {
        processKey(rawEvent->when, rawEvent->value!=0, scanCode, usageCode);
      }
    }  
  }
}
//down 是否是按键按下
void KeyboardInputMapper::processKey(nsecs_t when, bool down, int32_t scanCode, int32_t usageCode) {
  //可能是按下或者松开事件，通过参数down描述。当一个按键被按下或松开时，可能有其他的按键正处于被按下的状态，这些按键保存在mKeyDowns中
  //如果是按下，那么keyCode可能要根据屏幕方向处理一下（rotateKeyCode）
  //findKeyDown事件用来判断是否重复按下同一个按键，如果是就将键盘码恢复位上一次的键盘码，否则就封装成一个KeyDown结构体，保存到mkeyDowns中。
  //updateMetaStateIfNeeded用来更新系统当前的组合键状态，Alt和shift的按下状态。
  //封装成NotifyKeyArgs，交给listener处理（InputListenr.cpp中的QueuedInputListener）
  NotifyKeyArgs args(when, getDeviceId(), mSource, polifyFlags, down?AKEY_EVENT_ACTION_DOWN : AKEY_EVENT_ACTION_UP, AKEY_EVENT_FLAG_FROM_SYSTEM, keyCode, scanCode, keyMetaState, downTime);
  getlistener()->notifyKey(&args);
}

//frameworks/native/services/inputflinger/InputListener.cpp
void QueuedInputListener::notifyKey(const NotifyKeyArgs* args) {
  mArgsQueue.push(new NotifyKeyArgs(*args));
  //由于loopOnce最后还调用了flush方法，所以最后还要触发QueuedInputListener::flush方法，交给
  //NotifyKeyArgs.notify(mInnerListener)，其中mInnerListenr是InputDispatcher，最后触发的是InputDispatcher::notifyKey方法
}
//frameworks/native/services/inputflinger/InputDispatcher.cpp
void InputDispatcher::notifyKey(const NotifyKeyArgs* args) {
  //mPolicy为NativeInputManager，intercept方法会将截获到的键盘交给WMS中的PowerManagerService处理，以便设置屏幕亮度管理电源
  KeyEvent event;
  event.initialize(xxxx);
  mPolicy->interceptKeyBeforeQueueing(&event, policyFlags);
  //封装成KeyEntry并添加到mInboundQueue队列中
  KeyEntry* newEntry;
  bool needWake = enqueueInboundEventLocked();
  //唤醒InputDispatcher
  if(needWake) {
    mLooper->wake();
  }
}
```

##### 14.4.2 InputDispatcher分发键盘事件

InputDispatcher被唤醒后会调用dispatchOnceInnerLocked分发

```c++
//frameworks/native/services/inputflinger/InputDispatcher.cpp
void InputDispatcher::dispatchOnceInnerLocked(nsecs_t* nextWakeupTime) {
  //,,,忽略前面
  switch(mPendingEvent->type) {
    case EventEntry::TYPE_KEY: {
      KeyEntry* typedEntry = ...;
      //处理appswitch情况
      done = dispatchKeyLocked(time, typedEntry,xxx);
    }  
  }
}
//省略中间,最终会调到Connection->inputPublisher.publishKeyEvent()
//frameworks/native/libs/input/InputTransport.cpp
status_t InputPublisher::publishKeyEvent() {
  return mChannel->sendMessage(&msg);
}
status_t InputChannel::sendMessage() {
  //写入一个字符signal，以便通知系统当前激活的APP窗口接收一个键盘事件
  do{    
    nWrite = ::send(mFd,msg,msgLength, MSG_DONTWAIT|MSG_NOSIGNAL);
  }while(nWrite != -1 && errno == EINTR);
}
```

##### 14.4.3 系统当前激活的APP窗口获得键盘消息

会触发InputDispatcher::handleReceiveCallback事件

```c++
//frameworks/native/services/inputflinginer/InputDispatcher.cpp
int InputDispatcher::handleReceiveCallback(int fd, int events, void* data) {
  //走到finishDispatchCycleLocked->onDispatchCyleFinishedLocked->doDispatchCycleFinishedLockedInterruptible->startDispatchCycleLocked->inputPulisher.publishKeyEvent->InputChannel::receiveMessage->InputConsumer::consume()->NativeInputEventReceiver::consumeEvents
  //最后调用NativeInputEventReceiver::consumeEvents
}
//frameworks/base/services/core/jni/android_view_InputEventReceiver.cpp
//register_android_view_InputReceiver方法中关联对应jni回调
status_t NativeInputEventReceiver::consumeEvents() {
  //省略...
  //mServiceObj InputEventReceiver,
  //gServiceClassInfo.dispatchInputEvent dispatchInputEvent方法
  env->CallObjectMethod(mServiceObj, gInputEventReceiverClassInfo.dispatchInputEvent,...,...);
}
```

**InputEventReceiver.dispatchInputEvent是个抽象类，实际是ViewRootImpl$WindowInputReceiver。WindowInputRecever.enqueueInputEvent->doProcessInputEvents->deliverInputEvent->EarlyPostImeinputState(NativePostImeStage(ViewPostImeState)).onProcess->ViewPostImeState.processKeyEvent->mView.dispatchKeyEvent**。

mView是通过ViewRootImpl.setView设置，而ViewRootImpl.setView是在WindowManagerGlobal.addView中调用的，WindowManagerImpl.addView中会调用mGlobal.addView。而Window在调用setWindowManager方法时会给mWindowManager设置一个WindowManagerImpl对象。Window的实现类是PhoneWindow。Activity.attach方法中会生成一个PhoneWindow并mWindow.setWindowManager。由此可知，上面的**mView.dispatchKeyEvent实际就是DecorView.dispatchKeyEvent事件**。

```java
//DecorView.java
public boolean dispatchKeyEvent(KeyEvent event) {
  //...
  //mWindow是PhoneWindow,cb是在Activity.attach中通过mWindow.setCallback(this)设置的  
  final Window.Callback cb = mWindow.getCallback();
 	//mFeatureId=-1表示是DecorView，所以这里往cb.dispatchKeyEvent走，而cb就是Activity，即Activity.dispatchKeyEvent
  final boolean handled = cb != null && mFeatureId<0?cb.dispatchKeyEvent(event):super.dispatchKeyEvent(event);
  if(handled) {
    return true;
  }
  return isDown?mWindow.onKeyDown(xxx):mWindow.onKeyUp(xxx);
}
```

接着是Activity.dispatchKeyEvent

```java
//Activity.java
public boolean dispatchKeyEvent(KeyEvent event) {
  onUserInteraction();
  //这里会触发Activity.onKeyDown(Up/LongPress)
  return event.dispatch(this, decor!=null?decor.getKeyDispatcherState():null, this);
}

```

最后再回到InputEventReceiver.nativeFinishInputEvent，调用frameworks/base/core/jni/android_view_InputvEventReceiver.cpp的NativeInputEventReceiver::nativeFinishInputEvent方法->NativeInputEventReceiver::consumeEvents，最后调用frameworks/native/libs/input/InputTransport.cpp中的InputConsumer::sendFinishedSignal

##### 14.4.4 InputDispacher获得键盘事件处理完成通知

略

#### 14.5 InputChannel的注销过程

##### 14.5.1 销毁应用程序窗口

Activity.finish->ActivityStackSupervisor.removeTaskByIdLocked->TaskRecord.performClearTaskAtIndexLocked->ActivityStack.finishActivityLocked->destroyActivityLocked->DestroyActiviyItem.execute->ActivityThread.handleDestroyActivity

ActivityThread.handleDestroyActivity会调用performDestroyActivity实际上就是**走一段activity关闭的生命周期，pause->stop->destroy，并且调用PhoneWindow.closeAllPanels**。然后wm.removeViewImmediate(decorView)->ViewRootImpl.die->doDie。doDie会调用dispatchDetachFromWindow->DecorView.onDetachedFromWindow。onDetachedFromWindow会触发**Activity.onDetachedFromWindow。ViewRootImpl.dispatchDetachedFromWindow还会调用unscheduleTraversals以及InputEventReceiver.dispose, InputQueue.dispose, InputChannel.dispose(实际上是这三个类的nativeDispose)**

一个APP窗口在创建时，会被封装成一个WindowState并保存。当窗口销毁时，需要通过Session.remove来实现。最终会走到WindowState.unregisterInputChannel->InputManagerService.unregisterInputChannel->nativeUnregisterInputChannel。从native层的InputDispatcher.mConnectionsByReceiveFd中已移除。

