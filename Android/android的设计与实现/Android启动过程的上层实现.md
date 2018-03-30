## Android启动过程的上层实现

### 主要过程

> 1. bootloader加载内核
> 2. 内核启动1号进程init（system/core/init/init.cpp）
> 3. init执行action和service (通过Parser解析import, on ,service)

### Android启动流程

> 1. init启动的核心daemon服务包括Android世界的第一个dalvik虚拟机zygote。
> 2. zygote中定义了一个socket，用于接收AMS启动应用程序的请求。
> 3. zygote通过fork系统调用创建system_server
> 4. 在system_server的进程的init1和init2阶段分别启动Native System Service和Java System Service。
> 5. 系统服务启动后会自动注册到ServiceManager中，用于Binder通信。
> 6. AMS进入systemReady状态。
> 7. 在systemReady状态，AMS会和zygote的socket通信，请求启动Home。
> 8. zygote收到AMS的连接请求，执行runSelectLoopMode处理请求。
> 9. zygote处理请求通过forkAndSpecialize启动新的应用进程，并最终启动Home。

### init.rc中zygote相关的信息

service zygote /system/bin/app_process -Xzygote /system/bin --zygote --start-system-server

> 1. app_process调用方法在frameworks/base/cmds/app_process/app_main.cpp中。
> 2. app_process.main方法，解析参数后最终执行到runtime.start("com.android.internal.ZygoteInit"，args , zygote);
> 3. runtime为AppRuntime, AndroidRuntime的子类。AppRuntime.start方法会触发startVm, startReg, 接着会由jni层回调ZygoteInit.main方法

### AppRuntime.startVm, AppRuntime.startReg

>1. startVm实际调用了AndroidRuntime.startVm，用来解析系统属性，最后通过jni.h的JNI_CreateJavaVm方法创建虚拟机。
>2. startReg实际调用了AndroidRuntime.startReg，通过将gRegJNI数组中的RegJNIRec的mProc方法指针来注册JNI方法。

### ZygoteInit.main

> 1. 通过ZygoteServer.registerServerSocket来建立与zygote的socket连接。
> 2. 调用preload方法，其中会调用preloadClasses()去加载system/etc/preloaded-classes中的类。接着preloadResources，会加载preloaded_drawables, preloaded_color_state_list, preloaded_freeeform_multi_window_drawables(按需)。nativePreloadAppProcessHALs，ZygoteInit.cpp->GraphicBufferMapper.cpp->Gralloc2.cpp->android::hardware::preloadPassthroughService方法。preloadOpenGL。preloadSharedLibraries，包括android, compiler_rt jnigraphics三个库。接着preloadextResources。WebViewFactory.prepareWebViewInZygote->WebViewLibraryLoader.reserveAddressSpaceInZygote
> 3. 接着调用ZygoteInit.forkSystemServer->Zygote.forkSystemServer。如果只在child process，调用handleSystemServerProcess->ZygoteInit.zygoteInit。zygoteInit方法会掉哦那个RuntimeInit.redirectLogStreams来重定向log输出，RuntimeInit.commonInit来处理一些其他的初始化事情，ZygoteInit.nativeZygoteInit->AndroidRuntime.com_android_internal_os_ZygoteInit_nativeZygoteInit->AppRuntime.onZygoteInit->AppRuntime.onZygoteInit->ProcessState.startThreadPool开启Binder通信通道。
> 4. RuntimeInit.applicationInit->findStaticMain->com.android.server.SystemServer.main和返回MethodAndArgsCaller对象，该对象实现了Runnable接口

### SystemServer.main

> SystemServer.run->System.loadLibrary("android_servers")，createSystemContext启动ActivityThread(), 注册SystemServiceManager，startBootstrapService，startCoreServices，startOtherServices
> 启动服务都是通过SystemServiceManager.startService方法：通过constructor反射生成SystemService对象，然后添加至mServices中，再调用onStart方法

