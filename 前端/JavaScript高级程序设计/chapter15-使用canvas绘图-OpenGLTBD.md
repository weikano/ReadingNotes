### 15.1 基本用法
要使用canvas元素，必须先设置其width和height
```JavaScript
var drawing = document.getElementById("drawing");
if(drawing.getContext){
    var context = drawing.getContext("2d");
    //取得图像的数据URI
    var imageURI = drawing.toDataURI("image/png");
    //显示图像
    var image = document.createElement("img");
    image.src = imageURI;
    document.body.appendChild(image);
}
```
### 15.2 2D上下文
#### 15.2.1 填充和描边
```JavaScript
//...省略获取context
//描边颜色
context.strokeStyle = "red";
//填充颜色
context.fillStyle = "#0000ff";
```
#### 15.2.2 绘制矩形
```javascript
//...省略获取context和设置strokeStyle, fillStyle
//在(10,10)处绘制一个长宽为50的矩形
context.fillRect(10,10,50,50);
//设置半透明
context.fillStyle = "rgba(0,0,255,0.5)";
//在(30,30)处绘制一个长宽为50的矩形叠加
context.fillRect(30,30,50,50);
```
#### 15.2.3 绘制路径
```JavaScript
var drawing = document.getElementById("drawing");
//确定浏览器支持<canvas>元素
if (drawing.getContext){
var context = drawing.getContext("2d");
//开始路径
context.beginPath();
//绘制外圆
context.arc(100, 100, 99, 0, 2 * Math.PI, false);
//绘制内圆
context.moveTo(194, 100);
context.arc(100, 100, 94, 0, 2 * Math.PI, false);
//绘制分针
context.moveTo(100, 100);
context.lineTo(100, 15);
//绘制时针
context.moveTo(100, 100);
context.lineTo(35, 100);
//描边路径
context.stroke();
}
```
#### 15.2.4 绘制文本
略
#### 15.2.5 变换
- rotate(angle)：围绕原点旋转angle弧度
- scale(scaleX, scaleY)：缩放
- translate(x, y)：将坐标原点移动到(x, y)
- transform()：矩阵变换
#### 15.2.6 绘制图像
context.drawImage()
#### 15.2.9 模式
重复的图像，用来填充或描边图形
```JavaScript
var image = document.images[0];
//需要一个img元素和一个表示如何重复的字符串
var pattern = context.createPattern(image, "repeat");
context.fillStyle = pattern;
```
#### 15.2.10 使用图像数据
略
#### 15.2.11 合成
### 15.3 WebGL
#### 15.3.1 类型化数组
#### 15.3.2 WebGL上下文
```JavaScript
var gl = drawing.getContext("experimental-webgl");
```
**TBD**