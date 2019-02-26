### 第06章-Ashmem匿名共享内存系统

ashmem可以划分为若干小块，基于临时临时文件系统tmpfs实现。

传统的Linux提供通过一个整数来标志一块共享内存，Android则使用文件描述符。这样可以**方便的将它映射到内存空间，也能使用Binder进程间通信机制来传输这个文件描述符，从而实现在不同进程共享一块匿名内存**。

#### 6.1 Ashmem驱动程序

主要使用到了ashmem_area，ashmem_range和ashmem_pin三个结构体

##### 6.1.1 基础数据结构

- ashmem_area：描述一块匿名共享内存

  ```c++
  #define ASHMEM_NAME_PREFIX "dev/ashmem/"
  #define ASHMEM_NAME_PREFIX_LEN (sizeof(ASHMEM_NAME_PREFIX)-1)
  #define ASHMEM_FULL_NAME_LEN (ASHMEM_NAME_PREFIX_LEN + ASHMEM_NAME_LEN)
  struct ashmem_area {
    char name[ASHMEM_FULL_NAME_LEN];//一块匿名共享内存的名称，会被写入/proc/<pid>/maps
    struct list_head unpinned_list;//解锁内存块列表，内存小块的内存地址互不交互
    struct file* file;//临时文件系统tmpfs中的一个文件
    size_t size;//file的大小
    unsigned long prot_mask;//file的访问保护位，默认为PROT_MASK(PROT_EXEC|PROt_READ|PROT_WRITE)
  };
  ```

- ashmem_range：一小块处于解锁状态的内存。都是从一块匿名共享内存中划分出来的，通过成员变量unpinned链入到宿主匿名共享内存列表unpinned_list中。

  ```c++
  struct ashmem_range {
    struct list_head lru;//链入到全局ashmem_lru_list中，由于解锁状态的内存都是不再需要使用的，所以系统会按LRU原则回收保存在ashmem_lru_list中的内存块
    struct list_head unpinned;
    struct ashmem_area* asma;
    size_t pgstart;//解锁状态的内存的开始地址
    size_t pgend;//解锁状态的内存的结束地址
    unsigned int purged;//是否已被回收。如果是，那么ASHMEME_WAS_PURGED
  };
  ```

- ashmem_pin：Ashmem驱动程序定义的IO控制命令ASHMEM_PIN和ASHMEM_UNPIN的参数，用来描述一小块即将被锁定或解锁的内存

  ```c++
  struct ashmem_pin {
    _u32 offset;//在宿主ashmem中的偏移值
    u32 len;//要被锁定或解锁的大小
  };
  ```

##### 6.1.2 Ashmem设备的初始化过程

```c++
//drivers/staging/android/ashmem.c
static struct miscdevice ashmem_misc = {
  .minor = MISC_DYNAMIC_MINOR,
  .name = "ashmem",
  .fops = &ashmem_fops,
};
static int __init ashmem_init(void) {
  //...
  misc_register(&ashmem_misc);
  //向内存管理系统注册一个内存回收函数
  register_shrinker(&ashmem_shrinker);
}
```

##### 6.1.3 ashmem设备文件的打开过程

```c++
//drivers/staging/android/ashmem.c
static int ashmem_open(struct inode* inode, struct file* file) {
  struct ashmem_area asma = xxx///创建并设置ashmem_area参数;
  file->private_data = asma;
}
```

##### 6.1.4 ashmem设备文件的内存映射过程

```c++
//drivers/stating/android/ashmem.c
static int ashmem_mmap(struct file* file, struct vm_area_struct* vma) {
  struct ashmem_area* asma = file->private_data;
  //在tmpfs中创建临时文件并保存值asma的成员变量file中
  shmem_file_setup();
  //设置映射文件以及内存操作方法&shmem_vm_ops;
  shmem_set_file();
}
```

##### 6.1.5 ashmem内存块的锁定和解锁过程

处于解锁阶段的内存块是可以被内存管理系统回收的。通过ASHMEM_PIN和ASHMEM_UNPIN来锁定和解锁一块ashmem。

##### 6.1.6 ashmem内存块的回收过程

```c++
//drivers/staging/android/ashmem.c
static int ashmem_release() {
  //释放ashmem
  ashmem_release();
}
```

#### 6.2 运行时库cutils的ashmem访问接口

```c++
//system/core/libcutils/ashmem-dev.cpp
//ashmem_create_region, ashmem_pin_region, ashmem_unpin_region, ashmem_set_prot_region, ashmem_get_size_region
```

- ashmem_create_region：请求ashmem驱动程序创建一块ashmem，其中name和size分别表示ashmem的名称和大小
- ashmem_pin_region：锁定一小块ashmem。fd是前面打开设备文件/dev/ashmem所得到的一个文件描述符；offset和len用来指定要锁定的内存块在其宿主ashmem中的偏移和长度。
- ashmem_unpin_region：解锁一小块ashmem。fd是前面打开设备文件/dev/ashmem所得到的一个文件描述符；offset和len用来指定要锁定的内存块在其宿主ashmem中的偏移和长度。
- ashmem_set_prot_region：修改ashmem的访问保护位。fd是前面打开设备文件/dev/ashmem所得到的一个文件描述符；prot为PROT_EXEC, PROT_READ,PROT_WRITE或其组合值。
- ashmem_get_size_region：获取一块ashmem的大小。fd是前面打开设备文件/dev/ashmem所得到的一个文件描述符。

#### 6.3 ashmem的C++访问接口

如果一个进程需要与其他进程共享一块完整的ashmem，使用MemoryHeapBase来创建这块ashmem；如果只希望共享一部分，使用MemoryBase来创建。

##### 6.3.1 MemoryHeapBase

如果一个进程需要与其他进程共享一块完整的ashmem，使用MemoryHeapBase来创建这块ashmem。

//todo

##### 6.3.2 MemoryBase

MemoryBase类内部有一个类型位IMemoryHeap的强指针mHeap，指向MemoryHeapBase服务。MemoryBase内部维护的ashmem只是MemoryHeapBase维护的ashmem的一小块，通过offset和size来描述。

##### 6.3.3 应用实例

//todo

#### 6.4 ashmem的Java访问接口

##### 6.4.1 MemoryFile

native层代码在frameworks/base/core/jni/android_os_MemoryFile.cpp。通过SharedMemory.nCreate方法创建一个ashmem，native层代码在frameworks/base/core/jni/android_os_SharedMemory.cpp

#### 6.5 ashmem的原理

