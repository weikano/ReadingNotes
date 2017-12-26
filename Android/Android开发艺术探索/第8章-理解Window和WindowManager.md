> Window表示一个窗口的概念.  
> Window是一个抽象类, 具体实现是PhoneWindow. 创建一个window需要通过WindowManager完成, WindowManagerService为具体实现. Android中所有视图都是通过Window来呈现的, 不管Activity, Dialog还是Toast. 

# Window和WindowManager
- **FLAG_NOT_FOCUSABLE**
> 表示Window不需要获取焦点, 与不需要接收各种输入事件, 此标记会同时启用FLAG_NOT_TOUCH_MODAL, 最终事件会直接传递给下层的具有焦点的Window.   
- **FLAG_NOT_TOUCH_MODAL**
> 此模式下, 系统会将当前Window区域以外的单击事件传递给底层的Window, 当前Window区域以内的单击事件则自己处理. 一般来说都需要开启此标记, 否则其它Window将无法收到单击事件.
- **FLAG_SHOW_WHEN_LOCKED**
> 开启此模式可以让Window显示在锁屏界面上.
  
Type参数表示Window的类型, Window有三种类型, 分别是应用Window, 子Window和系统Window. 应用类Window对应着一个Activity. 子Window不能单独存在, 它需要附属在特定的父Window之中, 比如常见的Dialog就是一个子Window. 系统Window需要声明权限才能创建Window, 比如Toast和系统状态栏都是这些系统的Window.  
Window是分层的, 每个Window都有对应的z-order, 层级大的会覆盖在层级下的上面. 在三类Window中, 应用Window的层级范围是1~99, 子Window范围是1000~1999, 系统Window是2000~2999, 这些层级范围对应着WindowManager.LayoutParams的type. 如果Window想要位于最顶层, 那么采用较大的层级即可. 很显然系统的Window层级最大. 一般我们可以选用TYPE_SYSTEM_OVERLAY或者TYPE_SYSTEM_ERROR. 如果采用TYPE_SYSTEM_ERROR, 只需type参数指定这个层级, 同时声明权限.

# Window的内部机制
每一个Window都对应着一个View和一个ViewRootImpl, Window和View通过ViewRootImpl建立联系, 因此Window并不是实际存在的, 它是以View的形式存在.

## Window的添加过程
WindowManagerImpl为WindowManager的实现类, 委托WindowManagerGlobal来实现addView. WindowManagerGlobal的addView方法主要分为以下几步:
1. 检查参数是否合法, 如果是子Window那么需要调整一些布局参数.
2. 创建ViewRootImpl并将View添加到列表中.
3. 通过ViewRootImpl更新界面并完成Window的添加过程.
> 这个步骤通过ViewRootImpl的setView方法来完成. setView内部会通过requestLayout来完成异步刷新请求. schedualTraversals实际是View绘制的入口. 接着会通过WindowSession最终来完成Window的添加过程, 真正的实现类是Session, 而Sessionn内部会通过WindowManagerService去处理.

## Window的删除过程
类似添加过程, WindowManagerImpl -> WindowManagerGlobal -> ViewRootImpl. die分为异步和同步删除, 异步使用Handler发送MSG_DIE, 调用doDie; 同步直接doDie. 真正删除View的逻辑在dispatchDetachedFromWindow, 主要做了四件事.
1. 垃圾回收, 比如清除数据,移除回调.
2. 通过session的remove方法删除Window, 同样是IPC方法.
3. 调用View的dispatchDetachedFromWindow方法. onDetachedFromWindow 可以做一些资源回收, 比如动画停止和线程停止.
4. 调用WindowManagerGlobal的doRemoveView方法刷新数据.

## Window的更新过程
WindowManagerGlobal首先更新LayoutParams, 然后通过setLayoutParams来调用requestLayout或scheduleTraversals.

# Window的创建过程
## Activity的Window创建过程
在Activity.attach方法里, 会创建PhoneWindow. 然后通过Window.setContentView添加界面. PhoneWindow的setContentView大致遵循以下几个步骤:
1. 没有DecorView就创建.
2. 将View添加到DecorView的mContentPanel中.
3. 回调Activity的onContentChanged方法通知Activity视图发生改变.

经过以上步骤, DecorView已经创建并初始化完毕, 布局文件也已经添加好, 但是DecorView还没有被WindowManager添加到Window中. 在ActivityThread.handleResumeActivity中, 首先会调用Activity的onResume, 接着WindowManager会将DecorView添加进去并设置VISIBLE. 

## Dialog的Window创建过程
1. 创建Window.
> 构造函数中.
2. 初始化DecorView并将Dialog的视图添加到DecorView中.
> setContentView.
3. 将DecorView添加到Window中并显示.
> show中调用.

普通的Dialog必须使用Activity的context. 如果不想用Activity的context, 必须将Dialog的type指定为系统类型还得加上权限.

## Toast的创建过程
Toast内部有两类IPC过程, 第一类是Toast访问INotificationManager, 第二类是INotificationManager回调Toast里的TN接口.  
Toast属于系统Window, 它内部的视图有两种. 一种为默认, 另一种是通过setView来指定. 对应Toast的mNextView.  
从代码得知, 显示和隐藏Toast都通过NotificationManagerService来实现.  
从NotificationManagerService中的mService可以得知, Toast的显示和隐藏过程是通过Toast中的TN来实现的. 由于TN是通过IPC调用的, 所以在Binder线程中, 必须通过Handler切换到Toast请求所在的线程, 所以内部使用了Handler.