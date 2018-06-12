**任何没有try-catch处理的错误都会触发window.error事件**
```JavaScript
//不能使用addEventListener方法
window.onerror = function(message, url ,line) {

}
```