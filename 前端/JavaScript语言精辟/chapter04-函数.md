#### 函数对象
JavaScript中的函数就是对象。函数对象连接到Function.prototype(该对象本身连接到Object.prototype)。每个对象创建时会附加两个隐藏属性：函数上下文和实现函数行为的代码。
#### 函数字面量
```JavaScript
var add = function(a, b) {
    return a+b;
}
```
#### 调用
除了声明时定义的形参，诶个函数还接收两个附加的参数：this和arguments。this在面向对象编程中非常重要，它的值取决于调用的模式。在JavaScript中一共由4中调用模式：
- 方法调用模式
- 函数调用模式
- 构造器调用模式
- apply调用模式
如果实际参数的个数与形参的个数不匹配，不会导致运行时错误。如果实参过多，超过的会被忽略；如果实参过少，缺失的值会被替换为undefined。

#### 方法调用模式
> 当一个函数被保存为对象的一个属性时，称为一个方法。当一个方法被调用时，this被绑定到该对象。方法可以使用this访问自己所属的对象。

#### 函数调用模式
当一个函数并非一个对象的属性时，就是一个函数来调用的。
```JavaScript
//this被绑定到全局对象
var sum = add(3,4);
```
这是一个语言设计上的错误。当内部函数被调用时，this应该绑定到外部函数的this变量才对。当时可以通过某些方法变通
```JavaScript
myObject.double = function() {
    var that = this;
    var helper = function() {
        that.value = add(that.value, that.value);
    }
    helper();
}
myObject.double();
console.log(myObject.value);
```
#### 构造器调用模式
```JavaScript
var Quo = function(string) {
    this.status = string;
};
Quo.prototype.getStatus = function() {
    return this.status;
}
var myQuo = new Quo("confused");
console.log(myQuo.getStatus());
```

#### Apply调用模式
```JavaScript
var array = [3,4];
var sum = add.apply(null, array);

var statusObject = {
    "status":"A-ok";
}
//statusObject没有集成自Quo.prototype，但我们可以在statusObject上调用getStatus方法
var status = Quo.prototype.getStatus.apply(statusObject);

```
#### 参数
函数可以通过arguments访问所有它被调用时传递给它的参数列表，包括那些没有被分配给函数声明时定义的形参的多余参数。arguments并不是一个真正的数组。arguments拥有一个length属性，但它没有任何数组方法。
#### 扩充类型的功能
```JavaScript
//通过给Function.prototype增加一个method方法，我们下次给对象增加方法的时候就不必键入prototype这几个字符
Function.prototype.method = function(name, func) {
    if(!this.prototype[name]) {
        this.prototype[name] = func;
    }
    return this;
}
Number.method("integer", function() {
    return Math[this<0? 'ceil' : 'floor'](this);
});
console.write((-10/3).integer());
```
#### 作用域
JavaScript不支持块级作用域。
#### 闭包
内部函数可以访问定义它们的外部函数的参数和变量(除了this和arguments)
```JavaScript
//通过定义一个返回对象字面量的匿名函数，然后调用该匿名函数返回一个对象
//因为value是function中的，对其他的程序来说是不可见的
var myObject = (function() {
    var value = 0;
    return {
        increment: function(inc) {
            value += (typeof inc === 'number' ? inc : 1);
        },
        getValue: function() {
            return value;
        }
    }
}());//()调用匿名函数
```
之前通过Quo构造器产生的对象并不是十分严谨。因为status本来就可以直接访问，何必再用一个getter方法去访问呢？
```JavaScript
//改良版
var quo = (function(status){
    return {
        getStatus : function() {
            return status;
        }
    }
}());
var myQuo = quo("amazed");
console.log(myQuo.getStatus());
```
#### 模块
模块是一个提供接口却隐藏状态和实现的函数或对象。可以使用函数或者闭包来构造模块。
```JavaScript
String.method('deentityify', function() {
    var entity = {
        quot:'"',
        lt:'<',
        gt:'>'
    };
    return function() {
        return this.replace(/&([^&;]+);/g, function(a, b) {
            var r = entity[b];
            return typeof r === 'string' ? r : a;
        })
    };
}());
```
模块模式的一般形式是：一个定义了私有变量和函数的函数；利用闭包创建可以访问私有变量和函数的特权函数；最后返回这个特权函数，或者把它们保存到一个可访问到的地方。
#### 级联
方法返回this就可以启用级联。
#### 柯里化
```JavaScript
Function.method('curry', function(){
    var slice = Array.prototype.slice,
        args = slice.apply(arguments),
        that = this;
    return function() {
        return that.apply(null, args.concat(slice.apply(arguments)));
    };
});
```
#### 记忆
```JavaScript
var fibonacci = function() {
    var memo = [0,1];
    var fib = function(n) {
        var result = memo[n];
        if(typeof result !== 'number') {
            result = fib(n-1) + fib(n-2);
            memo[n] = result;
        };
        return result;
    }
    return fib;
}();
//没看懂
var memorizer = function(memo, formula) {
    var recur = function(n) {
        var result = memo[n];
        if(typeof result !== 'number') {
            result = formula(recur, n);
            memo[n] = result;
        }
        return result;
    }
    return recur;
}();

var newfib = memorizer([0,1], function(recur, n) {
    return recur(n-1) + recur(n-2);
});
var fractorial = memorizer([1,1], function(recur, n){
    return n* recur(n-1);
});
```
