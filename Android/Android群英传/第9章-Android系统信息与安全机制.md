# PackageManager

![image](https://raw.githubusercontent.com/weikano/NoteResources/master/14.png)
最里面的框代表了整个Activity的信息, Activityinfo.  
  
    
最外面的框代表整个Manifest文件中节点的信息, PackageInfo.  
  
    
- ActivityInfo  
封装了在Manifest中<Activity>节点和<receiver>节点中的所有信息, 包括name, icon, label和launchMode等. 
- ServiceInfo  
封装了<service>之间的所有信息.
- ApplicationInfo
封装了<application>之间的信息, 并且包含了很多Flag.
- PackageInfo
PackageInfo包含了所有的Activity, Service等信息.
- ResolveInfo
封装<intent>信息的上一级信息, 所以它可以返回ActivityInfo, ServiceInfo等包含的<intent>信息, 用于帮助查找包含特定条件的信息, 比如分享功能.  
  
  
# ActivityManager

- ActivityManager.MemoryInfo
系统可用内存availMem, 总内存totalMem, threshold低内存阈值, 是否处于地内存lowMemory. 
- Debug.MemoryInfo
类似上面, 用于统计进程下的内存信息.
- RunningAppProcessInfo
运行进程的信息, 包括processName, pid, uid, pkgList该进程下所有的包. 
- RunningServiceInfo
用于封装运行的服务信息. 第一次被激活的时间aciveSince, 方式, foreground等.

# 解析Packages.xml获取系统信息

在系统初始化时, PackageManagerService会去扫描系统中的特定目录, 解析其中的APK文件. 同时将信息保存至XML中, 当作应用花名册, 当系统中的apk安装, 更新, 删除时都会更新. 这个文件位于/data/system/packages.xml.

# ~~Android安全机制~~
屁事没说