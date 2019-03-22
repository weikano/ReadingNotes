### Glide

#### 一、API

```kotlin
//加载图片
Glide.with(context).load(url).into(imageView)
//取消加载。不是必须的，fragment或activity中加载的图片，在销毁时会自动取消加载并且回收
Glide.with(context).clear(imageView)
```

#### 二、在ListView和RecyclerView中的使用

```kotlin
//对于任何可复用的 View 或 Target ，如果它们在之前的位置上，用 Glide 进行过加载操作，那么在新的位置上要去执行一个新的加载操作，或调用 clear() API 停止 Glide 的工作
fun onBindViewHolder(holder:ViewHolder, position:Int) {
  if(isImagePosition(position)) {
    String url = urls.get(position);
    Glide.with(fragment).load(url).into(holder.imageView)
  }else {
    Glide.with(fragment).clear(holder.imageView)
    holder.imageView.setImageDrawable(specialDrawable)
  }
}
```

#### 三、尺寸

Glide通过ViewTarget.getSize方法获取尺寸。逻辑如下：

1. View的布局尺寸（paramSize）>0并且paramSize比padding大，则使用paramSize - padding
2. View的尺寸（viewSize）>0且viewSize-padding>0，则使用viewSize - padding
3. 如果view的布局尺寸为WRAP_CONTENT并且至少已经发生过一次layout，则建议使用Target.SIZE_ORIGINAL或者通过override指定固定尺寸，并使用屏幕尺寸作为该请求尺寸
4. 其他情况下（MATCH_PARENT, 0 或者 WRAP_CONTENT且没有layout过），则等待布局完成，然后回溯步骤1

有时候在RecyclerView中，view可能被复用且保持前一个位置的尺寸，但在当前位置发生了改变。可以如下处理

```kotlin
override fun onBindViewHolder(holder:VH, position:Int) {
  Glide.with(fragment).load(url).into(DrawableImageViewTarget(holder.imageView, /*waitForLayout*/ true))
}
```

#### 四、注册组件

1. ModelLoader：用于加载自定义的Model（URL，URI或者任意的POJO）和data（InputStream，FileDiscriptor）
2. ResourceDecoder：用于对新的Resources（Drawable、Bitmap）或新的Data类型（InputStream, FileDiscriptor）进行解码
3. Encoder：用于向Glide的磁盘缓存写data
4. ResourceTranscoder：用于不同资源类型转换，比如BitmapResource转换为DrawableResource
5. ResourceEncoder：用于向磁盘缓存写resources（BitmapResource, DrawableResource）。

组件通过Registy类注册，如下：

```kotlin
@GlideModule
class YourAppGlideModule: AppGlideModule() {
  override fun register(context:Context, registry:Registry) {
    registry.append(Photo.class, InputStream.class, CustomeModelLoader.Factory())
  }
}
```

#### 五、剖析一个请求

一个请求粗略的由以下步骤组成

1. Model→Data，由ModelLoader处理
2. Data→Resource，由ResourceDecoder处理
3. Resource→TranscodedResource，由ResourceTranscoder处理

Encoder可以在步骤2之前往Glide的磁盘缓存写入数据，ResourceEncoder可以在步骤3之前往Glide磁盘缓存写入资源

#### 六、排序组件

##### prepend()

假如自定义的ModelLoader或者ResourceDecoder在某些地方失效了，你想将已有数据交由Glide的默认行为处理，可以使用prepend()。prepend确保你的ModelLoader先于之前注册的其他组件首先被执行。如果你的ModelLoader失败了，其他的ModelLoader可以按照注册顺序、一次一个执行，作为fallback方案

##### append()

append跟prepend相反，是在之前注册的其他组件失效后，交由append方法中的ModelLoader处理

##### replace()

完全替换Glide的默认行为。在替换网络逻辑时有用，比如确保仅OkHttp被调用

#### 七、变换

Transformation可以获取资源并修改它，然后返回被修改的资源。一般是用来对bitmap的裁剪或者应用过滤器，也可以用来转换GIF图片或者自定义的资源类型

##### 内置类型

包括CenterCrop、FitCenter、CircleCrop

```kotlin
val otpions = RequestionOptions().apply {
  centerCrop()
}
Glide.with(fragment).load(url).apply(options).into(imageView)
```

##### 多重变换

默认情况下，每个transform调用都会替换掉之前的变换。如果想在单次加载中使用多个变换，使用MultiTransformation类。注意，MultiTransformation构造函数中的顺序决定了应用顺序

```kotlin
Glide.with(fragment).load(url).apply(MultiTransformation(FitCenter(), YourCustomTransformation())).into(imageView)
```

##### 定制变换

如果只需要变换bitmap，最好继承BitmapTransformation。

```kotlin
class FillSplace : BitmapTransformation() {
  companion object {
    final ID = "com.bumptech.glide.transformation.FillSpace"
    final ID_BYTES = ID.getBytes("UTF-8")
  }
  override fun transform(pool:BitmapPool, toTransform:Bitmap, outWidth:Int, outHeight:Int):Bitmap {
    if(toTransform.width == outWidth && toTransform.height == outHeight) {
      return toTransform
    }
    return Bitmap.createScaledBitmap(toTransform, outWidth, outHeight, true)
  }
  override fun equals(o:Any) {
    return o is FillSpace
  }
  override fun hashCode():Int = ID.hashCode()
  override fun updateDiskCacheKey(digest:MessageDigest) {
    digest.update(ID_BYTES)
  }  
}
```

##### 必需的方法

对于任何Transformation的子类，equals()、hashCode()和updateDiskCacheKey必须实现以确保磁盘和内存缓存可以正常工作。参考内置的RoundedCorners，圆角弧度也需要参与到上述三个方法中。

##### 重用变换

Transformation是无状态的，因此尽量采用单例来实现。

加载中，Glide可能会根据ImageView的scaleType来自动应用FitCenter或者CenterCrop。

#### 八、缓存

默认情况下，glide会在一个请求里检查多级缓存：

1. 活动资源（ActiveResources）- 是否有另一个View正在展示这张图片
2. 内存缓存（MemoryCache）- 是否最近被加载过并且仍存在于内存
3. 资源类型（Resource）- 是否之前被解码、转换并写入过磁盘缓存
4. 数据来源（Data）- 构建这个图片的资源是否之前被写入过文件缓存

前两步从内存中获取图片，后两部检查磁盘，异步返回图片。

##### 缓存键（Cache Keys）

所有缓存键都至少包含两个元素：请求加载的Model（File,Url,URI）和一个可选的签名（Signature）。步骤1~3的缓存键还包含其他数据：宽高、可选的变换、额外添加的选项以及请求的数据类型（Bitmap、Gif或其他）