### 第13章-android应用程序的消息处理机制

android系统主要通过MessageQueue、Looper和Handler来实现消息机制。MessageQueue用来描述消息队列，Looper用来创建消息队列以及进入消息循环，Handler用来发送消息和处理消息。

#### 13.1 创建线程消息队列

android在c++层中也有一个对象的Looper类和NativeMessageQueue类。Java层的Looper类和MessageQueue类是通过c++层的looper类和NativeMessageQueue实现的。

Java层中的每个looper对象内部都有一个MessageQueue的成员变量mQueue。而在c++层中，每个NativeMessageQueue都有一个类型为Looper的成员变量mLooper，指向c++层的Looper对象。

Java层的每一个MessageQueue有一个mptr，保存了c++层中的一个NativeMessageQueue对象的地址值。

c++层中的looper对象有两个类型为int的成员变量mWakeEventFd，mEpollFd。

```c++
//system/core/libutils/looper.cpp为utils/looper.h的实现
//frameworks/base/native/android/looper.cpp
#include <android/looper.h> //定义了结构体ALooper和一些方法
#include <utils/looper.h> //定义了class Looper: public RefBase以及一些对应的方法
```

Looper.java中prepare解析

```java
//Looper.java
public static void prepare(){
  //最终会调用重载prepare方法
  sThreadLocal.set(new Looper(quitAllowed));
}
private Looper(boolean quitAllowed) {
  mQueue = new MessageQueue(quitAllowed);
  mThread = Thread.currentThread();
}
//MessageQueue.java
MessageQueue(boolean quitAllowed){
  mQuitAllowed = quitAllowed;
  //具体实现在frameworks/base/core/jni/android_os_MessageQueue.cpp中，mPtr其实是一个指向NativeMessageQueue的指针的地址
  //而NativeMessageQueue的构造函数中有创建了一个c++层的Looper对象
  mPtr = nativeInit();
}
```

```c++
//system/core/libutils/Looper.cpp
Looper::Looper(bool allowNonCallbacks) {
  //创建了一个eventfd对象，可以用做用户控件应用程序的事件等待/通知机制，内核以通知用户控件应用程序的事件
  mWakeEventFd = eventfd(0, EFD_NONBLOCK | EFD_CLOEXEC);
  rebuildEpollLocked();
}

//EPOLL_CTL_ADD：注册event事件
//EPOLL_CTL_MOD：event事件进行修改
void Looper::rebuildEpollLocked() {
  //创建一个epoll句柄
  mEpollFd = epoll_create(EPOLL_SIZE_HINT);
  struct epoll_event eventItem;
  memset(&eventItem, 0, sizeof(epoll_event));
  //EPOLLIN 读事件
  //EPOLLOUT 写事件
	//其他暂时不管  
  eventItem.events = EPOLLIN;
  eventItem.data.fd = mWakeEventFd;
  //epoll的事件注册函数，监听事件类型EPOLL_CTL_ADD
  epoll_ctl(mEpollFd, EPLL_CTL_ADD, mWakeEventFd, &eventItem);
  for(size_t i=0;i<mRequests.size();i++) {
    int epollResult = epoll_ctl(mEpollFd, EPOLL_CTL_ADD, request.fd, &eventItem);
  }
}

```

#### 13.2 线程消息循环过程

Looper.java中的loop解析

```java
//Looper.java
public static void loop(){
  for(;;){
    Message msg = queue.next();
  }
}

//MessageQueue.java
Message next() {
  //当线程发现的它的消息队列中没有新的消息要处理时，并不是马上进入睡眠等待状态，而是先调用注册到消息队列中的IdleHandler处理空闲时的操作。
  //nextPollTimeoutMillis表示当消息队列中没有新消息需要处理时，线程需要进入睡眠等待状态的时间。如果0，表示不需要进入睡眠等待。-1表示无限期睡眠，知道被其他线程唤醒。
  for(;;) {
    //nativePollOnce指向frameworks/base/core/jni/android_os_MessageQueue.cpp#android_os_MessageQueue_nativePollOnce->void NativeMessageQueue::pollOnce->system/core/libutils/Looper.cpp#Looper::pollOnce->Looper::pollInner
    //
  	nativePollOnece(ptr, nextPollTimeoutMillis);//ptr为指向NativeMessageQueue指针的地址
  //获取IdleHandler并执行 idler.queueIdle();   
  }  
}
```

```c++
//system/core/libutils/Looper.cpp
int Looper::pollInner(int timeoutMillis) {
  struct epoll_event eventItems[EPOLL_MAX_EVENTS];
  int eventCount = epoll_wait(mEpollFd, eventItems, EPOLL_MAX_EVENTS, timeoutMills);
  for(int i=0;i<eventCount;i++) {
    int fd = eventItems[i].data.fd;
    uint32_t epollEvents = eventItems[i].events;
    if(fd == mWakeEventFd) {
      if(epollEvents & EPOLLIN) {
        awoken();
      }
    }else {
      //mResponses为Vector<Response>
      pushResponse(events, mRequests.valueAt(requestIndex));
    }
  }
Done::
  while(xxx) {
    const MesssageEnvelope& envelope = mMessageEnvelopes.itemAt(0);
    //xxx
    sp<Messagehandler> handler = envelope.handler;
    Message message = envelope.message;
    mMessageEnvelopes.removeAt(0);
    handler->handleMessage(message);
  }
}

void Looper::awoken() {
  //epoll读取
  TEMP_FAILURE_RETRY(read(mWakeEventFd, &counter, sizeof(uint64_t));
}
```

#### 13.3 线程消息发送过程

Handler内部有mLooper和mQueue。Handler.sendMessage最终会触发MessageQueue.enqueuMessage方法

```kotlin
fun enqueueMessage(msg:Message, when:Long):Boolean {
  //分四种情况
  //1. 目标消息队列是一个空队列
  //2. 插入的消息的处理时间 == 0
  //3. 插入的消息的处理时间小于保存在目标消息队列头的消息的处理时间
  //4. 插入的消息的处理时间大于保存在目标消息队列头的消息的处理时间
  //前3情况，消息都要保存在目标消息度列的头部。
  msg.when = when;
  Message p = mMessages;
  //消息插入后，需要将目标线程唤醒。1. 插入的消息在队列中部；2. 消息的队列头部
  if(p == null || when == 0 || when < p.when) {
    msg.next = p;
    mMessages = msg;
    //在头部时，因为头部的消息发生变化，需要将目标线程唤醒，以便对头部的新消息进行处理。mBlocked表示目标线程是否正处于睡眠等待状态
    needWeak = mBlocked;
  } else {
    //第四种情况，消息要保存在队列中的某个位置，忽略...
    //在中部时，不需要唤醒
    needWake = false;
  }  
  if(needWake){
    nativeWake(mPtr);//最终由system/core/libutils/Looper.cpp#Looper::wake处理
  }  
}
```

```c++
//system/core/libutils/Looper.cpp
void Looper::wake() {
  //调用epoll的write函数写入IO事件唤醒
  TEMP_FAILURE_RETRY(write(mWakeEventFd, &inc, sizeof(uint64_t)));
}
```

#### 13.4 线程消息处理过程

普通线程消息通过Handler.dispatchMessage()。

线程空闲消息：线程在空闲时主动发送，通过IdleHandler处理。

