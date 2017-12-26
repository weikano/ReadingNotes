## Android Studio 简单入门

### 1. 项目结构

```xml
<!-- 最顶层的project -->
project 
	--module a <!--不同的module-->
        ----build.gradle <!-- module a的build.gradle文件 -->
	--module b <!-- 不同的module-->
        ----build.gradle <!-- module b的build.gradle文件 -->
	build.gradle <!-- project的build.gradle -->
```

project可以看作eclipse中的workspace，module是project。每个module既可以是一个独立的apk项目，也可以是一个library。

project目录下的build.gradle是用来配置整个project的，module下的是单独某个module适用的。

### 2. gradle和gradle-wrapper

- android studio的编译是基于gradle的，gradle是由groovy实现的，groovy是JVM语言。
- gradle-wrapper是提供给没有安装gradle的机器上使用的。**推荐使用gradle-wrapper**来执行gradle任务。因为gradle存在版本差异，某些方法可能在不同版本上会不存在。
- gradle-wrapper的使用：命令行中执行gradlew.bat 或者 linux 中./gradlew 。
- gradlew命令调用的是project/gradle/wrapper/gradle-wrapper.jar，其中gradle-wrapper.properties是wrapper的配置信息，包含对应的gradle版本。如果本机上面找不到对应的gradle版本，会先去下载。如果提示下载中，你也可以ctrl+c终止执行，先去properties文件中下载对应的gradle文件，然后到gradle默认的本地路径（一般是C:\Users\Yourname\\.gradle）中的wrapper\\dists找到对应的版本，把下载号的rar文件放进去。再执行gradlew就会自动解压执行命令。
- gradle的默认缓存地址可以通过配置环境变量GRADLE_USER_HOME来改变。
- android studio版本控制中，不要把project/gradle文件夹忽略在版本控制外。

### 3. project的build.gradle

```groovy
buildscript {
  repositories {
    //定义应该从哪寻找依赖库
  }
  dependencies {
    //语法为 classpath "${group}:${name}:${version}"
    classpath 'com.android.tools.build:gradle:2.3.0' //androidstudio需要使用到的依赖，版本跟android studio一样，对你使用的gradle版本有要求。
    classpath 'org.greenrobot:greendao-gradle-plugin:3.2.2'
    //其他library执行的自定义task可能需要另外的依赖，比如buterknife, tinker, retrolambda 等等……
    //可以把它当作java环境变量中的classpath，执行gradle任务时，会先加载classpath中定义好的依赖，一般可以在gradle_user_home/.gradle/caches/modules-2/files-2.1/对应的group+name找到对应的jar文件。这里以greendao为例。
  }
}

allprojects { //对project下的module都生效
  repositories {
    //repository
  }  
}

//还可以在这里定义一些变量，在module的build.gradle中使用
```

module的build.gradle

