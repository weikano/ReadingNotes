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

## 二、回收复用

`RecyclerView`的回收复用机制是通过内部的`Recycler`和`RecycledViewPool`来实现的，具体实现可参考`LayoutManager`。关键点在于`onLayoutChildren`以及其本身的`Recycler`，具体应用可以看`getViewForPosition`方法。`LayoutManager`一般都是通过`LayoutState.next(Recycler)`来获取对应的`View`，这就涉及到了`getViewForPosition`方法中的`ViewHolder`复用

```java
//Recycler
View getViewForPosition(int position, boolean dryRun) {
  return tryGetViewHolderForPositionByDeadline(position, dryRun, FOREVER_NS).itemView;
}
ViewHolder tryGetViewHolderForPositionByDeadline(position, dryRun, deadlineNs) {
  boolean fromScrapOrHiddenOrCache = false;
  ViewHolder holder = null;
  //第一次进来，返回Null
  if(mState.isPreLayout()) {
    holder = getChangedScrapViewForPosition(position);
    fromScrapOrHiddenOrCache = holder != null;
  }
  if(holder == null) {
    holder = getScrapOrHiddenOrCachedHolderForPosition(position, dryRun);
    if(holder != null) {
      if(!validateViewHolderForOffsetPosition(holder)) {
        if(!dryRun) {
          holder.addFlags(ViewHolder.FLAG_INVALID);
          if(holder.isScrap()) {
            removeDetachedView(holder.itemView, false);
            holder.unScrap();
          }else if(holder.wasReturnedFromScrap()) {
            holder.clearReturnedFromScrapFlags();
          }
          recyclerViewHolderInternal(holder);
          holder = null;
        }
      }else {
        fromScrapOrHiddenOrCache = true;
      }
    }
    
    if(holder == null) {
      final int offsetPosition = mAdapterHelper.findPositionOffset(position);
      final int type = mAdapter.getItemViewType(offsetPosition);
      if(mAdapter.hasStableIds()) {
        holder = getScrapOrCachedViewForId(mAdapter.getItemId(offsetPosition), type, dryRun);
        if(holder != null) {
          holder.mPosition = offsetPosition;
          fromScrapOrHiddenOrCache = true;
        }
      }
    }
    if(holder == null && mViewCacheExtension != null) {
      final View view = mViewCacheExtension.getViewForPositionAndType(this, positon, type);
      if(view != null) {
        holder = getChildViewHolder(view);
      }
    }
  }
  if(holder == null) {
    holder = getRecycledViewPool().getRecycledView(type);
    if(holder != null) {
      holder.resetInternal();
    }
  }
  if(holder == null) {
    holder = mAdapter.createViewHolder(RecyclerView.this, type);
  }
  //....
  boolean bound = false;
  if(!holder.isBound() || holder.needsUpdate() || holder.isInvalid()) {
    final int offsetPosition = mAdapterHelper.findPositionOffset(position);
    bound = tryBindViewHolderByDeadline(holder, offsetPosition, position, deadlineNs);
  }
  final ViewGroup.LayoutParams lp = holder.itemView.getLayoutParams();
  final LayoutParams rvLayoutParams;
  if (lp == null) {
    rvLayoutParams = (LayoutParams) generateDefaultLayoutParams();
    holder.itemView.setLayoutParams(rvLayoutParams);
  } else if (!checkLayoutParams(lp)) {
    rvLayoutParams = (LayoutParams) generateLayoutParams(lp);
    holder.itemView.setLayoutParams(rvLayoutParams);
  } else {
    rvLayoutParams = (LayoutParams) lp;
  }
  rvLayoutParams.mViewHolder = holder;
  rvLayoutParams.mPendingInvalidate = fromScrapOrHiddenOrCache && bound;
  return holder;
}
```

首先通过`getChangedScrapViewForPosition`方法获取`ViewHolder`，如果是第一次执行，肯定返回Null

**`mChangedScrap`是通过`Recycler.scrapView`->`Recycler.getScrapOrHiddenOrCachedForPosition`来填充数据**

