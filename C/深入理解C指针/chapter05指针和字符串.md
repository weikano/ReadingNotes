### 一、字符串基础
字符串是以ASCII字符NUL结尾的字符序列。ASCII字符NUL表示为\0。C中中有两种类型的字符串：
- 单字节字符串：由char数据类型组成的序列
- 宽字符换：以wchar_t数据类型组成的序列

#### 1. 字符串声明
```c
"Hello"；//字面量
char header[12];//字符数组
char* header;//字符指针
```
#### 2. 字符串字面量池
定义字面量时通常会将其分配在字符串字面量池中，这个区域保存了组成字符串的字符序列。多次用到同一个字面量时，字面量池中通常只有一份副本，**除非编译器关闭字面量池的选项**。
#### 3. 字符串初始化
1. 初始化char数组
```c
char header[]="Media Player";
char header[13];//需要13是不能少\0这个终止符
strcpy(header,"Media Player");
```
2. 初始化char指针
```c
char* header = (char*)malloc(strlen("MediaPlayer")+1);
strcpy(header,"Media Player");
char* header = "Media Player";
```
### 二、标准字符串操作
```c
//比较字符串
int strcmp(const char*, const char*);
//复制字符串
char* strcpy(char* s1, const char* s2);
//拼接字符串，把s2复制到s1的结尾。函数不会分配内存，所以s1要足够大
char* strcat(char* s1, const char* s2);
```

### 三、传递字符串
#### 1. 传递简单字符串
#### 2. 传递字符常量的指针
#### 3. 传递需要初始化的字符串

### 四、返回字符串
### 五、函数指针和字符串