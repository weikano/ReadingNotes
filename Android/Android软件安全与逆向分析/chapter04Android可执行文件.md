### 4.1 Android程序的生成步骤
1. 打包资源文件，生成R文件。aapt位于android-sdk\platform-tools目录下，源码在frameworks/base/tools/aapt目录。主要调用了Resource.cpp文件中的buildResources函数，然后调用makeFileResources对资源文件进行处理，包括向资源表table添加条目等，处理完之后调用compileResourceFile函数编译res与assets目录下的资源并生成resources.arsc，compileResourceFile位于ResourceTable.cpp中，该函数最终会调用parseAndAddEntry函数生成R.java文件。编译完成后，接着调用compileXmlFile对res目录下的子目录下的xml文件进行编译加密，最后将resources.arsc文件以及加密过的AndroidManifest文件打包压缩成resources.ap_文件。
2. 处理aidl文件。工具在sdk/platform-tools目录下的aidl，源码在frameworks/base/tools/aidl目录下
3. 编译源码，生成class文件。调用javac；编译native代码。
4. 生成dex文件。sdk/platform-tools/dx。
5. 打包apk文件。使用sdk/tools/apkbuilder，实际调用的是sdk/tools/lib/sdklib，源码位于sdk/sdkmanager/libs/sdk/sdklib/src/。
6. 签名。使用jarsigner或者build/tools/signapk目录下的signapk工具
7. 对齐处理。使用sdk/tools/zipalign，源码位于build/tools/zipalign目录。验证是否对齐过由ZipAlign.cpp文件的verify完成，对齐工作由process完成。

[image](001.png)

### 4.2 安装流程
> 最终都会调用PMS的scanPackageLI方法，然后调用mInstaller.install方法来安装程序，Installer通过socket向/system/bin/installd发送install指令（frameworks/base/cmds/installd），由installd.c中的do_install方法最终调用install，后者在commands.c中有它的实现代码。

### 4.3 dex文件格式
> dex文件的结构声明在dalvik/libdex/DexFile.h中
[image](002.png)