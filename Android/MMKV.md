# MMKV

## 1. 获取`java`层`MMKV`对象

`MMKV`实现了SharedPreference.Editor接口

```kotlin
//MMKV文件所在地址
//默认为"${context.getFilesDir().absolutePath}/mmkv"
val dir = MMKV.initialize(context)
val mmkv = MMKV.defaultMMKV()	
```

其中`MMKV.initialize`方法用来初始化并通过`ReLinker`来加载`so`库，设置`rootDir`并且调用`jniInitialize方法`

```c++
//native-bridge.cpp
MMKV_JNI void jniInitialize(JNIEnv* env, jobject obj, jstring rootDir) {
  if(!rootDir) {
    return;
  }
  const char *kstr = env->GetStringUTFChars(rootDir, nullptr);
  if(kstr) {
    MMKV::initializeMMKV(kstr);
    env->ReleaseStringUTFChars(rootDir, kstr);
  }
}
//MMKV.cpp
void MMKV::initializeMMKV(const std::string &rootDir) {
    static pthread_once_t once_control = PTHREAD_ONCE_INIT;
  //调用initialize方法
    pthread_once(&once_control, initialize);
  //static 变量，全局
    g_rootDir = rootDir;
    char *path = strdup(g_rootDir.c_str());
    if (path) {
      //创建文件夹，貌似没必要放在native层
        mkPath(path);
        free(path);
    }
    MMKVInfo("root dir: %s", g_rootDir.c_str());
}

void initialize() {
    g_instanceDic = new unordered_map<std::string, MMKV *>;
  //用于线程锁
    g_instanceLock = ThreadLock();
    //testAESCrypt();
    MMKVInfo("page size:%d", DEFAULT_MMAP_SIZE);
}

```

接下来看`MMKV`对象。一般来说，都是通过持有native层的指针来进行操作

```java
public static MMKV defaultMMKV() {
  if (rootDir == null) {
    throw new IllegalStateException("You should Call MMKV.initialize() first.");
  }
	//native层的MMKV对象的指针
  long handle = getDefaultMMKV(SINGLE_PROCESS_MODE, null);
  return new MMKV(handle);
}
```

接下来转至`native`层

```c++
//逻辑很简单，通过MMKV::defaultMMKV方法获取MMKV指针，然后返回
MMKV_JNI jlong getDefaultMMKV(JNIEnv *env, jobject obj, jint mode, jstring cryptKey) {
  MMKV *kv = nullptr;

  if (cryptKey) {
    string crypt = jstring2string(env, cryptKey);
    if (crypt.length() > 0) {
      kv = MMKV::defaultMMKV((MMKVMode) mode, &crypt);
    }
  }
  if (!kv) {
    kv = MMKV::defaultMMKV((MMKVMode) mode, nullptr);
  }

  return (jlong) kv;
}
//DEFAULT_MMAP_ID为mmkv.default
//DEFAULT_MMAP_SIZE=getpagesize，内存分页大小
MMKV *MMKV::defaultMMKV(MMKVMode mode, string *cryptKey) {
    return mmkvWithID(DEFAULT_MMAP_ID, DEFAULT_MMAP_SIZE, mode, cryptKey);
}

MMKV *MMKV::mmkvWithID(
    const std::string &mmapID, int size, MMKVMode mode, string *cryptKey, string *relativePath) {

    if (mmapID.empty()) {
        return nullptr;
    }
  //自动锁，通过构造函数调用lock，析构函数调用unlock方式来对方法加锁，用于线程同步
    SCOPEDLOCK(g_instanceLock);
    //根据mmapID和relativePath获取key
    auto mmapKey = mmapedKVKey(mmapID, relativePath);
  //根据key从unordered_map中获取缓存的MMKV指针（initialize时创建的map）
    auto itr = g_instanceDic->find(mmapKey);
    if (itr != g_instanceDic->end()) {
      //找到了直接返回
        MMKV *kv = itr->second;
        return kv;
    }
    if (relativePath) {
        auto filePath = mappedKVPathWithID(mmapID, mode, relativePath);
        if (!isFileExist(filePath)) {
            if (!createFile(filePath)) {
                return nullptr;
            }
        }
        MMKVInfo("prepare to load %s (id %s) from relativePath %s", mmapID.c_str(), mmapKey.c_str(),
                 relativePath->c_str());
    }
  //创建MMKV指针，放到g_instanceDic中，返回指针
    auto kv = new MMKV(mmapID, size, mode, cryptKey, relativePath);
    (*g_instanceDic)[mmapKey] = kv;
    return kv;
}
```

`native`层的`MMKV`对象是所有功能实现的基础。`MMKV`提供了两种方式的内存持久化方式，普通文件方式`MMAP_FILE`和`ashmem`方式`MMAP_ASHMEM`，然后将文件`mmap`至内存。二者都是通过`MMapedFile`来实现