```java
ViewHolder getChangedScrapViewForPosition(int position) {
  final int changedScrapSize;
  //mChangedScrap中保存的是被attach到RecyclerView但是具备rebinding和reuse的ViewHolder，通过
  //Recylcer.scrapView(view)来添加
  //如果没有过，那么第一次进来时就为null
  if(mChangedScrap == null || (changedScrapSize = mChangedScrap.size()) == 0) {
    return null;
  }
  for(int i=0;i<changedScrapSize;i++) {
    final ViewHolder holder = mChangedScrap.get(i);
    if(!holder.wasReturnedFromScrap() && holder.getLayoutPosition == position) {
      holder.addFlags(ViewHolder.FLAG_RETURNED_FROM_SCRAP);
      return holder;
    }
  }
  if(mAdapter.hasStableIds()) {
    final int offsetPosition = mAdapterHelper.findPositionOffset(position);
    if(offsetPosition > 0 && offsetPostion < mAdapter.getItemCount()) {
      final long id = mAdapter.getItemId(offsetPosition);
      for(int i=0;i<mChangedScrapSize;i++) {
        final ViewHolder holder = mChangedScrapSize.get(i);
        if(!holder.wasReturnedFromScrap() && holder.getItemId() == id) {
          holder.addFlags(ViewHolder.FLAG_RETURNED_FROM_SCRAP);
          return holder;
        }
      }
    }
  }
  return null;
}
```

接下来交给`getScrapOrHiddenOrCacheHolderForPosition`

`mAttachedScrap`也是在`Recycler.scrapView`中填充数据

```java
//从attach scrap、hidden children或者cache中获取ViewHolder
ViewHolder getScrapOrHiddenOrCachedHolderForPosition(position, dryRun) {
  for (int i = 0; i < scrapCount; i++) {
    final ViewHolder holder = mAttachedScrap.get(i);
    if (!holder.wasReturnedFromScrap() && holder.getLayoutPosition() == position
        && !holder.isInvalid() && (mState.mInPreLayout || !holder.isRemoved())) {
      holder.addFlags(ViewHolder.FLAG_RETURNED_FROM_SCRAP);
      return holder;
    }
  }

  if (!dryRun) {
    //从mHiddenViews中找到view所在ViewHolder的position相等的ViewHolder（从View的LayoutParams.mViewHolder），并且要满足!holder.isInValid()&&!holder.isRemoved，然后返回对应的view
    View view = mChildHelper.findHiddenNonRemovedView(position);
    if (view != null) {
      // This View is good to be used. We just need to unhide, detach and move to the
      // scrap list.
      final ViewHolder vh = getChildViewHolderInt(view);
      //将view从mHiddenViews中移除
      mChildHelper.unhide(view);
      int layoutIndex = mChildHelper.indexOfChild(view);
      if (layoutIndex == RecyclerView.NO_POSITION) {
        throw new IllegalStateException("layout index should not be -1 after "
                                        + "unhiding a view:" + vh + exceptionLabel());
      }
      //RecyclerView.detachViewFromParent，即从mChildren中移除index对应的view
      mChildHelper.detachViewFromParent(layoutIndex);
      //将view放到mChangedScrap或者mAttachedScrap中
      scrapView(view);
      vh.addFlags(ViewHolder.FLAG_RETURNED_FROM_SCRAP
                  | ViewHolder.FLAG_BOUNCED_FROM_HIDDEN_LIST);
      return vh;
    }
  }

  // Search in our first-level recycled view cache.
  final int cacheSize = mCachedViews.size();
  for (int i = 0; i < cacheSize; i++) {
    final ViewHolder holder = mCachedViews.get(i);
    // invalid view holders may be in cache if adapter has stable ids as they can be
    // retrieved via getScrapOrCachedViewForId
    if (!holder.isInvalid() && holder.getLayoutPosition() == position) {
      if (!dryRun) {
        mCachedViews.remove(i);
      }
      if (DEBUG) {
        Log.d(TAG, "getScrapOrHiddenOrCachedHolderForPosition(" + position
              + ") found match in cache: " + holder);
      }
      return holder;
    }
  }
  return null;
}
```

