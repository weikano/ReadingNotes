### 第10章-GUI系统之WMS

WMS至少要完成两个功能:

- 全局的窗口管理（Output）：窗口管理属于输出。应用程序的显示请求在SurfaceFlinger和WMS的协助下有序地输出给物理屏幕
- 全局的事件分发（Input）：WMS要管理输入事件的派发

#### 10.1 WMS综述

##### 1. WMS

- 由SystemServer启动：WMS启动时机比较晚，那么开机画面都是由BootAnimation通过OpenGL ES与SurfaceFlinger的配合完成。因此如果要在Android中显示UI界面，不一定要通过WMS，可以参考bootanimation来协议个不基于WMS的Linux程序（framworks/base/cmds/bootanimation/bootanimation_main.cpp）
- 直到系统关机时才退出
- 发生异常时必须能自动重启

##### 2. SurfaceFlinger

##### 3. 有图形显示需求的程序

除了常见的以Activity组件构成的应用程序外，Android系统自身也有显示界面的请求，**不同类型的用户创建的窗口是有优先级的**：

- Application Window：针对普通应用的窗口，优先级较低
- System Window：系统顶部的状态栏、壁纸等属于系统窗口
- Sub Window：点击menu时出现的应用程序菜单

##### 4. InputManagerService

当IMS收到一个按键或者触摸事件时，它需要通过WMS找到一个最合适的窗口来处理

##### 5. AMS

AMS管理系统中所有的Activity

##### 6. Binder通信

无论SurfaceFlinger、AMS或者其他应用进程，与WMS之间都需要进行Binder通信。对WMS来说，只要包含一下功能：

- 窗口的添加与删除：当某个进程有显示需求时，通过WMS请求添加一个窗口，不需要时再移除
- 启动窗口：添加一个新窗口时，某些条件下需要添加一个“启动窗口”
- 窗口动画：窗口间切换时，窗口动画是可以定制的
- 窗口大小：
- 窗口层级：即Z-Order
- 事件派发

###### 10.1.1 WMS的启动

```java
//SystemServer.java
void startOtherService() {
  //main方法通过BlockingRunnable方法来调用WMS的构造方法
  wm = WindowManagerService.main(context,inputManager, haveInputMethod, !mFirstBoot, mOnlyCore, new PhoneWindowManager());
  ServiceManager.addService(Context.WINDOW_MANAGER, wm, false, dumpFlag);
  wm.onInitReady();
}
```

###### 10.1.2 WMS的基础功能

比如获得显示屏的大小；判断是否SystemBar；是否有NavigationBar；锁定屏幕；截取屏幕等

###### 10.1.3 WMS的工作方式

- 事件投递：通过mH（Handler）发送消息来处理事件，比如showStrictModeViolation
- 直接调用：不需要将事件发送至mH，直接处理，比如isKeyguardLocked

###### 10.1.6 WMS、AMS与Activity之间的联系

1. IPC通信

```java
//ViewRootImpl.java
public ViewRootImpl(Context context, Display display) {
  //获取一个IWindowSession
  mWindowSession = WindowManagerGlobal.getWindowSession();  
}
//WindowManagerGlobal.java
public static IWindowSession getWindowSession() {
  synchronized(WindowManagerGlobal.class) {
    if(sWindowSession == null) {
      try {
        INputMethodManager imm = InputMethodManager.getInstance();
        IWindowManager wm = getWindowManagerService();
        sWindowSession = wm.openSession(new IWindowSessionCallback.Stub(){
          @Override public void onAnimatorScaleChanged(float scale) {
            ValueAnimator.setDurationScale(scale);
          }
        }, imm.getClient(), imm.getInputContext());
      } catch(RemoteException e) {
        throw e.rethrowFromSystemServer();
      }
    }
    return sWindowSesson;
  }
}
```

WMS是通过匿名Binder来访问应用程序的。首先通过openSession来建立与WMS的私有连接，紧接着调用IWindowSession::relayout(IWindow window)方法，window由应用程序提供，用于WMS回访应用程序进程的匿名Binder server。IWindow的实现类是W。W提供了resized、dispatchAppVisibility、dispatchScreenSize等回调接口来通知WMS

2. 内部组织方式：Activity启动时，会在AMS中注册一个对应的ActivityRecord和WindowState，使用AppWindowToken来对应AMS中的一个ActivityRecord。

#### 10.2 窗口属性

