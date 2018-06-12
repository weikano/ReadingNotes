### 22.1 高级函数
#### 22.1.1 安全的类型检测
```JavaScript
function isArray(value) {
    return Object.prototype.toString.call(value) == "[object Array]";
}
function isFunction(value) {
    return Object.prototype.toString.call(value) == "[object Function]";
}
function isRegExp(value) {
    return Object.prototype.toString.call(value) == "[object RegExp]";
}
```
#### 22.1.2 作用域安全的构造函数
```JavaScript
function Person(name, age, job){
    this.name = name;
    this.age = age;
    this.job = job;
}
var person = new Person("Nicholas", 29, "Software Engineer");
//当没有使用new来调用构造函数时，由于this是在运行时绑定的，所以直接调用Person()，this会映射到全局对象window上
//修改为以下形式
function Person(name, age, job){
    if (this instanceof Person){
        this.name = name;
        this.age = age;
        this.job = job;
    } else {
        return new Person(name, age, job);
    }
}
```
#### 22.1.3 惰性载入函数
#### 22.1.4 函数绑定
### 22.2 防篡改对象
#### 22.2.1 不可扩展对象
```JavaScript
var person = {name:"Hi"};
person.age = 20;
Object.preventExtensions(person);
person.job = "test";
alert(person.job);//undefined
```
#### 22.2.2 密封的对象
密封对象不可扩展，而且已有成员不能删除属性和方法
```JavaScript
var person = {name :"hi"};
Object.seal(person);
person.age = 29;
alert(person.age);//undefined
delete person.name;
alert(person.name);//hi
```
#### 22.2.3 冻结的对象
不可扩展、密封，而且对象属性的[[writable]]特性为false。如果定义[[Set]]函数，访问器属性仍然是科协的。
```JavaScript
var person = { name: "Nicholas" };
Object.freeze(person);
person.age = 29;
alert(person.age); //undefined
delete person.name;
alert(person.name); //"Nicholas"
person.name = "Greg";
alert(person.name); //"Nicholas"
```
### 22.3 高级定时器
```JavaScript
setTimeout(function(){
    //处理中
    setTimeout(arguments.callee, interval);
}, interval);
```
### 22.4 自定义事件
```JavaScript
function EventTarget() {
    this.handlers = {};
}
EventTarget.prototype = {
    constructor : EventTarget,
    addHandler: function(type, handler) {
        if(typeof this.handlers[type] == "undefined") {
            this.handlers[type] = [];
        }
        this.handlers[type].push(handler);
    },
    fire: function(event) {
        if(!event.target) {
            event.target = this;
        }
        if(this.handlers[event.type] instanceof Array) {
            var handlers = this.handlers[event.type];
            for(var i=0;i<handlers.length;i++){
                handlers[i](event);
            }
        }
    },
    removeHandler:function(type, handler) {
        if(this.handlers[type] instanceof Array){
            var handlers = this.handlers[type];
            for(var i=0;i<handlers.length;i++) {
                if(handlers[i] == handler) {
                    break;
                }
            }
            handlers.splice(i,1);
        }
    }
}
```
### 22.5 拖放
**TBD**
