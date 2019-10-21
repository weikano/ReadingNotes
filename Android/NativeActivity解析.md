# NativeActivity解析

## 1. Java层代码NativeActivity

```java
void onCreate() {
  //libname和funcname都可以在Androidmanifest中通过meta-data配置
  //默认加载libmain.so
  String libname = "main";
  //onCreate中通过loadNativeCode方法调用funcname方法
  String funcname = "ANativeActivity_onCreate";    
  BaseDexClassLoader classLoader = getClassLoader();
  String path = classLoader.findLibrary(libname);
  mNativeHandler = loadNativeCode(path, funcname, xxx);
}

void onDestroy() {
  unloadNativeCode(mNativeHandler);
  super.onDestroy();
}
//其他生命周期也会调用对应的native方法
void onPause() {
  super.onPause();
  onPauseNative(mHandler);
}
```

## 二、如何使用

由`Java`层代码可知，一般来说`NativeActivity`首先调用的`Native`层方法是`ANativeActivity_onCreate`

而`Android`的`NDK`本身提供了静态`library`：`native_app_glue`来便于实现相关功能，其代码位于ANDROID_NDK/sources/android/native_app_glue。

如需使用：

```cmake
//CMakeLists.txt
add_library(native_app_glue STATIC
	${ANDROID_NDK}/sources/android/native_app_glue/android_native_app_glue.c)
# 导出符号表
set(CMAKE_SHARED_LINKER_FLAGS
    "${CMAKE_SHARED_LINKER_FLAGS} -u ANativeActivity_onCreate")
target_link_libraries(xxx
		native_app_glue
)	
```

然后在`native`层代码中：

```c++
//xxx.cpp
#include<android_native_app_glue.h>
//实现下述方法
void android_main(struct android_app* app) {
}
```

## 三、原理

`NativeActivity`最先调用的是`ANativeActivity_onCreate`方法，那么`android_main`又是何物？

答案在于`android_native_app_glue`这个静态库，其只有`android_native_app_glue.h`和`android_native_app_glue.c`两个文件

`android_native_app_glue.h`中只是定义了相关的结构体以及一些需要实验的方法，那么对应的`JNI`层代码是如何关联至`java`层的？答案在于头文件`android/native_activity.h`

而`android/native_activity.h`这个头文件，又是关联在`android_runtime/android_app_NativeActivity.h`中，交由`android_app_NativeActivity.cpp`来实现关联

```c++
//android_app_NativeActivity.cpp
static const JNINativeMethod g_methods = {
  {"loadNativeCode", 
  "(Ljava/lang/String;Ljava/lang/String;Landroid/os/MessageQueue;Ljava/lang/String;Ljava/lang/String;Ljava/lang/String;ILandroid/content/res/AssetManager;[BLjava/lang/ClassLoader;Ljava/lang/String;)J",
        (void*)loadNativeCode_native},
  { "onResumeNative", "(J)V", (void*)onResume_native },
  //XXX其他方法
};

static const char* const kNativeActivityPathName = "android/app/NativeActivity";
//动态注册jni方法
//在AndroidRuntime.cpp中调用该方法
int register_android_app_NativeActivity(JNIEnv* env)
{
    //ALOGD("register_android_app_NativeActivity");
    jclass clazz = FindClassOrDie(env, kNativeActivityPathName);

    gNativeActivityClassInfo.finish = GetMethodIDOrDie(env, clazz, "finish", "()V");
    gNativeActivityClassInfo.setWindowFlags = GetMethodIDOrDie(env, clazz, "setWindowFlags",
                                                               "(II)V");
    gNativeActivityClassInfo.setWindowFormat = GetMethodIDOrDie(env, clazz, "setWindowFormat",
                                                                "(I)V");
    gNativeActivityClassInfo.showIme = GetMethodIDOrDie(env, clazz, "showIme", "(I)V");
    gNativeActivityClassInfo.hideIme = GetMethodIDOrDie(env, clazz, "hideIme", "(I)V");

    return RegisterMethodsOrDie(env, kNativeActivityPathName, g_methods, NELEM(g_methods));
}
```

由`NativeActivity.java`可知，`ANativeActivity_onCreate`方法是通过`loadNativeCode`方法调用的，而`g_methods`中`loadNativeCode`对应的是`loadNativeCode_native`方法

