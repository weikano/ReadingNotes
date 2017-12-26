# 初识ViewRoot和DecorView
ViewRoot对应于ViewRootImpl类, 是连接WindowManager和DecorView的纽带, View的三大流程都是通过ViewRoot来完成的. 在ActivityThread中, 当activity对象被创建时, 会将DecorView添加到Window中, 同时创建ViewRootImpl对象, 并将ViewRootImpl和DecorView建立关联.  
  
View的绘制流程从ViewRoot的performTraversals方法开始, 它经过measure, layout和draw三个过程.
![image](https://raw.githubusercontent.com/weikano/NoteResources/master/58999540ab64413b8000405e.png)

# 理解MeasureSepc
## MeasureSpec
 - UNSPECIFIED 父容器不对View有任何限制, 要多大给多大. **一般用于系统内部, 标识一种测量状态.**
 - EXACTLY 父容器已经检测出View说需要的精确大小, 这个时候View的最终大小就是SpecSize的值. **它对应于LayoutParams中的match_parent和具体的数值.**
 - AT_MOST 父容器指定一个可用大小即SpecSize, View的大小不能大于这个值, 具体是多少要看不同View的具体实现. **对应于LayoutParams中的wrap_content.**
 
## MeasureSpec和LayoutParams的对应关系
- 对于DecorView, 其MeasureSpec由窗口尺寸和其自身的LayoutParams来确定.

```
private static int getRootMeasureSpec(int windowSize, int rootDimension) {
    int measureSpec;
    switch (rootDimension) {

    case ViewGroup.LayoutParams.MATCH_PARENT:
        // Window can't resize. Force root view to be windowSize.
        measureSpec = MeasureSpec.makeMeasureSpec(windowSize, MeasureSpec.EXACTLY);
        break;
    case ViewGroup.LayoutParams.WRAP_CONTENT:
        // Window can resize. Set max size for root view.
        measureSpec = MeasureSpec.makeMeasureSpec(windowSize, MeasureSpec.AT_MOST);
        break;
    default:
        // Window wants to be an exact size. Force root view to be that size.
        measureSpec = MeasureSpec.makeMeasureSpec(rootDimension, MeasureSpec.EXACTLY);
        break;
    }
    return measureSpec;
}
```


- 对于普通的View, 其MeasureSpec由父容器的MeasureSpec和自身的LayoutParams来确定. 
> ViewGroup.measureChildWithMargins -> View.getChildMeasureSpec .

```
protected void measureChildWithMargins(View child,
            int parentWidthMeasureSpec, int widthUsed,
            int parentHeightMeasureSpec, int heightUsed) {
    final MarginLayoutParams lp = (MarginLayoutParams) child.getLayoutParams();

    final int childWidthMeasureSpec = getChildMeasureSpec(parentWidthMeasureSpec,
            mPaddingLeft + mPaddingRight + lp.leftMargin + lp.rightMargin
                    + widthUsed, lp.width);
    final int childHeightMeasureSpec = getChildMeasureSpec(parentHeightMeasureSpec,
            mPaddingTop + mPaddingBottom + lp.topMargin + lp.bottomMargin
                    + heightUsed, lp.height);

    child.measure(childWidthMeasureSpec, childHeightMeasureSpec);
}

public static int getChildMeasureSpec(int spec, int padding, int childDimension) {
    int specMode = MeasureSpec.getMode(spec);
    int specSize = MeasureSpec.getSize(spec);
    //子元素大小为父元素大小减去padding
    int size = Math.max(0, specSize - padding);

    int resultSize = 0;
    int resultMode = 0;

    switch (specMode) {
    // Parent has imposed an exact size on us
    case MeasureSpec.EXACTLY:
        if (childDimension >= 0) {
            resultSize = childDimension;
            resultMode = MeasureSpec.EXACTLY;
        } else if (childDimension == LayoutParams.MATCH_PARENT) {
            // Child wants to be our size. So be it.
            resultSize = size;
            resultMode = MeasureSpec.EXACTLY;
        } else if (childDimension == LayoutParams.WRAP_CONTENT) {
            // Child wants to determine its own size. It can't be
            // bigger than us.
            resultSize = size;
            resultMode = MeasureSpec.AT_MOST;
        }
        break;

    // Parent has imposed a maximum size on us
    case MeasureSpec.AT_MOST:
        if (childDimension >= 0) {
            // Child wants a specific size... so be it
            resultSize = childDimension;
            resultMode = MeasureSpec.EXACTLY;
        } else if (childDimension == LayoutParams.MATCH_PARENT) {
            // Child wants to be our size, but our size is not fixed.
            // Constrain child to not be bigger than us.
            resultSize = size;
            resultMode = MeasureSpec.AT_MOST;
        } else if (childDimension == LayoutParams.WRAP_CONTENT) {
            // Child wants to determine its own size. It can't be
            // bigger than us.
            resultSize = size;
            resultMode = MeasureSpec.AT_MOST;
        }
        break;

    // Parent asked to see how big we want to be
    case MeasureSpec.UNSPECIFIED:
        if (childDimension >= 0) {
            // Child wants a specific size... let him have it
            resultSize = childDimension;
            resultMode = MeasureSpec.EXACTLY;
        } else if (childDimension == LayoutParams.MATCH_PARENT) {
            // Child wants to be our size... find out how big it should
            // be
            resultSize = View.sUseZeroUnspecifiedMeasureSpec ? 0 : size;
            resultMode = MeasureSpec.UNSPECIFIED;
        } else if (childDimension == LayoutParams.WRAP_CONTENT) {
            // Child wants to determine its own size.... find out how
            // big it should be
            resultSize = View.sUseZeroUnspecifiedMeasureSpec ? 0 : size;
            resultMode = MeasureSpec.UNSPECIFIED;
        }
        break;
    }
    //noinspection ResourceType
    return MeasureSpec.makeMeasureSpec(resultSize, resultMode);
}
```
![image](https://raw.githubusercontent.com/weikano/NoteResources/master/1.png)

- 当View使用固定大小时, 不管父容器的MeasureSpec是什么, View的MeasureSpecMode都是EXACTLY并且size为LayoutParams中设置值的大小.
- 当View设置为match_parent时, 如果父容器为EXACTLY, 那么View的SpecMode为EXACTLY, size为父容器的剩余空间;如果父容器为AT_MOST, 那么View的pecMode也为AT_MOST, 且size不能超过父容器的剩余空间.
- 当View设置为wrap_content时, 不管父容器的SpecMode是EXACTLY还是AT_MOST, View的SpecMode都为AT_MOST, 且size不超过父容器的剩余空间.
- UNSPECIFIED模式主要用于系统能够内部多次Measure的情形, 一般不需要关注此模式. 

# View的工作流程
## measure过程
### View的measure过程
> View.measure->View.onMeasure->setMeasuredDimension.

```
protected void onMeasure(int widthMeasureSpec, int heightMeasureSpec) {
    setMeasuredDimension(getDefaultSize(getSuggestedMinimumWidth(), widthMeasureSpec),
            getDefaultSize(getSuggestedMinimumHeight(), heightMeasureSpec));
}
```
setMeasuredDimension会设置View的宽高测量值.所以我们只需看getDefaultSize这个方法即可.

```
public static int getDefaultSize(int size, int measureSpec) {
    int result = size;
    int specMode = MeasureSpec.getMode(measureSpec);
    int specSize = MeasureSpec.getSize(measureSpec);

    switch (specMode) {
    case MeasureSpec.UNSPECIFIED:
        result = size;
        break;
    case MeasureSpec.AT_MOST:
    case MeasureSpec.EXACTLY:
        result = specSize;
        break;
    }
    return result;
}
```
简单的理解, 其实getDefaultSize返回的就是measureSpec中的specSize, 而这个specSize就是View测量后的大小. 之所以说是测量大小, 因为View最终的大小是在layout阶段确定的.  
至于UNSPECIFIED这种情况, 返回的是getSuggestedMinimumWidth/Height();

```
// 返回背景的最小宽度和android:minWidth二者中较大的值, getSuggestedMinimumHeight相同
protected int getSuggestedMinimumWidth() {
    return (mBackground == null) ? mMinWidth : max(mMinWidth, mBackground.getMinimumWidth());
}
```
直接继承View的自定义控件需要重写onMeasure方法, 并设置wrap_content时的自身大小, 否则在布局中使用wrap_content就相当于使用match_parent, 因为它的specMode为AT_MOST, 并且specSize为父容器的剩余空间的大小.解决方法如下:

```
private int minWidth, minHeight; //View的最小宽高.可以参考TextView和ImageView的源码
@Override
protected void onMeasure(int widthMeasureSpec, int heightMeasureSpec) {
    super.onMeasure(widthMeasureSpec, heightMeasureSpec);
    int widthSpecMode = MeasureSpec.getMode(widthMeasureSpec);
    int heightSpecMode = MeasureSpec.getMode(heightMeasureSpec);
    int widthSpecSize = MeasureSpec.getSize(widthMeasureSpec);
    int heightSpecSize = MeasureSpec.getSize(heightMeasureSpec);

    int desiredWidth = widthSpecMode == MeasureSpec.AT_MOST ? minWidth : widthSpecSize;
    int desizedHeight = heightSpecMode == MeasureSpec.AT_MOST ? minHeight : heightSpecSize;
    setMeasuredDimension(desiredWidth, desizedHeight);

}
```
### ViewGroup的measure过程
ViewGroup是一个抽象类, 并没有重写View的onMeasure方法, 而是交给实现类来重写, 但是它提供了一个叫做measureChildren的方法.

## layout过程
layout方法的大致流程如下: 首先会通过setFrame方法来设定View的四个顶点位置;接着调用onLayout方法, 这个方法用于父容器确定子元素的位置, 和onMeasure方法类似, onLayout的具体实现同样和具体的布局有关, 所以Viewh哦ViewGroup都没有真正实现onLayout方法.

## draw过程
1. 绘制背景(background.draw(canvas)). 
2. 绘制自己(onDraw).
3. 绘制children(dispatchDraw).
4. 绘制装饰(onDrawScrollBars). 

# 自定义View
## 自定义View的分类
1. **继承View重写onDraw方法**
> 这种方法主要用于实现一些不规则的效果, 即这种效果不方便通过布局达到, 往往需要静态或者动态的显示一些不规则的图形. **采用这种方法需要自己支持wrap_content并且padding也需要自己处理.**

2. **继承ViewGroup派生特殊的Layout**
> 主要用于实现自定义布局. **需要合适的处理ViewGroup的测量和布局两个过程, 并同时处理子元素的测量和布局**.

3. **继承特定的View (比如TextView)**
> 用于扩展已有的View的功能. **不需要自己支持wrap_content和padding**.

4. **继承特定的ViewGroup (比如LinearLayout)**
> 一般来说2能实现的这种方式也能实现, 只是2更接近View的底层.

## 自定义View的需知
1. 让View支持wrap_content. 
> 如果不在onMeasure中对wrap_content进行特殊处理, 那么就无法达到预期的效果.

2. 如果有必要, 让View支持padding.
> 直接继承View的空间, 如果不在draw方法中处理padding, 那么padding是无法其作用的; 继承自ViewGroup的空间需要在onMeasure和onLayout中考虑padding和子元素margin对其造成的影响.

3. 尽量不要在View中使用Handler, 因为View本身提供了post方法
4. View中如果有线程或动画, 要及时停止, 参考onDetachedFromWindow.
5. View带有滑动嵌套情形时, 需要处理好滑动冲突.

## 自定义View示例
> 参考chapter4代码. 

## 自定义View的思想
> 掌握View的弹性滑动, 滑动冲突解决, 绘制原理等.