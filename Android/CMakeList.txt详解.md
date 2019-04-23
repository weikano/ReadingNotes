### CMakeList.txt详解

#### 一、单个源文件

```cmake
#CMAKE的最低版本
cmake_minimum_version (VERSION 3.1.2)
#设置项目名称为Test
project (Test)
#将main.cpp编译生成一个名称为Test1的可执行文件
#也可以添加多个cpp文件
add_executable(Test1 main.cpp)
```

#### 二、多个源文件

```cmake
#参考单个源文件中的设置，仅仅修改add_executable
#比如目录下有main.cpp MathFunctions.h MathFunctions.cpp
add_executable(Test1 main.cpp MathFunctions.cpp)
#或者使用aux_source_directory来实现相同效果
# 将dir下的所有原文件给保存至variable中
# aux_source_directory(<dir> <variable>)
# DIR_SRCS保存所有当前目录下的源文件，包括main.cpp，MathFunctions.cpp
aux_source_directory(. DIR_SRCS)
add_executable(Test1 ${DIR_SRCS})
```

##### 三、多个目录多个源文件

比如.下有main.cpp，math目录下有MathFunctions.cpp, MathFunctions.h
这时候需要在根目录和math目录下分别编写CMakeList.txt文件。一般是将math目录下的编写为静态库再由main函数调用。

- 首先在根目录下的CMakeLists.txt中通过add_subdirectory方法添加子目录
- 子目录中添加CMakeLists.txt，然后通过add_library编译成库
- 根目录中的CMakeLists.txt中通过target_link_libraries添加库

```cmake
# 根目录下的CMakeLists.txt
aux_source_directory(. DIR_SRCS)
#添加math子目录
add_subdirectory(math)
add_executable(Test1 ${DIR_SRCS})
#添加链接库 target_link_libraries(<executable_name> <library_name>)
#executable_name为add_executable指定的name，library_name为add_library中的name
target_link_libraries(Test1 MathFunctions)
```

math目录下的CMakeLists.txt

```cmake
# math/CMakeLists.txt
aux_source_directory(. DIR_LIB_SRCS)
add_library(MathFunctions ${DIR_LIB_SRCS})
```

##### 四、自定义编译选项

- 通过configure_file设置，自动生成一个头文件，通过set方法设置参数然后使用到configure_file中
- option添加一个选项

比如说通过config.h.in来生成一个config.h

```CMAKE
#config.h.in文件，下面是内容
#cmakedefine会在config.h中根据USE_MYMATH的值(ON, OFF)来确定是否生成#define USE_MYMATH
#USE_MY_MATH是通过option命令生成option(<option_name> "desc" ON/OFF)
#cmakedefine USE_MYMATH
# MAJOR_V和MINOR_V是通过set命令生成set(<variable_name> value)
#define MAJOR_VERSION @MAJOR_V@
#define MINOR_VERSION @MINOR_V@
```

接下来看CMakeLists.txt的内容

```cmake
# 忽略版本等
# 将MAJOR_V和MINOR_V设置为1和0
set(MAJOR_V 1)
set(MINOR_V 0)
# USE_MYMATH 为ON
option(USE_MYMATH "DESC OF USE_MYMATH" ON)
# configure_file(<in_file> <out_file>)
# input指定输入， out指定输出文件
configure_file("${PROJECT_SOURCEC_DIR}/config.h.in" "${PROJECT_BINARY_DIR}/config.in")
# 其他的不变
```

接着reload CMakeLists.txt文件，可以看到生成的config.h

```c++
#define USE_MYMATH
#define MAJOR_VERSION 1
#define MINOR_VERSION 0
```

接下来用USE_MYMATH来进行条件编译，在根目录的CMakeLists.txt中

```cmake
option(USE_MYMATH "DESC OF USE_MYMATH" ON)
configure_file("${PROJECT_SOURCEC_DIR}/config.h.in" "${PROJECT_BINARY_DIR}/config.in")
#因为我们在main.cpp中会用到config.h，而config.h并不在当前目录中，所以要include "config.h"，必须给cmake添加一个搜索h的目录
#include_directories会把该目录添加给compiler,用来搜索include files
#如果是在PROJECT_SOURCE_DIR中就不需要
include_directories("${PROJECT_BINARY_DIR}")  
# 如果设置了USE_MYMATH
if (USE_MYMATH) 	
  # 添加math子目录，会编译成target为MathFunction的库
  add_subdirectory(math)
  # 设置一个EXTRA_LIBS的变量，指向MathFunction，在后面使用
  set(EXTRA_LIBS MathFunction)
endif (USE_MYMATH)

aux_sourcce_directory(. DIR_SRCS)

add_executable(Config ${DIR_SRCS})
target_link_libraries(Config ${EXTRA_LIBS})
```

接下来编写main.cpp

```c++
#include <iostream>
//切记需要在CMakeLists.txt中将config.h所在的目录通过include_directories命令添加进去
#include "config.h"
#ifdef USE_MYMATH
	#include "math/MathFunctions.h"
#else
  #include <math.h>
#endif
int main() {
  using namespace std;
#ifdef USE_MYMATH
  cout<<"Use MyMath"<<endl;
#else
  cout<<"Use Standard Libray"<<endl;
#endif  
  cout<<pow(3,2)<<endl;
  return 0;
}
```