##### 10.2.1 窗口层级与类型

1. Application Window：普通的应用程序都属于这一类

| type                          | 说明                                                         |
| ----------------------------- | ------------------------------------------------------------ |
| FIRST_APPLICATION_WINDOW = 1  | 应用程序窗口的起始值                                         |
| TYPE_BASE_APPLICATION = 1     | 应用窗口类型的基础值，其他窗口类型以此为基础                 |
| TYPE_APPLICATION = 2          | 普通应用程序的窗口类型                                       |
| TYPE_APPLICATION_STARTING = 3 | 应用程序的启动窗口类型，他不能由应用程序本身使用，而是Android系统为应用程序启动设计的窗口-当真正的窗口启动后就消失 |
| LAST_APPLICATION_WINDOW = 99  | 应用窗口类型的最大值                                         |

2. Sub Window：这类窗口附着在其他window中

| type                                                | 说明                                                         |
| --------------------------------------------------- | ------------------------------------------------------------ |
| FIRST_SUB_WINDOW=1000                               | 子窗口类型的起始值                                           |
| TYPE_APPLICATION_PANEL=FIRST_SUB_WINDOW             | 应用程序的panel子窗口，在它的父窗口上显示                    |
| TYPE_APPLICATION_MEDIA = FIRST_SUB_WINDOW+1         | 用于显示多媒体内容的子窗口，在其他父窗口之下                 |
| TYPE_APPLICATION_SUB_PANEL = FIRST_SUB_WINDOW+2     | 位与父窗口以及所有TYPE_APPLICATION_PANEL子窗口之上           |
| TYPE_APPLICATION_ATTACHED_DIALOG=FIRST_SUB_WINDOW+3 | Dialog子窗口，如menu类型                                     |
| TYPE_APPLICATION_MEDIA_OVERLAY=FIRST_SUB_WINDOW+4   | 多媒体窗口的覆盖层，位于TYPE_APPLICATION_MEDIA和应用程序窗口之间，通常需要是透明的才有意义，目前未开放 |
| LAST_SUB_WINDOW=1999                                | 子窗口结束值                                                 |

3. System Window

   | type                                         | 说明                                                         |
   | -------------------------------------------- | ------------------------------------------------------------ |
   | FIRST_SYSTEM_WINDOW = 2000                   | 系统窗口的起始值                                             |
   | TYPE_STATUS_BAR = FIRST_SYSTEM_WINDOW        | 系统状态栏窗口                                               |
   | TYPE_SEARCH_BAR = FIRST_SYSTEM_WINDOW+1      | 搜索条窗口                                                   |
   | TYPE_PHONE=FIRST_SYSTEM_WINDOW+2             | 通话窗口，特别是来电。通常位于系统状态栏之下，其他应用程序窗口之上 |
   | TYPE_SYSTEM_ALERT = FIRST_SYSTEM_WINDOW+3    | Alert窗口，如电量不足警告窗口，通常位于所有应用程序之上      |
   | TYPE_KEYGUARD = FIRST_SYSTEM_WINDOW+4        | 屏保窗口                                                     |
   | TYPE_TOAST = FIRST_SYSTEM_WINDOW+5           | Toast窗口                                                    |
   | TYPE_SYSTEM_OVERLAY = FIRST_SYSTEM_WINDOW+6  | 系统覆盖层窗口，不能接受Input焦点，否则会与屏保发生冲突      |
   | TYPE_PRIORITY_PHONE = FIRST_SYSTEM_WINDOW+7  | 电话优先窗口，屏保状态下的来电显示                           |
   | TYPE_SYSTEM_DIALOG = FIRST_SYSTEM_WINDOW + 8 | 比如RecentAppsDialog就是这样的窗口                           |
   | TYPE_KEYGUARD_DIALOG = FIRST_SYSTEM_WINDOW+9 | 屏保时显示的对话框                                           |

   //todo

WMS需要根据具体情况来调整窗口的Z-Order。数值越大的，在WMS中的优先级越高，最终在屏幕上显示时就越靠近用户。

```java
//WindowManagerSerivce.java
public int addWindow(xxx) {
  //最终调用WindowContainer.assignChildLayers来实现
	win.getParent().assignChildLayers()
}
```

#### 10.3 窗口的添加过程

##### 10.3.1 系统窗口的添加过程

StatusBar添加过程

