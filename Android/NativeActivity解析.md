### NativeActivity解析

#### 1. Java层代码NativeActivity

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

#### 二、native代码

##### 2.1 JNI相关注册

```c++
//android_app_NativeActivity.cpp以及
int register_android_app_NativeActivity(JNIEnv* env) {
  //注册对应的方法
}
//AndroidRuntime.cpp
REG_JNI(register_android_app_NativeActivity);
int AndroidRuntime::startReg(JNIEnv* env) {
  if(register_jni_procs(gRegJNI,xxx)) {
    env_PopLocalFrame(NULL);
  }
}
```

##### 2.2 相关h和cpp文件结构

android_app_NativeActivity.cpp包含android_app_NativeActivity.h，android_app_NativeActivity.h包含native_acitivity.h。

native_activity.cpp包含了android_runtime/android_app_NativeActivity.h。

##### 2.3 native_app_glue

native_app_glue.c中实现了ANativeActivity_onCreate方法

```java
//${NDK-BUNDLE}/sources/anroid/native_app_glue/native_app_glue.c
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

    activity->instance = android_app_create(activity, savedState, savedStateSize);
}
//android_app_create方法优惠出发android_app_entry，android_app_entry又会出发android_main方法，这个方法需要自己在include <native_app_glue.h>后自己实现
//native_app_glue.h
//一般实现时会设置app->onAppCmd和app->onInputEvent来相应Activity对应的状态
//onAppCmd一般是Activity生命周期变化
//onInputEvent是输入事件响应
extern void android_main(struct android_app* app);

```





