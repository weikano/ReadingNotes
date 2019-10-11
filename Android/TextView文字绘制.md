# TextView文字绘制

## 1. onMeasure

`TextView`的`onMeasure`过程跟`Layout`有相当大的关系，其通过`makeNewLayout`来获取`Layout`对象

`onMeasure`过程如下

1. 获取`layout`对象
2. 如果`MeasureSpec.EXACT`，那么返回对应的height，不然就通过`getDesiredHeight`方法由`Layout`计算出对应的高度

```java
public void onMeasure() {
  //一般来说TextView都设置为width=MATCH_PARENT, height=WRAP_CONTENT
  //所以width可以确定为下列
  int width = MeasureSpec.getSize(widthMeasureSpec);
  int want = width - getCompoundPaddingLeft() - getCompoundPaddingRight();
  int hintWant = want;
  int hintWidth = (mHintLayout == null) ? hintWant : mHintLayout.getWidth();//现在是hintWant
  if(mLayout == null) {
    makeNewLayout(want, hintWant, boring, hintBoring, width-getCompoundPaddingLeft()-getCompoundPaddingRight(), false);
  }
  //获取TextView的高度
  if(heightMode == MeasureSpect.AT_MOST) {
    height = heightSize;
    mDesiredHeightAtMeasure = -1;
  }else {
    int desired = getDesiredHeight();
    height = desired;
    mDesiredHeightAtMeasure = desired;
    if(heightMode == MeasureSpect.AT_MOST) {
      height = Math.min(desired, heightSize);
    }
  }
  setMeasureDimension(width, height)
}

private int getDesiredHeight() {
  return Math.max(getDesiredHeight(mLayout, true), getDesiredHeight(mHintLayout, mEllipsize != null));
}

private int getDesiredHeight(Layout layout, boolean cap) {
  //重点，获取layout的高度
  int desired = layout.getHeight(cap);
  //左右的Drawable的高度需要考虑
  desired = Math.max(Math.max(desired, dr.mDrawableHeightLeft), dr.mDrawableHeightRight);
  //获取行数
  int linecount = layout.getLineCount();
  //加上textview的上下padding
  desired += paddingTop+paddingBottom;
  //这种情况只有setHeight(), mMaxium = mMinimum = pixels
  if(mMaxMode != LINES) {    
    desired = Math.min(desired, mMaxium);
  }else if(cap && linecount>mMaxium &&(layout instanceof DynamicLayout || layout instanceof BoringLayout)) {
    //如果当前是setMaxLines并且行数比mMaxium还大
    //mMaxium是maxLines 
    //**重点**
    //只获取layout到mMaxium行的高度
    desired = layout.getLineTop(mMaxium);
    desired = Math.max(Math.max(desired, dr.mDrawableHeightLeft), dr.mDrawableHeightRight);
    desired += padding;
    linecount = mMaxium;
  }
  
  if(mMinMode == LINES) {
    if(linecount < mMinimum) {
      //getLineHeight() == FastMath.round(mTextPaint.getFontMetricsInt(null)*mSpaceMult + mSpacingAdd)
      //如果当前行数不够mMinimum，那么要凑齐高度
      desired += getLineHeight() *(mMinimum - linecount)
    }
  }else {
    //这里是setHeight设置的desired为大于mMinimum小于mMaxium
    desired = Math.max(desired, mMinimum)
  }
  desired = Math.max(desired, getSuggestedMinimumHeight());
  return desired;
}
```

## 2. makeNewLayout

