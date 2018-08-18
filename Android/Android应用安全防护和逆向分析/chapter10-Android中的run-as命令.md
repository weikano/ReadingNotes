### 一、命令的分析和使用
run-as命令源码在aosp/system/core/run-as/run-as.cpp
```c
int main(int argc, char* argv[])
{
    //检查uid
    if(getuid() != AID_SHELL && getuid() != AID_ROOT) {
        //报错
    }
    //根据包名解析包信息
    pkg_info info;
    memset(&info, 0, sizeof(info));
    info.name = pkgname;
    if(!packagelist_parse(packagelist_parse_callback, &info)) {
        //包解析失败
    }
    if(info.uid == 0) {
        //unknown package
    }
    //忽略下面检查uid
    //检查是否debuggable
    if(!info.debuggable) {
        //报错
    }
    //检查data目录是否可用
    if(!check_data_path(info.data_dir, userAppId)) {
        //报错
    }
}
```

### 二。Linux中的setuid和setgid概念
#### 1.Linux中的setuid和setgid概念
> Linux下的可执行文件一旦被设置了setuid标记，使用该执行程序的进程就将拥有该执行文件的所有者权限。普通用户执行这个命令，可使自己升级成root权限

### 三、Android中setuid和setgid的使用场景
#### 1. zygote降权
Android中每个app的启动都是通过AMS发送socket消息通知zygote，然后zygote会fork一个进程。由于Linux中父进程fork出来的子进程具有相同的uid，所以如果zygote是root，那么每个进程都是root了。因此，在forkAndSpecialize中会通过setuid和setgid进行降权。
```c
static pid_t forkAndSpecializeCommon(const u4* args, bool isSystemServer) {
    pid_t puid;
    uid_t uid=(uid_t)args[0];
    gid_t gid=(gid_t)args[1];
    ArrayObject* gids = (ArrayObject*)args[2];
    u4 debugFlags = args[3];
    //省略...
    int error = setgid(gid);
    //省略...
    error = setuid(uid);

}

```
#### 2.su工具原理
root原理：一个普通的进程通过执行su，从而获得一个具有root权限的进程。su是fork一个子进程来做真正的事。由于su的uid是root，所以它fork出来的子进程的uid也是root。

#### 3.run-as命令
### 四、run-as命令的作用
run-as命令除了能够在非root设备上查看debug模式的应用沙盒数据外，还能为Android中的调试做基础。
Android中的调试系统是gdb，通过gdb和gdbserver来调试APP。gdbserver通过ptrace附加到目标APP进程，然后gdb再通过socket或者pipe来链接gdbserver，并且向它发出命令来对APP进行调试，以下关键点：
- 每一个需要调试的apk在打包的时候都会附带上gdbserver，因为手机上面没有gdbserver。gdbserver用来通过ptrace命令附加到要调试的APP进程上。
- 要注意ptrace的调用。一般来说只有root和具有相同gid、uid的进程可以通过ptrace向目标注入一个so.
- 如何在非root设备上让gdbserver与目标APP的uid相同。只有通过run-as命令提权。

**因此，要调试一个程序，首先要是debug模式**

### 五、调用系统受uid限制的API
略