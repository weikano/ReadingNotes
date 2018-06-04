### 一、动态内存分配
使用动态内存分配的基本步骤：
1. malloc类的方法分配内存
2. 用这些内存支持应用程序
3. free函数释放内存
```C
int *pi = (int*)malloc(sizeof(int));
*pi=5;
cout<<*pi<<endl;
free(pi);
```

### 二、动态内存分配函数
- malloc：从堆上分配内存
- realloc：从之前分配的内存块的基础上，将内存重新分配为更大或更小的部分
- calloc：从堆上分配内存并清零
- free：将内存块返回堆

#### 1. 使用malloc函数
```C
//返回值为void指针，所以一般都需要强转至声明的指针类型。如果内存不足或者分配失败，会返回NULL。
//另外此函数不会清空或者修改内存，所以我们认为新分配的内存包含垃圾数据。
void* malloc(size_t);
//使用如下
int *pi = (int*)malloc(sizeof(int));
```
#### 2. 使用calloc函数
```C
//calloc函数会根据numElements和elementSize的乘积来分配内存，并返回指向内存第一个字节的指针。如果失败会返回NULL。calloc函数会内存清零。
void* calloc(size_t numElements, size_t elementSize);
//使用如下
int* pi = (int*)calloc(10, sizeof(int));
//用malloc+memset实现相同效果
int* pi=malloc(5*sizeof(int));
memset(pi,0, 5*sizeof(t));
```
#### 3. 使用realloc函数
```C
//realloc返回指向内存块的指针。第一个参数指向原内存块的指针，第二个是请求的大小。如果第二个参数为0，那么就释放内存。如果无法分配内存，那么原来的内存块就保持布标，返回值为NULL。
void* realloc(void* ptr, size_t size);
```
#### 4.alloca函数和变长数组

### 三、用free函数释放内存
```C
//free之后，仍然可能包含原值，这种情况称为迷途指针
void* free(void* ptr);
```
#### 1. 将已释放的指针赋值为NULL
```C
int* p = malloc(sizeof(int));
*p = 100;
cout<<*p<<endl;
free(p);
//显示的给释放后的指针赋值为NULL，后续使用这种指针时会造成运行期异常
p=NULL;
```
#### 2. 重复释放
```C
int* pi=malloc(sizeof(int));
*pi=5;
free(pi);
//重复释放会导致运行期异常
free(pi);

int* p1=malloc(sizeof(int));
int* p2=p1;
free(p1);
//p1和p2是同一个指针，重复释放仍然会导致异常
free(p2);
```

#### 3. 堆和系统内存
#### 4. 程序结束前释放内存

### 四、迷途指针
**如果内存已经释放，而指针还在引用原始内存，这样的指针就被成为迷途指针：**
- 如果访问内存，则行为不可预测
- 如果内存不可访问，则是段错误
- 潜在的安全威胁
- 访问已释放的内存
- 返回的指针指向的是上次函数调用中的自动变量

#### 1. 迷途指针示例
类似重复释放中的代码：pi被free掉，仍然可以访问；p1被free掉，通过p2访问；还有如下代码块中：
```C
int* pi;//可以是全局变量也可以是局部变量
{
    int tmp=100;
    pi=&tmp;
}
//代码块被当做一个栈帧，tmp变量分配在栈帧上，之后在块语句退出后会出栈。pi指针现在指向一块最终可能被其他活跃记录覆盖的内存区域
```
#### 2. 处理迷途指针
- 释放之后设置为NULL
- 写一个特殊的函数代替free
- 用第三方工具检测

### 五、动态内存分配技术
#### 1. C的垃圾回收
#### 2. 资源获取即初始化
#### 3. 使用异常处理函数