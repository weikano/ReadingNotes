### Android Lifecycle Component
> AppcompatActivity继承自SupportActivity, SupportActivity实现了LifecycleOwener接口(getLifecycle()方法)，并提供了LifecycleRegestry作为getLifecycle()方法的返回值。

#### 一. LIfecycle Component原理
1. SupportActivity.onCreate方法中调用了ReportFragment.injectIfNeededIn(LifecyleOwner)
2. ReportFragment.injectIfNeededIn方法实际上是给SupportActivity添加了一个ReportFramgent
3. ReportFragment的作用是在其生命周期中调用dispatch方法来获取当前activity(实现了LIfecycleOwener接口)的lifecyle(即SupportFragment.LifecyleRegistry)，然后调用LifecycleRegistry.handleLifecycleEvent()->sync()，接着会根据state来调用ObserverWthState.dispatchEvent方法。ObservableWithState是通过LifecycleRegistry.addObserver()方法来添加进去，而现在系统中唯一的调用放是LiveData.observe()
4. 也就是说Activity的生命周期的变化，现在会通知到实现android.arch.lifecycle.Observer接口的并通过AppcompatActivity.getLifecyle().addObserver()添加的对象。
5. 实际上会触发ObserverWrapper.activeStateChanged()，然后根据条件调用LiveData.onInactive()或者ObserverWrapper.dispatchValue()->Observer.onChanged()(即LiveData.observer()方法传入的observer)
6. LifecycleRegistry.addObserver(LifecycleObserver observer)的使用。observer可以直接集成LifecycleObserver，然后用OnLifecycleEvent注解标识对应Event要调用的方法(Lifecycling.getCallback->ReflectiveGenericLifecycleObserver中会收集用注解标识过的方法，并在恰当的实际调用)；也可以使用它的实现类。
7. 当生命周期到Destroy的时候，LiveData会remove掉observer(具体方法在onStateChanged()中)

#### 二、LiveData原理
```java
LiveData<Product> getProduct() {xxx}
//监听LiveData的变化
getProduct().observe{
  void onChanged(T t) {
    //
  }
}
//修改LiveData的值
getProduct().setValue();//主线程
getProduct().postValue();//子线程
```
1. 修改LiveData的值
> 不管是setValue还是postValue(实际上还是调用的setValue方法)，都会触发dispatchingValue方法，然后调用considerNotify()->Observer.onChanged()
2. 何时停止监听LiveData的变化
> - 手动调用removeObserver方法
> - LiveData会在LifecycleEvent为DESTROY时自动调用removeObserver方法
> - 接着LiveData.onInactive()方法会调用
3. onInactive和onActive
> - onActive:如果之前没有活跃的observer，那么当shouldBeActive()为true时，会触发onActive()
> - onInactive：当前没有活跃的observer并且shouldBeActive()为false时才会触发

4. MutableLiveData

> `MutableLiveData`就是将`set\postValue`两个方法设置为`public`，`set\postValue`后会回调`obsever.onChanged`方法

4. MediatorLiveData

`MediatorLiveData`类似于`RxJava`中的`combineLatest`，通过`addSource(LiveData, Observer)`将多个`LiveData`的变化通过其对应的`observer`设置给`MediatorLiveData`，然后多个`LiveData`最终的结果合并交给`MediatorLiveData`的`observer`来观察

比如一个购物车需要监控两个商品的数量来计算商品总数，那么就有

```kotlin
val stock1 = MutableLiveData<Int>()
val stock2 = MutableLiveData<Int>()
val merge = MediatorLIveData<Int>()
merge.addSource(stock1,  { count ->
  merge.setValue(count + stock2.getValue())
})
merge.addSource(stock2, { count->
  merge.setValue(count + stock1.getValue())
})
merge.observe(lifecycle, {vallue ->
  label.text = "商品数量为$value"
})

//这样每次stock1、stock2有数据变化时，merge都会收到通知更新界面
```

#### 三、ViewModel原理
```java
//一般ViewModel类都是继承AndroidViewModel，并设置LiveData field
public class ProductViewModel extends AndroidViewModel {
    public LiveData<Product> getProduct() { .... }
}
//使用ViewModel
ProductViewModel model = ViewModelProviders.of(fragment or activity).get(ProductViewModel.class);
//监听ViewModel中LiveData的数据变化
model.getProduct().observe {
    show(it)
}
//更新ViewModel中的数据
model.getProduct().setValue()//主线程使用
model.getProduct().post()//其他线程使用
```
1. ViewModel是怎么创建出来的
> ViewModelProviders.of()会生成一个ViewModelProvider.AndroidViewModelFactory类，另外还会根据当前的fragment或者activity是否实现了ViewModelStore接口来创建一个ViewModelStore来创建一个ViewModel的缓存，接着由store和factory生成一个ViewModelProvider
2. ViewModelProvider是怎么通过class来生成的ViewModel的
> 通过AndroidViewModelFactory.create方法反射生成一个ViewModel类
3. ViewModel是如何与生命周期关联起来的
> ViewModelStore在FragmentActivity/v4.Fragment.onDestroy方法中会调用clear()方法，该方法会调用ViewModel.onCleared()方法

#### 四、总结
1. Lifecycle是LiveData和ViewModel的基础
2. ViewModel中使用LiveData作为数据的存储和变化
3. 不管是LiveData还是ViewModel，都会对生命周周期的变化做出反应，而observer就是回调的地方。

