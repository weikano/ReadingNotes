### 一，基础命令
1. cat
> 查看文件内容
2. echo/touch
> 写文件

### 二、非she'll命令
> 需要提前使用adb shell命令运行的命令叫做shell命令，直接用adb shell命令运行的叫做非shell命令

1. adb shell dumpsys activiy top
> 说明：查看当前应用的Activity信息
> 用法：adb shell dumpsys activity top
2. adb shell dumpsys package [pkgname]
> 查看指定包名应用的详细信息，即AndroidManifest.xml中的内容
3. adb shell dumpsys meminfo
> 查看指定进程的内存信息, adb shell dumpsys meminfo [pname/pid]
4. adb shell dumpsys dbinfo
> 查看指定进程的数据库信息，包括最近使用过的SQL语句 adb shell dumpsys dbinfo [pname/pid]
5. adb shell screencap
> 截屏 adb shell screencap -p [保存的路径]
6. adb shell screenrecord
> 录屏 adb shell screenrecord [保存的路径]
7. adb shell input text
> 在EditText控件获取到焦点后，使用该命令可以输入文本, adb shell input text [文本内容]
8. adb logcat
> 查看当前的日志信息
> adb logcat -s [tag]
> adb logcat | grep [pname/pid/keyword]
   
### 三、shell命令
1. run-as
> 可以在非root设备中查看指定debug模式的包名应用沙盒数据
> run-as qiu.statusbarcompat
2. ps
> ps | grep 过滤内容
> ps -t [pid]查看指定进程的线程信息
3. pm clear
> pm clear [pkgname]清除指定应用的数据
4. am start
> 启动一个应用
> am start -n [pkgname]/[pgkname].[activity.name]
> am start -n com.android.browser/com.android.browser.BrowserActivity
5. am startservice
> 启动某个服务
> am startservice -n [pkgname]/[pkgname].[service.name]
6. am broadcast
> am broadcast -a [action]
7. netcfg
> 查看设备的ip地址
8. netstat
> 查看设备的端口号
9.  app_process
> 运行Java代码
> app_process [代码运行目录] [运行主类]
> 需要先使用dx命令把dex文件转成jar包，实际上它不是一个正常的jar，而是包含了classes.dex文件的压缩文件
10. dalvikvm
> 运行一个dex文件
> dalvikvm -cp [dex文件] [运行主类]
> dalvikvm -cp /data/demo.dex cn.wjdiankong.Main
11. top
> 查看当前CPU的消耗信息
> top [-n/-m/-d/-s/-t]
> | 命令 | 描述 |
> | --- | ---- | 
> | -m | 最多显示多少个进程 |
> | -n | 刷新次数 |
> | -d | 刷新间隔(默认5秒) |
> | -s | 按哪列排序 |
> | -t | 显示线程信息而不是进程信息 |
12. getprop
> 查看系统属性
> getprop ro.debuggable

### 三、操作apk命令
1. aapt(SDK目录下build—tools中)
> aapt dump xmltree [apk包] [需要查看的资源文件xml]
> aapt dump xmltree demo.apk AndroidManifest.xml
2. dexdump(sdk目录中build-tools下)
> dexdump [dex文件]

### 四、进程命令
1. 查看当前进程的内存加载情况
> cat /proc/[pid]/maps
2. 查看进程的状态信息
> cat /proc/[pid]/status
3. 查看当前应用使用的端口信息
> cat /proc/[pid]/net/[tcp/tcp6/udp/udp6]