```c++
//file或者ashmem的fd保存至m_fd, SIZE保存至m_segmentSize,  mmap后的ptr保存至m_segmentPtr
MmapedFile::MmapedFile(const std::string & path, size_t size, bool fileType)
  :m_name(path), m_fd(-1), m_segmentPtr(nullptr), m_segmentSize(0), mfileType(fileType) {
    //MMAP_FILE形式，普通文件mmap
    if(m_fileType == MMAP_FILE) {
      m_fd = open(m_name.c_str(), O_RDRW|O_CREAT, S_IRWXU);
      //忽略进程锁、线程锁
      struct stat st = {};
      //这段话为啥不同Math.max
      if(fstat(m_fd, &st) != -1) {
        m_segmentSize = static_cast<size_t>(st.st_size);
      }
      if(m_segmentSize < DEFAULT_MMAP_SIZE) {
        m_segmentSize = static_cast<size_t>(DEFAULT_MMAP_SIZE);
        //将m_fd指向的文件修改为大小DEFAULT_MMAP_SIZE
        ftruncate(m_fd, m_segmentSize);
        //将文件重置为空，所有字节都为0
        zeroFillFile(m_fd, 0, m_segmentSize);
      }
      //mmap使得进程之间通过映射同一个普通文件实现内存共享。普通文件被映射到进程地址空间后，进程可以像访问普通文件一样对文件进行调用，不需要read、write操作
      //mmap操作提供了一种机制，让用户程序直接访问设备内存，这种机制，相比较在用户空间和内核空间互相拷贝数据，效率更高
      //mmap(start, length, prot, flags, fd, offset);
      //start如果是nullptr，那么就交由系统决定映射的内存起始地址
      m_segmentPtr =
                (char *) mmap(nullptr, m_segmentSize, PROT_READ | PROT_WRITE, MAP_SHARED, m_fd, 0);
    }else {
      //MMAP_ASHMEM 匿名共享内存形式
      m_fd = AsharedMemory_create(m_name.c_str(), size);
			m_segmentSize = static_cast<size_t>(size);
      m_segmentPtr = (char*)mmap(nullptr, m_segmentSize, PROT_READ|PROT_WRITE, MAP_SHARED, m_fd, 0);
    }
  }

//MmapFiled.cpp
typedef int (*AShmem_create_t)(const char *name, size_t size);
int ASharedMemory_create(const char *name, size_t size) {
    int fd = -1;
    if (g_android_api >= __ANDROID_API_O__) {
      //加载libandroid.so
      //dlopen(libandroid.so, RTLD_LAZY | RTLD_LOCAL);
        static auto handle = loadLibrary();
        static AShmem_create_t funcPtr =
            (handle != nullptr)
                ? reinterpret_cast<AShmem_create_t>(dlsym(handle, "ASharedMemory_create"))
                : nullptr;
        if (funcPtr) {
            fd = funcPtr(name, size);
            if (fd < 0) {
                MMKVError("fail to ASharedMemory_create %s with size %z, errno:%s", name, size,
                          strerror(errno));
            }
        } else {
            MMKVWarning("fail to locate ASharedMemory_create() from loading libandroid.so");
        }
    }
    if (fd < 0) {
        fd = open(ASHMEM_NAME_DEF, O_RDWR);
        if (fd < 0) {
            MMKVError("fail to open ashmem:%s, %s", name, strerror(errno));
        } else {
            if (ioctl(fd, ASHMEM_SET_NAME, name) != 0) {
                MMKVError("fail to set ashmem name:%s, %s", name, strerror(errno));
            } else if (ioctl(fd, ASHMEM_SET_SIZE, size) != 0) {
                MMKVError("fail to set ashmem:%s, size %zu, %s", name, size, strerror(errno));
            }
        }
    }
    return fd;
}
```

## 2. 数据操作`put`/`get`

### 2.1 `putBoolean(String key, boolean value)`

```java
public Editor putBoolean(String key, boolean value) {
  //encodeBool转至native层
  encodeBool(nativeHandle, key, value);
  return this;
}
@Override
public boolean commit() {
  sync(true);
  return true;
}
@Override
public void apply() {
  sync(false);
}
```

`native`层处理数据保存

```c++
//native-bridge.cpp
MMKV_JNI jboolean encodeBool(JNIEnv* env, jobject , jlong handle, jstring oKey, jboolean value) {
  MMKV* kv = reinterpret_cast<MMKV*>(handle);
  if(kv && oKey) {
    string key = jstring2string(env, oKey);
    return (jboolean)kv->setBool(value, key);
  }
  return (jboolean) false;
}
```

`MMKV`处理`boolean`值

```c++
//MMKV.cpp
bool MMKV::setBool(bool value, const std::string & key) {
  if(key.emty()) {
    return;
  }
  //size = 1
  size_t size = pbBoolSize(value);
  MMBuffer data(size);
  CodedOutputData output(data.getPtr(), size);
  output.writeBool(value);
  //通过std::move方法转移，之后data原来的值会传至方法setDataForKey的形参data中，而原来的data的值变为nullptr?
  return setDataForKey(std::move(data), key);
}
```

`MMBuffer`是用来保存`value`的地方

`CodedOutputData`是用来写入数据到`MMBuffer`中的`ptr`：通过`malloc`分配`length`字节的内存`ptr`，然后将数据写入`ptr`

