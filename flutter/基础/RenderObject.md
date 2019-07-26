# RenderObject

RenderObject是render tree中的元素，是render的核心

RenderTree的根节点是RenderView

```dart
///view.dart
class RenderView extends RenderObject {
  ///RendererBinding.initRenderView中调用
  void scheduleInitialFrame() {
    scheduleInitialLayout();//给下面的node（RenderObject）中的_relayoutBoundary设置为this
  }
}
///rendering/binding.dart
mixin RendererBinding on ServicesBinding {
  //initInstances在runApp时调用
  @override 
  void initInstances() {
    super.initInstances();
    initRenderView();
  }
  
  void initRenderView() {
    assert(renderView == null);
    renderView = RenderView(configuration: createViewConfiguration(), window: window);
    renderView.scheduleInitialFrame();
  }
  
  ViewConfiguration createViewConfiguration() {
    final double devicePixelRatio = window.devicePixelRatio;
    return ViewConfiguration(
      size: window.physicalSize / devicePixelRatio,
      devicePixelRatio: devicePixelRatio,
    );
  }
}

void runApp(Widget app) {
  WidgetsFlutterBinding.ensureInitialized()
    ..attachRootWidget(app)
    ..scheduleWarmUpFrame();
}
class WidgetsFlutterBinding extends BindingBase {
  static WidgetsFlutterBinding ensureInitialized() {
    if(WidgetsBinding.instance == null) {
      //这里会触发BindingBase的构造函数
      WidgetsFlutterBinding();
    }
    return WidgetsBinding.instance;
  }
}
abstract BindingBase {
  BindingBase() {
    ///就是这里
    initInstances();
    initServiceExtensions();
    developer.postEvent('Flutter.FrameworkInitialization', <String, String>{});
    developer.Timeline.finishSync();
  }
}

```

RenderObject作为一个结点，是AbstractNode的子类。在被添加到RenderTree（PipelineOwner)时会触发attach方法

RenderObject.attach方法则会开始layout, paint

```dart
///object.dart
class PipelineOwner {
  set rootNode(AbstractNode value) {
    if(_rootNode == value) {
      return;
    }
    _rootNode.detach();
    _rootNode = value;
    _rootNode.attach();
  }
}
class RenderObject {
  bool _needsLayout = true;
  RenderObject _relayoutBoundary;//在scheduleInitialLayout时设置为this
  @override void attach(PipelineOwner owner) {
    super.attach(owner);
    if(_needsLayout && _relayoutBoundary != null) {
      _needsLayout = false;
      markNeedsLayout();
    }
    if(_needsPaint && _layer != null) {
      _needsPaint = false;
      markNeedsPaint();
    }
  }
}
```

RenderObject有一个parent，有一个用来保存child-specific数据的parentData的slot，比如child的位置

RenderObject也实现了基本的layout和paint协议

RenderObject并没有定义child模型，比如是否有节点，也没有定义坐标系或者特殊的layout协议

RenderBox子类使用了笛卡尔坐标系

##  实现一个RenderObject的子类

大多数情况下继承自RenderBox是一个更好的起点，RenderObject有些太繁琐

但是如果一个RenderObject不打算使用笛卡尔坐标系，那么就应该直接继承RenderObject，这样就可以通过实现一个新的Constraints子类，而不是使用BoxConstraints来规范layout

RenderBox可以在么有完全layout时测量出child的Size

大多数情况下，实现一个RenderBox子类和实现一个RenderObject子类差不多。最大的区别就在layout和hit testing。

## Layout

layout协议由Constraints开始

对RenderBox及其子类来说，performLayout方法是通过BoxConstraints，然后经过计算得到一个Size。

如果parent在layout方法中parentUsesSize为true，那么performLayout计算后的Size只应该被parent用到

不管是合适何处，只要是render object中会影响到layout的变化发生，都应该调用markNeedsLayout

```dart
///object.dart
class RenderObject {
  void layout(Constaints constraints, {bool parentUsesSize = false}) {
    RenderObject relayoutBoundary;
    if (!parentUsesSize || sizedByParent || constraints.isTight || parent is! RenderObject) {
      relayoutBoundary = this;
    } else {
      final RenderObject parent = this.parent;
      relayoutBoundary = parent._relayoutBoundary;
    }
    //不需要重新layout&&constraints没变&&relayoutBoundary没变，那么不不管了
    if (!_needsLayout && constraints == _constraints && relayoutBoundary == _relayoutBoundary) {
      return;
    }
    _constraints = constraints;
    _relayoutBoundary = relayoutBoundary;
    if(sizedByParent) {
      performResize();
    }
    performLayout();
    markNeedsPaint();
  }
}
```

那么Constraints是从哪来的呢？

由上可知RenderView作为根节点，控制着layout，那么

```dart
///view.dart
class RenderView {
  @override performLayout() {
    //configuration从构造方法中传入，也有通过set方法
    _size = configuration.size;
    if(child != null)
      child.layout(BoxConstraints.tight(_size));
  }
}
```

RenderView的构造方法在RendererBinding.initRenderView中调用，configuration为

```dart
///binding.dart
ViewConfiguration createViewConfiguration() {
  final double devicePixelRatio = window.devicePixelRatio;
  return ViewConfiguration(
    size: window.physicalSize / devicePixelRatio,
    devicePixelRatio: devicePixelRatio,
  );
}
```

因此最初的Constraints为BoxConstraints.tight(_size)，即屏幕尺寸

## Hit Testing

有上面可以直到RenderView是根节点，那么

```dart
///view.dart
class RenderView {
  bool hitTest(HitTestResult result, {Offset position}) {
    //child是RenderObject
    if(child != null) {
      child.hitTest(BoxHitTestResult.wrap(result), position:position);  
    }
    result.add(HitTestEntry(this));
    return true;
  }
}
///object.dart
class RenderObject {
  bool hitTest(result, {@required Offset position}) {
    if (_size.contains(position)) {
      if (hitTestChildren(result, position: position) || hitTestSelf(position)) {
        result.add(BoxHitTestEntry(this, position));
        return true;
      }
    }
    return false;
  }
}
```