### 16.1 跨文档消息传递(XDM)
```JavaScript
var iframeWindow = document.getElementById("myFrame").contentWindow;
iframeWindow.postMessage("A Secret", "https://www.baidu.com");
//处理消息
window.addEventListener("message", function(event){
    if(event.origin == "https://www.origin.com") {
        processMessage(event.data);
        event.source.postMessage("Received","https://www.baidu.com");
    }
});
```
### 16.2 原生拖放
#### 16.2.1 拖放事件
dragstart, drag, dragend
#### 16.2.2 自定义放置目标
重写dragenter和dragover事件，调用EventUtil.preventDefault(event)
#### 16.2.5 可拖动
H5为所有元素规定了一个draggable属性，默认为false

### 16.3 媒体元素
audio和video两个标签
### 16.4 历史状态管理
history.pushState()方法
```javascript
//三个参数分别为状态对象、新状态的标题和可选的相对URL
history.pushState({name:"Nico"}, "Nico's page", "nico,html");
//执行完pushState后，新的状态信息会加入历史状态栈，而浏览器地址烂也会编程新的相对URL。**但是浏览器并不会真的向服务器发送请求，即使状态改变之后查询locatio.href也会返回与地址栏相同的地址**
//pushState之后后退按钮可以使用，按下后退会出发window的popstate事件。
window.addEventListener("popState", function(event) {
    processState(event.state);
})
```