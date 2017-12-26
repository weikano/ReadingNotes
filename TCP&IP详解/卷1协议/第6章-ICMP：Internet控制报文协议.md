## ICMP：Internet控制报文协议

### 1、引言

ICMP传递差错报文以及其他需要注意的信息，一般认为是IP层的一个部分。

ICMP通常被IP层或更高层协议（TCP或UDP）使用能够。一些ICMP报文把差错报文返回给用户进程。

ICMP报文是在IP数据报内部被传输的。ICMP被封装在IP数据报内部，紧接着IP首部。

![image](https://raw.githubusercontent.com/weikano/NoteResources/master/TCP%26IP/009.png)

ICMP报文的格式如图：

- 所有报文的前4个字节都是一样的
- 类型字段可以有15个不同的值，用来描述特定类型的ICMP报文。某些ICMP报文还使用代码字段的值来进一步描述不同的条件。
- 检验和字段覆盖整个ICMP报文，使用的算法与IP首部检验和算法相同的。检验和是必须的。

![image](https://raw.githubusercontent.com/weikano/NoteResources/master/TCP%26IP/010.png)



### 2、ICMP报文的类型

![image](https://raw.githubusercontent.com/weikano/NoteResources/master/TCP%26IP/011.png)

当发送一份ICMP差错报文时，报文始终包含IP的首部和产生ICMP差错报文的IP数据报的前8个字节。这样，接收ICMP差错报文的模块就会把它与某个特定的协议（根据IP数据报首部中的协议字段）和用户进程（根据IP数据报首部前8个字节中的TCP或UDP报文首部中的TCP/UDP端口号来确定）联系起来。

下列各种情况不会产生ICMP差错报文：

- ICMP差错报文（非ICMP查询报文）。
- 目的地址是广播地址或多播地址的IP数据报。
- 作为链路层广播的数据报。
- 不是IP分片的第一片。
- 源地址不是单个主机的数据报。即源地址不能为零地址、环回地址、广播或多播地址。



### 3、ICMP地址掩码请求与应答

### 4、ICMP时间戳请求与应答

### 5、ICMP端口不可达差错