```java
//StatusBar.java
public void start() {
  //忽略其他，下面的方法会调用addStatusBarWindow
  createAndAddWindow();
}
private void addStatusBarWindow() {
  //初始化mStatusBarWindow
  makeStatusBarView();
  mStatusBarWindowManager = Dependency.get(StatusBarWindowManager.class);
  //...忽略其他
  mStatusBarWindowManager.add(mStatusBarWindow, getStatusBarHeight());
}
//Dependency.java
public void start() {
  mProviders.put(StatusBarWindowManager.class, ()-> new StatusBarWindowManager(mContext));
}
//StatusBarWindowManager.java
public void add(View statusBarView, int barHeight) {
  //设置WindowManager.LayoutParams等属性
  //mWindowManager.addView最终会调用WindowManagerGlobal.addView方法
  mWindowManager.addView(mStatusBarView,mLp);
}
//WindowManagerGlobal.java
public void addView(view, params, display, parentWindow) {
  //检查type，是否有重复添加
  ViewRootImpl root = new ViewRootImpl(view.getContext(), display);
  view.setLayoutParams(params);
  mViews.add(view);
  mRoots.add(root);
  mParams.add(params);
  //panelParentView只有在params.type是Sub Window时才会去mViews中查找
  root.setView(view,params,panelParentView)
}
//ViewRootImpl.java
public void setView(view, WindowManager.LayoutParams attrs, panelParentView) {
  if(mVIew == null) {
    mView = view;
    //...
    //在添加至WindowManager之前先发起layout请求
    requestLayout();
    //...
    //mWindowSession = WindowManagerGlobal.getWindowSession()，ViewRootImpl构造函数中调用，mWindowSession实际上是Session对象
    //mWindow = new W(this);
    res = mWindowSession.addToDisplay(mWindow, mSeq, mWindowAttributes, getHostAbility(), mDisplay.getDisplayId(), mWinFrame, ....);
  }
}
//android.server.wm.Session.java
public int addToDisplay() {
  //调用WMS.addWindow方法
  return mService.addWindow(this, window, seq, ...);
}
//WindowManagerService.java
public int addWindow(Session session, IWindow client, int seq, ...){
  //一系列检查……
  final WindowState win = new WindowState(this, session, client, token, parentWindow, ...);
  res = mPolicy.prepareAddWindowLw(win, attrs);
  win.mToken.addWindow(win);
  win.getParent().assignChildLayers();
}
```

##### 10.3.2 Activity窗口的添加过程

```java
//ActivityThread.java
public void handleResumeActivity() {
  final ActivityClientRecord r = performResumeActivity(token, finalStateRequest, reason);
  if(r.window == null && !.a.mFinished && willBeVisible) {
    View decor = r.window.getDecorView();
    decor.setVisibility(View.INVISIBLE);
    ViewManager wm = a.getWindowManager();
    if(a.mVsiibleFromClient) {
      if(!a.mWindowAdded) {
        a.mWindowAdded = true;
        wm.addView(decor, l);
      }
    }
  }
  if(!r.activity.mFinished && willBeVisible && r.activity.mDecor != null && !r.hideForNow) {
    if(r.activity.mVsiibleFromClient) {
      ViewManager wm = a.getWindowManager();
      View decor = r.window.etDecorView();
      wm.updateViewLayout(decor, l);
    }
  }
}
```

##### 10.3.3 窗口添加实例

#### 10.4 Surface管理

##### 10.4.1 Surface申请流程(relayout)

参考StatusBar添加至Window的流程中，有用到ViewRootImpl.requestLayout()。

