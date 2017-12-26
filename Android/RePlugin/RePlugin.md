Replugin资源管理
Loader.loadDex 257开始获取插件的resource并添加至Plugin.FILENAME_2_RESOURCES中
RePlugin.attachBaseContext->PMF.callAttach->PmBase.callAttach->Plugin.load-

> Plugin.loadLocked->Plugin.doLoad初始化Loader(mLoader)->Loader.loadDex将Resource保存

至Plugin.FILENAME_2_RESOURCES中

RePlugin.fetchResource(pluginName)->PmLocalImpl.queryPluginResources(pluginName)
首先调用Plugin.queryCachedResources中查找缓存起来的Resources。如果不存在，

PmBase.loadResourcePlugin()->PmBase.loadPlugin()->Plugin.load()->Plugin.loadLocked-

> Plugin.doLoad()->Loader.loadDex

加载内置插件信息
RePlugin.attachBaseContext->PMF.init()->PmBase.init()->PmBase.initForPersistent()-

> Builder.builder()->Finder.search()->FinderBuiltin.loadPlugins()-

> FinderBuiltin.readConfig()

启动插件Activity
使用RePlugin.createIntent(pluginName, targetActivityName)获取Intent,intent中的

component的pkgName替换成pluginName了。RePlugin.startActivity()-

> PmLocalImpl.startActivity()->PmInternalImpl.startActivity()。
> 如果Factory2.isDynamicClass（动态注册的类PmBase.isDynamicClass），直接调用

context.startActivity;反之获取PmLocalImpl.loadPluginActivity获取ComponentName,并将

intent替换到坑位。

RePluginClassLoader.loadClass()->PMF.loadClass()->PmBase.loadClass()

- 加载activity，由PluginProcessPer.resolveActivityClass()-

> PluginContainers,lookupByContainer来获取对应的ActivityState，state==null返回

ForwardActivity.class;state!=null，返回Plugin.Loader.mClassLoader.loadClass。如果返

回为null，PmBase.loadClass返回DummyActivity.class

PluginContainers.mStates、PmBase.mContainerXXX由PluginProcessPer构造方法-

> PluginContainer.init方法确定
> Loader.mClassLoader由RePluginCallbacks.createPluginClassLoader生成



### ClassLoader替换

> RePlugin.attachBaseContext()->PMF.init()->PatchClassLoaderUtils.patch()将RePluginClassLoader利用反射将原application中默认的ClassLoader对象替换为RePluginCallbacks.createClassLoader方法生成的RePluginClassLoader对象。

### loadClass

> RePluginClassLoader.loadClass()->PMF.loadClass()->PluginDexClassLoader.loadClass()

apk安装后保存的路径

> PluginInfo.getApkFile

### 插件Application初始化

Plugin.callAppLocked()<-Plugin.callApp()<-Plugin.load()，两种情况：

- PmBase.callAttach()<-PMF.callAttach()<-RePlugin.attachBaseContext()
- PmBase.loadPlugin()<-PmBase.loadAppPlugin()<-PluginProcessPer.bindActivity()<-PluginProcessPer.allocActivityContainers()<-PmLocalImpl.loadPluginActivity()<-Factory.loadPluginActivity()<-PmInternalImpl.startActivityForResult()<-Factory2.startActivityForResult()

RePlugin.startActivity()也会在PmInternalImpl.starActivity里面调用PmLocalImpl.loadPluginActivity()，最终会调用到Plugin.callAppLocked()。



### 插件独立进程

需要在插件的AndroidManfiest中新增meta-data如下

```xml
<!-- 第一个from to将整个插件的process转换到host:p1进程，第二个将插件中进程为packageName:a的切换到host:p2进程 -->
<meta-data
  android:name="process_map"
  android:value="[{'from':'当前插件的packageName','to':'$p1'},{'from':'当前插件的pckageName:a','to':'$p2'}]"/>
```

进程切换的相关代码在Loader.adjustPluginProcess()。



### host常驻进程GuardService

相关service可以查找GuardService关键字

### RePlugin如何绕过AMS检查的(将intent中的componentName替换为占坑activity)

PmLocalImpl.loadPluginActivity()->getActivityInfo()->PluginProcessPer.allocActivityContainer()->bindActivity()->PluginContainers.alloc()->allocLocked()

### RePlugin从占坑Activity还原为targetActivity

然后在RePluginClassLoader.loadClass()->PMF.loadClass()->PmBase.loadClass()->PluginProcessPer.resolveActivityClass