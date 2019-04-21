### 第11章-Zygote和System进程的启动

#### 11.1 Zygote进程的启动脚本

Zygote进程是由android的第一个进程init启动起来的，init进程是在内核加载完成后就启动起来的。init在启动过程中会读取脚本文件init.rc。

```shell
# system/core/rootdir/init.rc
# zygote以service方式启动，对应的文件为/system/bin/app_process
# start-system-server表示要将System进程也启动
service zygote /system/bin/app_process -Xzygote /system/bin --zygote --start-system-server socket zygote stream 666
```

#### 11.2 Zygote进程的启动过程

```cpp
// frameworks/base/cmds/app_procecss/app_main.cpp
int main(int argc, char* const argv[]) {
  //...
  AppRuntime runtime(argv[0], computeArgBlockSize(argc, argv));
  //启动中参数加了--zygote --start-system-server，所以 
  zygote = true;
  startSystemServer = true;
  args.add(String8("start-system-server"));
  runtime.start("com.android.internal.os.ZygoteInit", args, zygote);  
}
//AppRuntime继承自AndroidRuntime
//frameworks/base/core/jni/AndroidRuntime.cpp
static AndroidRuntime* gCurRuntime = NULL;
AndroidRuntime::AndroidRuntime(char* argBlockStart, const size_t argBlockLength) {
  //...
  SkGraphics::Init();
  assert(gCurRuntime == NULL); //one per process
  gCurRuntime = this;
}
void AndroidRuntime::start(const char* className, const Vector<String8>& options, bool zygote) {
  setenv("ANDROID_ROOT", rootDir, 1); //rootDir = "/system"
  //开启vm
  JniInvocation jni_invocation;
  jni_invocation.Init(NULL);
  startVm(&mJavaVM, &env, zygote);
  onVmCreate(env);//这里回调至frameworks/base/cmds/app_process/app_main.cpp中的AppRuntime的onVmCreate方法
  //注册jni方法
  startReg(env);
  //调用start方法中传入的class的main方法
  jmethodID startMeth = env->GetStaticMethodID(startClass,"main","([Ljava/lang/String;])V");
  env->CallStaticVoidMethod(startClass,startMeth,strArray)''
}

int AndroidRuntime::startReg(JNIEnv* env) {
  //gRegJNI是REG_JNI宏的数组，NELEM是获取REG_JNI数组的长度
  register_jni_procs(gRegJNI, NELEM(gRegJNI), env);
}

static int register_jni_procs(const RegJNIRec array[], size_t count, JNIEnv* env) {
  array[i].mProc(env);
}

#define REG_JNI(name) {name, #name}
struct RegJNIRec {
  int (*mProc)(JNIEnv*);
  const char* mName;//指向方法
}
//以REG_JNI(register_com_android_intenral_os_RuntimeInit)为例，最终startReg会遍历gRegJNI中的结构体，调用jniRegisterNativeMethods方法关联jni
int register_com_android_internal_os_RuntimeInit(JNIEnv* env) {
  const JNINativeMethod methods[] = {
    {"nativeFinishInit","()V",(void*) com_android_intenral_os_RuntimeInit_nativeFinishInit}_os_RuntimeInit_nativeFinishInit
  },{
    "nativeSetExitWithoutCleanup","(Z)V",(void*)com_android_internal_os_RuntimeInit_nativeSetExitWithoutCleanup
  },}:
	return jniRegisterNativeMethods(env,"com/android/internal/os/RuntimeInit",methods,NELEM(methods));
}
```

即app_process.cpp的main方法会通过AppRuntime.start启动VM，通过startReg方法关联JNI，然后调用**ZygoteInit.main**方法。

```java
//com.android.internal.os.ZygoteInit.java
public static void main(String[] args) {
  boolean startSystemServer = true;
  String socketName = "zygote";//默认为zygote，--zygote-name xxx修改socketName
  ZygoteServer zygoteServer = new ZygoteServer();
  //通过LocalServerSocket关联到"ANDROID_SOCKET_$socketName"的fd
  zygoteServer.registerServerSocketFromEnv(socketName);
  //preload IcuCahe、classes、资源、opengl、sharedlibrary、textResource
  preload();
  //通过Zygote.forkSystemServer->frameworks/base/core/jni/com_android_intenral_os_Zygote.cpp#com_android_internal_os_Zygote_nativeForkSystemServer方法，实际上通过fork方法启动一个子进程。最后调用handleSystemServerProcess
  Runnable r = forkSystemServer(abiList, socketName, zygoteServer);
  r.run();
  //循环接收命令，最后会触发ZygoteConnection.processOneCommand方法来fork出app对应的进程，比如在AMS启动activity时会通过Process.start方法创建新的进程，最终还是通过触发到这里来真正的创建新进程
  Runnable caller = zygoteServer.runSelectLoop(abiList);
  caller.run();
}

private static Runnable forkSystemServer() {
  Zygote.forkSystemServer();
  //"--nice-name=system_server  com.android.server.SystemServer"
  return handleSystemServerProcess();
}

private static Runnable handleSystemServerProcess() {
  performSystemServerDexOpt();
  if(parsedArgs.invokeWith != null) {
    //调用WrapperInit.main方法，创建classloader，调用RuntimeInit.applicationInit
    WrapperInit.execApplication();
  }else {
    //调用RuntimeInit.commonInit()以及RuntimeInit.applicationInit方法调用SystemServer.main()
    ZygoteInit.zygoteInit();
  }  
}
```

**SystemServer.main**

```java
public static void main(String[] args){
  new SystemServer().run();
}

private void run() {
  createSystemContext();//ActivityThread.systemMain();
  mSystemServiceManager = new SystemServiceManager();
  LocalServices.addService(SystemServiceManager.class, mSystemServiceManager);
  //开启系统服务
  startBootstrapServices();
  startCoreServices();
  startOtherServices();  
  //消息队列开始loop
  Looper.loop();
}
```

AMS的启动

```java
//SystemServer.java
//SystemServiceManager.startService方法会触发Lifecycle.onStart方法，接着是AMS.start方法
mSystemServiceManager.startService(ActivityManagerService.Lifecycle.class).getService();
//ActivityManagerService.java
private void start() {
  LocalServices.addService(ActivityManagerInternal.class, new LocalService());
}
//从上可知，每次启动Service后，都会添加到LocalServices中
```

PMS的启动

```java
//跟AMS不一样，PMS是通过PMS.main方法启动
public static PackageManagerService main() {
  PMS m = new PMS();
  ServiceManager.addService("package",m);
}
//frameworks/base/core/java/android/os/ServiceManager.java
public static void addService(String name, IBinder service) {
  //重载的方法，最后会调用
  getIServiceManager().addService(name, service, allowIsolated, dumpPriority);
}
private static IServiceManager getIServiceManager() {
  //忽略
  return ServiceManagerNative.asInterface(Binder.allowBlocking(BinderInternal.getContextObject()));
}
```

