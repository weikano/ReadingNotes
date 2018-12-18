### 第4章-CPU：速度与负载的博弈

#### 4.1 原理

CPU问题无非分为以下三种：

- CPU资源冗余使用：可能是算法太粗糙，一次遍历的却遍历两次；可以是没使用cache，明明解码过的图片还要再解码一次；明明int就足够，偏偏要用long。
- CPU资源争抢：抢主线程的CPU资源：最经典的例子就是主线程Handler的优化；抢音视频CPU资源：第一、排除非核心业务的消耗，第二，优化自身的性能消耗，把CPU负载转化为GPU负载，比如使用renderscript来处理视频中的影像信息；第三、大家平等，相互抢。控制核心数、控制线程数。
- CPU资源利用率低：除了磁盘和网络的IO外，还有锁操作、sleep等。其中锁的优化，一般是在锁的范围上。

#### 4.2 工具集

**TOP软件**

> 依靠adb shell top 就可以简单列出进程的各种信息，缺点是top本身的性能消耗就不低。
>
> 排除0%的进程信息：adb shell top | grep -v "0% S"
>
> 只打印一次按CPU排序的TOP 10的进程信息：adb shell top -m 10 -s cpu -n 1
>
> 指定进程的CPU、内存消耗并设置刷新间隔：adb shell top -d 1 | grep com.kingsunedu.sdk.sample

**PS软件**

> adb shell ps --help 查看帮助

**proc下的CPU信息**

> cat /proc/[pid]/stat

**dumpsys cpuinfo**

> adb shell dumpsys cpuinfo

**Systrace、Traceview和Trepn**

> Systrace、TraceView用于定位，Trepn用于耗电分析

#### 4.3 案例A：音乐播放后台的卡顿问题

通过cat /sys/devices/system/cpu/cpu0/cpufreq/scaling_cur_freq发现，小米3手机锁屏后会降频，所以在这时就需要控制好CPU消耗，避免类似绘制歌词、解析歌词之类的功能还在运行。

#### 4.4 案例B：注意Android中的低效API

https://www.zhihu.com/question/50981262?guide=1

#### 4.5 案例C：用renderscript来减少图像处理的CPU消耗