### SystemServer.startBootstrapService
> startBootstrapServer用于启动紧要的服务。
> 包括Installer, DeviceIdentifiersPolicyService, AMS, PowerManagerService, RecoverySystemService, LightsService, DisplayManagerService, 接着创建PackageManagerService并添加到ServiceManager中，创建OtaDexoptService，启动UserManagerService, OverlayManagerService, 最后调用startSensorService。

### SystemServer.startCodeServices
> DropBoxManagerService, BatteryService, UsageStatsService, UsageStatsManagerInternal, WebViewUpdateService

### SystemServer.startOtherServices
> KeyAttestationApplicationIdProviderService, KeyCHainSystemService, SchedulingPolicyService, TelecomLoaderService, TelephonyRegistry, EntropyMixer，AccountManagerService.Lifecycle(onStart方法中生成AccountManagerService对象并发布)， ContentService(同上), installSystemProviders发布系统ContentProvider，VibratorService, ConsumerIrService, AlarmManagerService, 初始化Watchdog(Watchdog.getInstance().init()), InputManagerService, WindowManagerService.main(), VrManagerService, BluetoothService, IpConnectivityMetrics, PinerService, InputMethodManagerService(类似AccountManagerService), AccessibilityManagerService, StorageManagerService(类似AccountManagerServicec), UiModeManagerService, LOckSettingService.Lifecycle, PersistentDataBlockService, OemLockService, DeviceIdleController, DevicePolicyManagerService, StatusBarManagerService, ClipboardService, NetworkManagementService.create(), IpService.create(), TextServicesManagerService.Lifecycle, NetworkScoreService,NetworkStatsSErvice.create(), NetworkPolicyManagerService, WifiService, WifiScanningService, RttService, WifiAwareService, WifiP2pService, LowpanService, EthernetService, ConnectivityService, NsdService.create(), UpdateLockService,  NotificationManagerService, DeviceStorageMonitorService, LocalManagerService, CountryDetectorService, SearchManagerService.Lifecycle, WallpaperManagerService.Lifecycle, AudioService.Lifecycle, BroadcastRadioService, DockObserver, ThermalObserver, MidiService.Lifecycle, UsbService.Lifecycle, SerialService, HardwarePropertiesManagerService, TwilightService, NightDisplayService, JobSchedulerService, SoundTriggerService, TrustManagerService, BackupManagerService.Lifecycle, AppWidgetService, VoiceInteractionManagerService, GestureLauncherService, SensorNotificationService,  ContextHubSystemService, DiskStatsService, RulesManagerService.Lifecycle, NetworkTimeUpdateService, CommonTimeManagementService, EmergencyAffordanceService, DreamManagerService, GraphicsStatsService,  CoverageService, PrintManagerService, 
CompanionDeviceManagerService, RestrictionsManagerService, MediaSessionService, HdmiControlService, TvInputManagerService, MediaResourceMonitorService, TvRemoteService, MediaRouterService, FingerprintService, BackgroundDexOptService.schedule(), PruneInstantAppsJobService.schedule(), ShortcutService, LauncherAppsService, MediaProjectionManagerService, WearConnectivityService, WearDisplayService, WearTimeService, WearLeftyService, CameraServiceProxy, MmsServiceBroker, AutofillManagerService, 接下来就是某些服务的systemReady和systemRunning方法的调用

### ZygoteServer.runSelectLoop
> 返回一个Runnable(ZygoteConnection.processOneCommand -> ZygoteInit.zygoteInit-> RuntimeNit.applicationInit, 最后就是启动传入的参数中的class的main方法)

### SystemServer.startOtherServices中AMS.systemReady

> AMS.startSystemUi -> ActivityStack.resumeTopActivityLocked->AMS.startHomeActivityLocked->ActivityStack.startActivityLocked->resumeTopActivityLocked->startSpecificActivityLocked->startProcessLocked->Process.start->ZygoteProcess.start->ZygoteProcess.startViaZygote->zygoteSendArgsAndGetResult->建立与ZygoteServer的socket连接，此时ZygoteServer正处在runSelectLoop下，最终还是通过RuntimeInit.applicationInit方法调用ActivityThread.main方法
>
> 最终还会回到RuntimeInit.applicationInit，只不过这次传递的是android.app.ActivityThread.main方法来启动Home。