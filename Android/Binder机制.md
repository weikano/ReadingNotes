### Binder机制

#### 一、引言

目前Linux支持的IPC包含传统的管道、System V IPC（消息队列/共享内存/信号量）以及socket。

- socket作为通用接口，传输效率低、开销大，主要用于跨网络的进程间通信和本机上的低速通信。
- 消息队列和管道采用存储-转发方式，数据先从发送方的缓存区copy到内核开辟的缓存区，然后从内核缓存区中copy到接收方缓存区，至少两次copy过程。
- 共享内存无需copy，但控制复杂，难以使用。

安全性：传统IPC没有任何安全措施，完全依赖上层协议的实现。首先传统的IPC无法获取进程可靠的PID和UID，无法鉴别对方身份。android为每个安装好的APP分配了UID，所以进程UID是鉴别进程身份的标志。传统的IPC只能在数据包里填入UID和PID，但这样不可靠。其次传统的IPC访问接入点是开放的，无法建立私有连接。比如命名管道的名称，socket的ip或者文件名，无法阻止恶意程序通过猜测获得连接。

Binder基于CS通信方式，传输过程只需一次copy，为发送方添加PID和UID，既支持实名也支持匿名Binder，安全性高。

![](https://img-blog.csdn.net/20180107175213902?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvcWl5ZWkyMDA5/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)

Binder驱动通信：

1. 通过binder_open打开Binder驱动
2. 通过binder_mmap将Binder驱动设备与应用程序建立内存映射，这样用户控件可以直接操作Binder设备上的数据。

**所以从上图可知，要将B中的数据copy到A中，主需要在Binder驱动中执行一次copy_from_user复制一次数据，这样A就可以在mmap之后使用B中的数据**。

Binder使用CS方式：一个进程作为server；多个进程作为client向server发起服务请求，获得所需要的服务。要实现CS架构必须实现以下两点：

1. server必须有确定的访问接入点或地址来接收client的请求，并且client可以通过某种途径获取server的地址。
2. 指定command-reply协议来传输数据。

**对server而言，Binder可以看成server提供的实现某个特定服务的访问接入点，client通过这个地址向server发送请求来使用服务。对client而言，Binder是通向server的管道入口，要想和某个server通信首先必须建立这个管道并获得管道入口**。

Binder是一个实体位于Server中的对象，该对象提供了一套方法用以实现对服务的请求，就象类的成员函数。遍布于client中的入口可以看成指向这个binder对象的‘指针’，一旦获得了这个‘指针’就可以调用该对象的方法访问server。在Client看来，通过Binder‘指针’调用其提供的方法和通过指针调用其它任何本地对象的方法并无区别，尽管前者的实体位于远端Server中，而后者实体位于本地内存中。**Client中的Binder也可以看作是Server Binder的‘代理’，在本地代表远端Server为Client提供服务**。

**Binder对象是一个可以跨进程引用的对象，它的实体位于一个进程中，而它的引用却遍布于系统的各个进程之中**。最诱人的是，这个引用和java里引用一样既可以是强类型，也可以是弱类型，而且可以从一个进程传给其它进程，让大家都能访问同一Server，就象将一个对象或引用赋值给另一个引用一样。

#### 三、Binder通信模型

##### 3.1 Binder驱动

工作于内核态，负责进程之间Binder通信的建立、Binder在进程间的传递、Binder引用计数管理、数据报在进程间的传递和交互等一系列底层支持。

##### 3.2 ServiceManager与实名Binder

ServiceManager作用是将字符形式的Binder转化成client中对该Binder的引用，是的client能够通过Binder名字获得对server中Binder实体的引用。注册了名字的Binder叫实名Binder。server创建了Binder实体，为其取一个名字，将这个Binder联通名字以数据包的形式通过Binder驱动发送给ServiceManager。驱动为这个穿过进程变的Binder创建位于内核中的实体结点以及ServiceManager对实体的引用。ServiceManager收到数据包后，从中取出名字和引用填入一张查找表中。

**ServiceManager和其他进程同样采取Binder通信，ServiceManager是server端，有自己的Binder对象（实体），其他进程都是client，需要通过这个Binder引用来实现Binder的注册、查询和获取。ServiceManager的Binder比较特殊，它没有名字也不需要注册，当一个进程使用BINDER_SET_CONTEXT_MGR将自己注册成ServiceManager时，Binder驱动会自动为它创建Binder实体。其次这个Binder的引用在所有client中都固定为0而无需通过其他手段获得。即一个server若要向ServiceManager注册自己的Binder就必须通过0这个引用和ServiceManager通信**。

##### 3.3 client获得实名Binder的引用

ServiceManager从数据包中取得Binder的名字，在查找表中找到该名字对应的条目，从条目中取得Binder的引用，将该引用作为回复发送给发起请求的client。现在这个Binder对象有了两个引用：一个位于ServiceManager，一个位于发起请求的client进程。这些引用都是强类型，从而确保只要有引用，Binder实体就不会被释放掉。

##### 3.4 匿名Binder

server端可以通过已经建立的Binder连接将创建的Binder实体传给client，这条Binder连接必须通过实名Binder创建。匿名Binder为通信双方建立了一个私密通道，只要server没有把匿名Binder发给别的进程，别的进程就无法通过穷举或者猜测的方式获得该Binder的引用。

#### 四、Binder协议

**控制协议**：

| 命令                     | 说明                                          | 参数类型          |
| :----------------------- | :-------------------------------------------- | :---------------- |
| BINDER_WRITE_READ        | 读写操作，IPC进程就是通过这个命令进行数据传递 | binder_write_read |
| BINDER_SET_MAX_THREADS   | 设置进程支持的最大线程数                      | size_t            |
| BINDER_SET_CONTEXT_MGR   | 设置自身为ServiceManager                      | 无                |
| BINDER_THREAD_EXIT       | 通知驱动Binder线程退出                        | binder_version    |
| BINDER_VERSION           | 获取驱动的版本号                              | binder_version    |
| BINDER_SET_IDLE_PRIORITY | 未用到                                        |                   |
| BINDER_SET_IDLE_TIMEOUT  | 未用到                                        |                   |

驱动协议：

- binder_driver_command_protocal：进程发送给驱动Binder的命令
- binder_driver_return_protocal：Binder驱动发送给进程的命令

#### 五、Binder表述

#### 六、Binder内存映射和接收缓存区管理

传统的IPC数据是怎样从发送端到达接收端：发送方将准备好的数据存放在缓存区中，调用API通过系统调用进入内核中。内核服务程序在内核空间分配内存，将数据从发送方缓存区复制到内核缓存区中。接收方读数据时也要提供一块缓存区，内核将数据从内核缓存区拷贝到接收方提供的缓存区中并唤醒接收线程，完成一次数据发送。**这种方法有两个缺陷：1、两次数据copy；2、接收方不知道要多大的缓存才够用，只能开辟尽量大的空间或先调用API接收消息头获得消息体大小**。

Binder采用了一种全新策略：由Binder驱动负责管理数据接收缓存。

```c++
fd = open("/dev/binder", O_RDWR);
mmap(NULL, MAP_SIZE, PROT_READ, MAP_PRIVATE, fd,0);
```

Binder接收方就有了一片大小为MAP_SIZE的接收缓存区。mmap返回的是内存映射在用户空间的地址，这段空间是由驱动管理，用户控件不能直接访问（PROT_READ，只读映射）。

接收缓存区映射好后就可以做为缓存池接收和存放数据了。前面说过，接收数据包的结构为binder_transaction_data，但这只是消息头，真正的有效负荷位于data.buffer所指向的内存中。这片内存不需要接收方提供，恰恰是来自mmap()映射的这片缓存池。在数据从发送方向接收方拷贝时，驱动会根据发送数据包的大小，使用最佳匹配算法从缓存池中找到一块大小合适的空间，将数据从发送缓存区复制过来。要注意的是，存放binder_transaction_data结构本身以及表4中所有消息的内存空间还是得由接收者提供，但这些数据大小固定，数量也不多，不会给接收方造成不便。映射的缓存池要足够大，因为接收方的线程池可能会同时处理多条并发的交互，每条交互都需要从缓存池中获取目的存储区，一旦缓存池耗竭将产生导致无法预期的后果。

有分配必然有释放。接收方在处理完数据包后，就要通知驱动释放data.buffer所指向的内存区。在介绍Binder协议时已经提到，这是由命令BC_FREE_BUFFER完成的。

驱动为接收方分担了最为繁琐的任务：分配/释放大小不等，难以预测的有效负荷缓存区，而接收方只需要提供缓存来存放大小固定，最大空间可以预测的消息头即可。在效率上，由于mmap()分配的内存是映射在接收方用户空间里的，所有总体效果就相当于对有效负荷数据做了一次从发送方用户空间到接收方用户空间的直接数据拷贝，省去了内核中暂存这个步骤，提升了一倍的性能。顺便再提一点，Linux内核实际上没有从一个用户空间到另一个用户空间直接拷贝的函数，需要先用copy_from_user()拷贝到内核空间，再用copy_to_user()拷贝到另一个用户空间。**为了实现用户空间到用户空间的拷贝，mmap()分配的内存除了映射进了接收方进程里，还映射进了内核空间。所以调用copy_from_user()将数据拷贝进内核空间也相当于拷贝进了接收方的用户空间，这就是Binder只需一次拷贝的‘秘密’**。

#### 七、Binder接收线程管理

应用程序通过BINDER_SET_MAX_THREADS告诉驱动最多可以创建几个线程。以后每个线程在创建、进入主循环、退出主循环时都分别使用BC_REGISTER_LOOP、BC_ENTER_LOOP、BC_EXIT_LOOP告知驱动，以便驱动收集和记录当前线程池状态。

Binder驱动还做了一点小小的优化。当进程P1的线程t1向进程P2的发送请求时，驱动会先查看t1是否正在处理来自P2某个线程请求但未完成（没有发送回复）。加入在P2中有这样的线程，比如t2，就会要求t2来处理t1的这次请求。因为t2既然向t1发送了请求但未收到回包，表示t2阻塞在读取返回包的状态。可以让t2顺便做点事情，比闲着好。如果t2不是线程池中的线程，还可以帮线程池分档部分工作，减少线程池使用率。

#### 八、数据报接收队列与（线程）等待队列管理

[队列管理]: https://www.cnblogs.com/qingchen1984/p/5212755.html

