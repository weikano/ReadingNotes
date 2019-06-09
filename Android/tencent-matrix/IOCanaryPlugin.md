### IOCanaryPlugin原理

#### 1. IPlugin接口方法调用时机

- init会在Matrix的私有构造函数中触发
- start会在各个plugin中手动调用，Matrix.startAllPlugins也会调用
- stop
- destroy

#### 2. IOCanaryPlugin流程

1. init时获取包名，初始化core（IOCanaryCore）

2. IOCanaryPlugin.start实际上起作用的时IOCanaryCore，根据IOConfig中的配置决定是否检测IO和Closeable。

   ```java
   IOCanaryJniBridge.install(config, listener);
   mCloseGuardHooker = new GloseGuardHooker(issueListener);
   mCloseGuardHooker.hook()
   ```

#### 3. CloseGuardHooker流程

> 通过反射调用CloseGuard.setEnable和setReporter方法来实现hook。
>
> CloseGuard在FileInputStream时会调用guard.open方法持有一个Throwable，在close时会调用guard.close方法释放当前持有的throwable，然后在FileInputStream.finalize方法中会根据是否持有throwable来判断有没有调用close方法。

#### 4. IOCanaryJniBridge流程

##### 4. 1. 调用install方法

```java
public static void install() {
  //加载io-canary库，触发io_canary_jni.cc中的JNI_OnLoad方法
  if(!loadJni()) {
    return;
  }
  //setConfig和enableDetector
  //通过native方法设置config，比如buffer大小、重读读文件的次数限制等
  //调用doHook方法
  doHook();
}
```

```c
//io_canary_jni.cc
jint JNI_OnLoad() {
  //获取JNIEnv以及一些Java层的globalref对象
  if(!InitJniEnv(vm)) {
    return -1;
  }
  //OnIssuePublish是一个方法指针，用来上报issue
  //Get方法返回一个IOCanary对象
  //OnIssuePublish被赋值给issued_callback_，Issued_callback_在Detect时会调用
  iocanary::IoCanary::Get().SetIssuedCallback(OnIssuePubilsh);
}

void enableDetector() {
  //向IOCanary中的detectors中添加对应的FileIODetector
  //分为主线程FileIOMainThreadDetector、buffer过小FileIOSmallBufferDetector、重复读FileIORepeatReaderDetector
  iocanary::IOCanary::Get().RegiesterDetector(static_cast<DetectorType>(type));
}

jboolean doHook() {
  //TARGET_MODULES中保存了libopenjdkjvm.so、libjavacore.so、libopenjdk.so
  for(auto lib:TARGET_MODULES) {
    //通过elfhook_open找到对应的so在内存中的位置
    loaded_soinfo* soinfo = elfhook_open(lib);
    //将原有的open方法替换成ProxyOpen，并把原有的open方法保存至original_open
    //ProxyOpen/Close主要是记录threadname和stack，然后调用IOCanary.OnOpen和onClose
    //ProxyWrite/Read主要是记录write/read函数的好事以及size，然后调用IOCanary.OnRead方法
    //OnOpen/Close/Write/Read都是交给IOInfoCollector来处理。
    elfhook_replace(soinfo, "open", (void*)ProxyOpen, (void**)&original_open);
    elfhook_replace(soinfo, "open64", (void*)ProxyOpen64, (void**)&original_open64);
    bool is_libjavacore = xxx;//判断当前是否在hook libjavacore.so
    if(is_libjavacore) {
      //如果hook不了read，就hook__read_chk；同样hook其中的write函数，hook不了就找__write_chk
      elfhook_replace(soinfo, "read", (void*)ProxyRead, (void**)&original_read);
    }
    //hook其中的close函数
    elfhook_replace(so_info, (void*)ProxyClose, (void**)&original_close);
    elfhoo_close(soinfo);
  }
}

void ProxyClose() {
  iocanary::IOCanary::Get().OnClose(fd, ret);
}
//io_canary.cc
void OnClose(fd, ret) {
  std::shared_ptr<IOInfo> info = collector_.OnClose(fd, ret);
  if(!info) {
    OfferFileIOInfo(info);
  }
}

void OfferFileIOInfo(info) {
  //加锁
  queue_.push_back(info);
  //Detect中调用TakeFileIOInfo时有用到它，当queue中没有数据时用来阻塞线程
  queue_cv_.notify_one();
}

void Detect(vector<Issue> published_issues, shared_ptr<IOInfo> info) {
  while(true) {
    published_issues.clear();
    int ret = TakeFileIOInfo(info);
    for(auto detector:detectors_) {
      //通过detector检测IOInfo是否出现问题
      detector->Detect(env_, *info, published_issues);
    }
    if(issued_callback_ && !published_issues.empty()) {
      //通过issued_callback_回调出现问题的地方
      issued_callback_(published_issues);
    }
  }
}

//io_info_collector.cc
void OnOpen(pathname, flags, mode, open_ret, java_context) {
  std::shared_ptr<IOInfo> info = std::make_shared<IOInfo>(pathname, java_context);
  //open_ret为原open函数返回的文件fd
  //将当前打开的文件信息放入info_map_中
  if(info_map_.find(open_ret) == info_map.end()) {
    info_map_.insert(std::make_pair(open_ret, info));
  }
}
//read_ret返回实际读取的字节数，read_cost为read函数消耗的时间
void OnRead(fd, buf, size, read_ret, read_cost) {
  //OnWrite 一样处理，只是type不一样
  CountRWInfo(fd, FileOpType::kRead, size, read_cost);
}

void CountRWInfo(int fd, type, long op_size, long rw_cost) {
  IOInfo info = info_map_[fd];
  //不管读写都会增加
  info->op_cnt_ ++;
  info->op_size += op_size;
  info->rw_cost_us_ += rw_cost;
  //更新单次读写最长的时长
  if(rw_cost>info->max_once_rw_cost_time_us) {
    info->max_once_rw_cost_time_us = rw_cost;
  }
  //更新持续读写时长
  if(info->last_rw_time_us > 0 && (now - info->last_rw_time_us) < kContinualThreshold) {
    info->current_continual_rw_time_us += rw_cost;
  }else {
    info->current_continual_rw_time_us = rw_cost;
  }
  info->last_rw_time_us = now;
  //更新单次最大读写的字节数
  info->buffer_size = max(info->buffer_size, op_size);
  if(info->op_type == FileOpType::kInit) {
    info->op_type = type;
  }
}

std::shared_ptr<IOInfo> OnClose(int fd, close_ret) {
  IOInfo info = info_map_[fd];
  //start_time_us是构造函数中默认复制为now的
  info->total_cost_us = now - info->start_time_us;
  //通过stat函数获取文件的字节数
  info->file_size = GetFileSize(info.path.c_str);
  std::shared_ptr<IOInfo> infor = info;
  info_map_.erase(fd);
  return infor;
}

```

