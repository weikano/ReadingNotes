# Animator和Transition

## 一、Animatior

`Animator`是3.0后动画实现的基础，一般使用的是子类`ValueAnimator`、`ObjectAnimator`和`AnimatorSet`

### ValueAnimator

`ValueAnimator`提供了简单的时序引擎来计算动画中的值并且将它们设置给目标，以此来运行动画

`ValueAnimator`使用`AnimationHandler` + `Choreograhper`作为时序引擎

```java
//AnimationHandler.java
private final Choreograher.FrameCallback mFrameCallback = new Choreograhper.Cllback() {
  public void doFrame(long frameTimeNanos) {
    //ValueAnimator实现了AnimationHandler.AnimationFrameCallback接口
    //doAnimationFrame会调用ValueAnimator.doAnimationFrame
    doAnimationFrame(getProvider().getFrameTime());
    if(mAnimationCallbacks.size() >0) {
      //getProvider返回MyFrameCallbackProvider，持有Choreographer.getInstance()
      getProvider().postFrameCallback(this);
    }
  }
};
//ValueAnimator.java
public final boolean doAnimationFrame(long frameTime) {
  //各种情况判断，比如pause、resume之类，省略
  boolean finished = animateBasedOnTime(currentTIme);
  if(finished) {
    endAnimation();
  }
  return finished;
}
//关键是调用了animateValue
boolean animateBasedOnTime(long currentTime) {
  if(mRunning) {
    //省略前面的判断
    //这里的关键在于调用了AnimatorUpdateListener.onAnimationUpdate(ValueAnimator)
    animateValue(currentIterationFraction);
  }
  return done;
}
```

由上述代码可知，`ValueAnimator`使用`Choreographer`作为时序引擎来计算每个frame对应的属性值，然后通过`AnimatorUpdateListener`的`onAnimationUpdate`方法回调给调用对象。

具体使用如下：

```kotlin
//控制alpha值变化
val view = findViewById(R.id.target)
val animator = ValueAnimator.ofInt(0,1) //由0到1之间变化
animator.addUpdateListener(object : UpdateListener() {
  fun onAnimationUpdate(animation:ValueAnimator) {
    view.alpha = animation.getAnimatedValue().toInt()
  }
})
```

### ObjectAnimator

`ObjectAnimator`为`ValueAnimator`的子类，它是通过`ObjectAnimator.ofXXX(target, propertyName, values)`方法来生成。由`ValueAnimator`可知，`animateValue`是回调的关键方法

```java
//ObjectAnimator.java
void animateValue(float fraction) {
  final Object target = getTarget();
  //调用ValueAnimator.animateValue来修改对应的值以及进行回调
  super.animateValue(fraction);
  //接下来就是给target设置属性
  int numValues = mValues.length;
  for(int i=0;i<numValues;++i) {
    //PropertyValueHolder.setAnimatedValue
    mValues[i].setAnimatedValue(target);
  }
}
```

`PropertyValuesHolder`用于保存`target`和`propertyName`之间的对应关系，通过调用`setupSetterAndGetter`方法，找到`target`对应`propertyName`的`set/get`方法。接下来就通过`setter`方法设置target对应的属性

示例如下：

```kotlin
val view = xxx
//view必须有对应的setAlpha和getAlpha方法
ObjectAnimator.ofInt(view, "alpha", 0, 1).start()
```

### AnimatorSet

`AnimatorSet`是用于规定多个`Animator`的播放顺序，比如`playTogether(...)`和`playSequentially`

## 二、Transition

`Transtion`保存了在`Scene`变化引起的动画中，它的`target`的一些信息。任何`Transtion`有两个必须的职责：获取对应的属性值(`captureStart\EndValues`)、根据属性值的变化进行动画(`createAnimator`)，以及返回`propertyName`，即当前`Transition`关心的属性变化有哪些（`getTransitionProperties`）。

一个自定义的`Transtion`必须要决定它对`View`的哪些属性感兴趣，并且要决定怎么使用动画来改变这些属性值

比如`Fade`，作为`Visibility.java`的子类，大部分工作交给了`Visibility`来做

```java
//Fade.java
//只对view.getTranstionAlpha感兴趣
public void captureValues(TransitionValues values) {
  super.captureValues(values);
  values.values.put(PROPNAME_TRANSITION_ALPHA, values.view.getTransitionAlpha());
}
```

`Transition`在`SurfaceView`和`TextureView`上并不能完美的生效。对于`SurfaceView`，问题在于它是通过`non-UI`线程更新界面。所以`moving`或者`resize`可能不能同步的展示；对于`TextureView`来说，适配性比`SurfaceView`更好，但是比如`Fade`之类的可能就不太妙，因为他们基于`ViewOverlay`来工作，而`TextureView`当前并不能跟`ViewOverlay`来很好的配合

