### 3.5.2 ART优化机制基础
> - dalvik：dex2opt
> - ART：dex2oat
> - 二者优化代码路径及文件名都为/data/dalvik-cache/app/data@app@{packageName}.apk@classes.dex
> - compiler：主要负责dalvik字节码到本地代码的转换，编译为libart-compiler.so
> - dex2oat：完成dex文件到elf文件转换，编译为dex2oat
> - runtime：ART运行时源代码，编译为libart.so