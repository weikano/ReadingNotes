Android的消息机制主要是指Handler的运行机制, Handler的运行需要底层MessageQueue和Looper来支撑.   
  
  
MessageQueue为消息队列, 采用单链表来存储消息列表.  
  
  
Looper无限循环去判断MessageQueue中是否有新消息, 如果有就处理, 如果没有就一直等待.  
  
  
线程默认是没有Looper的, 如果需要使用Handler就必须为线程创建Looper.  
  
  
我们经常提到主线程, 它就是ActivityThread, ActivityThread被创建时就初始化Looper, 这也是在主线程中默认可以使用Handler的原因. 

# Android的消息机制概述
Handler的主要作用是将一个任务切换到某个指定的线程中执行.
![image](https://raw.githubusercontent.com/weikano/NoteResources/master/6.png)

# Android的消息机制分析
## ThreadLocal的工作原理
ThreadLocal是一个线程内部的数据存储类, 通过它可以在指定的线程中存储数据. 数据存储之后, 只有在指定线程中可以获取到存储的数据, 其他线程则无法获取.  
  
  
**一般来说, 当某些数据是以线程为作用域并且不同线程具有不同的数据副本时, 可以考虑采用ThreadLocal.** 比如Handler, 它需要获取当前线程的Looper, 这个时候就可以通过ThreadLocal实现Looper在线程中的存取.  
  
  
**ThreadLocal的另一使用场景是复杂逻辑下的对象传递, 比如监听器的传递,** 有些时候一个线程中的人物过于复杂, 比如函数调用栈比较深或者代码如果多样, 这种情况下, 我们有需要监听器能够贯穿整个线程的执行过程, 就可以采用ThreadLocal.

## 消息队列(MessageQueue)的工作原理
MessageQueue主要包含两个操作: 插入和读取. 读取操作本身会伴随着删除操作, 插入和读取对应的方法分别为enqueueMessage和next. **可以发现next方法是一个无线循环的方法, 如果队列中没有消息, 那么next方法会一直阻塞. 有新消息到来时, next方法会返回这条消息并将其从单链表中移除. **

## Looper的工作原理
**在子线程中, 如果手动创建了Looper, 那么在所有事件完成之后应该调用quit方法来终止. 否则这个子线程就会一直处于等待状态.**  
  
  
Looper最重要的一个方法是Looper.loop(), 只有调用了loop后, 消息循环系统才真正的起作用. 

## Handler的工作机制
Handler的工作主要包含消息的发送和接收过程.  
  
  
Handler处理消息的过程如下:  
首先检查Message的callback是否为null, 不为null就通过handleCallback方法来处理. Message的callback其实是一个Runnable对象, 实际上就是Handler的post方法所传递的Runnable参数.  
其次检查mCallback是否为null, 不为null就调用mCallback的handleMessage来处理消息.  
最后调用Handler的handleMessage来处理消息.
![image](https://raw.githubusercontent.com/weikano/NoteResources/master/7.png)

# 主线程的消息循环
Android的主线程就是ActivityThread, 主线程的入口为main, 在main方法中会通过Looper.prepareMainLooper()来创建主线程的Looper和MessageQueue, 并通过Looper.loop()来开启主线程的消息循环.  
主线程的消息循环开始后, 还需要一个Handler来处理, 这个Handler就是ActivityThread.H.  
ActivityThread通过ApplicationThread和AMS进行IPC, AMS以IPC方式完成ActivityThread的请求后, 会毁掉ApplicationThread的Binder方法, ApplicationThread会向H发送消息, H收到消息后会将ApplicationThread中的逻辑切换到ActivityThread来执行, 即切换到主线程. 

