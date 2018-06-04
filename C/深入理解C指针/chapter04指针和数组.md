### 一、数组概述
#### 1. 一维数组
```C
int vector[5];
int arrlen = sizeof(vector)/sizeof(int);
```
#### 2. 二维数组
```c 
int matrix[2][3]={{1,2,3},{4,5,6}};
```
#### 3. 多维数组
略
### 二、指针表示法和数组
我们可以用指针指向已有数组，也可以从堆上分配内存然后把这块内存当做一个数组使用。数组表示法和指针表示法并不完全相同。
```c
int vector[5]={1,2,3,4,5};
int* pv=vector;//pv是指向数组第一个元素而不是指向数组本身的指针。给pv赋值是把数组第一个元素的地址赋给pv。
//也可以写成这样
pv=&vector[0];
```
**数组和指针的区别**
- 生成的机器码不一样
- sizeof操作符返回的结果不一样
- 指针是一个左值，必须能修改。数字名字不是左值，不能被修改。
```c
pv=pv+1;
vector=vector+1;//语法错误
```

### 三、用malloc创建一维数组
```c
int* pv = malloc(5*sizeof(int));
for(int i=0;i<5;i++) {
    pv[i]=i;
    //或者使用下面的方式
    *(pv+i)=i;
}

```
### 四、使用realloc调整数组长度
略
### 五、传递一维数组
#### 1. 用数组表示法
```c
void display(int arr[], int size) {
    for(int i=0;i<size;i++) {
        cout<<arr[i]<<endl;
        //或者
        cout<<*(arr+i)<<endl;
    }
}
```
#### 2. 指针表示法
```c
void display(int* pv, int size) {
    for(int i=0;i<size;i++) {
        cout<<pv[i]<<endl;
        //或者
        cout<<*(pv+i)<<endl;
    }
}
```
### 六、使用指针的一维数组
**TBD**

### 七、指针和多维数组
**TBD**

### 八、传递多维数组
```c
void display2Darray(int arr[][5], int rows);
void display2Darray(int (*arr)[5], int rows);
```
**TBD 狗屎语法**
### 九、动态分配二维数组
**TBD 狗屎语法**
### 十、不规则数组和指针
**TBD 狗屎语法**