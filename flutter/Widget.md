### Widget
> Widget是用来说明Element配置
> 一个widget是不可变的, Widget可以inflate成一个element，element是用来管理render tree。
> widget本身都是不可变的属性。如果你想要关联一个可变状态到widget，考虑使用StatefulWidget，通过StatefulWidget.createState方法关联了一个State来响应state的变化。
> 一个指定的widget可以被包含在render tree中0次或多次。每次widget被加入到render tree后，都会被inflate成一个element。
> 'key'属性控制一个widget如何替换另一widget。当两个widget的key和runtimeType都相等时，新的widget会替换就的widget，通过Element.update方法。旧的widget会被移除。

#### StatelessWidget