那么`Transition`又是怎样跟`Scene`一起工作呢？以`TranstionManager.go()`方法为例

```java
//TransitionManager.java
public static void go(Scene scene, Transition transition) {
  changeScene(scene, transition);
}
private static void changeScene(Scene scene, Transition transition) {
  //其实就是把sceneRoot上之前的view给remove掉，然后换上新的
  scene.enter();
  //
  sceneChangeRunTransition(sceneRoot, transition.clone());
}

private static void sceneChangeRunTransition(final ViewGroup sceneRoot, final Transition transition) {
  //接下来的一些都在MultiListener的接口实现中
  MultiLIstener listener = new MultiListener(transition, sceneRoot);
  sceneRoot.addOnAttachStateChangeListener(listener);
  sceneRoot.getViewTreeObserver().addOnPreDrawListener(listener);
}

//MultiListener实现
private static class MultiListener implements ViewTreeObserver.OnPreDrawListener, View.OnAttachStateChangeListener {
  public boolean onPreDraw() {
    mTransition.captureValues(mSceneRoot, false);
    mTransition.playTransition(mSceneRoot);
  }
}
```

接下来的活交给`Transition.playTransition`

```java
//Transition.java
void playTransition(ViewGroup sceneRoot) {
  //前面是设置AnimationInfo等信息，忽略
  //首先调用createAnimator方法
  createAnimators(sceneRoot, mStartValues, mEndValues, mStartValuesList, mEndValuesList);
  //运行所有animator
  runAnimators();
}
```

## 三、Transition的使用场景

### 3.1 Activity之间的`SharedElement`

```kotlin
//比如从ActivityA跳转到B，并且想把A中的ImageView和TextView做为sharedelement
private fun openB() {
  val intent = Intent(this@A, B::java.class)
  val options = ActivityOptionsCompat.makeSceneTransitionAnimation(this@A, Pair<View, String>(icon,"icon"), Pair<View, String>(titleView, "title"))
  ActivityCompat.startActivity(this@A, intent, options.toBundle())
}
//接下来在B里面
public fun onCreate() {
  val icon = findViewById(R.id.xxx)
  val titleView = findViewById(R.id.xxx1)
  //跟ActivityOptionsCompat中的String要对应
  ViewCompat.setTransitionName(icon,"icon")
  ViewCompat.setTransitionName(titleView, "title")
  //接下来加载对应的数据
  loadItem()
}
private void loadItem() {
  titleView.setText(xxx)
  //这里有个小技巧就是，如果能够运行Transition，那么首先加载缩略图，等transition运行完之后再加载真正的图
  if(Build.VERSION.SDK_INT>=Build.VERSION_CODES.LOLLIPOP && addTransitionListener()) {
    loadFullImage()
  }else {
    loadThumbnail()
  }
}
private fun addTransitionListener():Boolean {
  val transition = window.getSharedElementEnterTransition()
  if(transition != null) {
    transition.addListener(object: Transition.TransitionListener() {
      onTransitionEnd() {
        loadFullImage();
        transition.removeListener(this)
      }
      onTransitionCancel() {
        transition.removeListener(this)
      }
    })
    return true
  }
  return false
}
```

### 3.2 Layout中的Transition-1

如果想要在同一个布局中，移动相同的`View`到不同的位置，应该使用`Scene`

比如在布局中有一个`FrameLayout`，id为`root`。它包含一个`layout`，`id`为`scene1`，其中有ABC三个`View`分别位于左上、右上、右下。另外一个`id`为`scene2`的`layout`呢，也同样包含ABC三个`View`，其`id`要跟`scene1`中一一对应。`scene1`和`scene2`两个的ABC都是放在一个`id`为`container`的`RelativeLayout`中

假设当前`Activity/Fragment`的布局如下

```xml
<!-- 当前界面只展示scene1，打算从scene1变成scene2 -->
<LinearLayout>
  <Button id="@+id/btn1" text="展示scene1" />
  <Button id="@+id/btn2" text="展示scene2" />
  <FrameLayout id="@+id/root">
    <include layout="@layout/scene1" />
  </FrameLayout>
</LinearLayout>
```

接下来看代码实现

```kotlin
//这个是作为Scene的sceneRoot
val root = findViewById(R.id.root)
val scene1 = Scene(root, findViewById(R.id.container) )//现在找到是对应的scene1的container
val scene2 = Scene.getSceneForLayout(root, R.layout.scene2, activity)
//接下来是变换事件
//展示scene1
private void onClickScene1() {
  //这样就会根据layout中的id记住对应信息，然后TransitionManager就会使用默认的Transition来展示动画
  TransitionManager.go(scene1)
}
private void onClickScene2() {
  TransitionManager.go(scene2)
}
```

### 3.3 Layout中的Transition-2

