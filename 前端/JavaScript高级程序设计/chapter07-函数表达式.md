### 7.1 递归
```javascript
function frac(num) {
    if(num<=1) {
        return 1;
    }else{
        return num*frac(num-1);
        // return num*arguments.callee(num-1);
    }
}

var another = frac;
frac = null;
console.log(another(5));//报错，因为frac已经不是一个function了，要想成功调用，改成callee形式
//或者使用以下形式，因为严格模式下无法从脚本访问arguments.callee
var frac = (function f(num){
    if(num<=1) {
        return 1;
    }else{
        return num*f(num-1);
        // return num*frac(num-1);
        // return num*arguments.callee(num-1);
    }
})
```

### 7.2 闭包
#### 7.2.1 闭包与变量
- 闭包只能取得包含函数中任何变量的最后一个值。
- this变量一般为其包含作用域或者外部作用域的this对象，下面的例子除外
#### 7.2.2 关于this对象
```javascript
var name = "Window";
var object = {
    this.name = "object";
    this.sayName = function() {
        return function() {
            return this.name;
        }
    }
}
alert(object.sayName());//window
```
#### 7.2.3 内存泄露
**闭包会引用包含函数的整个活动对象**
### 7.3 模仿块级作用域
- JavaScript没有块级作用域的概念。因此下面示例中的i，从它有定义开始，就可以在函数内部随处访问它。
- javasc从来不会告诉你是否多次声明同一个变量。它只会对后续的声明视而不见(会执行后续声明中的变量初始化)
```javascript
function output(count) {
    for(var i=0;i<count;i++){
        console.log(i);
    }
    console.log(i);//打印出count的值
}
```
- 用匿名函数模仿块级作用域
```javascript
(function(){
    //块级作用域
})(;
//或者
var someFunction = function() {
    //块级作用域
};
someFunction();
```
### 7.4 私有变量
任何在函数中定义的变量，都可以认为是私有变量，因为不能在函数外部访问。
#### 7.4.1 静态私有变量
略
#### 7.4.2 模块模式
即单例模式
```javascript
var application = function() {
    //私有变量
    var components = new Array();
    //初始化
    components.push(new BaseComponent());
    //字面量形式返回一个Object对象实例
    return {
        getComponentCount: function() {
            return components.length;
        },
        registerComponent: function(component) {
            if(typeof component == "object") {
                components.push(component);
            }
        }
    }
}
```
#### 7.4.3 增强的模块模式
类似上面的模块模式实现
```javascript
var singleton = function() {
    //私有变量和私有方法
    var privateValue = 10;
    function privateFuntion() {
        return false;
    }
    var object = new CustomType();
    object.publicProperty = true;
    object.publicMethod = function() {
        privateValue ++;
        return privateFuntion();
    }
    return object;
}
```
