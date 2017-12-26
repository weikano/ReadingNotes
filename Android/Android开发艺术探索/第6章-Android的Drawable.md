# Drawable简介
Drawable的内部宽高这个参数比较重要, 通过getIntrinsicWidth/Height来获取. 并不是所有的Drawable都有内部宽高, 比如一张图片形成的Drawable, 内部宽高是图片的宽高; 但是一个颜色形成的Drawable是没有的. **Drawable的内部宽高并不等同于它的大小, 用作View背景时, Drawable会被拉伸.**

# Drawable的分类
## BitmapDrawable
- android:src 
> 图片的资源id. 
- android:antialias
> 抗锯齿, 应该开启. 
- 是否开启抖动效果.可以让高质量的图片在低质量的屏幕上保持较好的显示效果, 应该开启.
- android:filter
> 当图片被拉伸或压缩时, 开启过滤性效果可以保持较好的显示效果. 应该开启. 
- android:gravity
> 图片小于容器尺寸时, 图片的定位.
- android:mipMap
> 不常用, 默认为false.
- android:tileMode
> 平铺模式. 
> 1. repeat 水平和竖直方向上的平铺.
> 2. mirror 水平和竖直方向上的镜像投影. 
> 3. clamp 图片四周的像素会扩展至周围区域. 

## ShapeDrawable
用<shape>创建的Drawable, 其实体类是GradientDrawable.
- android:shape
> 表示图形的形状
- <corners>
> 表示shape的四个角的弧度, 只适用于shape. 
- <gradient>
> 与<solid>标签互斥
- <solid>
> 纯色填充, 通过android:color设置填充的颜色.
- <stroke>
> shape的描边. 如果android:dashGap和android:dashWidth任意一个为0, 虚线效果失效.
- <padding> 
> 表示包含它的View的空白.
- <size>
> shape的大小. 并不是最终显示的大小.

## LayerDrawable
对应标签为<layer-list>, 表示一种层次化的Drawable集合, 通过将不同的Drawable放置在不同的层上面达到一种叠加后的效果.

## StateListDrawable
对应标签为<selector>, 每个Drawable对应着View的一种状态.

## LevelListDrawable
对应标签为<level-list>, 同样表示一个Drawable集合, 集合中的每个Drawable都有一个level概念. 根据不同的level, LevelListDrawable会切换不同的Drawable. 

## TransitionDrawable
对应标签为<transition>, 它用于实现两个Drawable之间的淡入淡出效果.

## InsetDrawable
对应标签为<inset>, 它可以将其它Drawable内嵌到自己当中, 并可以在四周流出一定的间距. 当一个View希望自己的背景比自己的实际区域小时, 可以采用InsetDrawable. 

## ScaleDrawable
对应标签为<scale>, 根据自己的level将指定的Drawable缩放到一定的比例. level为0时不显示.

## ClipDrawable
对应标签为<clip>, 根据level来裁剪一个Drawable. 裁剪方式通过android:clipOrientation和android:gravity来共同控制. level 0表示完全裁剪, 10000表示不裁剪. 

# 自定义Drawable
- 当自定义的Drawable有默认大小时, 最好重写getIntrinsicWidth/Height两个方法, 它会影响到View的wrap_content布局.
- Drawable的实际大小可以通过getBounds方法来得到, 一般来说它和View的尺寸相同. 