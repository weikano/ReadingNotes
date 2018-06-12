### 13.1 事件流
#### 13.1.1 事件冒泡
事件开始时由最具体的元素接收，然后主机向上传播到较为不具体的节点。
#### 13.1.2 事件捕获
不太具体的节点应该更早接收到事件，而最具体的节点应该最后接收到事件。
#### 13.1.3 DOM事件流
### 13.2 事件处理程序
#### 13.2.1 HTML事件处理程序
#### 13.2.2 DOM0级事件处理程序
```javascript
var btn = document.getElementById("btn");
btn.onclick = function() {
    //todo
};
//删除事件
btn.onclick = null;
```
#### 13.2.3 DOM2级事件处理程序
addEventListener, removeEventListener。接收三个参数，最后一个boolean值如果为true，表示在捕获阶段调用事件处理程序；false表示在冒泡阶段处理程序。
```javascript
var btn = document.getElementById("btn");
btn.addEventListener("click",function(){
    //something
}, false);
```
### 13.3 事件对象
#### 13.3.1 DOM中的事件对象
load和unload事件