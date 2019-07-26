# Provider分析

InheritedWidget.updateShouldNotify->InheritedElement.update->ProxyElement.updated->notifyClients->InheritedElement.notifyClients->notifyDependent->Element.didChangeDependencies->markNeedBuild->scheduleBuildFor->BuildOwner->buildScope->Element.rebuild->performRebuild->ComponentElement.performRebuild->build->StatelessElement/StatefulElement.build->Widget/State.build



ChangeNotifier.notifyListeners

_ListenableDelegateMixin.initDelegate->startListening->listenable.addListener(Listenable继承自ChangeNotifier), 现在的listener为listener=()=>setState(()=>buildCount++), 即每次notifyListener后都会出发setState方法.  StateDelegate.\_setState是在\_DelegateWidgetState(继承自State)中的\_mountDelegate中将\_setState赋值为State的setState方法. State的setState方法会触发Element.markNeedsBuild, 使widget重新build

StatefulWidget->DelegateWidget->ValueDelegateWidget->

## 1. 使用说明

### 暴露一个值

最简单的方法就是将整个Application包装到一个Provider中,然后将值传递给Provider的value

```dart
Provider<String>.value(
  value:"Hello World",
  child:MaterialApp(
    home:MyHome();//在MyHome中通过Provider.of<String>(context)可以获取到对应的值
  ),
);
```

对复杂的对象,比如需要通过查找一些本地数据之类的

```dart
Provider<MyComplexClass>(
	builder:(context)=>MyComplexClass.from(context),
  dispose:(context, value)=>value.dispose(),
  child:SomeWidget(),//SomeWidget中通过Provider.of<MyComplexClass>(context)获取值
);
```

## 2.读取一个值

最简单的方式是通过下面的代码

```dart
Provider.of<ProviderType>(context);
```

但是上述方法需要用到BuildContext. 如果在某些场景下很难获取到BuildContext, 可以只用Consumer来做到

```dart
Provier<String>.value(
	value:"Hello world",
  child:Comsumer<String>(
  	builder:(context, value, child)=>Text(value),//value即为暴露的String值zhi
  ),
);
```

Provider也可以将不同类型的值通过不同的provider

```dart
Provier<String>(
	value:"Hello World",
  child:Provier<Int>(
  	value:2,
    child:SomeWidget(),
  ),
);
```

## 3.MultiProvider

上述读取时有提到可以组合多个Provider, 但如果是一个比较大的项目,可能就会存在多重嵌套,所以提供了MultiProvider

```dart
MultiProvider(
	providers:[
    Provider<Foo>.value(value:foo),
    Provider<Bar>.value(value:bar),
    Provider<Barz>.value(value:baz),
  ],
  child:someWidget,
);
```

## 4. 已有的Provider

| 类名                    | 说明                                                         |
| ----------------------- | ------------------------------------------------------------ |
| Provider                | 最基础的Provider,接受任意值                                  |
| ListenableProvider      | 针对Listenable对象的Provider                                 |
| ChangeNotifierProvider  | ListenableProvider的一个特殊类型, 接收ChangeNotifier类参数. 会自动调用ChangeNotifier.dispose方法 |
| ValueListenableProvider | 监听ValueListenable并且只暴露ValueListenable.value           |
| StreamProvider          | 监听Stream并只暴露最新emit的值                               |
| FutureProvider          | 接收一个Future类型的参数并且在future完成时更新dependents     |

## 5. 原理分析

### 5.1 ValueStateDelegate

> 由StateDelegate名字可以猜测该类一定是跟State有关,而State最重要的即是生命周期
>
> StateSetter 其实是一个方法,里面是一个VoidCallback. 真正的实现实际是State.setState方法, 即StateSetter为setState((){VoidCallback()}), 在setState中调用VoidCallback回调
>
> 那么StateDelegate是在哪用到的呢? DelegateWidget
>
> 在delegate_widget.dart中可以看到DelegateWidget有一个delegate参数, 而DelegateWidget又是一个StatefulWidget, 因此其实真实作用地点在_DelegateWidgetState中
>
> 因此对于Provider()方法中的builder, 正是在State.initState中触发,然后将返回值赋值给delegate.value, 而context和setState也是在此赋值
>
> **而DelegateWidget正是Provider的父类**

