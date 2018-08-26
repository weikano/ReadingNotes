### 一、动态调试步骤
1. 反编译apk
> 修改debuggable属性，并在入口处加上waitForDebug方法，进行debug等待
2. 回编译apk
3. 将反编译smali工程导入IDE
4. 设置远程调试
5. 调试apk程序
6. 编写代码实现核心逻辑