`scrapView`的实现

```java
void scrapView(View view) {
  final ViewHolder holder = getChildViewHolderInt(view);
  if (holder.hasAnyOfTheFlags(ViewHolder.FLAG_REMOVED | ViewHolder.FLAG_INVALID)
      || !holder.isUpdated() || canReuseUpdatedViewHolder(holder)) {
    if (holder.isInvalid() && !holder.isRemoved() && !mAdapter.hasStableIds()) {
      throw new IllegalArgumentException("Called scrap view with an invalid view."
                                         + " Invalid views cannot be reused from scrap, they should rebound from"
                                         + " recycler pool." + exceptionLabel());
    }
    holder.setScrapContainer(this, false);
    mAttachedScrap.add(holder);
  } else {
    if (mChangedScrap == null) {
      mChangedScrap = new ArrayList<ViewHolder>();
    }
    holder.setScrapContainer(this, true);
    mChangedScrap.add(holder);
  }
}
```

如果`getScrapOrHiddenOrCachedHolderForPosition`也找不到，即`scrap\hidden list\cache`里面也没有，接下来就判断`adapter.hasStableIds`，来根据`getItemId`来获取`ViewHolder`

```java
//先从mAttachedScrap中找
//然后从mCachedView中找
//只有符合id，type的ViewHolder才返回
ViewHolder getScrapOrCachedViewForId(id, type, dryRun) {
  final int count = mAttachedScrap.size();
  for(int i=count-1;i>=0;i--) {
    final ViewHolder holder = mAttachedScrap.get(i);
    if(holder.getItemId() == id && !holder.wasReturnedFromScrap()) {
      if(type == holder.itemViewType) {
        holder.addFlags(ViewHolder.FLAG_RETURNED_FROM_SCRAP);
        return holder;
      }else if(!dryRun) {
        mAttachedScrap.remove(i);
        removeDetachedView(holder.itemView, false);
        quickRecycleScrapView(holder.itemView)
      }
    }
  }
   // Search the first-level cache
  final int cacheSize = mCachedViews.size();
  for (int i = cacheSize - 1; i >= 0; i--) {
    final ViewHolder holder = mCachedViews.get(i);
    if (holder.getItemId() == id) {
      if (type == holder.getItemViewType()) {
        if (!dryRun) {
          mCachedViews.remove(i);
        }
        return holder;
      } else if (!dryRun) {
        recycleCachedViewAt(i);
        return null;
      }
    }
  }
  return null;
}
```

接下来交给`ViewCacheExtension`，一般都不会用到？`ViewCacheExtension`一般是开发者自定义的额外的`View`的缓存

```java
final View view = mViewCacheExtension.getViewForPositionAndType(this, position, type);
if(view != null) {
  holder = getChildViewHolder(view);
}
```

接下来交给`RecycledViewPool`，从`RecycledViewPool`中获取一个`ViewHolder`

`RecycledViewPool`通过调用`putRecycledView`方法添加`mScrapData`，而`Recycler.addViewHolderToRecycledViewPool->recycleCachedViewAt`

```java
//RecycledViewPool.java
public ViewHolder getRecycledView(int viewType) {
  final ScrapData sd = mScap.get(viewType);
  if(sd != null && !sd.mScrapHeap.isEmpty()) {
    final ArrayList<ViewHolder> scrapHeap = sd.mScrapHeap;
    return scrapHeap.remove(scrapHeap.size() - 1);
  }
  return null;
}
```

如果`RecycledViewPool`也拿不到对应的`ViewHolder`，那么只能通过`Adapter.createViewHolder`来处理。

获取到对应的`ViewHolder`后，调用`Adapter.bindViewHolder`方法来绑定数据

所以`Recycler`的缓存机制如下：

1. 首先通过`mChangedScrap`查找对应的`ViewHolder`
2. 接下来从`mAttachedScrap`、`mHiddenViews`或者`mCacheViews`中获取`ViewHolder`
3. 然后交给`ViewCacheExtension`，由开发者实现的`View`的缓存中查找，一般来说不会用到
4. 然后由`RecycledViewPool`中查找
5. 最后交由`Adapter.createViewHolder`
6. 最后调用`Adapter.bindViewHolder`方法

