# View基础知识
## 什么是View
> View是界面层控件的一种抽象.

## View的位置参数
> 1. left, top, right, bottom都是相对于View的父容器来说的, 是一种相对坐标.
> 2. x, y是View左上角的坐标; translationX和translationY是View左上角相对父容器的偏移量, 默认为0. 四者都是相对坐标. 
> 3. x=left + translationX, y=top+translationY.
> 4. 需要注意的是, View在平移过程中, top和left表示的是原始左上角的位置信息, 不会改变, 此时改变的是x,y,translationX,translationY.

## MotionEvent和TouchSlop
 1. MotionEvent
 > getX和getY返回的是相对于当前View左上角的x和y坐标, getRawX和getRawY返回的是相对于手机屏幕左上角的x和y坐标.
 2. TouchSlop
 > 系统能识别出的被认为是滑动的最小距离ViewConfiguration.get(context).getScaledTouchSlop.

## VelocityTracker, GestureDetector和Scroller
 1. VelocityTracker: 手指逆着坐标系滑动,这速度为负值.

    VelocityTracker tracker = VelocityTracker.obtainer();
    tracker.addMovement(event);
    tracker.computeCurrentVelocity(1000);
    int xV = (int) tracker.getXVelocity();
    int yV = (int) tracker.getYVelocity();
    tracker.clear();
    tracker.recycle();

 2. GestureDetector
 3. Scroller
 > 搭配View.computeScroll使用, 参考ScrollView.

 ![image](https://raw.githubusercontent.com/weikano/NoteResources/master/8.png)

# View的滑动
> 滑动实现方式有3种: 
> 1. 通过View本身提供的scrollTo/scrollBy方法; 
> 2. 第二种是通过动画给View施加平移效果; 
> 3. 第三种是通过改变View的LayoutParams使得View重新布局.

## 使用scrollTo/scrollBy
> 如果从左向右滑动, 那么mScrollX为负值; 如果从上往下滑动, 那么mScrollY为负值.
> **使用scrollTo/scrollBy来实现View的滑动, 只能将View的内容进行移动, 并不能将View本身进行移动. 也就是说, 不管怎么滑动, 也不可能将当前View滑动到附近View所在的区域**.

## 使用动画
 > View动画是对View的影像进行操作,并不能真正改变View的位置参数, 包括宽高.
 
## 改变布局参数
 > 通过修改LayoutParams
 
## 各种滑动方式对比
 - scrollTo/scrollBy: 操作简单, 适合对View内容的滑动.
 - 动画: 操作简单, 适用于没有交互的View和实现复杂的动画效果. 
 - 改变布局参数: 操作稍微复杂, 适用于有交互的View. 


# 弹性滑动

## 使用Scroller
 Scroller本身并不能实现View滑动, 它需要配合View的computeScroll方法才能完成弹性滑动的效果, 它不断让View重绘, 而每一次重绘距滑动起始时间会有一个时间间隔, 通过这个时间间隔, Scrollerk可以得出View当前的滑动位置, 从而调用scrollTo方法来完成View的滑动.
 
## 通过动画Animator

## 使用延时策略
通过发送一系列延时消息从而达到一种渐进式的效果, 具体可以用Handler或View的postDelayed方法.

# View的事件分发机制

## 点击事件的传递规则
 
```
//伪代码
    public boolean dispatchTouchEvent(MotionEvent ev){
        boolean consume = false;
        if (onInterceptTouchEvent(ev)){
            consume = onTouchEvent(ev);
        }else{
            consume = child.dispatchTouchEvent(ev);
        }
        return consume
    }
```

 1. 对于一个根ViewGroup来说,点击事件产生后,首先传递给它,然后它的dispatchTouchEvent会被调用, 如果这个ViewGroup的onInterceptTouchEvent方法返回true, 就表示它要拦截当前事件, 接着事件就交给ViewGroup的onTouchEvent处理; 如果ViewGroup的onInterceptTouchEvent返回false, 那么当前事件就继续传递给它的子View, 接着调用子View的dispatchTouchEvent事件, 如此反复直到事件被最终处理.
 2. 当一个View设置了OnTouchListener, 那么Listener中的onTouch会被回调, 如果onTouch返回false, 那么onTouchEvent会被调用.
 3. 顺序: OnTouchListener.onTouch -> View.onTouchEvent -> OnClickListener.onClick. 
 4. 当一个点击事件产生后, 传递顺序为Activity -> Window (PhoneWindow) -> 顶层View .
 5. 如果View的onTouchEvent返回false, 那么父容器的onTouchEvent将会被调用, 依次如果所有View都不处理这个事件, 那么Activity的onTouchEvent会被调用. 

**一些结论**
 1. 同一个事件序列是指从手指接触屏幕到手指离开屏幕的期间所产生的一系列事件, 这个序列以down开始, 最终以up结束.
 2. 正常情况下, 一个事件序列只能被一个View拦截和消耗.
 3. 某个View一旦决定拦截, 那么这一个事件序列都只能由它来处理, 并且它的onInterceptTouchEvent不会被调用. 
 4. 某个View一旦开始处理事件, 如果它不消耗down事件, 那么同一个事件序列中的其它event都不会再交给它来处理, 并且事件将重新交给它的父容器处理, 即父容器的onTouchEvent会被调用.
 5. 如果View不消耗除down事件以外的其它事件, 那么这个点击事件会消失, 此时父容器的onTouchEvent并不会被调用, 而且当前View可以持续收到后续的事件, 最终这些消失的点击事件会传递给Activity处理.
 6. ViewGroup默认不拦截任何事件.
 7. View没有onInterceptTouchEvent方法, 一旦事件传递给它, onTouchEvent就会调用. 
 8. View的onTouchEvent默认都会消耗事件(返回true), 除非它是不可点击的(clickable和longClickable同时为false). 
 9. View的enable不影响onTouchEvent的返回值. 
 10. onClick发生的前提是View是可点击的, 并且它会收到down和up事件. 
 11. 事件传递过程是由外向内, 即总是先传递给父容器, 然后父容器分发给子View, 通过requestDisallowInterceptTouchEvent方法可以在子View中干预父容器事件分发过程, 但是down事件除外. 
 

## 事件分发的源码解析
 - ViewGroup的dispatchTouchEvent会根据是否intercept, 通过dispatchTransformedTouchEvent将事件传递给child的dispatchTouchEvent或者View.dispatchTouchEvent(即由自己的onTouchEvent或者TouchListener来处理).
 - View的dispatchTouchEvent事件简单, 参照源码即可.

# View的滑动冲突
## 常见的滑动冲突场景
 - 外部滑动方向和内部不一致.
 - 外部滑动方向和内部一致.
 - 上面两种嵌套.
![image](https://raw.githubusercontent.com/weikano/NoteResources/master/58997909ab64413f92003da7.png)      

## 滑动冲突的处理规则
 - 对于场景一: 当用户左右滑动时, 我们需要让外部View拦截点击事件; 当用户上下滑动时, 我们需要让内部View拦截事件.
 - 对于场景二: 一般根据业务需求来确定哪个View在什么情况下拦截事件.
 - 类似场景二.

## 滑动冲突的解决方法
 1. 外部拦截法
 > 点击事件都先经过父容器的拦截处理, 如果父容器需要此事件就拦截, 如果不需要就不拦截. 重点在于重写父容器的onInterceptTouchEvent, 在内部进行相应的拦截即可. **注意在处理down事件时, 必须返回false, 否则后续的时间都会直接交由父容器处理, 这时事件没法再传递给子View了.**

 2. 内部拦截法
 > 父容器不拦截任何事件, 所有事件都传递给子View. 如果子View需要此事件就直接消耗掉, 否则交由父容器处理. **需要配合requestDisallowInterceptTouchEvent方法才能工作. 除了子View需要做处理之外, 父容器也要默认拦截除了down以外的其他事件, 这样当子View调用parent.requestDisallowInterceptTouchEvent(false)时, 父容器才能继续拦截所需事件. **
 