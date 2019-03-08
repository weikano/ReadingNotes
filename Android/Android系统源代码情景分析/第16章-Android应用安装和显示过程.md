### 第16章-Android应用安装和显示过程

Android系统在启动过程中，会扫描特定目录，以便可以对保存在里面的应用程序进行安装。PMS主要完成两件事：解析AndroidManifest.xml，为APP分配Linux用户ID和组ID。

android系统启动的最后一步是启动一个home应用。

#### 16.1 应用程序的安装过程

SystemServer在startBootstrapServices中会调用PackageManagerService.main方法创建一个PMS，在startOtherServices中会调用updatePackagesIfNeeded和systemReady。PMS在构造函数中会调用scanDirTracedLI函数扫描/vendor/overlay、/product/overlay、/ANDROID_ROOT/system/framework、/ANDROID_ROOT/priv-app

、/VENDOR_ROOT/vendor/priv-app、/VERDOR_ROOT/vendor/app、/ODM_ROOT/odm、/ODM/odm/app、/OEM_ROOT/oem/app、/PRODUCT_ROOT/product/priv-app以及/PRODUCT_ROOT/product/app目录。

scanDirTracedLI函数最终会调用scanDirLI

```java
//PackageManagerService.java
private void scanDirLI(File scanDir, int parseFlags, int scanFlags, long currentTime) {
  final File[] files = scanDir.listFiles();
  int fileCount = 0;
  try(ParallelPackageParser parser = ParallelPackageParser(xxx)) {
    for(File file:files) {
      //调用parser.parsePackage，走到PackageParser.parseMonolithicPackage()，然后将结果转换成byte数组保存至mCacheDir，在PMS构造函数中设置为ANDROID_DATA/data/system/1
     	parser.submit(file, parseFlags);
    	fileCount ++; 
    }  
    for(;fileCount>0;fileCount--) {
      ParseResult result = parser.take();
      //忽略错误码检测
      scanPackageChildLI(result.pkg, parseFlags, scanFlags, currentTime, null);
    }
  }  
}
private Package scanPackageChildLI(Package pkg, int parseFlags, int scanFlags, time, UserHandler user) {
  //将pgk添加到mPackages中
  addForInitLI();
}

//PackageParser.java
public Package parseMonolithicPackage(File file, int flags) {
  //PackageLite包含一个ApkLite，里面描述了codePath,packageName,version,debuggable等信息
  //PackageLite还包含了file.path
  final PackageLite lite = parseMonolithicPackageLite(file, flags);
  final Package pkg = parseBaseApk(file, assetManager, flags);
  return pkg;
}
private Package parseBaseApk(File file, AssetManager am, int flags) {
  //调用parseBaseApkCommon，解析application的tag属性  
  parseBaseApplication(pkg,xxx,xxx);
}
private boolean parseBaseApplication() {
  //persistent属性：SystemServer会在startBootstrapService中通过mSystemServiceManager.startService(AMS.Lifecycle.class)方法构造一个AMS，在startOtherService中会调用AMS.systemReady，其中会调用AMS.startPersisentApps方法来调用PMS.getPersistentApplications搜索所有persistent属性为true的app，并且通过addAppLocked方法来启动  
  //parseIntent会在parseActivity/Service/ProviderTags中调用,parseMetadata也会
  //Activity为PackageParser$Activity类，包含ActivityInfo，用来解析activity和receiver标签  
  parseActivity();
  //PackageParser$Service，包含ServiceInfo
  parseService();
  //PackageParser$Provider，包含ProviderInfo
  parserProvider();
  parseActivityAlias();
  parseMetadata();  
 	//Activity/Service/Provider都是Coponent的子类，构造函数中会调用parsePackageItemInfo解析name等属性
  //parse之后添加到Package对应的activities/services/providers中
}
```

PMS有一个Settings类型的变量mSettings，用来管理APP的安装信息。PMS在Android系统每次启动时都会重新安装一遍系统中的应用程序，每次安装时，APP的信息都要保持一致，都过mSettings.readLP来实现。mSettings中的信息都保存在文件/ANDROID_DATA/data/system/packages(-bakup).xml中, readLPw时会读取其中的信息。以readPackgeLPw时会调用mpackages.put

#### 16.2 应用的显示过程

AMS.systemReady会调用startHomeActivityLocked->ActivityStackController.startHomeActivity。对应的Intent中mTopAction = Intent.ACTION_MAIN，category为Intent.CATEGORY_HOME，而mTopData和mTopComponent根据是非工厂测试模式和高级工程测试模式有不同。非工厂测试模式：mTopData = null, mTopComponentName = null