## 三、LayoutManager

由上节可知，具体的实现在`onLayoutChildren`，不同的`LayoutManager`有不同的实现

**TBD**

## 四、ItemDecoration

`ItemDecoration`是一个抽象类，一般是用来给`RecyclerView`子`View`添加装饰效果，比如分割线、高亮等。

```java
public abstract static class ItemDecoration {
  public void onDraw(Canvas c, RecyclerView parent, State state);
  public void onDrawOver(Canvas c, RecyclerView parent, State state);
  public void getItemOffset(Rect out, View child, RecyclerView parent, State state);
}
```

其在`RecyclerView`中只在三处地方有使用到：`draw`、`onDraw`和`getItemDecorInsetsForChild`

```java
public void draw(Canvas c) {
  super.draw(c);//触发onDraw()
  final int count = mItemDecorations.size();
  for(int i=0;i<count;i++) {
    mItemDecorations.get(i).onDrawOver(c, this, mState);
  }
  //省略
}

public void onDraw(Canvas c) {
  super.onDraw(c);
  final int count = mItemDecorations.size();
  for(int i=0;i<count;i++) {
    mItemDecorations.get(i).onDrawOver(c, this, mState);
  }
}

Rect getItemDecorInsetsForChild(View child) {
  final LayoutParams lp = (LayoutParams) child.getLayoutParams();
  if (!lp.mInsetsDirty) {
    return lp.mDecorInsets;
  }

  if (mState.isPreLayout() && (lp.isItemChanged() || lp.isViewInvalid())) {
    // changed/invalid items should not be updated until they are rebound.
    return lp.mDecorInsets;
  }
  final Rect insets = lp.mDecorInsets;
  insets.set(0, 0, 0, 0);
  final int decorCount = mItemDecorations.size();
  for (int i = 0; i < decorCount; i++) {
    mTempRect.set(0, 0, 0, 0);
    mItemDecorations.get(i).getItemOffsets(mTempRect, child, this, mState);
    insets.left += mTempRect.left;
    insets.top += mTempRect.top;
    insets.right += mTempRect.right;
    insets.bottom += mTempRect.bottom;
  }
  lp.mInsetsDirty = false;
  return insets;
}

public void measureChildWithMargins(View child, int widthUsed, int heightUsed) {
  final LayoutParams lp = (LayoutParams) child.getLayoutParams();
  final Rect insets = mRecyclerView.getItemDecorInsetsForChild(child);
  widthUsed += insets.left + insets.right;
  heightUsed += insets.top + insets.bottom;
  final int widthSpec = getChildMeasureSpec(getWidth(), getWidthMode(),
                                            getPaddingLeft() + getPaddingRight()
                                            + lp.leftMargin + lp.rightMargin + widthUsed, lp.width,
                                            canScrollHorizontally());
  final int heightSpec = getChildMeasureSpec(getHeight(), getHeightMode(),
                                             getPaddingTop() + getPaddingBottom()
                                             + lp.topMargin + lp.bottomMargin + heightUsed, lp.height,
                                             canScrollVertically());
  if (shouldMeasureChild(child, widthSpec, heightSpec, lp)) {
    child.measure(widthSpec, heightSpec);
  }
}

void layoutChunk() {
 	//...
  measureChildWithMargins();
  //....
  layoutDecoratedWithMargins();
}

public void layoutDecoratedWithMargins(View child, int left, int top, int right,
                int bottom) {
  final LayoutParams lp = (LayoutParams) child.getLayoutParams();
  final Rect insets = lp.mDecorInsets;
  child.layout(left + insets.left + lp.leftMargin, top + insets.top + lp.topMargin,
               right - insets.right - lp.rightMargin,
               bottom - insets.bottom - lp.bottomMargin);
}
```

从`draw`和`onDraw`方法可以看出，`ItemDecoration`的`onDraw`方法在`onDrawOver`之前执行。

而`getItemOffset`方法设置的`Rect`值，实际上在`measureChild`时被当成`View`的`padding`

