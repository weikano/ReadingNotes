### chapter03-Communicating with Native Code using JNI

#### Operations on Rereference Types

引用类型是直接将引用传递给native code，他们不能被直接使用和修改。JNI提供了一系列的方法来操作这些引用类型。包含：Strings、Arrays、NIO Buffers、Fields、Methods

##### String Operations

java String 是作为引用类型来被JNI处理的。他们不能像native C string一样被使用。JNI提供方法将就是jstring转换为c strings。因为java string是不可变的，jni没有提供任何修改jstring内容的方法

```c++
//new string
jstring javaString;
javaString = env->NewStringUTF("Hello world");
//Converting a Java String to C String
const jbyte* str;
jboolean isCopy;
str = env->GetStringUTFChars(env, javaString, &isCopy);
if(0 != str) {
  printf("Java String :%s", str);
  if(JNI_TRUE = isCopy) {
    printf("C String is a copy of the Java String");
  }else {
    printf("C String points to actual String");
  }
}
//释放
env->ReleaseStringUTFChars(javaString, str);
```

##### Array Operations

```c++
//new array
jintArray array = env->NewIntArray(10);
//copy
jint nativeArray[10] = env->GetIntArrayRegion(array, 0, 10 ,nativeArray);
//设置回去
env->SetIntArrayRegion(array, 0, 10, nativeArray);
//操作指针
jint* nativeDirectArray;
jboolean isCopy;
nativeDirectArray = env->GetIntArrayElements(array, &isCopy);
//进行操作……
//释放,最后一个参数：
//0表示copy back the content and free native array
//JNI_COMMIT：copy back content but do not free the native array.
//JNI_ABORT：free native array without copying its content
env->ReleaseIntArrayElements(array, nativeDirectArray, 0);
```

##### Direct Byte Buffer

```c++
//创建一个direct byte buffer
unsigned char* buffer = static_cast<unsigned char*>(malloc(1024));
//...其他对buffer的操作
jobject directBuffer = env->NewDirectByteBuffer(buffer, 1024);
free(buffer);
//获取一个direct buffer
unsigned char* buffer = env->GetDirectBufferAddress(directBuffer);
```

##### Accessing Fields

```c++
public class JavaClass {
  private String instanceField = "instance Field";
  private static String staticField = "static Field";
  private String instanceMethod() {
    return "instanceMethod";
  }
  private static String staticMethod() {
    return "Instance MEthod";
  }
}

jclass clazz = env->GetObjectClass(instance);
jfieldID instanceField = env->GetFieldID(clazz, "instanceField", "Ljava/lang/String;");
jfieldID staticField = env->GetStaticFieldID(clazz, "staticField", "Ljava/lang/String;");
```

##### Calling method

```c++
jmethodID instanceMethod = env->GetMethodID(clazz, "instancecMethod", "()Ljava/lang/String;");
jmethodID staticMethod = env-GetStaticMethodID(clazz,"staticMethod","()Ljava/lang/String;");
jstring imr = env->CallStringMethod(instance, instanceMethodID);
```

#### Local And Global Reference

- LocalReference：大部分jni函数返回local reference。不能被缓存，native方法结束后就会被释放；也可以手动调用env->DeleteLocalRef释放

- GlobalReference：除非在native中手动释放，jvm不会去释放

- WeakGlobalRef：跟GlobalRef一样，但是不会阻止jvm释放持有的对象

  ```c++
  if(JNI_FALSE == env->IsSameObject(weakGlobalCLazz, NULL)) {
    //object is still alive and can be used
  }else {
    //object is gc and cannot be used
  }
  ```

#### Threading

- LocalRef 不能在多线程之间共享，只有globalref可以
- JNIEnv只在调用方法的线程中有效，不能被缓存给其他线程使用

```c++
if(JNI_OK == env->MonitorEnter(env, obj)) {
  //error_handle
}
//同步代码
if(JNI_OK == env->MonitorExit(env, obj)) {
  //error_handle
}
```

native端的线程必须关联到JVM

```c++
JavaVM* gVM;
JNIEnv× gEnv;
gVM->AttachCurrentThread(env, NULL);
//接下来可以通过JNIEnv跟Java端应用通信
gVM->DetachCurrentThread();
```

