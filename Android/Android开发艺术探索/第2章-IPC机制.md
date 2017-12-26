# 开启多进程模式
1. 在AndroidManifest.xml中设置android:process属性.
2. 通过JNI在native层fork一个新的进程. 

## process的命名区别
1. :的含义是指要在当前进程名前面附加上当前的包名,这是一种简写.
2. 不以:开头的为完整的命名方法,不会附加包名信息.
3. 以:开头的进程属于当前应用的私有进程,其它应用的逐渐不可以和它跑在同一个进程中.
4. 不以:开头的为全局进程, 其他应用通过shareUID方式可以和它跑在同一个进程中.
5. Android系统会为每个应用分配一个唯一的UID, 具有相同UID的应用才能共享数据
6. 两个应用通过shareUID跑在同一个进程中, 必须具有相同的UID和签名,才能访问对方的私有数据,比如data目录,组件信息等, 不管是否在同一个进程中.

## 多进程模式的运行机制

> **使用多进程会造成如下几个问题**:
> 1. 静态成员和单例模式完全失效
> 2. 线程同步机制完全失效
> 3. SharePreferences的可靠性下降
> 4. Application会多次创建

# IPC基础概念介绍
## Serializable接口
1. 使用ObjectOutputStream和ObjectInputStream来实现序列化和反序列化
2. serialVersionUID: 序列化时,系统会把当前类的serialVersionUID写入序列化的文件中,当反序列化时会去检测文件中的serialVersionUID, 如果一致就说明版本相同,可以成功反序列化; 否则说明当前的类和序列化的类相比, 发生了某些变换.
3. 静态成员变量不属于对象, 不会参与序列化过程. 用transient标记的成员变量不参与序列化过程.
4. 可以通过private void writeObject(ObjectOutputStream oos) throws IOException和private void readObject(ObjectInputStream ois) throws  IOException, ClassNotFoundException来重写默认的序列化和反序列化, 系统会通过反射调用这两个方法.

## Parcelable接口
1. describeContents: 几乎所有情况下都应该返回0, 仅当当前对象中存在FileDescriptor时, 返回CONTENTS_FILE_DESCRIPTOR(0x1).
2. List和Map也可以序列化,前提是它们里面的每个元素都是可以序列化的.

# Binder
> 直观来说Binder是Andorid中的一个类,它实现了IBinder接口; 从IPC角度来说, Binder是Android中的一种跨进程通信方式, Binder还可以理解为一种虚拟的物理设备, 它的设备驱动是/dev/binder; 从Android Framework角度来说, Binder是ServiceManager连接各种Manager和相应的ManagerService桥梁; 从Android应用层来说, Binder是客户端和服务端进行通信的媒介, 当bindService时, 服务端会返回一个包含了服务端业务调用的Binder对象, 通过这个Binder对象, 客户端就可以获取服务器端提供的服务或数据, 包括普通的服务和基于AIDL的服务. **Binder运行在服务端进程, 如果服务端进程异常终止, 这时候我们到服务端的Binder连接断裂, 会导致运程调用失败. 更为关键的是, 如果我们不知道Binder连接已经断裂, 那么客户端程序会受到影响. 为了解决这个问题,我们可以给Binder设置一个死亡代理DeathRecipient, 通过linkToDeath和unlinkToDeath监听.**

# Android中的IPC方式
## 使用Messenger
> Messenger是一种轻量级IPC方案,底层实现是AIDL(IMessenger). 通过Messenger.send(Message msg)来实现简单的进程间通信, 服务端需要设置Handler对Message进行处理, Message的replyTo可以设置本地接收方的Messenger. 

## 使用AIDL
 1. 服务端
 >服务端需要创建一个Service来监听客户端的连接请求, 然后创建AIDL文件并暴露给客户端, 然后在Service中实现这个AIDL中定义的接口即可.

 2. 客户端
 > 首先需要绑定服务端的Service,绑定成功后将服务端返回的Binder对象转换成AIDL接口所属的类型,接着调用AIDL中的方法即可.

 3. AIDL接口的创建
 > 如果AIDL中使用了自定义的Parcelable对象, 必须新建一个和它同名的AIDL文件, 并在其中声明它为Parcelable类型.

 4. 注册监听
 > 因为Binder会把客户端传来的对象重新转化生成一个新的对象,所以reigster和unregister时,使用普通的List无法实现(equals方法无效), 必须使用**RemoteCallbackList**

 5. 注意事项
 > 客户端调用远程服务的方法运行在服务器的Binder线程池中. 调用时客户端会挂起, 如果耗时较长会造成ANR, 所以要避免在UI线程中访问远程方法, 比如onServiceConnected和onServiceDisconnected都运行在UI线程中, 所以不可以在他们中调用.
 
 6. 程序的健壮性
 > 1. 设置DeathRecipient, binderDied方法在客户端的Binder线程回调.
 > 2. onServiceDisconnected方法中重新连接服务
 
 7. AIDL中的权限验证
 > 1. 使用自定义permission, 在服务的onBind方法中调用checkCallingOrSelfPermission(permission)来验证是否有权限,如果没有返回null.
 > 2. 在onTransact中验证permission, 还可以加入uid和pid相关的验证, 比如客户端包名限制等等. 

## 使用ContentProvider
 1. ContentProvider可以用File, Database甚至内存数据来实现.
 2. android:authorities是ContentProvider的唯一标识
 2. ContentProvider包含read和write两种permission
 3. ContentProvider通过Uri来区分外界要访问的内容, 需要用到UriMatcher来实现.
 4. 要观察ContentProvider中数据的变化,可以通过ContentResolver的registerContentObserver来注册观察者, 而对应的ContentProvider在数据变化时, 需要调用ContentResolver的notifyChange方法来通知数据变化. 
 5. query, update ,insert, delete四大方法是存在多线程访问的, 因此方法内部要做好线程同步. 
 
## 使用Socket
> ServerSocket和Socket实现

# Binder连接池
> 参考com.wkswind.androidart.chapter2.binderpool代码

# 选用合适的IPC方式
![image](https://raw.githubusercontent.com/weikano/NoteResources/master/589841f8ab64413b80002b38.png)