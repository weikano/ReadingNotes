### 一、逆向工具
#### 1. 反编译利器apktool
#### 2. dex2jar+jd-gui
#### 3. jeb和jadx
#### 4. hook神奇xposed框架
#### 5. native hook神奇cydia substrate
#### 6. 脱壳神器ZjDroid工具
#### 7. so相关的IDA pro

### 二、逆向的基本知识
smali语法、arm指令、ndk开发等

### 三、打开系统调试总开关(system/default.prop文件中的ro.debuggable)
在Android中一个应用是否可以被调试是这样判断的：当dalvik虚拟机从Android应用框架中启动时，如果系统属性ro.debuggable为1，那么所有应用都是可以被调试的。如果ro.debuggable为0，那么就根据AndroidManifest.xml中的android:debuggable属性来确定。

default.prop是init进程解析并加载进内存块中。只有init进程能够修改。以下三种方式修改：
- 直接修改default.prop文件中的值然后重启设备：修改后直接死机……
- 改写系统文件，重新编译系统镜像然后写入设备
- 注入init进程，修改内存中的值：使用ptrace注入init进程然后修改属性的值，只要init进程不重启就会一直生效

最后一种方式可以使用一个叫做mprop的执行文件：将mprop拷贝到设备中的目录下，然后执行命令
```shell
./mprop ro.debuggable 1

//然后重启adbd这个守护进程

stop;start
```
