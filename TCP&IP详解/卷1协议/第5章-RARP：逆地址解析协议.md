RARP：逆地址解析协议

1、引言

具有本地磁盘的系统引导时，一般是从磁盘上的配置文件读取IP地址。无盘机则是从接口卡上读取唯一的硬件地址，然后发送一份RARP请求，请求某个主机相应该无盘系统的IP地址。

2、RARP服务器的设计

2、1 作为用户进程的RARP服务器

RARP服务器的复杂性在于一个服务器一般要为多个主机提供硬件地址到IP地址的映射。该映射包含在一个磁盘文件中。由于内核一般不读取和分析磁盘文件，因此RARP服务器的功能就由用户进程来提供。

RARP请求是作为一个特殊类型的以太网数据帧来传送的。这说明RARP服务器必须能够发送和接收这种类型的以太网数据帧。

2、2 每个网络有多个RARP服务器

RARP请求是在硬件层上进行广播的，这意味着它们不经过路由器进行转发。通常一个网络上要提供多个RARP服务器。

因为每个服务器对每个RARP请求都要发送RARP应答，所以流量也随之增加。发送RARP请求的无盘系统一般采用最先收到的RARP应答。

3、小结

- RARP协议是许多无盘系统在引导时用来获取IP地址的。
- RARP的问题包括使用链路层广播，这样就阻止大多数路由器转发RARP请求，只返回很少信息：只是系统IP地址。


