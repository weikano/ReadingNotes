### 第12章-android应用程序进程的启动过程

AMS在启动一个APP时，如果发现组件需要的进程没有启动起来，就会请求zygote进程将这个应用程序进程启动起来。

zygote进程是通过fork的方式来创建一个新的应用程序进程的。因此，通过fork它而得到的应用程序进程就会有一个VM的实例拷贝。除了VM拷贝之外，还包括一个Binder线程池和一个消息循环。

#### 12.1 应用程序进程的创建过程

AMS.startProcessLocked到Process.start到ZygoteProcess.startViaZygote->zygoteSendArgsAndGetResult，其中entryPoint（processClass）是ActivityThread。ZygoteProcess.zygoteSendArgsAndGetResult通过LocalSocket连接到Zygote进程的LocalServerSocket来创建进程。

Zygote进程启动时，会在内部创建一个名称为zygote的Socket，这个socket与设备文件dev/socket/zygote班定在一起。

接下来zygote进程会在ZygoteServer.runSelectLoopMode中接收到一个创建新的应用程序进程的请求

```java
//ZygoteServer.java
private static void runSelectLoopMode() {
  zygoteConnection.processOneCommand();
}
//ZygoteConnection.java
Runnable processOneCommand() {
  Zygote.forkAndSpecialize()
} 
//Zygote.java
public static int forkAndSpecialize() {
  int pid = nativeForkAndSpecialize();
}
//frameworks/base/core/jni/com_android_internal_os_Zygote.cpp
static int com_android_internal_os_Zygote_nativeForkAndSpecialize() {
  pid_t pid = ForkAndSpecializeCommon();
}
static pid_t ForkAndSpecializeCommon(){
  //设置uid、pid、gid之类的...
  pid_t pid = fork();
  if(pid == 0) {
    //在子进程中调用Zygote.callPostForkChildHooks方法，最后调用RuntimeInit.applicationInit，通过findStaticMain方法执行ActivityThread.main
    env->CallStaticVoidMethod(gZygoteClass,//com.android.internal.os.Zygote
                              gCallPostForkChildHooks,//"callPostForkChildHooks"
                             xxx,);
  }
}

```

#### 12.2 Binder线程池的启动过程

一个新的应用程序进程在创建完成后，会调用ZygoteInit.zygoteInit方法，进而触发ZygoteInit.nativeZygoteInit来创建Binder线程池。

```c++
//frameworks/base/cmds/app_prcess/app_main.cpp
virtual void onZygoteInit() {
  sp<ProcessState> proc = ProcessState::self();
  proc->startThreadPool();
}
//frameworks/base/libs/binder/ProcesState.cpp
void ProcessState::startThreadPool(){
  spawnPooledThread(true);
}
void ProcessState::spawnPooledThread(bool isMain){
  sp<Thread> t = new PoolThread(isMain);
  t->run(name.string());
}
virtual bool threadLoop(){
  IPCThreadState::self()->joinThreadPool(mIsMain);
  return false;
}

```

#### 12.3 消息循环的创建过程

```java
//ActivityThread.java
public static void main(String[] args) {
  Looper.prepareMainLooper();
  ActivityThread thread = new ActivityThread();
  thread.attach(false, startSeq);
  lopper.loop();
}

```

