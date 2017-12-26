## Fresco使用指南（一）- 引入、快速使用及关键概念

### 1、引入Fresco

>  修改module的build.gradle文件，添加以下内容

```groovy
dependencies {
  //fresco核心
  compile "com.facebook.fresco:fresco:${version.fresco}"
  //根据需求添加以下依赖
  //API<14的机器上支持webp
  compile "com.facebook.fresco:animated-base-support:${version.fresco}"
  //支持GIF
  compile "com.facebook.fresco:animated-gif:${version.fresco}"
  //支持webp（静态+动态）
  compile "com.facebook.fresco:animated-webp:${version.fresco}"
  compile "com.facebook.fresco:webpsupport:${version.fresco}"
  //仅支持webp静态图
  compile "com.facebook.fresco:webpsupport:${version.fresco}"
}
```

### 2、开始使用

如果你仅仅是想简单下载一张网络图片，在下载完成之前，显示一张占位图，那么简单使用**SimpleDraweeView**即可

加载图片之前，必须初始化Fresco类。

```java
public class MyApplication extends Application {
  @Override
  public void onCreate() {
    super.onCreate();
    Fresco.initialize(this);
  }
}
```

接着在XML布局中使用SimpleDraweeView

```xml
<com.facebook.drawee.view.SimpleDraweeView
	xmlns:android="http://schemas.android.com/apk/res/android"
    xmlns:fresco="http://schemas.android.com/apk/res-auto"                                       
    android:id="@+id/my_image_view"
    android:layout_width="130dp"
    android:layout_height="130dp"
    fresco:placeholderImage="@drawable/my_drawable"
  />
```

加载图片

```java
Uri uri = Uri.parse("https://raw.githubusercontent.com/facebook/fresco/gh-pages/static/logo.png");
SimpleDraweeView draweeView = (SimpleDraweeView) findViewById(R.id.my_image_view);
draweeView.setImageURI(uri);
```

剩下的，Fresco会替你完成：

- 显示占位图片直至加载完成
- 下载图片
- 缓存图片
- 图片不再显示时，从内存中移除
- 等等等等

### 3、关键概念

#### Drawees

> Drawees负责图片的呈现。它由三个元素组成，类似MVP模式。

##### DraweeView

> 继承于View，负责图片的显示。
>
> 一般情况下，使用SimpleDraweeView即可。可以在XML或者Java代码中使用，通过setImageUri来加载图片。

##### DraweeHierarchy

> 用于组织和维护最终绘制和呈现的Drawable对象，相当于MVP中的M。
>
> 可以在Java代码中自定义图片的展示，具体的请参考[在Java代码中自定义显示效果](https://www.fresco-cn.org/docs/using-drawees-code.html)

##### DraweeController

> 负责和ImageLoader交互（Fresco默认为ImagePipeline，当然也可以指定为别的），通过它来实现对所要显示图片做更多的控制。
>
> **如果你需要对加载的图片进行额外的处理**，那么需要使用这个类。
>
> 用DraweeControllerBuilder来创建。创建之后不可修改。具体参见: [使用ControllerBuilder](https://www.fresco-cn.org/docs/using-controllerbuilder.html)

##### Listeners

> ControllerListener设置图片下载的监听

##### The Image Pipeline

> 负责图片的获取和管理。
>
> 在5.0以下，Image Pipeline使用pinned purgeables将bitmap数据避开Java heap，存在ashmem中。这要求图片不使用时，要显示的释放内存。
>
> SimpleDraweeView自动处理了这个释放过程，所以没有特殊情况，尽量使用SimpleDraweeView，在特殊的场合，如果有需要，也可以直接控制Image Pipeline。

### 4、支持的URI

**Fresco不支持相对路径的URI，所有的URI必须是绝对路径，并且带上URI的scheme**。

| 类型               | SCHEME                 | 示例                                       |
| ---------------- | :--------------------- | :--------------------------------------- |
| 远程图片             | HTTP://, HTTPS://      | `HttpURLConnection` 或者参考 [使用其他网络加载方案](https://www.fresco-cn.org/docs/using-other-network-layers.html) |
| 本地图片             | file://                | FileInputStream                          |
| Content Provider | content://             | ContentResolver                          |
| asset目录下的资源      | asset://               | AssetManager                             |
| res目录下的资源        | res://                 | Resources.openRawResource                |
| Uri中指定图片数据       | data:mime/type;base64, | 数据类型必须符合 [rfc2397规定](http://tools.ietf.org/html/rfc2397) (仅支持 UTF-8) |
只有图片资源才能使用在Image pipeline中，其他资源比如字符串或者XML Drawable在Image pipeline中没有意义。所以加载的资源不支持这些类型。

像ShapeDrawable这样声明在XML中的drawable可能引起困惑，因为不是图片。如果想显示drawable，可以把他设置为占位图，然后设置URI为null。