```java
//wantWidth为width-paddingHorizontal
//hintWidth最开始为wantWidth
//ellipsizeWidth也为wantWidth
//bringIntoView = false
public void makeNewLayout(int wantWidth, int hintWidth, BoringLayout.Metrics boding, hintBoring, int ellipsizeWidth, boolean bringIntoView) {
  mLayout = makeSingleLayout(wantWidth, boring, ellipsisWidth, alignment, shouldEllipsize,
                effectiveEllipsize, effectiveEllipsize == mEllipsize);
  mHintLayout = StaticLayout.Builder().xxx.xxx.build();
}

public boolean useDynamicLayout() {
  return isTextSelectable() || (mSpannable != null && mPrecomputed == null);
}

protected Layout makeSingleLayout(int wantWidth, BoringLayout.Metrics boring, int ellipsisWidth,
                                  Layout.Alignment alignment, boolean shouldEllipsize, TruncateAt effectiveEllipsize,
                                  boolean useSaved) {
  Layout result = null;
  //通常都会使用DynamicLayout
  if (useDynamicLayout()) {
    final DynamicLayout.Builder builder = DynamicLayout.Builder.obtain(mText, mTextPaint,
                                                                       wantWidth)
      .setDisplayText(mTransformed)
      .setAlignment(alignment)
      .setTextDirection(mTextDir)
      .setLineSpacing(mSpacingAdd, mSpacingMult)
      .setIncludePad(mIncludePad)
      .setUseLineSpacingFromFallbacks(mUseFallbackLineSpacing)
      .setBreakStrategy(mBreakStrategy)
      .setHyphenationFrequency(mHyphenationFrequency)
      .setJustificationMode(mJustificationMode)
      .setEllipsize(getKeyListener() == null ? effectiveEllipsize : null)
      .setEllipsizedWidth(ellipsisWidth);
    result = builder.build();
  } else {
    if (boring == UNKNOWN_BORING) {
      boring = BoringLayout.isBoring(mTransformed, mTextPaint, mTextDir, mBoring);
      if (boring != null) {
        mBoring = boring;
      }
    }

    if (boring != null) {
      if (boring.width <= wantWidth
          && (effectiveEllipsize == null || boring.width <= ellipsisWidth)) {
        if (useSaved && mSavedLayout != null) {
          result = mSavedLayout.replaceOrMake(mTransformed, mTextPaint,
                                              wantWidth, alignment, mSpacingMult, mSpacingAdd,
                                              boring, mIncludePad);
        } else {
          result = BoringLayout.make(mTransformed, mTextPaint,
                                     wantWidth, alignment, mSpacingMult, mSpacingAdd,
                                     boring, mIncludePad);
        }

        if (useSaved) {
          mSavedLayout = (BoringLayout) result;
        }
      } else if (shouldEllipsize && boring.width <= wantWidth) {
        if (useSaved && mSavedLayout != null) {
          result = mSavedLayout.replaceOrMake(mTransformed, mTextPaint,
                                              wantWidth, alignment, mSpacingMult, mSpacingAdd,
                                              boring, mIncludePad, effectiveEllipsize,
                                              ellipsisWidth);
        } else {
          result = BoringLayout.make(mTransformed, mTextPaint,
                                     wantWidth, alignment, mSpacingMult, mSpacingAdd,
                                     boring, mIncludePad, effectiveEllipsize,
                                     ellipsisWidth);
        }
      }
    }
  }
  if (result == null) {
    StaticLayout.Builder builder = StaticLayout.Builder.obtain(mTransformed,
                                                               0, mTransformed.length(), mTextPaint, wantWidth)
      .setAlignment(alignment)
      .setTextDirection(mTextDir)
      .setLineSpacing(mSpacingAdd, mSpacingMult)
      .setIncludePad(mIncludePad)
      .setUseLineSpacingFromFallbacks(mUseFallbackLineSpacing)
      .setBreakStrategy(mBreakStrategy)
      .setHyphenationFrequency(mHyphenationFrequency)
      .setJustificationMode(mJustificationMode)
      .setMaxLines(mMaxMode == LINES ? mMaximum : Integer.MAX_VALUE);
    if (shouldEllipsize) {
      builder.setEllipsize(effectiveEllipsize)
        .setEllipsizedWidth(ellipsisWidth);
    }
    result = builder.build();
  }
  return result;
}
```

## 3. onDraw

```java
protected void onDraw(Canvas canvas) {
  restartMarqueeIfNeeded();
  super.onDraw(canvas);
  //画周边drawable，省略
  Layout layout = mLayout;
  if(mHint != null && mText.length() == 0) {
    layout = mHintLayout;
  }
  //裁剪画布，之后的canvas.drawxxx只在[clipLeft, clipTop, clipRight, clipBottom]范围，之后的起始点(0,0)为原来的(clipLeft,clipTop)
  canvas.clipRect(clipLeft, clipTop, clipRight, clipBottom);
  //接下来是marquee效果，通过canvas.translate()实现。
  //marquee通过cherographer来刷新界面
  //省略
  //....
  //重点：显示text
  //一般来说mEditor!=null，mEditor.onDraw其实也是使用layout.draw来绘制文字
  if(mEditor != null) {
    mEditor.onDraw(canvas, layout, highlight, mHighlightPaint, cursorOffsetVertical);
  }else {
    layout.draw(canvas, highlight, mHighlightPaint, cursorOffsetVertical);
  }
}
```

## 4. Editor

```java
//Editor.java
void onDraw(canvas, layout, Path hightlight, Paint highlightPaint, int cursorOffsetVertical) {
  //绘制光标
  drawCursor(canvas, cursorOffsetVertical);
  if(mTextView.canHaveDisplayList() && canvas.isHardwareAccelerated()) {
    drawHardwareAccelerated(canvas, layout, highlight, highlightPaint, cursorOffsetVertical);
  }else {
    //其实也是layout.drawBackground + layout.drawText
    layout.draw(canvas, highlight, highlightPaint, cursorOffsetVertical);
  }
}

private void drawHardwareAccelerated() {
  layout.drawBackground(canvas, highlight, highlightPaint, cursorOffsetVertical,
                firstLine, lastLine);
  //忽略其他的
  layout.drawText(displayListCanvas, blockBeginLine, blockEndLine);
}
```

