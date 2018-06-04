### 一、指针的声明和初始化
#### 1. 不恰当的指针声明
```c
int* ptr1, ptr2;//ptr2为int而不是int*
//建议写法
typedef pint int*;
pint ptr1,ptr2;
```
#### 2. 使用之前未初始化
```c
int* pi;
printf("%d\n",pi);//野指针
```
#### 3. 处理未初始化指针
- 总是用NULL来初始化指针
- 用assert函数
- 第三方工具
```c
int* pi = NULL;
//...其他操作
if(pi) {
    //已经初始化
}else{
    //pi未初始化
}
//或者
assert(pi != NULL);
```

### 二、指针的使用问题
- 访问数组元素时没有检查边界
- 对数组指针做指针运算时没有检查边界
- 用gets这样的函数从标准输入输出读取字符串可能导致缓冲区溢出
- 误用strcpy和strcat这样不会内部分配内存的函数
#### 1. 测试NULL
```c
float* vector = (float*)malloc(sizeof(float)*20);
if(vector) {
    //分配成功
}else{
    //分配失败
}
```
#### 2. 误用解引用
```c
int num=10;
int* pi;
*pi = &num;//这里误用解引用符*，使得num的地址赋值给了pi
//正确写法v
pi=&num;
```
#### 3. 迷途指针
#### 4. 越过数组边界访问内存
#### 5. 错误计算数组长度
#### 6. 错误使用sizeof操作符
#### 7. 一定要匹配指针类型
#### 8. 有界指针
#### 9. 字符串安全问题
strcpy和strcat，稍不留神就会引发缓冲区溢出
#### 10. 指针算术运算与结构体
**我们应该只对数组使用指针算术运算，因为数组肯定会分配在连续的内存块上，指针算术运算可以得到有效的偏移量，不过不应该将它们用在结构体内，因为结构体的字段可能分配在不连续的内存区域**

#### 11. 函数针织的问题
如果函数和函数指针的签名不一致，不要把函数赋给函数指针，这样会导致未定义的行为

### 三、内存释放问题
#### 1. 重复释放
避免这类漏洞的简单方法是释放指针后将其置为NULL
```c
char* name = (char*)malloc(sizeof(char)*10);
//...operations
free(name);
name = NULL;
```
#### 2. 清除敏感数据
```c
char name[32];
int userId;
char* securityQuestion;
//赋值...
memset(name,0, sizeof(name));
userId=0;
memeset(sercurityQuestion, 0, strlen(securityQuestion));
free(securityQuestion);
securityQuestion = NULL;
```

### 四、使用静态分析工具
略