```cpp
static jlong
loadNativeCode_native(JNIEnv* env, jobject clazz, jstring path, jstring funcName,
        jobject messageQueue, jstring internalDataDir, jstring obbDir,
        jstring externalDataDir, jint sdkVersion, jobject jAssetMgr,
        jbyteArray savedState, jobject classLoader, jstring libraryPath) {
    if (kLogTrace) {
        ALOGD("loadNativeCode_native");
    }

    ScopedUtfChars pathStr(env, path);
    std::unique_ptr<NativeCode> code;
    bool needs_native_bridge = false;

    char* nativeloader_error_msg = nullptr;
  //打开对应的so文件
    void* handle = OpenNativeLibrary(env,
                                     sdkVersion,
                                     pathStr.c_str(),
                                     classLoader,
                                     nullptr,
                                     libraryPath,
                                     &needs_native_bridge,
                                     &nativeloader_error_msg);

    if (handle == nullptr) {
        g_error_msg = nativeloader_error_msg;
        NativeLoaderFreeErrorMessage(nativeloader_error_msg);
        ALOGW("NativeActivity LoadNativeLibrary(\"%s\") failed: %s",
              pathStr.c_str(),
              g_error_msg.c_str());
        return 0;
    }
	//funcPtr即为对应的ANativeActivity_onCreate方法指针
    void* funcPtr = NULL;
    const char* funcStr = env->GetStringUTFChars(funcName, NULL);
    if (needs_native_bridge) {
        funcPtr = NativeBridgeGetTrampoline(handle, funcStr, NULL, 0);
    } else {
        funcPtr = dlsym(handle, funcStr);
    }

    code.reset(new NativeCode(handle, (ANativeActivity_createFunc*)funcPtr));
    env->ReleaseStringUTFChars(funcName, funcStr);

    if (code->createActivityFunc == NULL) {
        g_error_msg = needs_native_bridge ? NativeBridgeGetError() : dlerror();
        ALOGW("ANativeActivity_onCreate not found: %s", g_error_msg.c_str());
        return 0;
    }

    code->messageQueue = android_os_MessageQueue_getMessageQueue(env, messageQueue);
    if (code->messageQueue == NULL) {
        g_error_msg = "Unable to retrieve native MessageQueue";
        ALOGW("%s", g_error_msg.c_str());
        return 0;
    }

    int msgpipe[2];
    if (pipe(msgpipe)) {
        g_error_msg = android::base::StringPrintf("could not create pipe: %s", strerror(errno));
        ALOGW("%s", g_error_msg.c_str());
        return 0;
    }
  //epoll
    code->mainWorkRead = msgpipe[0];
    code->mainWorkWrite = msgpipe[1];
    int result = fcntl(code->mainWorkRead, F_SETFL, O_NONBLOCK);
    SLOGW_IF(result != 0, "Could not make main work read pipe "
            "non-blocking: %s", strerror(errno));
    result = fcntl(code->mainWorkWrite, F_SETFL, O_NONBLOCK);
    SLOGW_IF(result != 0, "Could not make main work write pipe "
            "non-blocking: %s", strerror(errno));
    code->messageQueue->getLooper()->addFd(
            code->mainWorkRead, 0, ALOOPER_EVENT_INPUT, mainWorkCallback, code.get());

    code->ANativeActivity::callbacks = &code->callbacks;
    if (env->GetJavaVM(&code->vm) < 0) {
        g_error_msg = "NativeActivity GetJavaVM failed";
        ALOGW("%s", g_error_msg.c_str());
        return 0;
    }
    code->env = env;
    code->clazz = env->NewGlobalRef(clazz);

    const char* dirStr = env->GetStringUTFChars(internalDataDir, NULL);
    code->internalDataPathObj = dirStr;
    code->internalDataPath = code->internalDataPathObj.string();
    env->ReleaseStringUTFChars(internalDataDir, dirStr);

    if (externalDataDir != NULL) {
        dirStr = env->GetStringUTFChars(externalDataDir, NULL);
        code->externalDataPathObj = dirStr;
        env->ReleaseStringUTFChars(externalDataDir, dirStr);
    }
    code->externalDataPath = code->externalDataPathObj.string();

    code->sdkVersion = sdkVersion;

    code->javaAssetManager = env->NewGlobalRef(jAssetMgr);
    code->assetManager = NdkAssetManagerForJavaObject(env, jAssetMgr);

    if (obbDir != NULL) {
        dirStr = env->GetStringUTFChars(obbDir, NULL);
        code->obbPathObj = dirStr;
        env->ReleaseStringUTFChars(obbDir, dirStr);
    }
    code->obbPath = code->obbPathObj.string();

    jbyte* rawSavedState = NULL;
    jsize rawSavedSize = 0;
    if (savedState != NULL) {
        rawSavedState = env->GetByteArrayElements(savedState, NULL);
        rawSavedSize = env->GetArrayLength(savedState);
    }
		//调用createActivityFunc，即ANativeActivity_onCreate方法
    code->createActivityFunc(code.get(), rawSavedState, rawSavedSize);

    if (rawSavedState != NULL) {
        env->ReleaseByteArrayElements(savedState, rawSavedState, 0);
    }

    return (jlong)code.release();
}
```