## 五、ItemAnimator

`ItemAnimator`包含下列几个需要实现的方法：

- `animateDisappearance`
- `animateAppearance`：当一个`ViewHolder`加入`RecyclerView`时
- `animatePersistence`：当一个`ViewHolder`在之前和之后都在`RecyclerView`当中，并且`Adapter.notifyItemChanged`或者`Adapter.notifyDatasetChanged`没有影响到时
- `animateChange`：`ViewHolder`在之前和之后都存在于`RecyclerView`中并且`RecyclerView`有收到`Adapter.notifyItemChanged`的调用。如果是`Adapter.notifyDatasetChanged`并且`adapter`有`stable ids`，还是会触发。如果是通过`Adapter.notifyDatasetChanged`触发，那么有可能`ViewHolder`的内容并没有改变，你需要参考`DefaultItemAnimator`来跳过这些`ViewHolder`
- `runPendingAnimations`：如果上述`animateXXX`返回false，那么就动画就会在`runPendingAnimations`中执行
- `endAnimation：`当某个`View`的动画需要立即停止时调用。比如`RecyclerView`在滑动或者获得焦点时，动画中的`View`需要马上被设置为结束状态。
- `endAnimations`：针对所有动画中的`View`
- `isRunning`：当前是否有动画正在运行

`android`自带了`DefaultItemAnimator`实现，继承自`SimpleItemAnimator`

`SimpleItemAnimator`提供了各种列表来保存增删改中的`ViewHolder`以及其位置信息`MoveInfo`、`ChangeInfo`

以`animateAppearance`为例

```java
//SimpleItemAnimator.java
public boolean animateAppearance(RecyclerView.ViewHolder viewHolder, ItemHolderInfo preLayoutInfo, ItemHolderInfo postLayoutInfo) {
  if (preLayoutInfo != null && (preLayoutInfo.left != postLayoutInfo.left
                                || preLayoutInfo.top != postLayoutInfo.top)) {
    // slide items in if before/after locations differ
    return animateMove(viewHolder, preLayoutInfo.left, preLayoutInfo.top,
                       postLayoutInfo.left, postLayoutInfo.top);
  } else {
    return animateAdd(viewHolder);
  }
}
```

分为两种情况：一种是`slide items in`，一种是`Add`，以`animateAdd`为例

```java
//DefaultItemAnimator.java
public boolean animateAdd(final ViewHolder holder) {
  //实际上是调用了endAnimation(holder)方法终止holder相关的动画
  resetAnimation(holder);
  holder.itemView.setAlpha(0);
  //将需要add的ViewHolder放到mPendingAdditions中
  mPendingAdditions.add(holder);
  //返回true意味着需要在runPendingAnimations中进行动画
  return true;
}
```

接下来看`runPendingAnimations`如何处理动画