##### 4.2 FileIODetector解析

###### 4.2.1 IO执行在主线程FileIOMainThreadDetector

```c
//main_thread_detector.cc
void Detect(env, info, vector<Issue>& issues) {
  if(GetMainThreadId() == info.java_context_.thread_id) {
    int type = 0;
    //16ms的80%
    if(info.max_continueal_rw_cost_time_us > IOCanaryEnv::kPossibleNegativeThreshold){
      type = 1;
    }
    //kDefaultMainThreadTriggerThreshold = 500ms
    if(info.max_continueal_rw_cost_time_us > env.GetMainThreadThreshold()) {
      type |= 2;
    }
    //忽略
    //type = 1表示单次操作超过阈值
    //type = 2重复操作超过阈值
  }
}
```

###### 4.2.2 进程重复读取某个文件多次 RepeatReadInfo，相同文件在17ms内被重复读取

```c
//repeat_read_detector.cc
void Detect(env, info, issues) {
  string& path = info.path;
  if(observing_map_.find(path) == observing_map_.end()) {
    //如果持续读写时长不超过阈值就不管他
    if(info.max_continual_rw_cost_time_μs_ < env.kPossibleNegativeThreshold) {
      return;
    }
    observing_map_.insert(std::make_pair(path, vector<RepeatReadInfo>()));
  }
  vector<RepeatReadInfo>& repeat_info = observing_map_[path];
  if(info.op_type == FileOpType::kWrite) {
    repeat_info.clear();
  }
  //根据info创建一个RepeatReadInfo对象repeat_read_info
  if(repeat_info.empty()) {
    repeat_info.push_back(repeat_read_info);
    return;
  }
  //如果跟最后一个repeatreadinfo的op_timems(生成对象时赋值为GetTickCount)相差查过17ms，那么就清空
  if(GetTickCount() - repeat_info[repeat_info.size()-1].op_timems) > 17) {
   	repeat_info.clear(); 
  }
  bool found = false;
  int repeatCnt;
  for(auto& info_:repeat_infos) {
    if(info_ == repeat_read_info) {
      found = true;
      info.IncRepeatReadCount();
      repeatCnt = info.GetRepeatReadCount)();
      break;
    }
  }
  if(!found) {
    repeat_info.push_back(repeat_read_info);
    return;
  }
  if(repeatCnt >= env.GetRepeatThreadThreshold()) {
    //Publish
  }
}
```

###### 4.2.3  IO时buffer太小FileIOSmallBufferDetector

```c
//smaill_buffer_detector.cc
//1. 读写次数超过阈值
//2. 平均读写的字节大小小于阈值
//3. 持续读写时间超过阈值
//3者都满足才算问题
void Detect(env, info, issue) {
  //op_cnt为文件读写次数
  //op_size为每次读写操作的字节数的累加值
  if(info.op_cnt>env.kSmallBufferOpTimesThreshold 
     && info.op_size/info.opt_cnt < env.GetSmallBufferThreshold
     && info.max_continual_rw_cost_time_μs_ >= env.kPossibleNegativeThreshold) {
    //publish issue
  }
}
```

#### 重点

jni层的hook重点在与elfhook这个native层，Java层仅仅是参考了StrictMode的实现，采用了hook CloseGuard中的reporter，然后获取其中的某些关键字