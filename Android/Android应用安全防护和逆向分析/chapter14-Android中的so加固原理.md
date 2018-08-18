### 一、基于对so中的section加密实现so加固
#### 1. 技术原理
加密过程：找到so文件中的一个section的起始地址和大小就可以级加密了
解密过程：对一个so来说，当被加载到程序之后，在运行程序之前。__attribute__((constructor))，在main之前执行，类似于Java中的构造函数。
#### 2. 实现方案
编写一个简单的native代码：
- 将核心的native函数定义在一个自己的section中，__attribute__((section(".mytext)));其中.mytext是定义的section。
- 编写解密函数，用__attribute__((constructor));声明。

编写加密程序：
- 解析so文件，找到.mytext的起始地址和大小
- 找到.mytext段后进行加密再写入文件中。

#### 3. 代码实现
//TBD