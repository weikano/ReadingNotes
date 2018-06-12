### 6.1 理解对象
#### 6.1.1 属性类型
1. 数据属性
- [[Configurable]]：标识能否通过delete删除属性从而重新定义属性，能否修改属性的特性，或者能否把属性修改为访问器属性。
- [[Enumerable]]：标识能否通过for-in循环返回属性。
- [[Writable]]：表示能否修改属性的值。
- [[Value]]：包含这个属性的数据值。
```javascript
var person = {};
Object.defineProperty(person, "name", {
    writable: false,
    value : "Nico"
});
alert(person.name);//Nico
person.name = "Greg";
alert(person.name);//Nico
```
2. 访问器属性
它们包含一对getter和setter函数
```javascript
var book = {
    year:2004,
    edition:1
}
Object.defineProperty(book,"_year", {
    get: function() {
        return this.year
    }, 
    set: function(newValue) {
        if(newValue>2004) {
            this.year = newValue;
            this.edition += newValue - 2004;
        }
    }
})
book._year = 2005;
alert(book.edition);//2
```
#### 6.1.2 定义多个属性
#### 6.1.3 读取的属性的特性
```javascript
var descriptor = Object.getOwnPropertyDescriptors(book,"year");
alert(destcriptor.value);//2004;
alert(descriptor.configurable);//false 
```

### 6.2 创建对象
#### 6.2.2 构造函数模式
```javascript
function Person(name, age, job) {
    this.name = name;
    this.age = age;
    this.job = job;
    this.sayName = function() {
        console.log(this.name);
    }
}
//当做构造函数
var person1 = new Person("Hello",1,"World");
//作为普通函数，里面的this就是window对象了
Person("Greg",22,"Doctor");
//this现在是o
var o = new Object();
Person.call(o, "ken",22,"teacher");
```
1. 理解原型对象
> 只要创建了一个新函数，就会为该函数创建一个prototype属性，该属性指向函数的原型对象。默认情况下，所有原型对象都会自动获得一个constructor属性，该属性包含一个指向prototype所在函数的指针。即Person.prototype.constructor指向Person函数。**Person函数中所有属性，都是保存在Person.prototype中。当为一个对象实例添加一个属性时，这个属性会屏蔽掉prototype中的同名属性。当然可以通过delete来删除实例属性，比如delete person.name，这样就可以访问prototype中的name了。**
```javascript
person.hasProperty("name");//false，name属性在prototype中时为false，在对象实例中时为true
```
2. 原型与in操作符
- 作为in操作符使用：仅当对象实例可以访问给定属性时返回true，不管是通过prototype或者对象实例。下面的name属性可以通过person1访问，那么true。
- for-in循环：返回的是所有能够通过对象访问的、可枚举的属性。包括原型和实例中的属性。
```javascript
function Person() {}
Person.prototype.name = "1";
var person1 = new Person();
alert(person1.hasProperty("name"));//false
alert("name" in person1);//
```
3. 更简单的原型语法
```javascript
function Person() {}
Person.prototype = {
    //如果不加constructor，那么因为这样是重新定义了prototype属性，会覆盖掉本来的prototype。
    //默认的prototype.constructor指向的是Person，但是覆盖之后指向的是Object的构造函数
    constructor : Person,
    name : "hI",
    age : 29,
    sayHello : function() {
        console.log(this.name)
    }
};
```
4. 原型的动态性
![image](https://raw.githubusercontent.com/weikano/NoteResources/master/JavaScript%E9%AB%98%E7%BA%A7%E7%A8%8B%E5%BA%8F%E8%AE%BE%E8%AE%A1/01.png)
5. 原生对象的原型
可以类似kotlin一样，给引用类型添加扩展函数，比如
```javascript
String.prototype.startWith = function(text) {
    return this.indexOf(text) == 0;
}
```
6. 原型对象的问题
- 省略了为构造函数传递初始化参数这一环节，结果所有实例在默认情况下都将取得相同的属性值
- 原型中所有属性是被很多实例共享的。如果实例对象没有定义一个同名属性覆盖掉原型中的属性，那么对原型中的属性的修改会造成所有该类型的对象实例的属性变化

#### 6.2.4 组合使用构造函数和原型模式
```javascript
function Person(name, age) {
    this.name = name;
    this.age = age;
    this.friends = ["1","2"];
}
Person.prototype = {
    constructor : Person,
    sayName : function() {
        alert(this.name);
    }
}

var person1 = new Person("1",1);
var person2 = new Person("2",2);
person1.friends.push("van");//实例属性都在构造函数中定义的，共享属性为constructor和sayName方法。不会影响prototype中的friends属性
alert(person2.friends);//1,2
```
#### 6.2.5 动态原型模式
```javascript
//只在sayName方法不存在的情况下，才会将他添加到原型中
function Person(name) {
    this.name = name;
    if(typeof this.sayName != "function") {
        this.prototype.sayName = function() {
            alert(this.name);
        }
    }
}
```
#### 6.2.6 寄生构造函数模式
```javascript
function Person(name, age, job){
var o = new Object();
o.name = name;
o.age = age;
o.job = job;
o.sayName = function(){
alert(this.name);
};
return o;
}
var friend = new Person("Nicholas", 29, "Software Engineer");
friend.sayName(); //"Nicholas"
```

### 6.3 继承
#### 6.3.1 原型链
**TBD**

