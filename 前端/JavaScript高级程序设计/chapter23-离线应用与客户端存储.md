### 23.1 离线检测
```JavaScript
//单独使用navigator.onLine不能确定网络是否连通
if(navigator.onLine) {
    //正常工作
}else{
    //离线任务
}
//对online和offline事件进行监听
EventUtil.addHandler(window, "online", function(){
    //
});
EventUtil.addHandler(window,"offline", function(){

});
```
### 23.2 应用缓存
核心是applicationCache对象
### 23.3 数据存储
#### 23.3.1 Cookie
1. 限制：Cookie在性质上是绑定在特定的域名下的。每个域的Cookie总数是有限的。
2. Cookie的构成：名称，不区分大小写；值，必须被URL编码；域；路径：对于指定域中的那个路径，应该向服务器发送Cookie；失效时间；安全标志：只有在SSL连接的时候才发送。
3. JavaScript中的Cookie：document.cookie返回当前页面(根据域、有效时间、路径和安全设置)所有的Cookie，用;分割并且经过URL编码的，所以需要使用decodeURIComponent()解码。**设置document.cookie值并不会覆盖cookie，除非设置的Cookie名称已经存在**
4. 子Cookie
5. 关于Cookie的思考：http专有Cookie可以从浏览器或者服务器设置，但只能从服务器端读取。因为JavaScript无法获取http专有Cookie的值。

### 23.3.2 IE用户数据
略
### 23.3.3 web存储机制
1. storage类型
- clear()：清楚所有值e
- getItem(nam)：获取对应的值
- key(index)：获取index位置处值的名字
- removeItem(name)：删除指定的键值对
- setItem(name, value)：指定键值对
2. sessionStorage对象：存储特定于某个会话的数据，只保持到浏览器关闭
3. globalStorage对象
```JavaScript
globalStorage["baidu.com"].name="hello";
var name = globalStorage["baidu.com"].name;
```
4. localStorage对象：要访问同一个localStorage对象，页面必须来自同一个域名(子域名无效)，使用同一种协议，在同一个端口上。
5. storage事件：对storage对象进行修改会触发document.storage事件
```JavaScript
EventUtil.addHandler(document, "storage", function(event) {
    alert("storage changed for " + event.domain);
});
```
6. 限制：大小限制

### IndexedDB
```javascript
var indexedDB = window.indexedDB || window.msIndexedDB || window.mozIndexedDB || window.webkitIndexedDB;
```
1. 数据库
```JavaScript
var request, database;
request = indexedDB.open("admin");
request.onerror = function(event) {
    //错误处理
};
request.onsuccess = function(event) {
    database = event.target.result;
};
```
2. 对象存储空间
```javascript
var user = {
    id : 0,
    name : "he",
    job : "hehe"
};
//keyPath类似主键，大多数时候都要通过keyPath来访问数据
var store = database.createObjectStore("users", {keyPath:"id});
//users中保存中一批用户对象
var i = 0, len = users.length;
while(i<len) {
    store.add(users[i++]);
}
```
3. 事务
```javascript
//不传参数，只能通过事务来读取数据库中保存的对象
var transaction = database.transaction();
//保证只加载users存储空间中的数据
transaction = database.transaction("users");
//传入多个参数
transaction = database.transaction(["users","anotherStore"]);
//要想修改访问方式，必须在创建时传入第二个参数
var type = window.IDBTransaction||window.webkitIDBTranscation;
transaction = database.transaction("users", type.READ_WRITE);
var request = transaction.objectStore("users").get("0");
request.onerror = function(){};
request.onsuccess = function(event {
    var result = event.target.result;
    alert(result.name);//he
})
``` 
4. 使用游标查询
```JavaScript
var store = db.transction("users").objectStore("users"), 
    request = store.openCursor();
request.onsuccess = function(event) {
    var cursor = event.target.result;
    if(cursor) {
        console.log("Key", cursor.key +",Value" + JSON.stringify(cursor.value));
    }
};
request.onerror = function(event) {

}
```
5. 键范围 
6. 设定游标方向
7. 索引
8. 并发问题