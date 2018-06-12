### 8.1 window对象
表示浏览器的一个实例。既是通过JavaScript访问浏览器窗口的一个接口，又是ECMAScript规定的global对象。
#### 8.1.1 全局作用域
- 全局变量会成为window对象的属性(使用var语句添加的全局变阿玲属性有一个名为[[Configurable]]的特性为false)
- 全局变量不能通过delete操作符删除，而window对象上定义的属性可以
#### 8.1.2 窗口关系及框架
每个frame都有自己的window对象，并且保存在frames集合中
#### 8.1.3 窗口位置
```javascript
window.moveTo(0,0);
window.moveBy(0,100);//向下移动100像素
```
#### 8.1.4 窗口尺寸
```javascript
var pageWidth = window.innerWidth, pageHeight = window.innerHeight;
window.resizeTo(200,200);//调整到200x200
window.resizeBy(100,50);//调整到100x150;
```
#### 8.1.5 导航和打开窗口
window.open()方法既可以导航到一个特定的URL，也可以打开一个新的浏览器窗口。这个方法接收4个参数。
- 要加载的URL
- 窗口目标
- 特性字符串，不允许出现空格
- 标识新页面是否取代浏览器历史记录中当前加载页面的boolean
如果传递了第二个参数，而且该参数为已有窗口或frame的名称，那么就会在具有该名称的窗口或frame中加载URL，否则就会新建一个窗口并命名为该值；
```JavaScript
//在topFrame中打开或者创建一个新窗口并将其命名为topFrame
window.open("https://www.baidu.com","topFrame");
```
1. 弹出窗口
```JavaScript
var newWindow = window.open("https://www.baidu.com");
alert(newWindow.opener == window);
//如果在chrome中想在单独进程运行，则将opener设置为null即可
window.close();
```
2. 安全限制
3. 弹出窗口屏蔽程序
```JavaScript
var blocked = false;
try {
    var newWindow = window.open();
    if(newWindow == null) {
        blocked = true;
    }
}catch(ex) {
    blocked = true;
}
```
#### 8.1.6 间歇调用和超时调用
```JavaScript
//超时调用
var timeoutId = setTimeout(function() {
    alert("Hello");
}, 1000);
//JavaScript是单线程的解释器，因此一定时间内只能执行一段代码。为了控制要执行的代码，就有一个JavaScript任务队列。setTimeout的第二个参数告诉JavaScript再过多长时间把当前任务添加到队列中。如果队列为空，那么添加的代码会立即执行；不为空，需要等前面的代码执行完成后再执行
clearTimeout(timeoutId);
//间歇调用
var intervalId = setInterval(function(){
    alert(1);
},10000)
clearInterval(intervalId);
```
### 8.2 location对象
window.location == document.location
#### 8.2.2 位置操作
```JavaScript
//修改location属性等方法会在历史记录中生成一条记录, location.replace()则不会
//假设初始 URL 为 http://www.wrox.com/WileyCDA/
//将 URL 修改为"http://www.wrox.com/WileyCDA/#section1"
location.hash = "#section1";
//将 URL 修改为"http://www.wrox.com/WileyCDA/?q=javascript"
location.search = "?q=javascript";
//将 URL 修改为"http://www.yahoo.com/WileyCDA/"
location.hostname = "www.yahoo.com";
//将 URL 修改为"http://www.yahoo.com/mydir/"
location.pathname = "mydir";
//将 URL 修改为"http://www.yahoo.com:8080/WileyCDA/"
location.port = 8080;
```
### 8.3 navigator对象
#### 8.3.1 检测插件
遍历navigator.plugins
#### 8.3.2 注册处理程序

### 8.4 screen对象
### 8.5 history对象