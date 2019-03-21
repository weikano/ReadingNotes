### 第06章-进程间通信-Binder

Binder驱动→路由器

ServiceManager→DNS

Binder client→客户端

Binder server→服务器

#### 6.1 智能指针

//TODO

#### 6.2 进程间数据传递载体-Parcel

##### 6.1 Parcel设置先关

- dataSize：当前已经存储的数据大小
- setDataCapacity：设置Parcel的空间大小
- setDataPosition：改变Parcel的读写位置
- dataAvail：当前Parcel中 的可读数据大小
- dataCapacity：当前Parcel的存储能力
- dataPosition：当前的位置
- dataSize：Parcel包含的数据大小

##### 6.2 ActiveObjects

- Binder：利用Parcel将Binder写入，读取时就能取得原始的Binder对象或者他的特殊代理实现。先通过Parcel.writeInterfaceToken(String)写入Binder的DESCRIPTOR，然后调用writeStrongBinder写入Binder对象。
- FileDescriptor：Linux中的文件描述符。



