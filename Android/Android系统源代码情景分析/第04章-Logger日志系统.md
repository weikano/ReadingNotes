### 第04章-Logger日志系统

基于内核中的Logger日志驱动程序实现。Logger日志驱动程序在内部使用一个环形缓冲区保存日志。当缓冲区满了之后，新的日志会覆盖旧的日志。

日志的类型一共有4中: main, system, radio和events。这四种类型分别通过/dev/log/main, /dev/log/system, /dev/log/radio和/dev/log/events四个设备文件来访问。

main类型为应用级别，system为系统级别。由于系统级别的日志较为重要，因此把它们分开来可以避免系统日志被应用日志覆盖。radio与无线设备相关，量很大，因此单独记录。events为系统问题诊断，开发者不应该使用。

main, system和events分别通过android.util.Log, android.util.Slog和android.util.EventLog三个接口往Logger日志驱动中写入日志。C/C++中提供了LOGV, LOGD, LOGI, LOGW, LOGE来写入main；SLOGV, SLOGD, SLOGI, SLODW和SLOGE来写入system; LOG_EVENT_INT, LOG_EVENT_LONG, LOG_EVENT_STRING来写入events。不管是java端还是c/c++端，最终调用liblog来往logger驱动中写入日志。

#### 4.1 Logger日志格式

只需要关注events类型的日志格式，但是也不是那么重要, P84

#### 4.2 Logger日志驱动程序

主要代码为Android/kernel/goldfish/drivers/staging/android/logger.h(c)

##### 4.2.1 基础数据结构

- logger_entry：描述一个日志记录，最大长度为4K，记录的有效负载长度为4K减去结构体logger_entry的大小

  ```c++
  //kernel/goldfish/drivers/staging/android/logger.c
  #define LOGGER_ENTRY_MAX_LEN (4*1024)
  #define LOGGER_ENTRY_MAX_PAYLOAD (LOGGER_ENTRY_MAX_LEN - sizeof(struct logger_entry))
  
  struct logger_entry {
    _u16 len;//有效负载长度
    _u16 _pad;//用于4K对齐,2bytes
    _s32 pid;//进程id
    _s32 tid;//线程组id
    _s32 sec;  
    _s32 nsec;//nano seconds
    _char msg[0];//有效负载  
  };
  ```

- logger_log：日志缓冲区。每种类型都对应一个单独的日志缓冲区

  ```c
  //kernel/goldfish/drivers/staging/android/logger.c
  struct logger_log {
    unsigned char* buffer;//内核缓冲区，用来保存日志记录，循环使用，大小有size决定
    struct miscdevice misc;//日志设备
    wait_queue_head_t wq;//等待队列，用来记录正在等待读取新的日志记录的进程
    struct list_head readers;//列表，用来记录正在读取日志记录的进程
    struct mutex mutext;//互斥量，保护buffer的并发访问  
    size_t w_off;//下一条要写入的日志记录在日志缓冲区中的位置
    size_t head;//当一个新的进程读取日志时，应该从日志缓冲区的什么位置开始读取
    size_t size;//缓冲区大小  
  }
  ```

- logger_reader：一个正在读取某个日志缓冲区的日志记录的进程

  ```c
  //kernel/goldfish/drivers/staging/android/logger.c
  struct logger_reader {
    struct logger_log* log;//要读取的日志缓冲区
    struct list_head list;//所有读取同一种类型日志记录的进程    
    size_t r_off;  //当前进程要读取的下一条日志记录在缓冲区中的位置
  }
  ```

##### 4.2.2 日志设备的初始化过程

初始化在logger日志驱动程序的入口函数logger_init中进行。

```c++
//kernel/goldfish/drivers/staging/android/logger.c
#define DEFINE_LOGGER_DEVICE(VAR,NAME,SIZE) \
static unsigned char _buf_ ## VAR[SIZE]; \
static struct logger_log VAR = {\
  .buffer = _buf_ ## VAR, \
  .misc = {\
    .minor = MISC_DYNAMIC_MINOR,\
    .name = NAME,\
    .fops = &logger_fops, \
    .parent = NULL,
  },
  .ws = _WAIT_QUEUE_HEAD_INTITIALIZER(VAR, wq), \
  .readers = LIST_HEADE_INIT(VAR, readers), \
  .mutex = _MUTEX_INITIALIZER(VAR, mutex), \
  .w_off = 0, \
  .head = 0, \
  .size = SIZE, \
};
//文件操作函数表
static struct file_operations logger_fops = {
  .owner = THIS_MODULE,
  .read = logger_read,//读取
  .aio_write = logger_aio_write,//写入
  .poll = logger_poll,
  .unlocked_ioctl = logger_ioctl,
  .compat_ioctl = logger_iocl,
  .open = logger_open,//打开
  .release = logger_release  
};
static int _init init_log(struct logger_log* log){
  int ret;
  ret = misc_register(&log->misc);//将对应的日志设备注册到系统中misc_list
  if(unlikely(ret)){
    printk("failed");
    return ret;  
  }
  printk("success");  
  return ret;  
}

#define LOGGER_LOG_RADIO "log_radio"
#define LOGGER_LOG_EVENTS "log_events"
#define LOGGER_LOG_MAIN "log_main"

DEFINE_LOGGER_DEVICE(log_main, LOGGER_LOG_MAIN, 64*1024)//定义log_main，用于system和main
DEFINE_LOGGER_DEVICE(log_events, LOGGER_LOG_EVENTS, 256*1024)//定义log_events
DEFINE_LOGGER_DEVICE(log_radio, LOGGER_LOG_RADIO, 64*1024)//定义log_radio 
```

##### 4.2.3 日志设备文件的打开过程

```c++
static struct logger_log* get_log_from_minor(int minor){
  if(log_main.misc.minor == minor){
    return &log_main;
  }else if(log_events.misc.minor == minor) {
    return &log_events;
  }else if(log_radio.misc.minor == minor) {
    return &log_radio;    
  }else {
    return NULL;
  }
}
```

##### 4.2.4 日志记录的读取过程 logger_read(kernel/goldfish/drviers/staging/Android/logger.c)

因为缓冲区是环形的，所以表示日志负载长度的两个字节可能是一个在头一个在尾，也有可能是两个连在一起的字节。

##### 4.2.5 日志记录的写入过程 logger_aio_write(kernel/goldfish/drviers/staging/Android/logger.c)

P96

#### 4.3 运行时库层日志库liblog

无论写入什么类型的日志记录，最终都是通过调用函数write_to_log写入到Logger日志驱动。write_to_log是一个函数指针，最开始指向函数write_to_log_init执行初始化，然后指向\_\_write_to_log_kernel或者\_\_write_to_log_null

#### 4.4 C/C++日志写入接口(system/core/include/cutils/log.h中的LOGV,LOGD,LOGI,LOGE的四个宏)

#### 4.5 Java层日志写入接口(Log.java, SLog.java, EventLog.java)

#### 4.6 Logcat工具分析

logcat.cpp中的main函数为Logcat工具的初始化点。

