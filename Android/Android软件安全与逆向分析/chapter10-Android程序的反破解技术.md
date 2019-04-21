### 10.1 对抗反编译
### 10.2 对抗静态分析
#### 10.2.1 代码混淆
#### 10.2.2 NDK保护
#### 10.2.3 外壳保护
### 10.3 对抗动态分析
#### 10.3.1 检测调试器
> android.os.Debug.isDebuggerConnected()
#### 10.3.2 检测模拟器
> - ro.product.model：模拟器中值为sdk
> - ro.build.tags：模拟器中为test-keys
> - ro.kernel.qemu：模拟器中为1
### 10.4 防止重编译
#### 10.4.1 检查签名
```java
PackageManager pm = context.getPackageManager();
pm.getPackageInfo(pkgName, PackageManager.GET_SIGNATURES);
Signature[] ss = info.signatures;
int hash = ss[0].hashCode();
```

#### 10.4.2 校验保护
> 检查安装后的classes.dex文件的hash值来判断apk是否被重新打包过