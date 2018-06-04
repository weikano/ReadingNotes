### 一、介绍
```c
//定义
typedef struct _person {
    char* name;
    int age;
} Person;
//声明
Person person = {"Hello",2};
//或者使用指针的方式
Person* pp = (Person*)malloc(sizeof(Person));
pp->name=(char*)malloc(sizeof("Hello")+1);
pp->name="Hello";
pp->age=2;
```

### 二、结构体释放问题
类似上述的Person结构体，系统会回收掉person的内存，但是其中动态分配的name却不会自动回收，所以需要提供方法来手动回收掉
```c
void deallocate(Person* p) {
    free(p->name);
    p->name=NULL;
}
```

### 三、避免malloc/free开销
**TBD**

### 四、用指针支持数据结构
**TBD**