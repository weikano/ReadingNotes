#### 11.3.2 串谋权限攻击
> 比如A本身没有任何权限，它的组件想要联网下载文件并保存到SD卡上，这在正常情况下是不允许的。B拥有联网与写SD卡的权限，并实现了文件下载与保存的功能。这时候A就可以通过B来实现文件的下载，从而突破系统的权限控制。

#### 11.3.3 权限攻击检测
> drozer，原mercury，Android平台安全评估工具

### 11.4 Android组件安全
#### 11.4.1 Activity安全及劫持
> - Activity设置IntentFilter之后，默认是可以被外部程序访问的。如果设置android:exported=false来限制外部程序调用（**外部程序指的是签名不同、用户ID不同的程序，签名相同且用户ID相同的程序可以在同一个进程共享，彼此之间没有组件访问限制**），那么特定的程序要访问该组件就是不可能的了。通过android:permission来设置一个权限，可以解决上述问题。
> - 劫持：通常是判断前台运行的进程，符合条件的话就拉起一个伪装的Activity

#### 11.4.2 BroadcastReceiver安全
> 有序广播因为是根据优先级的，所以优先级高的，还可以更改广播的内容或者终止；无需广播无法通过abortBroadcast来终止。解决方法是在intent中设置指定的组件的类
```java
intent.setClass(context, TargetReceiver.class);
```

#### 11.4.3 service安全
> 处理方式类似Activity，通过android:exported=false一劳永逸或者设置android:permission

#### 11.4.4 ContentProvider安全
> 通过设置android:readPermission和android:writePermission以及对指定的uri设置权限来限制外部访问

### 11.5 数据安全
#### 11.5.1 外部存储安全
#### 11.5.2 内部存储安全
```java
//拿到com.pkg.b的context
Context ctx = context.createPackageContext("com.pkg.b", Context.CONTEXT_IGNORE_SECURITY);
InputStream is = ctx.openFileInput("config.txt");
//开始读取com.pkg.b中的config.txt的文件内容
```

#### 11.5.3 数据通信安全
> 通过Intent传递数据时，比如给broadcastreceiver传递数据时，如果没有指定组件，那么只要能收到该广播，就能拿到传递过去的数据