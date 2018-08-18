#### 引用
对象通过引用来传递，它们永远不会被复制
#### 原型
- 每个对象都连接到一个原型对象，并且可以从中继承属性。所有通过对象字面量创建的对象都连接到Object.prototype
```JavaScript
if(typeof Object.create !== "function") {
    Object.create = function(o) {
        var F = function(){};
        F.prototype = o;
        return new F();
    }
}
```
- 原型连接在更新时是不起作用的。当我们对某个对象做出改变时，不会触及该对象的原型。
- 原型连接只有在检索值时才被用到
#### 删除
delete运算符不会触及原型链中的任何对象
#### 减少全局变量污染
```javascript
var MYAPP = {};
MYAPP.stooge = {
    "first-name" : "Joe",
    "last-name":"Howard"
};
```