## 5. Layout

由`Text`的`makeLayout`可知，`Editor`中的`layout`为`DynamicLayout`, 涉及到的方法如下：

1. `getHeight()`，内部实现由`getLineTop(int line)`
2. `getLineCount()`
3. `layout.drawBackground()`
4. `layout.draw()`
5. `layout.drawText()`

以`DynamicLayout`为例分析上述方法

上述1-3方法其实都跟`PackgedIntVector`对象`mInts`有关，它在`generate`和`reflow`方法中会对文本进行分析，保存每一行的`ascent`和`descent`之类的信息，这样就能获取到对应的`lineTop`和`lineCount`

```java
public int getHeight() {
  return getLineTop(getLineCount());
}

public int getLineCount() {
  return mInts.size() -1;
}

public int getLineTop(int line) {
  return mInts.getValue(line, TOP);
}
```

上述代码的关键在于`mInts`，由`generate`方法赋值，而`generate`方法由`Builder`方法在`build`时调用

`mInts`是一个`PackedIntVector`类。`PackedIntVector`是把一个行列式用一个`int[]`来保存

```java
//mColumns为构造函数中传入的列数
//比如要往第2行第三列传入数据
//setValue(1,2,10)
//mValues[1*10+2]=10;
public void setValue(int row, int column, int value) {
  mValues[row*mColumns + column] = value;
}
public int size() {
  return mRows - mRowGapLength;
}
```

接下来继续看`mInts`

```java
//DynamicLayout.java
private void generate(Builder b) {
  mInts = new PackedIntVector(COLUMN_ELLIPSIZE or COLUMN_NORMAL);
  int[] start = new int[mInts.size()];
  final Paint.FontMestricsInt fm = b.mFontMetricsInt;
  start[DIR] = DIR_LEFT_TO_RIGHT << DIR_SHIFT;
  start[TOP] = 0;
  start[DESCENT] = fm.descent;
  mInts.insertAt(0, start);
  start[TOP] = fm.descent - fm.ascent;
  mInts.insert(1, start);
}
//mInts保存了每一行的Descent、Top等信息
public void reflow(CharSequence s, int where, int before, int after) {
  //起始行
  int startline = getLineForOffset(where);
  //起始行top
  int startv = getLineTop(startline);
  //结束行
  int endline = getLineForOffset(where + before);
  if (where + after == len)
    endline = getLineCount();
  //结束行top
  int endv = getLineTop(endline);
  int n = reflowed.getLineCount();
  for (int i = 0; i < n; i++) {
    final int start = reflowed.getLineStart(i);
    ints[START] = start;
    ints[DIR] |= reflowed.getParagraphDirection(i) << DIR_SHIFT;
    ints[TAB] |= reflowed.getLineContainsTab(i) ? TAB_MASK : 0;

    int top = reflowed.getLineTop(i) + startv;
    if (i > 0)
      top -= toppad;
    ints[TOP] = top;

    int desc = reflowed.getLineDescent(i);
    if (i == n - 1)
      desc += botpad;

    ints[DESCENT] = desc;
    ints[EXTRA] = reflowed.getLineExtra(i);
    objects[0] = reflowed.getLineDirections(i);

    final int end = (i == n - 1) ? where + after : reflowed.getLineStart(i + 1);
    ints[HYPHEN] = reflowed.getHyphen(i) & HYPHEN_MASK;
    ints[MAY_PROTRUDE_FROM_TOP_OR_BOTTOM] |=
      contentMayProtrudeFromLineTopOrBottom(text, start, end) ?
      MAY_PROTRUDE_FROM_TOP_OR_BOTTOM_MASK : 0;

    if (mEllipsize) {
      ints[ELLIPSIS_START] = reflowed.getEllipsisStart(i);
      ints[ELLIPSIS_COUNT] = reflowed.getEllipsisCount(i);
    }
	  
    mInts.insertAt(startline + i, ints);
    mObjects.insertAt(startline + i, objects);
  }

  updateBlocks(startline, endline - 1, n);
}

```

接下来的重点是`layout.drawBackground`和`layout.drawText`两个方法

```java
//Layout.java
//其实就是LineBackgroundSpan效果实现
public void drawBackground(canvas, highlight, highlightPaint, xxx) {
  //提取出mText中LineBackgroundSpan的相关信息
  mLineBackgroundSpans.init((Spanned)mText, 0, textLength);
  for(int n=0;n<spansLength;n++) {
    ((LineBackgroundSpan)spans[n]).drawBackground();
  }
}
```

`Backgound`绘制完毕后，接下来就是`layout.drawText`