```java
//DefaultItemAnimator.java
public void runPendingAnimations() {
  boolean removalsPending = !mPendingRemovals.isEmpty();
  boolean movesPending = !mPendingMoves.isEmpty();
  boolean changesPending = !mPendingChanges.isEmpty();
  boolean additionsPending = !mPendingAdditions.isEmpty();
  if (!removalsPending && !movesPending && !additionsPending && !changesPending) {
    // nothing to animate
    return;
  }
  // First, remove stuff
  for (RecyclerView.ViewHolder holder : mPendingRemovals) {
    animateRemoveImpl(holder);
  }
  mPendingRemovals.clear();
  //Next, move stuffs
  if(movesPending) {
    final ArrayList<MoveInfo> moves = new ArrayList<>();
    moves.addAll(mPendingMoves);
    mMoveList.add(moves);
    mPendingMoves.clear();
    Runnable mover = new Runnable {
      public void run() {
        for(MoveInfo moveInfo:moves) {
          animateMoveImpl(moveInfo.holder, moveInfo.fromX, moveInfo.fromY, moveInfo.toX, moveInfo.toY);
        }
        moves.clear();
        mMoveLists.remove(moves);
      }
    };
    if (removalsPending) {
      View view = moves.get(0).holder.itemView;
      ViewCompat.postOnAnimationDelayed(view, mover, getRemoveDuration());
    } else {
      mover.run();
    }
  }
  // Next, change stuff, to run in parallel with move animations
  if (changesPending) {
    final ArrayList<ChangeInfo> changes = new ArrayList<>();
    changes.addAll(mPendingChanges);
    mChangesList.add(changes);
    mPendingChanges.clear();
    Runnable changer = new Runnable() {
      @Override
      public void run() {
        for (ChangeInfo change : changes) {
          animateChangeImpl(change);
        }
        changes.clear();
        mChangesList.remove(changes);
      }
    };
    if (removalsPending) {
      RecyclerView.ViewHolder holder = changes.get(0).oldHolder;
      ViewCompat.postOnAnimationDelayed(holder.itemView, changer, getRemoveDuration());
    } else {
      changer.run();
    }
  }
  
  // Next, add stuff
  if (additionsPending) {
    final ArrayList<RecyclerView.ViewHolder> additions = new ArrayList<>();
    additions.addAll(mPendingAdditions);
    mAdditionsList.add(additions);
    mPendingAdditions.clear();
    Runnable adder = new Runnable() {
      @Override
      public void run() {
        for (RecyclerView.ViewHolder holder : additions) {
          animateAddImpl(holder);
        }
        additions.clear();
        mAdditionsList.remove(additions);
      }
    };
    if (removalsPending || movesPending || changesPending) {
      long removeDuration = removalsPending ? getRemoveDuration() : 0;
      long moveDuration = movesPending ? getMoveDuration() : 0;
      long changeDuration = changesPending ? getChangeDuration() : 0;
      long totalDelay = removeDuration + Math.max(moveDuration, changeDuration);
      View view = additions.get(0).itemView;
      ViewCompat.postOnAnimationDelayed(view, adder, totalDelay);
    } else {
      adder.run();
    }
  }
}
```

从`runPendingAnimations`可知，`animateAddImpl`实在`remove`、`move`和`change`之后执行的

因为是关注`add`，那么就看`animateAddImpl`如何实现

```java
//DefaultItemAnimator.java
void animateAddImpl(final RecyclerView.Holder holder) {
  final View view = holder.itemView;
  final ViewPropertyAnimator animation = view.animate();
  mAddAnimations.add(holder);
  animation.alpha(1).setDuration(xxx).setListener(new AnimatorListenerAdapter() {
    public void onAnimationStart() {
      dispatchAddStarting(holder);
    }
    public void onAnimationCancel() {
      view.setAlpha(0);
    }
    public void onAnimationEnd() {
      animation.setListener(null);
      dispatchAddFinished(holder);
      mAddAnimations.remove(holder);
      dispatchFinishedWhenDone();
    }
  }).start();
}
```

