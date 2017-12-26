- 找到正确的process，有些process无法在debugger中找到是因为android.os.Process这个类里面可以看到android进程启动过程有这么一句, 所以，建议调试framework需要使用模拟器或者原生系统.

```
if ((debugFlags & Zygote.DEBUG_ENABLE_DEBUGGER) != 0) {
    argsForZygote.add("--enable-debugger");
}
```

- 如果源码中没有包含所要调试的类(比如@hide)，或者不是SDK的类(比如系统app的源码)，可以去 https://android.googlesource.com/下载对应的代码，然后导入android studio，然后在导入的代码中断点。
- 如果行号不对应，使用方法断点。