### IOCanaryPlugin原理

#### IPlugin接口方法调用时机

- init会在Matrix的私有构造函数中触发
- start会在各个plugin中手动调用，Matrix.startAllPlugins也会调用
- stop
- destroy

#### IOCanaryPlugin流程

1. init时获取包名，初始化core（IOCanaryCore）

2. IOCanaryPlugin.start实际上起作用的时IOCanaryCore，根据IOConfig中的配置决定是否检测IO和Closeable。

   ```java
   IOCanaryJniBridge.install(config, listener);
   mCloseGuardHooker = new GloseGuardHooker(issueListener);
   mCloseGuardHooker.hook()
   ```

#### CloseGuardHooker流程

> 通过反射调用CloseGuard.setEnable和setReporter方法来实现hook。
>
> CloseGuard在FileInputStream时会调用guard.open方法持有一个Throwable，在close时会调用guard.close方法释放当前持有的throwable，然后在FileInputStream.finalize方法中会根据是否持有throwable来判断有没有调用close方法。

#### IOCanaryJniBridge.install流程

>1. 根据config调用native enableDetector(int)方法，会调用native层的IOCanary::Get().RegisterDetectory方法, 给IOCanary设置detectors_(FileIOMainThreadDetector、FileIOSmallBufferDetector、FileIORepeatReadDetector)
>2. io_canary_jni.cc中的JNI_OnLoad中获取一些Java层的变量，比如IOCanaryJniBridge中的JavaContext等field以及OnIssuePublish方法设置给IOCanary::Get().SetIssuedCallback
>3. 调用native doHook方法，对应的c实现在io_canary_jni.cc中的doHook方法，用来hook掉libopenjdkjvm.so、libjavacore.so、libopenjdk.so中的open、open64、read、\_\_read_chk、write、\_\_write_chk、close方法
>4. 以read方法为例，用ProxyRead取代原来的read方法，类似代理实现，记录下read方法的时间，然后通过iocanary::IOCanary::Get().onRead()方法上报记录（io_canary.cc、io_canary.h），由IOInfoCollector实现(io_info_collector.cc、io_info_collector.h)，将这些信息写入一个map中
>5. 最后在close方法中，调用OfferFileIOInfo方法，然后会触发detetors_遍历调用Detect方法，读取IOInfo信息，然后通过PublishIssue方法调用java层的IOCanaryJniBridge.onIssuePublish

#### 重点

jni层的hook重点在与elfhook这个native层，Java层仅仅是参考了StrictMode的实现，采用了hook CloseGuard中的reporter，然后获取其中的某些关键字