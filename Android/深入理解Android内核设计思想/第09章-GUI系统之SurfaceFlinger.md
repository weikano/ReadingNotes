### 第09章-GUI系统之SurfaceFlinger

#### 9.1 OpenGL ES与EGL

#### 9.2 Android的硬件接口-HAL

##### 1. 硬件抽象接口

每一个硬件魔抗都需要一个岷城为HAL_MODULE_INFO_SYM的变量，比如GPS中的实现

```c++
struct hw_module_t HAL_MODEUL_INFO_SYM = {
  .tag = HARDWARE_MODULE_TAG, //必须以此作为tag
  .version_major = 1,
  .version_minor = 0,
  .id = GPS_HARDWARE_MODULE_ID,
  .name = "Goldfish GPS Module",
  .author = "The Android Open Source Project",
  .methods = &gps_module_methods,
};
```

其次，此变量的结构体要以hw_module_t开头

##### 2. 接口的稳定性

##### 3. 灵活的使用方法

#### 9.3 Android终端显示设备的化身-Gralloc与Framebuffer

Android系统中，Framebuffer提供的设备文件节点是/dev/graphics/fb*。理论上支持多个屏幕，所以fb按数字序号进行排序，即fb0、fb1等.

##### 1. Gralloc模块的加载

##### 2. Gralloc提供的接口

//TODO 太复杂,以后再看
