# Android资源解析

`Android`中``res`目录下资源会通过`aapt`生成`R.java`，然后打包成`arsc`文件。`AndroidManifest.xml`文件也会被编译成二进制。

## 1. Resources从何而来

使用`Resources`来获取资源时，一般先需要通过`Context.getResources`来获取对应的`Resources`对象。而`Context`的实际实现类是`ContextImpl`

```java
//ContextImpl.java
public Resources getResources() {
  return mResources;
}
```

`mResources`是什么时候赋值的呢？通过`ContextImpl.setResources`方法

```java
//ContextImpl.java
void setResources(Resources r) {
  if(r instanceof CompatResources) {
    ((CompatResources)r).setContext(this);
  }
  mResources = r;
}
```

那么`ContextImpl.setResources`方法又是何时调用？

`ContextImpl.createApplicationContext`、`ContextImpl.createPackageContextAsUser`、`ContextImpl.createContextForSplit`、`ContextImpl.createConfigurationContext`、`ContextImpl.createDisplayContext`、`ContextImpl.createSystemContext`、`ContextImpl.createSystemUiContext`、`ContextImpl.createAppContext`、`ContextImpl.createActivityContext`

以其中的`ContextImpl.createActivityContext`为例往下查看

其调用路径为`ActivityThread.performLaunchActivity`->`ActivityThread.createBaseContextForActivity`，最后调用了`ContextImpl.createActivityContext`方法

```java
//ContextImpl.java
static ContextImpl createActivityContext(xxx) {
  ContextImpl context = new ContextImpl(/**container**/ null, mainThread, packageInfo, activityInfo.splitName,xxx);
  final ResourceManager rm = ResourceManager.getInstance();
  context.setResources(rm.createBaseActivityResources(xxx))
}
```

`ResourcesManager.createBaseActivityResources`方法创建了`Resources`对象

```java
//ResourcesManager.java
public Resources createBaseActivityResources(xxx) {
  final ResourcesKey = new ResourcesKey(xxx);
  synchronized(this) {
    //WeakHashMap中的key是weak的，但key不存在时，对应的entry会被移除掉
    //其中Key使用WeakReference保存的
    //Entry<K,V> extends WeakReference<Object>
    //WeakHashMap<IBinder, ActivityResources>保存了对应的activityToken及其ActivityResources，
    //getOrCreate方法用来确保有对应的对象
    getOrCreateActivityResourcesStructLocked(activityToken);
  }
  //刷新对应ActivityResources中的configuration
  updateResourcesForActivity(activityToken, overrideConfig, displayId,
                    false /* movedToDifferentDisplay */);
  return getOrCreateResources(activityToken, key, classLoader);
}