接下来看``animateDisappearance`

```java
//SimpleItemAnimator.java
public boolean animateDisappearance(holder, preLayoutInfo, postLayoutInfo) {
  int oldLeft = preLayoutInfo.left;
  int oldTop = preLayoutInfo.top;
  View disappearingItemView = viewHolder.itemView;
  int newLeft = postLayoutInfo == null ? disappearingItemView.getLeft() : postLayoutInfo.left;
  int newTop = postLayoutInfo == null ? disappearingItemView.getTop() : postLayoutInfo.top;
  if (!viewHolder.isRemoved() && (oldLeft != newLeft || oldTop != newTop)) {
    disappearingItemView.layout(newLeft, newTop,
                                newLeft + disappearingItemView.getWidth(),
                                newTop + disappearingItemView.getHeight());
    if (DEBUG) {
      Log.d(TAG, "DISAPPEARING: " + viewHolder + " with view " + disappearingItemView);
    }
    return animateMove(viewHolder, oldLeft, oldTop, newLeft, newTop);
  } else {
    if (DEBUG) {
      Log.d(TAG, "REMOVED: " + viewHolder + " with view " + disappearingItemView);
    }
    return animateRemove(viewHolder);
  }
}
```

`ViewHolder.isRemoved`受`RecyclerView.offsetPositionRecordsForRemove`影响，而`RecyclerView.offsetPositionRecordsForRemove`最终是通过`dispatchLayoutStep1()`，即在`RecyclerView.onLayout`阶段会触发

```java
//DefaultItemAnimator.java
//跟add一样处理方式，最后交给animateRemoveImpl
public boolean animateRemove(holder) {
  resetAnimation(holder);
  mPendingRemovals.add(holder);
  return true;
}
```

不相同的处理方式在`animateChange`

```java
public boolean animateChange(oldHolder, newHolder, fromX, fromY, toX, toY) {
  if(oldHolder == newHolder) {
    return animateMove(olderHolder, fromX, fromY, toX, toY);
  }
  final float prevTranslationX = oldHolder.itemView.getTranslationX();
  final float prevTranslationY = oldHolder.itemView.getTranslationY();
  final float prevAlpha = oldHolder.itemView.getAlpha();
  resetAnimation(oldHolder);
  int deltaX = (int) (toX - fromX - prevTranslationX);
  int deltaY = (int) (toY - fromY - prevTranslationY);
  // recover prev translation state after ending animation
  oldHolder.itemView.setTranslationX(prevTranslationX);
  oldHolder.itemView.setTranslationY(prevTranslationY);
  oldHolder.itemView.setAlpha(prevAlpha);
  if (newHolder != null) {
    // carry over translation values
    resetAnimation(newHolder);
    newHolder.itemView.setTranslationX(-deltaX);
    newHolder.itemView.setTranslationY(-deltaY);
    newHolder.itemView.setAlpha(0);
  }
  //为什么这里需要两个ViewHolder？
  //比如Crossfade时，就需要动画旧的ViewHolder和新的ViewHolder
  mPendingChanges.add(new ChangeInfo(oldHolder, newHolder, fromX, fromY, toX, toY));
}

void animateChangeImpl(final ChangeInfo changeInfo) {
  final RecyclerView.ViewHolder holder = changeInfo.oldHolder;
  final View view = holder == null ? null : holder.itemView;
  final RecyclerView.ViewHolder newHolder = changeInfo.newHolder;
  final View newView = newHolder != null ? newHolder.itemView : null;
  //两个View都要执行动画donghua
  if (view != null) {
    final ViewPropertyAnimator oldViewAnim = view.animate().setDuration(
      getChangeDuration());
    mChangeAnimations.add(changeInfo.oldHolder);
    oldViewAnim.translationX(changeInfo.toX - changeInfo.fromX);
    oldViewAnim.translationY(changeInfo.toY - changeInfo.fromY);
    oldViewAnim.alpha(0).setListener(new AnimatorListenerAdapter() {
      @Override
      public void onAnimationStart(Animator animator) {
        dispatchChangeStarting(changeInfo.oldHolder, true);
      }

      @Override
      public void onAnimationEnd(Animator animator) {
        oldViewAnim.setListener(null);
        view.setAlpha(1);
        view.setTranslationX(0);
        view.setTranslationY(0);
        dispatchChangeFinished(changeInfo.oldHolder, true);
        mChangeAnimations.remove(changeInfo.oldHolder);
        dispatchFinishedWhenDone();
      }
    }).start();
  }
  if (newView != null) {
    final ViewPropertyAnimator newViewAnimation = newView.animate();
    mChangeAnimations.add(changeInfo.newHolder);
    newViewAnimation.translationX(0).translationY(0).setDuration(getChangeDuration())
      .alpha(1).setListener(new AnimatorListenerAdapter() {
      @Override
      public void onAnimationStart(Animator animator) {
        dispatchChangeStarting(changeInfo.newHolder, false);
      }
      @Override
      public void onAnimationEnd(Animator animator) {
        newViewAnimation.setListener(null);
        newView.setAlpha(1);
        newView.setTranslationX(0);
        newView.setTranslationY(0);
        dispatchChangeFinished(changeInfo.newHolder, false);
        mChangeAnimations.remove(changeInfo.newHolder);
        dispatchFinishedWhenDone();
      }
    }).start();
  } 
}


```







