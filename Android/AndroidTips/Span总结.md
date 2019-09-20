## Span总结

> Span大体分为两种，字符级和段落级。字符级针对字符设置，段落级影响段落层次
>
> - 如果一个Span影响字符级的文本格式，继承CharacterStyle
> - 如果一个Span影响段落层级的文本格式，继承ParagraphSpan
> - 如果一个Span修改字符级的文本外观，实现UpdateAppearance
> - 如果一个Span修改字符级文本尺寸，实现UpdateLayout

## 0、FontMetrics简介

### 0.1 简介

`FontMetricsInt`用于描述字体，不同的字体SIZE会有不同的`FontMetricsInt`值

- baseline：基准点
- Ascent：baseline到字符最高处的距离
- Descent：baseline到字符最低处的距离
- leading：上一行文字的descent到下一行文字的ascent之间的距离（？）
- top：Ascent的最大值
- bottom：Descent的最大值

![](https://s1.51cto.com/attachment/201205/204735397.png)

![](https://s1.51cto.com/attachment/201205/204802837.png)

### 0.2 应用

- 字体的高度可通过`descent+Math.abs(ascent)`计算得到

- 字符串宽度可通过`paint.measureText("xxx")`获得

- 字符串的实际可视高度和宽度，可以通过`paint.getTextBounds`或者`textView.getLineBounds()`获得

- 行间距：通过`setLineSpacing或者lineSpacingExrta、lineSpacingMultiplier`定义；对于使用`paint`绘制的行间距调整，设置`Paint.FontMetrics.leading`属性
- 行高：`ascent+descent+leading`，即字符串高度加行间距，对于`TextView`可以直接通过`getLineHeight`获取
- 字符间距：`TextView/Paint.get/setLetterSpacing`

## 1、字符级Span解析-CharacterStyle

### 1.1 CharacterStyle

CharacterStyle是一个抽象类，其中有一个抽象方法`public abstract void updateDrawState(TextPaint tp)`

通过改变其中的TextPaint属性就可以得到不同的字符展示。

### 1.2 UpdateAppearance

如果一个Span修改字符级别的文本外观，则实现UpdateAppearance。比如：

`BackgroundColorSpan、ForegroundColorSpan、StrikethroughSpan、UnderlineSpan和MaskFilterSpan`。

都是通过实现`updateDrawState`方法，修改其中的TextPaint来实现。

### 1.3 UpdateLayout

如果一个Span修改字符级文本尺寸大小，则需要实现UpdateLayout。只有`MetricAffectingSpan`实现了该接口，其提供了`updateMeasureState(TextPaint)`方法来实现对字体大小的改变。参考`SubscriptSpan和SupercriptSpan`。

`MetricAffectingSpan`同样继承了`CharacterStyle`，从而可以实现改变字体外观。另外`updateMeasureState`方法控制字体大小的改变

#### 1.3.1 `SubscriptSpan和SuperscriptSpan`

用于显示上下标

```java
@Override 
public void updateDrawState(TextPaint tp) {
		tp.baselineShift += (int)(tp.ascent/2);//或者-=
}
public void updateMeasureState(TextPaint tp) {
    tp.baselineShift += (int)(tp.ascent/2);//或者-=
}
```

#### 1.3.2 `AbsoluteSizeSpan和RelativeSizeSpan`

用于改变字符的字体大小

```java
public void updateDrawState(TextPaint tp) {
   tp.setTextSize(xxx);
}

public void updateMeasureState(TextPaint tp) {
  tp.setTextSize(xxx);
}
```

#### 1.3.3 `ScaleXSpan`

在X轴上缩放字符

```java 
@Override
public void updateDrawState(TextPaint ds) {
  ds.setTextScaleX(ds.getTextScaleX() * mProportion);
}

@Override
public void updateMeasureState(TextPaint ds) {
  ds.setTextScaleX(ds.getTextScaleX() * mProportion);
}
```

#### 1.3.4 `StyleSpan、TypefaceSpan和TextAppearanceSpan`

都是对字体进行改变，也是修改`TextPaint`的对应属性`style、typeface`

### 1.4 `ReplacementSpan`

`ReplacementSpan`继承自`MetricAffectingSpan`，但是增加了两个抽象方法

```java
//返回所占的宽度。一般通过paint.measureText返回width
public abstract int getSize(Paint paint, CharSequence text,
                         int start, int end,
                         Paint.FontMetricsInt fm);
//start、end为Span的起始结束字符的index
//x为起始横坐标
//y为baseline对应的坐标
//top起始高度
//bottom结束高度
public abstract void draw(Canvas canvas, CharSequence text,
                          int start, int end, float x,
                          int top, int y, int bottom, Paint paint);
```

#### 1.4.1 `DynamicDrawableSpan`

抽象类继承自`ReplacementSpan`，可以使用`Drawable`代替相应的字符，代码实现如下

```java
@Override
public int getSize(Paint paint, CharSequence text,
                   int start, int end,
                   Paint.FontMetricsInt fm) {
  //getCachedDrawable为获取对应的Drawable对象
  Drawable d = getCachedDrawable();
  Rect rect = d.getBounds();
  if (fm != null) {
    fm.ascent = -rect.bottom; 
    fm.descent = 0; 

    fm.top = fm.ascent;
    fm.bottom = 0;
  }
  return rect.right;
}
```

具体实现类有`ImageSpan`

`DynamicDrawableSpan`在对齐方面只有`ALIGN_BOTTOM`和`ALIGN_BASELINE`，所以在使用时会出现`Span`没有居中显示的情形。这时候需要修改代码

```java
class CenteredImageSpan extends ImageSpan {
   @Override void draw() {
     drawable.draw(canvas);
   }
   @Override int getSize() {
     val rect = drawable.bounds
      var initialDescent = 0
        var extraSpace = 0
        if (rect.bottom - (it.descent - it.ascent) >= 0) {
          initialDescent = fm.descent
            extraSpace = rect.bottom - (fm.descent - fm.ascent)
        }
     fm.descent = extraSpace / 2 + initialDescent
       fm.bottom = fm.descent
       fm.ascent = -rect.bottom + fm.descent
       fm.top = fm.ascent
       return rect.right
   }
}
```

## 二、段落级Span解析-ParagraphSpan

`ParagraphSpan`是一个没有定义任何方法的接口，作用可能是用于标识这个Span为段落级别

- `LeadingMarginSpan`：用来处理段落中类似Word中符号的接口，比如`<ui>`和`<li>`等
- `AlignmentSpan`：段落对齐
- `LineBackgroundSpan`：处理一行背景
- `LineHeightSpan`：处理一行高度
- `TabStopSpan`：替换\\t为相应的空行

### 2.1 `LeadingMarginSpan`

```java
//返回段落的margin，first表示是否第一行
public int getLeadingMargin(boolean first);
//x 当前margin的位置
// dir margin的方向。负数则为text的右边，否则左边
// first 表示当前是否第一行
public void drawLeadingMargin(Canvas c, Paint p,
                                  int x, int dir,
                                  int top, int baseline, int bottom,
                                  CharSequence text, int start, int end,
                                  boolean first, Layout layout);
```

`LeadingMarginSpan2`继承自`LeadingMarginSpan`，还多出一个`public int getLeadingMarginLineCount`方法。`LeadingMarginSpan`只能控制第一行的margin，而`LeadingMarginSpan2`通过该方法指定了有多少行可以算作first。

#### 2.1.1 `BulletSpan`

```java
//无论是否第一行，都添加一个margin，所以整个段落都右移
public int getLeadingMargin(boolean first) {
    return 2 * BULLET_RADIUS + mGapWidth;
}
// 在第一行绘制一个bullet
public void drawLeadingMargin(Canvas c, Paint p, int x, int dir,
                              int top, int baseline, int bottom,
                              CharSequence text, int start, int end,
                              boolean first, Layout l) {
    if (((Spanned) text).getSpanStart(this) == start) {
        Paint.Style style = p.getStyle();
        int oldcolor = 0;

        if (mWantColor) {
            oldcolor = p.getColor();
            p.setColor(mColor);
        }

        p.setStyle(Paint.Style.FILL);

        if (c.isHardwareAccelerated()) {
            if (sBulletPath == null) {
                sBulletPath = new Path();
                // Bullet is slightly better to avoid aliasing artifacts on mdpi devices.
                sBulletPath.addCircle(0.0f, 0.0f, 1.2f * BULLET_RADIUS, Direction.CW);
            }

            c.save();
            c.translate(x + dir * BULLET_RADIUS, (top + bottom) / 2.0f);
            c.drawPath(sBulletPath, p);
            c.restore();
        } else {
            c.drawCircle(x + dir * BULLET_RADIUS, (top + bottom) / 2.0f, BULLET_RADIUS, p);
        }

        if (mWantColor) {
            p.setColor(oldcolor);
        }

        p.setStyle(style);
    }
}

```

#### 2.1.2 `QuoteSpan`

```java
public int getLeadingMargin(boolean first) {
    return STRIPE_WIDTH + GAP_WIDTH;
}
//整个段落画一条竖直的线
public void drawLeadingMargin(Canvas c, Paint p, int x, int dir,
                              int top, int baseline, int bottom,
                              CharSequence text, int start, int end,
                              boolean first, Layout layout) {
    Paint.Style style = p.getStyle();
    int color = p.getColor();

    p.setStyle(Paint.Style.FILL);
    p.setColor(mColor);

    c.drawRect(x, top, x + dir * STRIPE_WIDTH, bottom, p);

    p.setStyle(style);
    p.setColor(color);
}
```

#### 2.1.3 `TextRoundSpan`

继承自`LeadingMarginSpan2`，实现效果如下图

![](https://upload-images.jianshu.io/upload_images/924057-fdf5fff7b17a6258.png?imageMogr2/auto-orient/strip|imageView2/2/w/400/format/webp)

```Java

@Override
public int getLeadingMargin(boolean first) {
  if (first) {
    return margin;
  } else {
    return 0;
  }
}

@Override
public void drawLeadingMargin(Canvas c, Paint p, int x, int dir,
                              int top, int baseline, int bottom, CharSequence text,
                              int start, int end, boolean first, Layout layout) {}

//控制共有几行有margin
@Override
public int getLeadingMarginLineCount() {
  return lines;
}
```

### 2.3 `AlignmentSpan`

处理整个段落的对齐排列

```Java
//返回对弈方式
Layout.Alignment getAlignment();

//使用方法如下
private void rtl() {
    SpannableString string = new SpannableString("Text with opposite alignment");
    string.setSpan(new AlignmentSpan.Standard(Layout.Alignment.ALIGN_OPPOSITE), 0, string.length(), Spanned.SPAN_EXCLUSIVE_EXCLUSIVE);
}
```

### 2.4 `LineBackgroundSpan`

```Java
public void drawBackground(Canvas c, Paint p,
                               int left, int right,
                               int top, int baseline, int bottom,
                               CharSequence text, int start, int end,
                               int lnum);
```

### 2.5 `LineHeightSpan`

```java 
public void chooseHeight(CharSequence text, int start, int end,
            int spanstartv, int lineHeight,
            Paint.FontMetricsInt fm);
```

Android中提供了`DrawableMarginSpan`，其实现了`LineHeightSpan`，主要是用于在段落前面展示一个图片

![](https://upload-images.jianshu.io/upload_images/924057-6bef9fe09e4849f3.jpg?imageMogr2/auto-orient/strip|imageView2/2/w/500/format/webp)

```java
//v：当前行的起始垂直坐标
//descent为正数，baseline=0，Ascent为负数
//istartv为整个span的起始垂直坐标
//即减去整个TextView到这一行的高度
@Override
public void chooseHeight(@NonNull CharSequence text, int start, int end,
                         int istartv, int v,
                         @NonNull Paint.FontMetricsInt fm) {
  if (end == ((Spanned) text).getSpanEnd(this)) {
    int ht = mDrawable.getIntrinsicHeight();

    int need = ht - (v + fm.descent - fm.ascent - istartv);
    if (need > 0) {
      fm.descent += need;
    }

    need = ht - (v + fm.bottom - fm.top - istartv);
    if (need > 0) {
      fm.bottom += need;
    }
  }
  
  @Override
  public void drawLeadingMargin(@NonNull Canvas c, @NonNull Paint p, int x, int dir,
                                int top, int baseline, int bottom,
                                @NonNull CharSequence text, int start, int end,
                                boolean first, @NonNull Layout layout) {
    //Span的起始位置
    int st = ((Spanned) text).getSpanStart(this);
    int ix = (int) x;
    //获取Span起始所在位置的行top
    int itop = (int) layout.getLineTop(layout.getLineForOffset(st));
	
    int dw = mDrawable.getIntrinsicWidth();
    int dh = mDrawable.getIntrinsicHeight();

    // XXX What to do about Paint?
    mDrawable.setBounds(ix, itop, ix + dw, itop + dh);
    mDrawable.draw(c);
  }
}
```

### 2.6 `TabStopSpan`

用来将`\t`替换成相应的空行。普通情况下`\t`不会显示，使用`TabStopSpan`可以将其替换成相应长度的空白区域