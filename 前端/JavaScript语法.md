#### 1. 基本语法
```javascript
var name;//声明变量，初始值为undefined
name = 1;//赋值，可以为null,字符串等等
//声明函数
function myFun(a, b) {
  document.write(a+" and " + b);
  return a+b;//需要返回值就return
}
var arr = new Array();//空数组
var arr1 = new Array(4);//4个元素的数组
var arr2 = new Array("hello", "world", 42, true, null);//指定数组中的每个元素，可以是不同类型的元素
```

#### 2. 比较符
```javascript
var a = 1;
var b = "1";
var c = 1;
var rs = (a==b);//true比较的是值，会把值转为字符串进行比较
rs = (a===b);//false。要比较两个变量的类型和值
rs = (a===c);//true。类型相同，值也相等。
```

#### 3. 定义对象
```javascript
var car = new Object();//对象构造函数
//给对象定义属性
car.name = "Benz";
car.seats = 4;
car.showInfo=function() {
    window.alert("car is " + name +" and has "+seats+"seats");
}
car.showInfo();
//通过对象字面表示法
var plane = {
    model:"BIG",
    seats:100
};
//访问对象属性
car.name;//圆点表示法
car["name"];//括号表示法
```

#### 4. 对象结构
```javascript
//构造函数
function Car(seats, engine, radio) {
    this.seats = seats;
    this.engine = engine;
    this.radio = radio;
}
var myCar = new Car(4,"V8","TapeDeck");
//使用prototype添加属性
Car.prototype.locks = "auto";
alert(myCar.locks);
//同样可以在prototype中新增方法，略
//遍历对象属性
for(var proname in myCar) {
    alert(proname+" has value " + myCar[proname]);
}
```

#### 5. 一些技巧
1. window.onload加载多个函数
```javascript
function addLoadEvent(func) {
    var oldonload = window.onload;
    if(typeof oldonload != 'function') {
        window.onload = func;
    }else{
        window.onload = function {
            oldonload();
            func();
        }
    }
}
```