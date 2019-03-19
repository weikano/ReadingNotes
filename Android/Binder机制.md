### Binder机制

#### 一、引言

目前Linux支持的IPC包含传统的管道、System V IPC（消息队列/共享内存/信号量）以及socket。

- socket作为通用接口，传输效率低、开销大，主要用于跨网络的进程间通信和本机上的低速通信。
- 消息队列和管道采用存储-转发方式，数据先从发送方的缓存区copy到内核开辟的缓存区，然后从内核缓存区中copy到接收方缓存区，至少两次copy过程。
- 共享内存无需copy，但控制复杂，难以使用。

安全性：传统IPC没有任何安全措施，完全依赖上层协议的实现。首先传统的IPC无法获取进程可靠的PID和UID，无法鉴别对方身份。android为每个安装好的APP分配了UID，所以进程UID是鉴别进程身份的标志。传统的IPC只能在数据包里填入UID和PID，但这样不可靠。其次传统的IPC访问接入点是开放的，无法建立私有连接。比如命名管道的名称，socket的ip或者文件名，无法阻止恶意程序通过猜测获得连接。

Binder基于CS通信方式，传输过程只需一次copy，为发送方添加PID和UID，既支持实名也支持匿名Binder，安全性高。

![](https://img-blog.csdn.net/20180107175213902?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvcWl5ZWkyMDA5/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)

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