private Resources getOrCreateResources() {
  //创建对应的ResourcesImpl对象
  ResourcesImpl resourcesImpl = createResourcesImpl(key);
  if (resourcesImpl == null) {
    return null;
  }
  // Add this ResourcesImpl to the cache.
  mResourceImpls.put(key, new WeakReference<>(resourcesImpl));
  final Resources resources;
  if (activityToken != null) {
    resources = getOrCreateResourcesForActivityLocked(activityToken, classLoader,
                                                      resourcesImpl, key.mCompatInfo);
  } else {
    resources = getOrCreateResourcesLocked(classLoader, resourcesImpl, key.mCompatInfo);
  }
  return resources;
}
//获取对应的ResourcesImpl对象
private ResourcesImpl createResourcesImpl() {
  final AssetManager assets = createAssetManager(key);
  final ResourcesImpl impl = new ResourcesImpl(assets, xxx);
  return impl;
}
//创建Resources对象，并将ResourcesImpl赋值给resourcesImpl字段
private Resources getOrCreateResourcesForActivityLocked() {
  //省略……
  Resources resources = compatInfo.needsCompatResources() ? new CompatResources(classLoader)
    : new Resources(classLoader);
  resources.setImpl(impl);
  //省略……
  return resources;
}
```

**由上可知，当Activity启动时，是通过Resources的构造函数创建对象，然后赋值给ContextImpl。而Resources中的ResourceImpl对象是从ResourcesManager.createResourcesImpl中获取**

## 2. Resource如何获取资源

以`Resources.getString`为例

```java
public String getString(int id) throw NotFoundException {
  return getText(id).toString();
}
public CharSequence getText(int id) throw NotFoundException {
  CharSequence res = mResourceImpl.getAssets().getResourceText(id);
  if(res != null) {
    return res;
  }
  throw new NotFoundException();
}
```

其最终交由`mResourceImpl.getAssets`对象处理，而这个对象正是上述中的`AssetManager`

`AssetManager`是创建过程如下：

```java
//ResourcesManager.java
//通过loadApkAssets方法加载的是ApkAssets对象，代表这内存中的apk并且是不可变的。主要是通过native c++实现，因为不可变，所以可以在AssetsManager之间共享
//builder.addApkAssets会将ApkAssets添加到Builder的mUserApkAssets中
protected AssetManager createAssetManager(ResourcesKey key) {
  final AssetManager.Builder builder = new AssetManager.Builder();
  if(key.mResDir != null) {
    builder.addApkAssets(loadApkAssets(key.mResDir, /**sahredLib**/false, /**overlay**/ false));
  }
  if(key.mSplitResDirs != null) {
    for(final String splitResDir: key.mSplitResDirs) {
      builder.addApkAssets(loadApkAssets(splitResDir, false, false));
    }
  }
  if(key.mOverlayDirs != null) {
    for(final String idmapPath: key.mOverlayDirs) {
      builder.addApkAssets(loadApkAssets(idmapPath, false, true));
    }
  }
  if(key.mLibDirs != null) {
    for(final String libDir: key.mLibDirs) {
      builder.addApkAssets(loadApkAssets(libDir, true, false));
    }
  }
  return builder.build();
}
//AssetManager.Builder
public AssetManager build() {
  //系统的ApkAssets，通过读取/system/framework/framework-res.apk以及/data/resource-cache/overlays.list
  final ApkAssets[] systemApkAssets = getSystem().getApkAssets();
  //将systemApkAssets拷贝至apkAssets中，通过System.arraycopy
  //然后再将mUserApkAssets拷贝至apkAssets位于systemApkAssets之后的位置
  final AssetManager am = new AssetManager(/**sentinel**/false);
  am.mApkAssets = apkAssets;
  AssetManager.nativeSetApkAssets(am.mObject, apkAssets, false);
  return am;
}
```

看到`mObject`或者`nativePtr`就知道，`AssetManager`在创建过程中肯定有创建c++层的对象

```java
//AssetManager.java
private AssetManager(boolean sentinel) {
  mObject = nativeCreate();
}
```

其c++代码在`android_util_AssetManager.cpp`下，而这些方法都是通过`register_android_content_AssetManager`，然后在`AndroidRuntime.cpp`中通过`REG_JNI(register_android_content_AssetManager)`来对应JNI

```c++
//android_util_AssetManager.cpp
//nativeCreate对应NativeCreate
//nativeSetApkAssets对应NativeSetApkAssets
static jlong NativeCreate(JNIEnv*, jclass) {
  //其实就是一个AssetManager2的指针
  return reinterpret_cast<jlong>(new GuardedAssetManager());
}

static void NativeSetApkAssets() {
  //java层的ApkAssets通过构造方法总的nativeLoad方法创建了一个c层的ApkAssets对象
  //该方法就是将Java层的ApkAssets对象中的native层对象的指针，转换成native层的ApkAssets对象，
  //然后设置给native层的AssetManager2对象
}
```

接下来看native层的`AssetManager2`

```c++
//AssetManager2.cpp
bool AssetManager2::SetApkAssets() {
  apk_assets_ = apk_assets;
  //给每个shared library ApkAssets分配一个packageId
  BuildDynamicRefTable();
  RebuildFilterList();
  invalidateCaches();
  return true;
}
```

回到`Java`层的`AssetManager`，它通过`getResourceText`方法获取对应的`CharSequence`对象

```java 
CharSequence getResourceText(int resId) {
  synchronized(this) {
    final TypedValue outValue = mValue;
    if(getResourceValue(resId, 0, outValue, true)) {
      //如果outValue的type是String，那么直接返回String
      return outValue.coerceToString();
    }
    return null;
  }
}
```

终点在于`getResourceValue`方法

```java
//AssetManager.java
boolean getResourceValue(@AnyRes int resId, int densityDpi, @NonNull TypedValue outValue,
                         boolean resolveRefs) {
  synchronized (this) {
    ensureValidLocked();
    final int cookie = nativeGetResourceValue(
      mObject, resId, (short) densityDpi, outValue, resolveRefs);
    if (cookie <= 0) {
      return false;
    }

    // Convert the changing configurations flags populated by native code.
    outValue.changingConfigurations = ActivityInfo.activityInfoConfigNativeToJava(outValue.changingConfigurations);

    if (outValue.type == TypedValue.TYPE_STRING) {
      outValue.string = mApkAssets[cookie - 1].getStringFromPool(outValue.data);
    }
    return true;
  }
}
```

接下来转到native层

```c++
//android_util_AssetManager.cpp
static jint NativeGetResourceValue() {
  ScopedLock<AssetManager2> am(AsssetManagerFromLong(ptr));
  //根据resId等信息获取对应的ApkAssetsCookie
  ApkAssetsCookie cookie = am->GetResource(resid, false, density, &value /**Res_value**/,xxx );
  uint32_t ref = static_cast<uint32_t>(resid);
  if (resolve_references) {
    cookie = assetmanager->ResolveReference(cookie, &value, &selected_config, &flags, &ref);
    if (cookie == kInvalidCookie) {
      return ApkAssetsCookieToJavaCookie(kInvalidCookie);
    }
  }
  return CopyValue(env, cookie, value, ref, flags, &selected_config, typed_value);
}

