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
void InputReader::loopOnce() {
  size_t count = mEventHub->getEvents();
}
```



