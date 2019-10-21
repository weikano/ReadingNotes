# RecyclerView 原理

## 一、从View的三大周期看RecyclerView

### 1. onMeasure

```java
protected void onMeasure(int widthSpec, int heightSpec) {
  //mLayout.onMeasure基本上都是调用defaultOnMeasure方法，只有在leanback.GridLayoutManager才有修改
  //没有设置LayoutManager，调用defaultOnMeasure方法
  if(mLayout == null) {
    //min是最小宽高, desired为上下/左右padding之和
    //1. 如果specMode为EXACTLY，那么就设置为size
    //2.  如果是AT_MOST，那么就取Math.min(size, Math.max(desired, min))
    //3. 其他的返回Math.max(desired, min);
    defaultOnMeasure(widthSpec, heightSpec);
    return;
  }
  if(mLayout.isAutoMeasureEnabled()) {
    inal int widthMode = MeasureSpec.getMode(widthSpec);
    final int heightMode = MeasureSpec.getMode(heightSpec);

    /**
             * This specific call should be considered deprecated and replaced with
             * {@link #defaultOnMeasure(int, int)}. It can't actually be replaced as it could
             * break existing third party code but all documentation directs developers to not
             * override {@link LayoutManager#onMeasure(int, int)} when
             * {@link LayoutManager#isAutoMeasureEnabled()} returns true.
             */
    mLayout.onMeasure(mRecycler, mState, widthSpec, heightSpec);

    final boolean measureSpecModeIsExactly =
      widthMode == MeasureSpec.EXACTLY && heightMode == MeasureSpec.EXACTLY;
    if (measureSpecModeIsExactly || mAdapter == null) {
      return;
    }

    if (mState.mLayoutStep == State.STEP_START) {
      dispatchLayoutStep1();
    }
    // set dimensions in 2nd step. Pre-layout should happen with old dimensions for
    // consistency
    mLayout.setMeasureSpecs(widthSpec, heightSpec);
    mState.mIsMeasuring = true;
    dispatchLayoutStep2();

    // now we can get the width and height from the children.
    mLayout.setMeasuredDimensionFromChildren(widthSpec, heightSpec);

    // if RecyclerView has non-exact width and height and if there is at least one child
    // which also has non-exact width & height, we have to re-measure.
    if (mLayout.shouldMeasureTwice()) {
      mLayout.setMeasureSpecs(
        MeasureSpec.makeMeasureSpec(getMeasuredWidth(), MeasureSpec.EXACTLY),
        MeasureSpec.makeMeasureSpec(getMeasuredHeight(), MeasureSpec.EXACTLY));
      mState.mIsMeasuring = true;
      dispatchLayoutStep2();
      // now we can get the width and height from the children.
      mLayout.setMeasuredDimensionFromChildren(widthSpec, heightSpec);
    }
  }else {
    //adapter changes cannot affect the size of RecyclerView
    if(mHasFixedSize) {
      //默认调用recyclerview.defaultOnMeasure
      mLayout.onMeasure(mRecycler, mState, widthSpec, heightSpec);
      return;
    }
    if(mAdapterUpdateDuringMeasure) {
      //start和onEnter都是修改标志位
      startInterceptRequestLayout();
      onEnterLayoutOrScroll();
      //consumes adapter updates and calculates which type of animations we want to run
      processAdapterUpdatesAndSetAnimationFlags();
      onExitLayoutOrScroll();
      if (mState.mRunPredictiveAnimations) {
        mState.mInPreLayout = true;
      } else {
        // consume remaining updates to provide a consistent state with the layout pass.
        mAdapterHelper.consumeUpdatesInOnePass();
        mState.mInPreLayout = false;
      }
      stopInterceptRequestLayout(false/**performLayoutChildren**/);
    }else if(mState.mRunPredictiveAnimations) {
      setMeasuredDimension(getMeasuredWidth(), getMeasuredHeight());
      return;
    }
    if (mAdapter != null) {
      mState.mItemCount = mAdapter.getItemCount();
    } else {
      mState.mItemCount = 0;
    }
    startInterceptRequestLayout();
    mLayout.onMeasure(mRecycler, mState, widthSpec, heightSpec);
    stopInterceptRequestLayout(false);
    mState.mInPreLayout = false; //
  }
}
```

### 2. onLayout

`RecyclerView`将`onLayout`分为三个步骤：

```java
//layout要经历三个阶段：Step1、Step2和Step3
//Step1：处理adapter的updates、决定使用的动画、保存信息和当前的view、如果必要进行预测的layout并保存信息
//step2：真正进行layout的地方，将layout交给LayoutManager.onLayoutChildren处理。可能会执行多次，onMeasure时也会触发该方法
//Step3：保存动画相关信息，触发动画以及做一些必要的清理
void dispatchLayout() {
  if (mAdapter == null) {
    Log.e(TAG, "No adapter attached; skipping layout");
    // leave the state in START
    return;
  }
  if (mLayout == null) {
    Log.e(TAG, "No layout manager attached; skipping layout");
    // leave the state in START
    return;
  }
  mState.mIsMeasuring = false;
  if (mState.mLayoutStep == State.STEP_START) {
    dispatchLayoutStep1();
    mLayout.setExactMeasureSpecsFrom(this);
    dispatchLayoutStep2();
  } else if (mAdapterHelper.hasUpdates() || mLayout.getWidth() != getWidth()
             || mLayout.getHeight() != getHeight()) {
    // First 2 steps are done in onMeasure but looks like we have to run again due to
    // changed size.
    mLayout.setExactMeasureSpecsFrom(this);
    dispatchLayoutStep2();
  } else {
    // always make sure we sync them (to ensure mode is exact)
    mLayout.setExactMeasureSpecsFrom(this);
  }
  dispatchLayoutStep3();
}
```

### 3. onDraw

`onDraw`的代码很简单，涉及到了`ItemDecoration`

```java
public void onDraw(Canvas c) {
  super.onDraw(c);
  final int count = mItemDecorations.size();
  for(int i=0;i<count;i++) {
    mItemDecorations.get(i).onDraw(c, this, mState);
  }
}
```

## 二、LayoutManager

## 三、ItemDecoration