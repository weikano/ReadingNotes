### 第05章-Android进程/线程管理

#### 5.2 Handler、MessageQueue、Runnable与Looper

Looper调用loop方法不断获取MessageQueue中的Message，然后交给Handler处理。

handler：

- 每个thread对应一个Handler
- 每个Looper对应一个MessageQueue
- 每个MessageQueue中有N个Message
- 每个Message中最多指定一个handler处理消息

