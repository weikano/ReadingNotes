# View动画
## View动画的种类
![image](https://raw.githubusercontent.com/weikano/NoteResources/master/3.png)

## 自定义View动画
继承Animation, 重写initialize和applyTransformation方法. 很多时候需要采用Camera来简化矩阵变换的过程. 具体例子参考ApiDemos中的Rotate3dAnimation. 

## 帧动画
不同于View动画, 系统提供了另外一个类AnimationDrawable来使用帧动画. 对应的标签为<animation-list>

# View动画的特殊使用场景
## LayoutAnimation
1. 定义LayoutAnimation.

```
<layoutAnimation xmlns:android="http://schemas.android.com/apk/res/android"
    android:delay="0.5"
    android:animation="@anim/anim_item"
    android:animationOrder="normal"/>
```

2. 为子元素指定具体的入场动画.

```
<set xmlns:android="http://schemas.android.com/apk/res/android"
    android:duration="300"
    android:interpolator="@android:anim/accelerate_interpolator"
    android:shareInterpolator="true">
    <alpha android:fromAlpha="0.0"
        android:toAlpha="1.0" />
    <translate android:fromXDelta="500"
        android:toXDelta="0" />
</set>
```

3. 为ViewGroup指定android:layoutAnimation属性.

```
<ListView xmlns:android="http://schemas.android.com/apk/res/android"
    android:id="@+id/content"
    android:layout_width="match_parent"
    android:layout_height="match_parent"
    android:layoutAnimation="@anim/anim_layout"/>
```
除了在xml中设置layoutAnimation外, 可以通过LayoutAnimationController来代码设置.

## Activity的切换效果
overridePendingTransition(int enterAnim, int exitAnim)
- enterAnim Activity被打开时所需的动画资源id.
- exitAnim Activity被暂停时所需的动画资源id.
当启动一个Activity时, 可以通过如下方式为其添加自定义的切换效果.

```
Intent intent = new Intent(xxx);
startActivity(intent);
overridePendingTransition(enterAnim, exitAnim);
```
当Activity退出时, 也可以指定自己的切换效果.

```
@Override
public void finish() {
    super.finish();
    overridePendingTransition(enterAnim, exitAnim);
}
```
**overridePendingTransition必须位于start和finish方法后面, 否则无效.**  


Fragment也可以通过setCustomAnimations方法添加动画.  

# 属性动画
在实际开发中建议使用代码来实现属性动画, 这是因为很多时候一个属性的起始值是无法提前确定的.

## 理解插值器和估值器
TimeInterpolator为时间估值器, TypeEvaluator为类型估值器.

## 属性动画的监听器
AnimatorListener和AnimatorListenerAdapter

## 对任意属性做动画
我们对object的属性abc做动画, 需要满足以下条件:
1. object必须提供abc的set方法setAbc, 如果动画没有传递初始值, 那么还要提供get方法getAbc. **不满足这条就会crash.**
2. object的setAbc对属性abc所做的改变必须能通过某种方式反映出来, 比如会带来UI变化之类的. 

对于第2条, 有3种解决方案:
- 给你的对象添加get和set方法.
- 用一个类来包装原始对象, 间接提供set/get方法.
- 使用ValueAnimator来监听变化, 自己实现属性的改变.

以改变Button的width为例, 其不满足第2种条件.  
第1种方案无法实现.  
第2种方案代码如下.

```
button.setOnClickListener(new View.OnClickListener() {
    @Override
    public void onClick(View v) {
        ButtonWidthWrapper wrapper = new ButtonWidthWrapper(button);
        ObjectAnimator.ofInt(wrapper, "width", 600).setDuration(500).start();
    }
});

private class ButtonWidthWrapper {
    private Button button;
    public ButtonWidthWrapper(Button button){
        this.button = button;
    }
    public void setWidth(int width){
        button.getLayoutParams().width = width;
        button.requestLayout();
    }

    public int getWidth(){
        return button.getLayoutParams().width;
    }
}
```
第3种方案代码如下:

```
ValueAnimator animator = ValueAnimator.ofInt(600);
animator.setInterpolator(new LinearInterpolator());
animator.addUpdateListener(new ValueAnimator.AnimatorUpdateListener() {
    final IntEvaluator evaluator = new IntEvaluator();
    @Override
    public void onAnimationUpdate(ValueAnimator animation) {
        int current = (int) animation.getAnimatedValue();
        float fraction = animation.getAnimatedFraction();
        animateValueAnimator.getLayoutParams().width = evaluator.evaluate(fraction,current, 600);
        animateValueAnimator.requestLayout();
    }
});
animator.setDuration(500);
animator.start();
```

## 属性动画的工作原理
ValueAnimator.start() -> AnimationHandler.start() -> ValueAnimator.doAnimationFrame() -> ValueAnimator.animationFrame() -> ValueAnimatior.animateValue() -> PropertyValuesHolder.calculateValue(). 

# 使用动画的注意事项
1. OOM问题
> 主要出现在帧动画. 图片数量较多且图片较大时会出现. 因尽量避免使用帧动画. 
2. 内存泄漏
> 在属性动画中有一类无线循环的动画, 这类动画要在Activity退出时及时停止. View动画不存在此问题.
3. 兼容性问题
> 3.0之下属性动画.
4. View动画的问题
> 有时候会出现动画完成后View无法隐藏, 此时只需要调用clearAnimation()即可解决. 
5. 不用使用px
> px会导致不同设备效果不一致.
6. 动画元素的交互
> View动画的点击位置.
7. 硬件加速
> 硬件加速会提高动画的流畅性.
8. 