static jint CopyValue(JNIEnv* env, ApkAssetsCookie cookie, const Res_value& value, uint32_t ref,
                      uint32_t type_spec_flags, ResTable_config* config, jobject out_typed_value) {
  //gTypedValueOffsets是一个结构体，保存了Java层TypedValue中各字段在native层通过jni的引用
  //这个方法用来给Java层的TypedValue设置给中字段，代码在AssetManager.getResourceValue
  env->SetIntField(out_typed_value, gTypedValueOffsets.mType, value.dataType);
  env->SetIntField(out_typed_value, gTypedValueOffsets.mAssetCookie,
                   ApkAssetsCookieToJavaCookie(cookie));
  env->SetIntField(out_typed_value, gTypedValueOffsets.mData, value.data);
  env->SetObjectField(out_typed_value, gTypedValueOffsets.mString, nullptr);
  env->SetIntField(out_typed_value, gTypedValueOffsets.mResourceId, ref);
  env->SetIntField(out_typed_value, gTypedValueOffsets.mChangingConfigurations, type_spec_flags);
  if (config != nullptr) {
    env->SetIntField(out_typed_value, gTypedValueOffsets.mDensity, config->density);
  }
  return static_cast<jint>(ApkAssetsCookieToJavaCookie(cookie));
}
//android_util_AssetManager.cpp
constexpr inline static jint ApkAssetsCookieToJavaCookie(ApkAssetsCookie cookie) {
  return cookie != kInvalidCookie ? static_cast<jint>(cookie + 1) : -1;
}
//AssetManager2.h
enum : ApkAssetsCookie {
  kInvalidCookie = -1,
};

```

Java层收到的`int cookie`最终还是通过native层的`AssetManager2::GetResource`方法返回

```c++
ApkAssetsCookie AssetManager2::GetResource(uint32_t resid, bool may_be_bag,
                                           uint16_t density_override, Res_value* out_value,
                                           ResTable_config* out_selected_config,
                                           uint32_t* out_flags) const {
  FindEntryResult entry;
  //resid一般为0x7f010008类整型
  //FindEntry方法会根据(ResourceUtils.h)
  //package_id = get_package_id(resId)最高8位，即最高两个十六进制，即7f;
  //type_idx = get_type_id(resid)-1; 第3、 4两位十六进制数, 即01
  //entry_idx = get_entry_id(resid); 最后4位十六进制数，即0008
  //从ApkAssets中的LoadedPackage里面查找对应的资源
  ApkAssetsCookie cookie =
      FindEntry(resid, density_override, false /* stop_at_first_match */, &entry);
  if (cookie == kInvalidCookie) {
    return kInvalidCookie;
  }

  if (dtohs(entry.entry->flags) & ResTable_entry::FLAG_COMPLEX) {
    if (!may_be_bag) {
      LOG(ERROR) << base::StringPrintf("Resource %08x is a complex map type.", resid);
      return kInvalidCookie;
    }

    // Create a reference since we can't represent this complex type as a Res_value.
    out_value->dataType = Res_value::TYPE_REFERENCE;
    out_value->data = resid;
    *out_selected_config = entry.config;
    *out_flags = entry.type_flags;
    return cookie;
  }

  const Res_value* device_value = reinterpret_cast<const Res_value*>(
      reinterpret_cast<const uint8_t*>(entry.entry) + dtohs(entry.entry->size));
  out_value->copyFrom_dtoh(*device_value);

  // Convert the package ID to the runtime assigned package ID.
  entry.dynamic_ref_table->lookupResourceValue(out_value);

  *out_selected_config = entry.config;
  *out_flags = entry.type_flags;
  return cookie;
}
```

已经拿到Java层`AssetManager.getResourceValue`方法中需要的`cookie`值，接下来

```java
 /**
  if (outValue.type == TypedValue.TYPE_STRING) {
   outValue.string = mApkAssets[cookie - 1].getStringFromPool(outValue.data);
   }
 **/
//交给android_util_StringBlock.cpp
//obj为c++层ApkAssets->GetLoadedArsc()->GetStringPool()
private static native String nativeGetString(long obj, int idx);
```

**其他资源获取类似，也是通过native层的AssetManager获取到id对应的ApkAssets，然后根据LoadedPackage去获取对应的资源，然后写入由Java层传入的`TypedValue`对象，再根据不同的类型获取不同的值**