```java
public void drawText() {
  ParagraphStyle[] spans = NO_PARA_SPANS;
  TextLine tl = TextLine.obtain();
  for(int lineNum = firstLine; lineNum <= lastLine; lineNum ++ ) {
    //对每一行
    //1. 获取所有的ParagraphStyle
    //2. 处理其中的AlignmentSpan忽略
    //3. 处理LeadingMarginSpan2
    //4. 处理LeadingMarginSpan
    if(mSpannedText) {
      spans = getParagraphSpans();
      final int length = spans.length;
      for(int n=0;n<length;n++) {
        if(spans[n] instanceof LeadingMarginSpan2) {
          int count = spans[n].getLeadingMarginLineCount();
          int startLine = getLineForOffset(sp.getSpanStart(spans[n]));
          if(lineNum < startLine + count) {
            useFirstLineMargin = true;
            break;
          }
        }
      }
      for(int n=0;n<length;n++) {
        if(spans[n] instanceof LeadingMarginSpan) {
          //绘制margin所占用的区域
          margin.drawLeadingMargin();
          //以RTL为例子
          //获取偏移量
          right -= margin.getLeadingMargin(useFirstLineMargin);
        }
      }
    }
    //接下来就是根据上述span信息绘制文字
    //一般情况，不包含Span等，直接由canvas.drawText
    //否则交个TextLine.draw
    //ltop = previousLineBottom
    //lbottom = getLineTop(lineNum + 1)
    //lbaseline = lbottom - getLineDescent(linenum);
    if(direction == LTR  && !mSpannedTExt && !hasTab && !justfy) {
      canvas.drawText(buf, start, end, x, lbaseline, paint);
    }else {
      tl.set(paint, buf, start, end, dir, directions, hasTab, tabStops);
      tl.draw(canvas,x,ltop,lbaseline,lbottom);
    }
  }
}
```

## 5. TextLine

```java
//top 为ltop = previousLineBttom，上一行的bottom
//y为lbaseline = lbottom - getLineDescent(lineNum) 下一行的top减去当前行的descent
//bottom为lbottom = getLineTop(lineNum + 1) 下一行的top
void draw(canvas, x, top, y, bottom) {
  if (!mHasTabs) {
    if (mDirections == Layout.DIRS_ALL_LEFT_TO_RIGHT) {
      drawRun(c, 0, mLen, false, x, top, y, bottom, false);
      return;
    }
    if (mDirections == Layout.DIRS_ALL_RIGHT_TO_LEFT) {
      drawRun(c, 0, mLen, true, x, top, y, bottom, false);
      return;
    }
  }  
  int[] runs = mDirections.mDirections;
  int lastRunIndex = runs.length - 2;
  for(int i=0;i<runs.length;i+=2) {
   	h+= drawRun(c,xx,xx, runIsRtl, x+h, top, y, bottom, i!=lastRunIndex || j != mLen);
    if(codept == '\t') {
      h =mDir * nextTab(h * mDir);
    }
  }
}
//x=x+h, top = top, y = y, bottom = bottom
private float drawRun(canvas, start, limit, rtl, x, top, y, bottom, needWidth)  {
  if ((mDir == Layout.DIR_LEFT_TO_RIGHT) == runIsRtl) {
    float w = -measureRun(start, limit, limit, runIsRtl, null);
    handleRun(start, limit, limit, runIsRtl, c, x + w, top,
              y, bottom, null, false);
    return w;
  }
  return handleRun(start, limit, limit, runIsRtl, c, x, top,
                   y, bottom, null, needWidth);
}
//真正调用Span绘制
private float handleRun(start, measureLImit, limit, rtl, canvas, x, top, y, bottom ,fmi, needWidth) {
  final boolean needsSpanMeasurement;
  if (mSpanned == null) {
    needsSpanMeasurement = false;
  } else {
    mMetricAffectingSpanSpanSet.init(mSpanned, mStart + start, mStart + limit);
    mCharacterStyleSpanSet.init(mSpanned, mStart + start, mStart + limit);
    needsSpanMeasurement = mMetricAffectingSpanSpanSet.numberOfSpans != 0
      || mCharacterStyleSpanSet.numberOfSpans != 0;
  }
  if (!needsSpanMeasurement) {
    final TextPaint wp = mWorkPaint;
    wp.set(mPaint);
    wp.setHyphenEdit(adjustHyphenEdit(start, limit, wp.getHyphenEdit()));
    return handleText(wp, start, limit, start, limit, runIsRtl, c, x, top,
                      y, bottom, fmi, needWidth, measureLimit, null);
  }
  //接下来处理各种Span，忽略
  x += handleText();
}
//绘制文字
private float handleText() {
  drawTextRun();
}

private void drawTextRun() {
  canvas.drawTextRun();
}
```