```dart
abstract class StateDelegate {
  BuildContext _context;
  /// The location in the tree where this widget builds.
  ///
  /// See also [State.context].
  BuildContext get context => _context;
  StateSetter _setState;
  /// Notify the framework that the internal state of this object has changed.
  ///
  /// See the discussion on [State.setState] for more information.
  @protected
  StateSetter get setState => _setState;

  /// Called on [State.initState] or after [DelegateWidget] is rebuilt
  /// with a [StateDelegate] of a different [runtimeType].
  @protected
  @mustCallSuper
  void initDelegate() {}

  /// Called whenever [State.didUpdateWidget] is called
  ///
  /// It is guaranteed for [old] to have the same [runtimeType] as `this`.
  @protected
  @mustCallSuper
  void didUpdateDelegate(covariant StateDelegate old) {}

  /// Called when [DelegateWidget] is unmounted or if it is rebuilt
  /// with a [StateDelegate] of a different [runtimeType].
  @protected
  @mustCallSuper
  void dispose() {}
}
abstract class ValueStateDelegate<T> extends StateDelegate {
  T get value;
}
```

### 5.2 各类Provider分析

#### 5.2.1 Provider

Provider提供了两种构造方法, 其属性如下

- updateShouldNotify:一个接收两个T参数的方法, 用于决定Element.didChangeDependencies的返回值.如果为null,那么就会根据当前Provider.value和didChangeDependencies中传入的Provider的value做比较
- _value: 用于保存传入的值 
- delegate: ValueStateDelegate, 用于获取value值. Provider中的SingleValueDelegate中的get方法直接返回value. **而BuilderStateDelegate则稍微复杂一些,需要在某个时机通过builder方法获取到value值**

```dart
//provider.dart
//ValueDelegateWidget<-DelegateWidget<-StatefulWidget
//所以需要实现Widget build(BuildContext)方法
class Provider extends ValueDelegateWidget<T> {
  Provider<T>({
    @required ValueBuilder<T> builder,
    Dispose<T> dispose,
    Widget child,
  }):this._(
    key:key,
    delegate:BuilderStateDelegate<T>(builder, dispose:dispose),
    updateShouldNotify:null,
    child:child,
  );
  Provider<T>.value(
    @required T value,
    Widget child,
    UpdateShouldNotify updateShouldNotify,
  ):this._(
    key:key,
    delegate:SingleValueDelegate<T>(value),
    updateShouldNotify:updateShouldNotify,
    child:child,
  );

  Provider._({
    @required ValueStateDelegate<T> delegate,
    this.updateShouldNotify,
    this.child,
  }):super(key:key, delegate:delegate);
  
  @override Widget build(BuildContext context) {
    ///InheritedProvider<-InheritedWidget
    return InheritedProvider<T>(
      value:delegate.value, 
      updateShouldNotify:updateShouldNotify,
      child:child,
    );
  }
}
///provider.dart
///InheritedWidget中最重要的方法是updateShouldNotify
///用于通知framework中其他的widget是否需要rebuild
class InheritedProvider<T> extends InheritedWidget {
  ///在InheritedElement.update最后一直回溯到Element.didChangeDependencies,然后根据返回值来决定是否需要markNeedBuild.如果需要就丢进dirtyElements中,在scheduleBuildFor后一致回溯到Element.rebuild,最后走到StatelessElement/StatefulElement.build->Widget/State.build
  @override bool updateShouldNotify(InheritedProvider<T> oldWidget) {
    if(_updateShouldNotify != null) {
      return _updateShouldNotify(oldWidget._value, _value);
    }
    return oldWidget._value != _value;
  } 
}
```

#### 5.2.2 ListenableProvider

暂时没想到什么情况下会直接使用ListenableProvider

ListenableProvider跟Provider用法极为相似, 原理也是一样, 唯一的区别在于T和delegate不同以及没有提供UpdateShouldNotify参数(由自己实现)

- T必须是Listenable的子类, 而Listenable实际上提供了addListener和removeListener两种方法,类似观察者模式
- delegate为ListenableDelegateMixin, 其关键在于initDelegate时会调用startListening来出发addListener

