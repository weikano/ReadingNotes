### 3.1 dalvik虚拟机得到特点
#### 3.1.2 dalvik虚拟机与jvm的区别
1. JVM运行的是Java字节码，dalvik运行的是dalvik字节码
> dalvik字节码通过Java字节码转换而来，打包到一个dex文件中
2. dalvik可执行文件体积更小
> Android SDK中的dx工具负责将Java字节码转换为dalvik字节码。dx工具会将Java类文件重新排序，消除在类文件中出现的所有冗余信息，避免虚拟机初始化时出现重复的文件加载和解析过程。另外它将所有Java类文件中的常量池进行分解，消除其中的冗余信息，重新合并成一个常量池，所有的类共享一个常量池。
3. jvm与dalvik架构不同
> JVM基于栈架构。dalvik基于寄存器，指令少而且精简

#### 3.1.3 dalvik是如何执行程序的
