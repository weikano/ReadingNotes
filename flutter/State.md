### State
State包含了某些信息:
1. 在widget的build过程中可以被同步的读取
2. 在widget的生命周期中可能会变化。通过state.setState方法来告知widget来响应变化(将当前element添加到dirty element list中，在WidgetsBinding.drawFrame中会调用_buildScope)

#### 生命周期
- StatefulWidget通过createState方法生成一个state对象
- 新生成的state永久关联到一个BuildContext。这时state可以认为时mounted
- 调用initState方法。state的子类应该override initState方法来执行一个一次性的初始化。
- 调用didChangeDependencies。当BuildContext.inheritFromWidgetOfExactType调用时也会触发didChangeDependencies。
- 现在state已经初始化完成，可以调用build方法
- 开发时，热加载触发时会调用reassemble
- 当state被移除时，调用deactivate方法。然后dispose