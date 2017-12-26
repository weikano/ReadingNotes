## Fresco使用指南（二）- Drawee指南

```xml
<com.facebook.drawee.view.SimpleDraweeView
  android:id="@+id/my_image_view"
  android:layout_width="20dp"
  android:layout_height="20dp"
  fresco:fadeDuration="300"
  fresco:actualImageScaleType="focusCrop"
  fresco:placeholderImage="@color/wait_color"
  fresco:placeholderImageScaleType="fitCenter"
  fresco:failureImage="@drawable/error"
  fresco:failureImageScaleType="centerInside"
  fresco:retryImage="@drawable/retrying"
  fresco:retryImageScaleType="centerCrop"
  fresco:progressBarImage="@drawable/progress_bar"
  fresco:progressBarImageScaleType="centerInside"
  fresco:progressBarAutoRotateInterval="1000"
  fresco:backgroundImage="@color/blue"
  fresco:overlayImage="@drawable/watermark"
  fresco:pressedStateOverlayImage="@color/red"
  fresco:roundAsCircle="false"
  fresco:roundedCornerRadius="1dp"
  fresco:roundTopLeft="true"
  fresco:roundTopRight="false"
  fresco:roundBottomLeft="false"
  fresco:roundBottomRight="true"
  fresco:roundWithOverlayColor="@color/corner_color"
  fresco:roundingBorderWidth="2dp"
  fresco:roundingBorderColor="@color/border_color"
/>
```

### 不支持wrap_content

> Drawee永远会在getIntrinsicHeight/Width中返回-1
>
> 因为Drawee不像ImageView那样。它同一时间可能会显示多个元素。比如在从占位图渐变到目标图时，两张图会有同时显示的时候。再比如可能有多张目标图片（低清晰度、高清晰度两张）。如果这些图片都是不同尺寸，那么很难定义intrinsic尺寸。
>
> 如果我们要先用占位图的尺寸，等加载完成后再使用真实的尺寸，那么图片可能会显示异常。它可能会缩放、裁剪、变性。唯一防止这种事情的方式只有等图片加载完成后强制layout。但是这样会不仅仅影响性能，还会让应用界面突然变化。
>
> **所以Drawee必须指定尺寸或者用match_parent来布局**。

### 固定宽高比

