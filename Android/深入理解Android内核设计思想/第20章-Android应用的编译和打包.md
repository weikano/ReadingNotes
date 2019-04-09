### 第20章-Android应用的编译和打包

#### 20.3 apk编译过程安排

1. 首先.aidl文件通过aidl工具转换成Java接口文件
2. 资源文件被aapt编译成最终的resouces.arsc，并生成R.java
3. 生成dex文件
4. dex、resources.arsc以及其他资源通过apkbuilder生成最初的apk文件
5. apksigner进行前面和zipalign等优化

#### 20.4 信息安全基础概述