接下来找到`ANativeActivity_onCreate`方法

```c++
//native_activity.h
typedef void ANativeActivity_createFunc(ANativeActivity* activity,
        void* savedState, size_t savedStateSize);
//即ANativeActivity_onCreate是一个返回值为void，接收ANativeActivity*和void*以及size_t参数的方法
//extern表明在其他文件中申明
extern ANativeActivity_createFunc ANativeActivity_onCreate;
```

接下来回到`android_native_app_glue`库

```c++
//android_native_app_glue.c
JNIEXPORT
void ANativeActivity_onCreate(ANativeActivity* activity, void* savedState,
                              size_t savedStateSize) {
    LOGV("Creating: %p\n", activity);
    activity->callbacks->onDestroy = onDestroy;
    activity->callbacks->onStart = onStart;
    activity->callbacks->onResume = onResume;
    activity->callbacks->onSaveInstanceState = onSaveInstanceState;
    activity->callbacks->onPause = onPause;
    activity->callbacks->onStop = onStop;
    activity->callbacks->onConfigurationChanged = onConfigurationChanged;
    activity->callbacks->onLowMemory = onLowMemory;
    activity->callbacks->onWindowFocusChanged = onWindowFocusChanged;
    activity->callbacks->onNativeWindowCreated = onNativeWindowCreated;
    activity->callbacks->onNativeWindowDestroyed = onNativeWindowDestroyed;
    activity->callbacks->onInputQueueCreated = onInputQueueCreated;
    activity->callbacks->onInputQueueDestroyed = onInputQueueDestroyed;
		//将activity->instance设置为android_app_create方法的返回值
    activity->instance = android_app_create(activity, savedState, savedStateSize);
}
```

由`ANativeActivity_onCreate`方法可知，最后会调用`android_app_create`方法

```c++
//android_native_app_glue.c
static struct android_app* android_app_create(ANativeActivity* activity, void* savedState, size_t savedStateSize) {
  struct android_app* android_app = (struct android_app*)malloc(sizeof(struct android_app));
  memset(android_app, 0, sizeof(struct android_app));
  android_app->activity = activity;
  //mutex同步锁，忽略
  //启动线程调用android_app_entry方法
  pthread_create(&android_app->thread, &attr, android_app_entry, android_app);
}

static void* android_app_entry(void* param) {
  //忽略上述变量设置
  android_main(android_app);
  android_app_destroy(android_app);
}
```

最后触发的还是`android_main`方法

```c++
//android_native_app_glue.h
//即android_main方法需要自己实现
extern void android_main(struct android_app* app);
```

而在`android_main`中我们要如何处理生命周期呢？

```cpp
void android_main(android_app* state) {
  state->onAppCmd = handle_cmd;
  state->onInputEvent = handle_input;
}

static void handle_cmd(android_app* app, int32_t cmd) {
  switch (cmd) {
        case APP_CMD_SAVE_STATE:
            // The system has asked us to save our current state.  Do so.
            break;
        case APP_CMD_INIT_WINDOW:
            // The window is being shown, get it ready.          
            break;
        case APP_CMD_TERM_WINDOW:
            // The window is being hidden or closed, clean it up.
            break;
        case APP_CMD_GAINED_FOCUS:
            // When our app gains focus, we start monitoring the accelerometer.            
            break;
        case APP_CMD_LOST_FOCUS:
            // When our app loses focus, we stop monitoring the accelerometer.
            // This is to avoid consuming battery while not being used.            
            // Also stop animating.
            break;
    }
}
```