```java
//ViewRootImpl.java
public void requestLayout() {
  scheduleTraversals();
}
void scheduleTraversals() {
  //mTraversalRunnable会触发performTraversals();
  mChoreographer.postCallback(Choreograhper.CALLBACK_TRAVERSAL, mTraversalRunnable, null);
}
private void performTraversals() {
  //...
  //获取Surface
  if(mFirst || windowShouldResize || insetsChanged || viewVisbilityChanged || params != null || mForce) {
    relayoutResult = relayoutWindow(params, viewVisibility, insetsPadding);
  }
  //...
  if(!mStopped || mReportNextDraw) {
    //measure
    performMeasure(childWidthMeasureSpec, childHeightMeasureSpec);
    if(lp.horizontalWeight > 0.0f) {
      width += (int)((mWidth - width) * lp.horizontalWeight);
      childWidthMeasureSpec = MeasureSpec.makeMeasureSpec(width, MeasureSpec.EXACTLY);
      measureAgain = true;
    }
    if (lp.verticalWeight > 0.0f) {
      height += (int) ((mHeight - height) * lp.verticalWeight);
      childHeightMeasureSpec = MeasureSpec.makeMeasureSpec(height, MeasureSpec.EXACTLY);
      measureAgain = true;
    }
    //如果有weight，重新再measure
    if(measureAgain) {
      performMeasure(childWidthMeasureSpec, childHeightMeasureSpec);
    }
  }
  final boolean didLayout = layoutRequested && (!mStopped || mReportNextDraw);
  if(didLayout) {
    //layout
    performLayout(lp, mWidth, mHeight);
  }
  boolean cancelDraw = mAttachInfo.mTreeObserver.dispatchOnPreDraw() || !isViewVisible;
  if(!cancelDraw && !newSurface) {
    //draw
    performDraw();
  }else {
    if(isViewVisible) {
      schedualTraversals();
    }
  }
}
//一个空的Surface
public final Surface mSurface = new Surface();
private int relayoutWindow(params, visibility, insetsPadding) {
  int relayoutResult = mWindowSession.relayout(mWindow, mSeq, params, ..., mSurface);
}
//com.android.server.wm.Session.java
public int relayout(window, ...,...,outSurface) {
  int res = mService.relayoutWindow(this, window, ..., outSurface);
  return res;
}
//WMS.java
public int relayoutWindow() {
  //代替api23之前的performLayoutAndPlaceSurfacesLockedInner
  mWindowPlacerLocked.performSurfacePlacement(true);
  if(shouldRelayout) {
    result = win.relayoutVisibleWindow(result, attrChanges, oldVibility);
    result = createSurfaceControl(outSurface, result, win, winAnimator);
  }
}
private int createSurfaceControl(outSurface, result, win, winAnimator) {
  surfaceController = winAnimator.createSurfaceLocked(win.mAttrs.type, win.mOwnerUid);
  surfaceController.getSurface(outSurface);
  return result;
}
//WindowStateAnimator.java
WindowSurfaceController createSurfaceLocked(windowType, ownerUid) {
  mSurfaceController = new WindowSurfaceController(mSession.mSurfaceSession, attrs.getTitle().toString,..,..,this, windowType, ownerUid);
  return mSurfaceController;
}
//WindowSurfaceController.java
void getSurface(outSurface) {
  outSurface.copyFrom(mSurfaceControl);
}
```

##### 10.4.2 Surface的跨进程

ViewRootImpl：一开始就分配了一个空的Surface

WindowStateAnimator：通过createSurfaceLocked方法生成一个真正有效的Surface对象，并交由SurfaceControl管理

```java
//Surface.java
public void copyFrom(SurfaceControl other) {
  //获取native层的Surface对象的指针
  long newNativeObject = nativeGetFromSurfaceControl(other.mNativeObject);
  synchronized(mLock) {
    if(mNativeObject != 0) {
      //如果以前保留了指针，需要释放
      nativeRelease(mNativeObject);
    }
    //讲mNativeObject设置为newNativeObject
    setNativeObjectLocked(newNativeObject);
  }
}
//SurfaceControl.java
private SurfaceControl(SurfaceSession sesson, String name, ...) {
  //native层的SurfaceControl
  mNativeObject = nativeCreate(session, name, ...);
}
```

```c++
//android_view_SurfaceControl.cpp
static jlong nativeCreate(env, clazz, sessionObj, nameStr, xxx) {
  sp<SurfaceComposerClient> client(android_view_SurfaceSession_getClient(env, sessionObj));
  SurfaceControl *parent = reinterpret_cast<SurfaceControl*>(parentObject);
  sp<SurfaceControl> surface;
  status_t err = client->createSurfaceChecked(
    String8(name.c_str()), w, h, format, &surface, flags, parent, windowType, ownerUid);
  if (err == NAME_NOT_FOUND) {
    jniThrowException(env, "java/lang/IllegalArgumentException", NULL);
    return 0;
  } else if (err != NO_ERROR) {
    jniThrowException(env, OutOfResourcesException, NULL);
    return 0;
  }

  surface->incStrong((void *)nativeCreate);
  return reinterpret_cast<jlong>(surface.get());
}

//android_view_Surface.cpp
static jlong nativeGetFromSurfaceControl(JNIEnv* env, jclass clazz, jlong surfaceControlNativeObj) {
  sp<SurfaceControl> ctrl(reinterpret_cast<SurfaceControl*>(surfaceControlNativeObj));
  sp<Surface> surface(crrl->getSurface);
  if(surface != null) {
    surface->incStrong(&sRefBaseOwner);
  }
  return reinterpret_cast<jlong>(surface.get());
}
```