```dart
mixin _ListenableDelegateMixin<T extends Listenable> on ValueStateDelegate<T> {
  VoidCallback _removeListener;
  UpdateShouldNotify<T> updateShouldNotify;
  @override initDelegate() {
    super.initDelegate();
    if(value != null) startListening(value);
  }
  
  void startListening(T listenable) {
    var buildCount = 0;
    final setState = this.setState;
    final listener = ()=>setState(()=>buildCount++);
    var capturedBuildCount = buildCount;
    //UpdateShouldNotify只关注buildCount的值
    //listener会触发setState然后修改buildCount
    updateShouldNotify=(_,_) {
      fianl res = buildCount != capturedBuildCount;
      capturedBuildCount = buildCount;
      return res;
    };
    listenable.addListener(listener);
    _removeListener = (){
      listenable.removeListener(listener);
      _removeListener = null;
      updateShouldNotify = null;
    }
  }
}
```

#### 5.2.3 ChangeNotifierProvider

ChangeNotifierProvider是ListenableProvider的子类，T为ChangeNotifier的子类，最关键的点在于ChangeNotifier

```dart
class ChangeNotifier implements Listenable {
  ObserverList<VoidCallback> _listeners = ObserverList<VoidCallback>();//用来保存Listener
  //通知listener
  void notifyListeners() {
    final List<VoidCallback> localListeners = List<VoidCallback>.from(_listeners);
      for (VoidCallback listener in localListeners) {
        if (_listeners.contains(listener))
          listener();
      }  
  }
}
```

由上面ListenableProvider可知，每次调用notifyListeners都会触发setState方法更新buildCount

而ChangeNotifierProvider的基本用法如下

```dart
class User extends ChangeNotifier{
  String name;
  int age;
  void updateName(String name) {
    this.name = name;
    notifyListeners();
  }
  void updateAge(int age) {
    this.age = age;
    notifyListeners();
  }
}
//或者使用ValueBuilder的构造函数，也一样
ChangeNotifierProvider<User>.value(
	value : User(name:"Haha", age:20),
  child: SomeWidget(),//在SomeWidget中，可以通过Provider.of<User>(context).updateName("hehe")的方法更新User值，然后触发界面刷新
);
```

#### 5.2.4 ValueListenableProvider

ValueListenableProvider跟ChangeNotifier一样通过notifyListeners来刷新界面，但是又有不同

- value构造函数需要一个ValueListenable<T>类型的value参数
- 另外一个构造方法需要一个返回值为ValueNotifier<T>的builder参数

```dart
class ValueNotifier<T> extends ChangeNotifier implements ValueListenable<T> {
  ValueNotifier(this._value);
  set value(T newValue) {
    if(_value == newValue){
      return;
    }
    _value = newValue;
    //不需要手动触发notifyListeners，ValueListenableProvider会自动帮你调用
    notifyListeners();
  }
}

class ValueListenableProvider<T> extends ValueDelegateWidget<ValueListenable<T>> {
  Widget build(BuildContext) {
    //ValueListenableBuilder中会在initState将_valueChanged方法作为listener
    //实际上就是setState((){value = newValue})
    return ValueListenableBuilder<T>(
      valueListenable: delegate.value,
      builder: (_, value, child) {
        return InheritedProvider<T>(
          value: value,
          updateShouldNotify: updateShouldNotify,
          child: child,
        );
      },
      child: child,
    );
  }
} 
```

即传入的T需要通过ValueNotifier封装起来，类似TextEditingController的实现

```dart
class User1{
  String name;
  int age;
  User1(this.name, this.age);
}
class UserValueNotifier extends ValueNotifier<User1> {
  UserValueNotifier(User1 user):super(user);
}
void test(context) {
  var uvalue = UserValueNotifier(User1(name:"hehe", age:20));
	ValueListenableProvider<User1>.value(
  value:uvalue,
  widget:SomeWidget(),//通过Provider.of<User1>(context)获取值, 也可以通过修改uvalue.value=User1(name:"haha",age:21)来刷新界面
);  
}

```

#### 5.2.5 StreamProvider和FutureProvider

通过StreamBuilder和FutureBuilder来包装一个InheritedProvider

### 6. Consumer

Consumer中关键的因素在于builder。由源码可知，Consumer仅仅是封装了一层，然后在内部通过Provider.of方法将需要expose的值暴露给builder函数

```dart
class Consumer<T> extends StatelessWidget {
  Consumer({
    Key key,
    @required this.builder,
    this.child,
  }):super(key:key);
  final Widget Function(BuildContext context, T value, Widget child) builder;
  
  Widget build(BuildContext context) {
    return builder(
    	context,
      Provider.of<T>(context),
      child,
    );
  }
}
```

