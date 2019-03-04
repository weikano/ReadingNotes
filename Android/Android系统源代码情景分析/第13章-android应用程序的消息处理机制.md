### 第13章-android应用程序的消息处理机制

android系统主要通过MessageQueue、Looper和Handler来实现消息机制。MessageQueue用来描述消息队列，Looper用来创建消息队列以及进入消息循环，Handler用来发送消息和处理消息。

#### 13.1 创建线程消息队列

android在c++层中也有一个对象的Looper类和NativeMessageQueue类。Java层的Looper类和MessageQueue类是通过c++层的looper类和NativeMessageQueue实现的。

Java层中的每个looper对象内部都有一个MessageQueue的成员变量mQueue。而在c++层中，每个NativeMessageQueue都有一个类型为Looper的成员变量mLooper，指向c++层的Looper对象。

Java层的每一个MessageQueue有一个mptr，保存了c++层中的一个NativeMessageQueue对象的地址值。

c++层中的looper对象有两个类型为int