ViewRootImpl中的Surface是空的，然后通过relayoutWindow方法，通过Surface.copyFrom方法，读取出来SurfaceControl的mNativeObject然后再设置回去

##### 10.4.3 Surface的业务操作

SurfaceControl.open(close)Transaction，对应调用了native本地方法：SurfaceComposerClient::openGlobalTransaction();

起核心语句在于调用了ISurfaceComposer::setTransactionState()这个接口。openGlobalTransaction和closeGlobalTransaction之间的所有设置Surface属性的操作，都不是及时生效的，而是要等到事物关闭后才统一告知SurfaceFlinger。这样对于频繁变更属性的地方，可以提高效率以及避免画面出现不稳定

#### 10.5 performLayoutAndPlaceSurfacesLockedInner

API23之前WMS.performLayoutAndPlaceSurfacesLockedInner，API23之后使用WindowSurfacePlacer来代替

#### 10.6 窗口大小的计算过程

ViewRootImpl.performTraversals->Session.relayout->WMS.relayoutWindow->updateFocusedWindowLocked->DisplayContent.performLayout

```java
//DisplayContent.java
void performLayout(boolean initial, boolean updateInputWindows) {
  //...
  mService.mPolicy.beginLayout();
  forAllWindows(mPerformLayout, true);
  forAllWindows(mPerformLayoutAttached, true);
}
private final Consumer<WindowState> mPerformLayoutAttached = w -> {
  if(w.mLayoutAttached) {
   	mService.policy.layoutWindowLw(); 
  }
}
private final Consumer<WindowState> mPerformLayout = m-> {
  //忽略
  mService.policy.layoutWindowLw();
}
//PhoneWindowManager.java
//计算显示区域
public void beginLayoutLw() {
  if(displayFrames.mDisplayId == DEFAULT_DISPLAY) {
    boolean updateSysUiVisbility = layoutNavigationBar();
    updateSysUiVisibility |= layoutStatusBar();
  }
  if(updateSysUiVisibility){
    updateSystemUiVisibilityLw();
  }
  layoutScreenDecorWindows();
}
public void layoutWindowLw() {
  if(type == TYPE_WALLPAPER) {
    layoutWallpaper();
  }
}
```

#### 10.7 启动窗口的添加与销毁

##### 10.7.1 启动窗口的添加

AppWindowContainerController.addStartingWindow<-ActivityRecord.showStartingWindow<-ActivityStack.startActivityLocked

```java
//ActivityStack.java
void startActivityLocked() {
  if(SHOW_APP_STARTING_PREVIEW && doShow) {
    r.showStartingWindow();
  }
}
//ActivityRecord.java
void showStartingWindow() {
  final boolean shown = mWindowContainerController.addStartingWindow();
  if(shown) {
    mStartingWindowState = STATING_WINDOW_SHOWN;
  }
}
//AppWindowContainerController.java
void addStartingWindow() {
  mContainer.startingData = new SplashScreenStaringData();
  sheduleAddStaringWindow();
}
void scheduleAddStartingWindow() {
  mService.mAnimationHandler.postAtFrontOfQueue(mAddStartingWindow);
}
private final Runnable mAddStartingWindow = new Runnable() {
  StartingSurface surface = startingData.createStartingSurface(container);
}
//SplashScreenStaringData.java
StartingSurface createStartingSurface(AppWindowToken atoken) {
  return mService.mPolicy.addSplashScreen(atoken.token,xxx);
}
//PhoneWindowManager.java
public StartingSurface addSplashScreen() {
 	final PhoneWindow win = new PhoneWindow(context);
  win.setIsStartingWindow(true);
  win.setType(WindowManager.LayoutParams.TYPE_APPLICATION_STARTING);
  //根据主题给win添加一个contentView并设置北京
  addSplashScreenContent(win, context);
  wm.addView(win.getDecorView())
}
```

##### 10.7.2 启动窗口的销毁