如果只是在布局中更改View大小或其他属性，那么要怎么做？

```kotlin
//只要使用beginDelay即可
TransitionManager.beginDelay(root)
val square = root.findViewById(R.id.a)
val params = square.params
params.width = 100dp
params.height = 100dp
square.params = params
```

### 3.4 Fragment之间的Transition

比如说要从一个`GridFragment`点击图片跳转到一个`ImagePagerFragement`，里面是`ViewPager`

```kotlin
public void onItemClick(view:View, position:Int) {
  //需要记录点击的position，因为back返回还需要滑动到对应的位置显示，这样才能有captureEndValues
  MainActivity.currentPosition = position
  //这句话的作用是点击之后让被点击的view立马消失，而不是需要经过动画，这样效果会好一点，可以不加试试效果
  (fragment.exitTranstion as TransitionSet).excludeTarget(view, true)
  val transitionView = view.findViewById(R.id.card_image)
  fragment.fragmentManager.beginTransaction()
  	.setReorderingAllowed(true) //对share element 进行优化
  	.addSharedElement(transitionView, transitionView.transitionName)
    .replace(R.id.fragment_container, ImagePagerFragment(), ImagePagerFragment::class.java.simpleName)
   .addToBackStack(true).commit()
}
```

而在`GridFragment`中，我们要设置`exitTransition`

```kotlin
//GridFragment.kt
fun onCreateView():View {
  prepareTransitions()
  postponeEnterTransition()
  //忽略其他
}
private fun prepareTransitions() {
  //对id为card_view的view使用fade
  exitTranstion(TransitionInflater.from(context).inflateTranstion(R.transition.grid_exit_transition))
  exitSharedElementCallback((_, sharedElements)-> {
    val viewHolder = recyclerView.findViewHolderForAdapterPosition(MainActivity.currentPosition)
    if(viewHolder == null || viewHolder.itemView == null) {
      return
    }
    //这里的含义是将item中的图片跟viewpager中的fragment的图片作为shared element，所以在
    //viewpager中的ImageAdapterFragment，还需要设置一下对应的sharedElement
    sharedElements.put(names[0], vewHolder.itemView.findViewById(R.id.card_image))
  })
}

fun onViewCreated() {
  //根据MainActivity中的currentPosition，判断对应的View是否显示完全。如果没有完全显示，那么滚动到对应位置
  scrollToPosition()
}

private fun scrollToPosition() {
	recyclerView.addOnLayoutChangeListener(object:OnLayoutChangeListener() {
    fun onLayoutChange() {
      recyclerView.removeOnLayoutChangeListener(this)
      val manager = recyclerView.getLayoutManager()
      val viewAtPosition = manager.findViewByPosition(MainActivity.currentPosition)
      if(viewAtPosition == null || manager.isPartiallyVisible(viewAtPosition, false, true)) {
        recyclerView.post(()->recyclerView.scrollToPosition(MainActivity.currentPosition))
      }
    }
  })
}
```

接下来设置`SharedElement`。首先是`Grid`中`item`对应的`Image`和`ViewPager`中每个`Fragment`的`Image`

```kotlin
//ImagePagerFragment.kt
fun onCreateView() {
  //设置ViewPager
  prepareSharedElementTransition()
  if(savedInstance == null) {
    postponeEnterTransition()
  }
}
private fun prepareSharedElementTransition() {
  Transition transition = TransitionInflater.from(getContext())
  .inflateTransition(R.transition.image_shared_element_transition);
  setSharedElementEnterTransition(transition);
  setEnterSharedElementCallback(
    new SharedElementCallback() {
      @Override
      public void onMapSharedElements(List<String> names, Map<String, View> sharedElements) {
        // Locate the image view at the primary fragment (the ImageFragment that is currently
        // visible). To locate the fragment, call instantiateItem with the selection position.
        // At this stage, the method will simply return the fragment at the position and will
        // not create a new one.
        Fragment currentFragment = (Fragment) viewPager.getAdapter()
        .instantiateItem(viewPager, MainActivity.currentPosition);
        View view = currentFragment.getView();
        if (view == null) {
          return;
        }
        // Map the first shared element name to the child ImageView.
        sharedElements.put(names.get(0), view.findViewById(R.id.image));
      }
    });
}
```

因为在`GridFragment`中我们设置了`postponeEnterTransition`，所以如果不调用`startPostponeEnterTransisition`，那么所有变动都不会生效

```kotlin
//ImageFragment.kt，即ViewPager中对应的Fragment
fun onCreateView() {
  Glide.with(this).load(uri).listener(
    onLoadFailed() {
      parentFragment.startPostponeEnterTransition()
    }
    onLoadComplete() {
      parentFragment.startPostponeEnterTransition()
    }
  ).into(image)
}
```

