### 一、程序的栈和堆
#### 1. 程序栈
栈存放函数参数和局部变量，堆管理动态内存
#### 2. 栈帧的组织
- 返回地址：函数完成后要返回的程序内部地址
- 局部数据存储：为局部变量分配的内存
- 参数存储：为函数参数分配的内存
- 栈指针和基指针：运行时系统用来管理栈的指针。栈指针通常指向栈顶部

### 二、通过指针传递和返回数据
要在某个函数中修改数据，需要用指针传递数据。通过传递一个指向常量的指针，可以使用指针传递数据并避免其被修改。传递参数时(包括指针)，传递的是它们的值。也就是说传递给函数的是参数值的一个副本。
#### 1. 用指针传递数据
```C
//交换两个整数的值
void swap(int* a, int* b) {
    int tmp = *a;
    *a=*b;
    *b=tmp;
}
//调用
int m=1,n=2;
swap(m,n);
```
#### 2. 用值传递数据
类似上面的swap函数，如果参数是int a和int b，那么就不会交换
#### 3. 传递指向常亮的指针
```C
//将num2的值修改为num1，但是不能用解引用的方式反过来将*num1=*num2
void passAddressOfConstants(const int* num1, int* num2) {
    *num2=*num1;
}
```
#### 4. 返回指针
```C
int* allocateArray(int size, int value) {
    int* arr = (int*)malloc(size*sizeof(int));
    int i=0;
    for(i=0;i<size;i++) {
        arr[i] = value;
    }
    return arr;
}
//使用
int* vector = allocateArray(10,25);
```
尽管上面的方法可以正确工作，但从函数返回指针可能存在几个潜在的问题：
- 返回未初始化的指针
- 返回指向无效地址的指针
- 返回局部变量的指针
- 返回指针但没有释放内存
比如上allocteArray函数，必须使用者手动释放free(vector)，否则会产生内存泄露。
#### 5. 局部数据指针
如果我们将上述的allocateArray中的arr定义修改为int arr[size]，那么当allocateArray执行完成后，因为没有动态分配内存，返回的指针已经失效。如果想要修复，将arr变量声明为static……
#### 6. 传递空指针
```C
int* allocateArray(int* vector, int size ,int value) {
    if(vector) {
        //循环赋值
    }
    return vector;
}
//使用方法如下
int* vector=malloc(5*sizeof(int));
allocateArray(vector, 5, 10);
```
#### 7. 传递指针的指针
**TBD**

### 三、函数指针
#### 1. 声明函数指针
```C
int (*foo)(double) //表示接收一个double参数并且返回值为int的函数指针,foo为函数指针的名字，用在调用上
int value = foo(1.5);
```
#### 2. 使用函数指针
```C
typedef double (*area)(double width, double height);//声明一个函数指针
double rectangleArea(double width, double height) {
    return width*height;
}
double width=10,height=11;
area = rectangleArea;//将函数指针指向一个相同签名的函数
cout<<area(width, height)<<endl;//调用rectangleArea函数
```
#### 3. 传递函数指针
```C
typedef int (*op)(int,int);
int add(int a, int b) {
    return a+b;
}
int sub(int a, int b) {
    return a-b;
}
int compute(op o, int a, int b) {
    return o(a,b);
}
//使用如下
cout<<"add:"<<compute(add,1,2)<<endl;
cout<<"sub:"<<compute(sub,2,1)<<endl; 
```
#### 4. 返回函数指针
```C
op select(char opCode) {
    switch(opCode) {
        case '+' : return add;
        case '-' : return sub;
    }
}
```
#### 5. 使用函数指针数组
```C
typedef int (*operation)(int, int);
operation operations[128]={NULL};
//或者使用如下方式
int (*operations[128])(int,int)={NULL};
//接着初始化数组
void init() {
    operations['+']=add;
    operations['-']=sub;
}
//select函数可以修改为
operation select(char opCode) {
    return operations[opCode];
}
```
#### 6. 比较函数指针
使用==或者!=操作符
#### 7. 转换函数指针
- 运行时系统不会验证函数指针所用的参数是否正确
- 无法保证函数指针和数据指针相互转换后正常工作