```c++
//MMBuffer.cpp
MMBuffer::MMBuffer(size_t length) : ptr(nullptr), size(length), isNoCopy(MMBufferCopy) {
    if (size > 0) {
        ptr = malloc(size);
    }
}
//CodedOutputData.cpp
void CodedOutputData::writeBool(bool value) {
    this->writeRawByte(static_cast<uint8_t>(value ? 1 : 0));
}
void CodedOutputData::writeRawByte(uint8_t value) {
    if (m_position == m_size) {
        MMKVError("m_position: %d, m_size: %zd", m_position, m_size);
        return;
    }
    m_ptr[m_position++] = value;
}
```

接下来需要将`CodedOutputData`跟`MMKV`关联起来，通过`setDataForKey`，将`key`和`value`保存至`MMKV`的`m_output`中，并在`map m_dic`中保存一个键值对

```c++
//MMKV.cpp
bool MMKV::setDataForKey(MMBuffer &&data, const std::string & key) {
  //加锁以及验证，忽略
  checkLoadData();
  auto ret = appendDataWithKey(data, key);
  if(ret) {
    m_dic[key] = std::move(data);
    m_hasFullWriteback = false;
  }
  return ret;
}
//将数据写入到m_output中，也是一个CodedOutputData
bool MMKV::appendDataWithKey(const MMBuffer &data, const std::string &key) {
    size_t keyLength = key.length();
    // size needed to encode the key
    size_t size = keyLength + pbRawVarint32Size((int32_t) keyLength);
    // size needed to encode the value
    size += data.length() + pbRawVarint32Size((int32_t) data.length());
    SCOPEDLOCK(m_exclusiveProcessLock);
    bool hasEnoughSize = ensureMemorySize(size);
    if (!hasEnoughSize || !isFileValid()) {
        return false;
    }
    writeAcutalSize(m_actualSize + size);
    m_output->writeString(key);
    m_output->writeData(data); // note: write size of data
    auto ptr = (uint8_t *) m_ptr + Fixed32Size + m_actualSize - size;
    if (m_crypter) {
        m_crypter->encrypt(ptr, ptr, size);
    }
    updateCRCDigest(ptr, size, KeepSequence);
    return true;
}
```

### 2.2 `commit\apply`保存数据

`commit`和`apply`都是调用`native`层的`sync`方法，差别在于`jboolean sync`参数：`commit`为`true`，`apply`为`false`

```c++
MMKV_JNI void sync(JNIEnv* env, jobject instance, jboolean async) {
  //instance其实是Java层的MMKV对象，通过handle获取到native层的MMKV
  //其实可以在Java层的native方法中传入handle
	MMKV* kv = getMMKV(env, instance);
  if(kv) {
    kv->sync((bool)sync);
  }
}

void MMKV::sync(bool sync) {
    SCOPEDLOCK(m_lock);
    if (m_needLoadFromFile || !isFileValid()) {
        return;
    }
    SCOPEDLOCK(m_exclusiveProcessLock);
    auto flag = sync ? MS_SYNC : MS_ASYNC;
  //MMKV其实是通过mmap将内容暂时保存在内存，然后要写入文件，
  //关键在于msync函数
  //也可以通过munmap，但是这样就将m_fd的关联失效了
    if (msync(m_ptr, m_size, flag) != 0) {
        MMKVError("fail to msync[%d] [%s]:%s", flag, m_mmapID.c_str(), strerror(errno));
    }
}
```

### 2.3 `getBoolean`方法获取值

```c++
MMKV_JNI jboolean decodeBool(JNIEnv* env, jobject, jlong handle, jstring oKey, jboolean defaultValue) {
  MMKV* kv = reinterpret_cast<MMKV*>(handle);
  if(kv && oKey) {
    string key = jstring2string(env, oKey);
    return (jboolean) kv->getBoolForKey(key, defaultValue);
  }
  return defaultValue;
}

bool MMKV::getBoolForKey(const std::string &key, bool defaultValue) {
    if (key.empty()) {
        return defaultValue;
    }
    SCOPEDLOCK(m_lock);
    auto &data = getDataForKey(key);
    if (data.length() > 0) {
        CodedInputData input(data.getPtr(), data.length());
        return input.readBool();
    }
    return defaultValue;
}
//从m_dic中根据Key获取对应的MMBuffer
const MMBuffer &MMKV::getDataForKey(const std::string &key) {
    checkLoadData();
    auto itr = m_dic.find(key);
    if (itr != m_dic.end()) {
        return itr->second;
    }
    static MMBuffer nan(0);
    return nan;
}
```

### 2.4 `Parcelable`对象处理

首先将`Parcelable`对象通过`Parcel`转化成字节数组

```java
public boolean encode(String key, Parcelable value) {
  Parcel source = Parcel.obtain();
  value.writeToParcel(source, value.describeContents());
  byte[] bytes = source.marshall();
  source.recycle();
  return encodeBytes(nativeHandle, key, bytes);
}
```

## 3. 多进程

[官方介绍](https://github.com/Tencent/MMKV/wiki/android_ipc)