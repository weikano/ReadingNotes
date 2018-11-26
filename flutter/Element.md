### Element
#### Element 生命周期
- Widget.createElement: 初始化element
- Element.mount: 将element添加到render tree中。mount方法负责inflate所有的child widget并且在需要的时候调用attachRenderObject方法
- element现在已经是active状态，并且可能已经在屏幕上显示
- 在某些时刻，parent可能需要更改widget，比如parent因为以个新的state需要重建，通过update方法来更新
- 某些时候ancestor可能会决定移除这个element，通过调用deactivateChild方法，这时候Element.deactivate方法会触发
- 这时候element已经是inactive并且已经不在屏幕上显示了。仅仅在当前animation frame中，一个widget能保持inactive状态。在下一个animation frame中，所有inactive的element会被unmounted，调用element.unmounted方法

#### StatelessElement
StatelessElement继承自ComponentElement，在mount时会调用_firstBuild方法来触发rebuild, rebuild->performRebuild->build，StatelessElement的build方法又会触发widget.build方法

#### StatefulElement
StatefulElement类似StatelessElement, mount方法最后会调用state.build(StatefulElement)