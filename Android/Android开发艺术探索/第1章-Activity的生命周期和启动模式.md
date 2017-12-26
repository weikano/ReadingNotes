# Activity的生命周期全面分析
> 正常情况下Activity的生命周期

![image](https://raw.githubusercontent.com/weikano/NoteResources/master/10.png)
               
## 典型情况下的生命周期分析

1. onCreate(): Activity正在被创建.
2. onRestart(): 正在重新启动, 一般情况下是从不可见状态重新变换为可见状态时触发.
3. onStart():正在启动, 即将开始. 已经可见, 但是还没出现在前台, 无法交互.
4. onResume(): 可见可交互, 在前台. 
5. onPause(): 正在停止, 紧接着会触发onStop().
6. onStop():即将停止.
7. onDestroy():即将销毁.
  
  
onStart和onResume, onPause和onStop看起来差不多. 但是这两个配对回调分别表示不同的意义, onStart和onStop是从Activity是否可见的来回调, 而onResume和onPause是从Activity是否位于前台来回调.

## 异常情况下的生命周期分析
1. 资源相关的系统配置发生变化导致Activity被杀死并重新创建.
![image](https://raw.githubusercontent.com/weikano/NoteResources/master/11.png)
2. 系统内存不足导致低优先级的Activity被杀死.

> Activity按照优先级从高到低, 可以分为如下三种:
> 1. 前台Activity---正在和用户交互的Activity, 优先级最高.
> 2. 可见到非前台的Activity---比如被对话框遮挡一部分的Activity, 可见但无法和用户直接交互.
> 3. 后台Activity----已经被暂停的Activity, 比如执行了onStop.

# Activity的启动模式

## Activity的launchMode
1. standard: 每次启动都会创建一个新的Activity实例. 一个任务栈中可以有多个实例, 每个实例也可以属于不同的任务栈. **在这种模式下, 谁启动了这个Activity, 这个实例就运行在启动它的那个Activity所在的栈当中**.
2. singleTop: 如果新的Activity已经位于栈顶, 那么不会重新创建, 同时它的onNewIntent会被回调. 如果实例已存在但不是位于栈顶, 那么新的实例仍然会重新创建. 
3. singleTask: 栈内复用模式. 只要Activity在一个栈中存在, 那么多次启动Activity都不会创建实例, 智慧回调onNewIntent. 当一个具有singleTask模式的Activity请求启动后, 比如A, 系统首先回去寻找是否存在Ax想要的任务栈, 如果不存在就重新创建一个, 然后将A的实例放到栈中. 如果存在, 这时要看栈中是否有A的实例, 如果有实例, 那么系统就会把A调到栈顶并调用onNewIntent, 同时由于singleTask默认具有clearTop, 会导致A上所有的Activity出栈. 如果不存在, 就创建A并将A压入栈中.
4. singleInstance: 具有此模式的Actiity只能单独位于一个任务栈中. 当A启动后, 系统会创建一个新的任务栈, 然后A独自在这个新的任务栈, 由于栈内复用,, 后续的请求均不会创建新的Activity. ![image](https://raw.githubusercontent.com/weikano/NoteResources/master/12.png)
![image](https://raw.githubusercontent.com/weikano/NoteResources/master/13.png)
  
  
**什么是Activity所需要的任务栈呢?**
> TaskAffinity, 可以翻译为任务相关性. 这个参数标识了一个Activity所需要的任务栈的名字, **默认情况下为应用的包名**.
> 1. 当TaskAffinity和singleTask配合使用时, 它是具有该模式的Activity的目前任务栈的名字, 待启动的Actiity会运行在名字和TaskAffinity相同的任务栈中.
> 2. 当TaskAffinity和allowTaskReparenting结合的时候, 会产生特殊的效果. 当一个应用A启动应用B的某个Activity, 比如a时, 如果a的allowTaskReparenting为true, 那么当应用B被启动后, a会直接从A的任务栈直接转移到B的任务栈. **具体点说, 比如现在有两个应用A和B, A启动了B的一个Activity C, 然后按Home键回到桌面, 再单击B的桌面图标, 并没有启动B的主Activity, 而是重新显示了被应用A启动的C, 或者说C从A的任务栈转移到了B的任务栈. 可以这么理解, 由于A启动了C, 这个时候C只能运行在A的任务栈中, 但是C属于B, 正常情况下它的TaskAffinity值不可能和A的相同. 所以当B被启动后, B会创建自己的任务栈, 这个时候系统发现C原本想要的任务栈已经被创建了, 所以就把C从A的任务栈中转移过来**.

## ~~两个关于任务栈的属性(==有问题==)~~
1. **clearTaskOnLaunch**
> 需要跟singleTask搭配. 比如有A,B,C,D四个Activity, B为singleTask且clearTaskOnLaunch设置为true, 那么A启动B, B启动C, C启动D, 然后按HOME键回桌面, 点击图标进入后, BCD都出栈, 只剩A.
2. **finishOnTaskLaunch**
> 

# IntentFilter的匹配规则

**以下intent-filter指的是AndroidManifest.xml中的注册信息**

1. action匹配规则: 只要intent中的action跟intent-filter中的任意一个action相同即可匹配.
2. categorypipe规则: intent中所有的category, 必须都在intent-filter中. 如果intent中没有categor, 默认加上default.
3. data匹配规则: 跟action类似, intent中必须含有data数据, 并且data能够完全匹配通过intent-filter中的任一个data.
> 首先了解一下data的结构. data由两部分组成: mimeType和URI. URI的结构如下  
**<scheme>://<host>:<port>/[<path>|<pathPrefix|<patPattern>]**.  
**Scheme**: URI的模式, 比如http, file, content等. 如果没有scheme, 那么URI无效. intent-filter中若未指定scheme, 默认为content或file.  
**Host**: URI的主机名, 比如www.baidu.com, 如果未指定, 那么URI无效.  
**Port**: URI中的端口号, 仅当URI中制定了scheme和host时才有意义.  
**Path, pathPattern和pathPrefix**: Path表示完整的路径信息; pathPattern也是完整的路径信息, 但是可以包含通配符, 使用了正则表达式规则;pathPrefix表示路径的前缀信息.