```groovy
apply plugin: 'com.android.application' //android应用程序
apply plugin: 'org.greenrobot.greendao' //greendao会根据注解来aapt生成对应的class，如果要用到greendao，必须加这一句。
apply from : 'else.gradle' //apply plugin应用的都是插件，或者说是jar包之类的，apply from 是使用gradle文件，比如你自己在else.gradle中定义了一个任务，apply from之后就可以在module中使用了。

android {
    compileSdkVersion 25
    buildToolsVersion 25.0.2
    defaultConfig {
        applicationId "com.weme.as.sample" //应用程序的报名
        minSdkVersion 15
        targetSdkVersion 25
        versionCode 1
        versionName "1.0"
        //单元测试用的，不需要改动。
        testInstrumentationRunner "android.support.test.runner.AndroidJUnitRunner"
      	//还有很多其他的配置信息……
    }
  	//签名配置
  	signingConfigs {
        debug {
            keyAlias 'debug'
            keyPassword 'weme123'
            storeFile file('签名文件地址')
            storePassword 'weme123'
        }
        release {
          	keyAlias 'release'
            keyPassword 'weme123'
            storeFile file('签名文件地址')
            storePassword 'weme123'
        }
    }
  	//默认提供debug和release，一般有这两种就可以分别打出测试和正式包
    buildTypes {
        debug {
          	//控制混淆和删除未使用的资源和代码文件
            minifyEnabled false
            //设置proguard规则，写在module/proguard-rules.pro里面
            proguardFiles getDefaultProguardFile('proguard-android.txt'), 'proguard-rules.pro'
            //配置签名信息
            signingConfig signingConfigs.debug
          	//使用debug打包后，apk的包名变成defaultConfig.applicationId+aplicatinIdSuffix
          	applicationIdSuffix '.debug'
            //versionName同样会在后面添加versionNameSuffix
            versionNameSuffix '.debug'
        }
    }
  	//类似分渠道，这样不仅仅可以修改下面的配置信息，还可以有不同的代码和资源。只要在src目录下再新建yingyongbao/java(res)，360/java(res)等文件，便可以在不同渠道打包时采用不同的代码资源。
  	productFlavors {
        yingyongbao {
          	//设置proguard
            proguardFile rootProject.file('app/proguard-rules.pro')
          	//设置签名
            signingConfig signingConfigs.debug
            targetSdkVersion 23
            minSdkVersion 16
            versionCode 1
            versionName '1.0.0'
          	//其他配置信息……
        }
      	360 {
          	//xxxx类似上面
      	}
    
}
//greendao插件需要读取的配置信息
//greendao {
//    schemaVersion 4
//}

dependencies {
    //资源的依赖有多种方式
  //compile会对所有build type和product flavor都会参与编译并且打包到最终的apk中
  compile "${group}:${name}:${version}"
  //对所有build type和product flavor都会参与编译但不会打包到apk中
  provided "${group}:${name}:${version}"
  //仅仅是针对单元测试代码的编译编译以及最终打包测试apk时有效，而对正常的debug或者release apk包不起作用
  testCompile "${group}:${name}:${version}"
  //仅仅是针对debug模式的编译和最终打包有效
  //参考facebook的leakcanary，debugCompile时会打包完整功能来提供内存泄漏检测，而compile的时候只提供一个空依赖
  debugCompile "${group}:${name}:${version}"
  //仅仅针对release模式和最终的release-apk打包
  releaseCompile "${group}:${name}:${version}"
}
```

### 4. BuildVariant的概念

gradle提供了build type和productFlavors的概念。buildType和productFlavors相结合，组成了Build Variant。每创建一个buildType和productFlavor，都会生成对应的BuildVariant。例如上面的例子中就会出现360Debug，360Release，yingyongbaoDebug和yingyongbaoRelease。这些build variant会跟task结合，生成不同的任务，比如install360Debug(Release)，installYingyongbaoDebug(Release)等任务。而且不仅仅是task，还会作用到dependencies中，比如上面的例子中可以这样添加

```groovy
dependencies {
  //尽在flavor为yingyongbao时有效
  yingyongbaoCompile "应用宝相关的库"
  //仅在flavor为360时有效
  360Compile "360相关的库"
}
```

### 5. gradle中如何定义变量

android开发中，某些依赖库需要有共同的版本。比如google自己的appcompat，design，recyclerview等，大版本号需要跟compileSdk或者targetSdk一致；retrofit2的converter和adapter也需要和retrofit2一致。这样就需要在某个地方统一定义版本号，然后在dependencies中使用即可。

以下有两种比较常用的方式。

- 在project的build.gradle中定义

  ```groovy
  //project的build.gradle中定义如下
  buildscript {
    //省略
  }
  allprojects {
    //省略
  }
  ext {
      minSdkVersion = 16
      compileSdkVersion = 25
      targetSdkVersion = compileSdkVersion
      buildToolsVersion = '23.0.3'
    	appcompat = "25.0.0"
  }
  //以上为project中的修改
  //module下build.gradle
  //xxx省略
  android {
    //xxx省略
    defaultConfig {
      //xxx省略
      minSdkVersion rootProject.ext.minSdkVersion
      //xxx省略
    }
  }
  dependencies {
    //使用双引号，具体可以了解groovy的字符串相关内容
    compile "com.android.support:appcompat:${rootProject.ext.appcompat}"
  }

  ```

- 在project目录下的gradle.properties中定义

  ```groovy
  //gradle.properties中的内容会加载到project中
  MIN_SDK = 15
  //那么上述rootProject.ext.minSdkVersion就可以替换成
  minSdkVersion MIN_SDK
  ```