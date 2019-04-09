### 第18章-SystemUI

#### 18.1 SystemUI的组成元素

#### 18.2 SystemUI的实现

SystemUIService也是在SystemServer中启动起来的

```java
//SystemServer.java
static final void startSystemUi() {
  Intent intent = new Intent();
  intent.setComponent(new ComponentName("com.android.systemui", "com.android.systemui.SystemUIService"));
  context.startServerAsUser(intent, UserHandle.SYSTEM);
  //KeyguardServiceDeleate.bindService(context)
  windowManager.onSystemUiStarted();
}

private void startOtherServices() {
  startSystemUi();
}
```

SystemUIService做了什么

```kotlin
//SystemUIService.java
//onBind中什么都没做
//onStartCommand也什么都没做
override fun onCreate() {
  super.onCreate()
 	(getApplication() as SystemUIApplication).startServiceIfNeeded()  
}
//SystemUIApplication.java
fun startServicesIfNeed() {
  //在frameworks/base/packages/SystemUI/res/values/config.xml中定义了一些SystemUI的子类的全名的字符串集合
  //com.android.systemui.Dependency 主要用于将class与对应的实现类对应起来，比如PowerUI.WarningsUI.class对应PowerNotificationWarnings(mContext)对象
  //SystemBars 在start方法中调用了createStatusBarFromConfig方法来调用对应的(Car)StatusBar对象的.start方法来创建statusbar和navigationbar
  //Recents 负责跟RecentActivity的交互，多任务界面
  //...
 	val names = resouces.getStringArray(R.array.config_systemUIServiceComponents);
  startServicesIfNeed(names);
}
private fun startServicesIfNeeded(names:Array<String>) {
  services.forEach { it ->
    val cls = Class.forName(it)
    mServices[i] = cls.newInstance() as SystemUI 
    //...忽略其他
    mServices[i].start()
    if(mBootCompleted) {
      mServices[i].onBootCompleted()                  
    }
  }
  Dependency.get(PluginManager::java.class).addPluginListener(object:PLuginListener<OverlayPlugin> {
    override fun onPluginConnected(plugin:OverlayPlugin, pluginContext:Context) {
      val statusBar = getComponent(StatusBar::java.class)
      statusBar?.also {
        plugin.setup(it.statusBarWindow, it.navigationBarView)
      }
      if(plugin.holdStatusBarOpen()) {
        mOverlays.add(plugin)
        Dependency.get(StatusBarWindowManager::java.class).setStateListener { b ->
          mOverlays.forEach { o-> o.setCollapseDesired(b)}
          Dependency.get(StatusBarWindowManager::java.class).setForcePluginOpen(mOverlays.size != 0)                                                                   
        }
      }
    }
    override fun onPluginDisconnected(plugin:OverlayPlugin) {
      mOverlays.remove(plugin)
      Dependency.get(StatusBarWindowManager::java.class).setForcePluginOpen(mOverlays.size != 0)
    }
  }, OverlayPlugin::java.class, true)
}
```

#### 18.3 Android壁纸资源

略

