在ECMAScript中，引用类型是一种数据结构，用于将数据和功能组织在一起。它也常被称为类，但这种称呼并不妥当。尽管ECMAScript从技术上讲是一门面向对象的语言，但它并不具备传统的面向对象语言所支持的类和接口等机构。

引用类型有时候也被称为对象定义。

### 5.1 Object类型
创建Object实例的方式有两种：
1. 通过new操作符后跟object构造函数：
```javascript
var person = new Object();
person.name = "Roman";
person.age = 29;
```
2. 对象字面表示法
```javascript
var person = {
    name:"Roman",//属性名也可以使用字符串，比如"name":"Roman", 如果属性名用的是数值，会自动转换为字符串
    age:29
};
```

### 5.2 Array类型
```javascript
var colors = new Array();
colors = new Array(3);
colors = new Array("red","yellow","black");
colors = ["red","yellow","black"];
alert(colors[0]);//通过下标访问数组元素
alert(colors.length);//length属性获取数组长度
colors.length=4;//修改colors的长度为4，新增的元素默认为undefined
```
#### 5.2.1 检测是否数组
```javascript
Array.isArray(colors);
```
#### 5.2.3 栈方法
```javascript
var colors = new Array();
var count = colors.push("red","green");//count=2
count = colors.push("black");//count=3
var item = count.pop();//item = black
```

#### 5.2.4 队列方法
```javascript
var colors = new Array();
var count = colors.push("red","green");//count=2
count = colors.push("black");//count=3
var item = colors.shift();//取得第一项,即red
alert(colors.length);//2
```

#### 5.2.5 重排序方法
```javascript
var values = [1,2,3,4,5];
values.revserse();
alert(values);//5,4,3,2,1

values = [0,10,5,15,1];
values.sort();
alert(values);//0,1,10,15,5，sort用字符串顺序排序

function compare(prev, next) {
    if(prev>next) {
        return 1;
    }else if(prev<next) {
        return -1;
    }else{
        return 0;
    }
}
values.sort(compare);
alert(values);//0,1,5,10,15
```
#### 5.2.6 操作方法
```javascript
//concat：拼接生成一个新的数组
//slice：从原数组中的指定范围复制元素并生成一个新的数组
//splice：提供删除、插入和替换功能
//删除，splice(0,2)，删除数组中的前两项
//插入, splice(1,0, "gray","light")，在位置1插入两项
//替换, splice(1,1,"lime")，删除掉位置1，然后从位置1开始插入lime，也可以插入多想
```
#### 5.2.7 位置方法
```javascript
var numbers = [1,2,3,4,4,5];
alert(numbers.indexOf(4));//3
alert(numbers.lastIndexOf(4));//4
```
#### 5.2.8 迭代方法
- every：对每一项运行指定函数，都为true则返回true
- filter：对每一项运行给定函数，返回该函数执行后结果为true的元素组成的数组
- forEach：对每一项运行指定函数
- map：对每一项运行指定函数，返回每次函数调用的结果组成的数组
- some：如果有一项运行指定函数的结果为true，则返回true

#### 5.2.9 归并方法
- reduce：从第一项开始遍历，并把返回结果放在prev中
- reduceRight：从最后一项开始遍历，结果放在prev中
```javascript
var values = [1,2,3,4,5];
var sum = values.reduce(function(prev,cur, index, array){
    return prev+cur;
}); //sum = 15;
var sum1 = values.reduceRight(function(prev,cur, index, array) {
    return prev+cur;
});//15
```
### 5.3 Date类型
略
### 5.4 RegExp类型
```javascript
// var regx = / pattern / flags;
```
- pattern：正则表达式
- flags：g标识全局模式，即pattern被应用于所有字符串中，而非在第一个匹配项被发现时立即停止；i不区分大小写；m开启多行模式
```javascript
//匹配字符串中所有at的实例
var pattern1 = /at/g;
//匹配字符串中第一个bat或cat，不区分大小写
var pattern2 = /[bc]at/i;
//匹配字符串中所有at结尾的3个字符的组合，不区分大小写
var pattern3 = /.at/gi
```
#### 5.4.1 RegExp实例属性
- global：是否设置了g
- ignoreCase：是否设置了i
- lastIndex：开始搜索下一个匹配项的字符位置，从0算起
- multiline：是否设置了m
- source：正则的字符串表示
#### 5.4.2 RegExp实例方法
**TBD**
#### 5.4.3 RegExp构造函数属性
**TBD**

### 5.5 Function类型
#### 5.5.1 没有重载
同名函数后面定义的会覆盖签名定义的函数
```javascript
function add(num) {
    return num+100;
}
function add(num) {
    return num+200;
}
alert(add(100));//300
```
#### 5.5.2 函数声明与函数表达式

#### 5.5.5 函数属性和方法
每个函数都包含两个属性：length和prototype。
- length表示函数希望接收的命名参数的个数
- prototype保存它们所有实例方法的真正所在。比如toString()和valueOf()
- call()和apply()函数：扩充函数的作用域
```javascript
window.color = "red";
var o = {color:"blue"};
function sayColor() {
    alert(this.color);
}
sayColor();//red
o.sayColor();//blue
sayColor.call(this);//red;
sayColor.call(o);//blue
sayColor.call(window);//red
```

### 5.6 基本包装类型
Boolean、Number和String
#### 5.6.1 Boolean类型
```javascript
var falseObject = new Boolean(false);
var result = falseObject && true;
alert(result);//true，这行代码是对falseObject而不是对它的值(false)进行求值，所以是true

var falseValue = false;
result = falseValue && true;
alert(result);//false;
```
**永远不要使用Boolean**

#### 5.6.2 Number类型
```javascript
var numberObject = new Number(10);
var numberValue = 10;
alert(typeof numberObject);//object
alert(typeof numberValue);//number
alert(numberObject instanceof Number);//true
alert(numberValue instanceof Number);//false
```
#### 5.6.3 String类型
略
### 5.7 单体内置对象
由ECMAScript实现提供的、不依赖于宿主环境的对象，这些对象在ECMAScript程序执行之前就已经存在了。
#### 5.7.1 Global对象
1. URI编码方法
2. eval方法：可以看成eval("alert('hi')")，可以看成将这行代码替换成alert("hi")。严格模式下，eval外的代码无法访问eval中定义的变量
3. Global对象的属性
4. window对象：在全局作用域中声明的所有变量和函数，都成为了window对象的属性。
```javascript
//获取global对象
var global = function() {
    return this;
}
```
#### 5.7.2 Math对象