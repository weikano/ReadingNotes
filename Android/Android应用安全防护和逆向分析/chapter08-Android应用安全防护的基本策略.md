### 一、混淆机制
#### 1. 代码混淆
破解查看Java层代码的方式有两种
- 直接解压class.dex文件，用dex2jar转换成jar文件，再使用jd-gui查看
- apktool反编译后查看smali源码
  
#### 2. 资源混淆
类似微信的**AndResGuard**，混淆后资源的name会被替换称类似a,b,c,d之类的简写，减小resource.arsc文件的大小以及增加破解后查找资源的难度。
但是一般在反编译的Java代码中，看到的获取资源值的时候，并不是资源的name，而是资源对应的int值，可以通过对应的int值在public.xml中查找对应的name，然后再去strings.xml中根据name查找。

### 二。签名保护
```Java
//获取签名
PackageInfo info = context.getPackageManager().getPackageInfo(context,pkgName, PackageManager.GET_SIGNATURES);
StringBuilder sb = new StringBuilder();
for(String signature : info.signatures) {
    sb.append(signature);
}

```
上述在Java层的签名验证很容易在反编译之后破解

### 三、手动注册native方法
通过jni_onload方法调用registerNative方法来注册对应的native方法，相对来说破解起来更麻烦。

### 反调试检测
IDA进行so动态调试是基于进程的注入技术，然后使用Linux的ptrace机制，进行调试目标进程的附加操作。ptrace机制有一个特点，如果一个进程被调试了，那么在它进程的status文件中TracerPid会记录调试者的pid。
反调试：轮询遍历自己进程的status文件，读取TracerPid的值，如果大于0，那么就是在调试，立马退出。