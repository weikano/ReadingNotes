### 第04-Logger日志系统

基于内核中的Logger日志驱动程序实现。Logger日志驱动程序在内部使用一个环形缓冲区保存日志。当缓冲区满了之后，新的日志会覆盖旧的日志。

日志的类型一共有4中: main, system, radio和events。这四种类型分别通过/dev/log/main, /dev/log/system, /dev/log/radio和/dev/log/events四个设备文件来访问。

main类型为应用级别，system为系统级别。由于系统级别的日志较为重要，因此把它们分开来可以避免系统日志被应用日志覆盖。radio与无线设备相关，量很大，因此单独记录。events为系统问题诊断，开发者不应该使用。

main, system和events分别通过android.util.Log, android.util.Slog和android.util.EventLog三个接口往Logger日志驱动中写入日志。C/C++中提供了LOGV, LOGD, LOGI, LOGW, LOGE来写入main；SLOGV, SLOGD, SLOGI, SLODW和SLOGE来写入system; LOG_EVENT_INT, LOG_EVENT_LONG, LOG_EVENT_STRING来写入events。不管是java端还是c/c++端，最终调用liblog来往logger驱动中写入日志。

#### 4.1 Logger日志格式

只需要关注events类型的日志格式，但是也不是那么重要, P84

#### 4.2 Logger日志驱动程序

主要代码为Android/kernel/goldfish/drivers/staging/android/logger.h(c)