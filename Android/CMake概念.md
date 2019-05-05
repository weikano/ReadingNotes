### CMake概念

#### 一、add_executable

#### 二、add_library

##### 2.1 普通的library

语法：

```cmake
#添加一个名字叫name的library，源码由source1、source2…等
# SHARED：动态链接
# STATIC：libraries are archives of object files for use when linking other targets
# MODULE：libraries are plugins that are not linked into other targets but may be loaded dynamically at runtime using dlopen-like functionality 
add_library(<name> [STATIC|SHARED|MODULE]
	[EXCLUDE_FROM_ALL]
	[source1] [source2...]
)
```

##### 2.2 imported library（library不在当前project中）

语法：

```cmake
# imported library需要设置REPORTED_LOCATION属性
add_library(<name> [STATIC|SHARED|MODULE|OBJECT|UNKNOWN] IMPORTED
[GLOBAL])
```

示例：

```cmake
# 先将IMPORTED_DIR设置外部imported library的目录
set(IMPORTED_DIR ${CMAKE_CURRENT_SOURCE_DIR}/../../../../distribution)
# 添加imported library，名为lib_gmath，因为源码是.a，所以使用static
add_library(lib_gmath STATIC IMPORTED)
# 给imported_library lib_gmath设置IMPORTED_LOCATION属性
set_target_properties(lib_gmath PROPERTIES IMPORTED_LOCATION ${IMPORTED_DIR}/gmath/lib/${ANDROID_ABI}/libgmath.a)
# 添加运行程序test，源码为main.cpp
add_executable(test main.cpp)
# test添加头文件搜索目录
target_include_directories(test PRIVATE 
	${IMPORTED_LOCATION}/gmath/include
)
# 给test添加库 lib_gmath
target_link_libraries(test 
lib_gmath)
```

#### 三、set_target_properties

```cmake
#给target1、target2设置PROPERTIES
set_target_properties(taraget1 target2 ...
	PROPERTIES prop1 value1
	prop2 value2
	....
)
```

#### 四、target_include_directories

语法：

```cmake
#target必须是通过add_executable或者add_library方式创建且非别名
target_include_directories(<target> [SYSTEM] [BEFORE]
<INTERFACE|PUBLIC|PRIVATE> [items1...]
[<INTERFACE|PUBLIC|PRIVATE> [items2...] ...])
```

#### 五、target_link_libraries

语法：

```cmake
# 给target编译时添加依赖
# item可以是library target name，可以是library文件的完整路径，可以是link flag等等
target_link_libraries(<target> item1 ...)
```

#### 六、get_filename_component

语法：

```cmake
# 比如CMakeLists.txt是在~/clionproject/test下
# 以get_filename_component(TEST_DIR ${CMAKE_CURRENT_SOURCE_DIR}\math\MyMath.test.a.b.cpp mode)为例
# var变量名称, FileName 文件名称, mode如下：
# DIRECTORY：变量仅仅保存FileName的目录 TEST_DIR = ${CMAKE_CURRENT_SOURCE_DIR}\math
# NAME：var保存文件名称, TEST_DIR = MyMath.test.a.b.cpp
# EXT：保存最长的扩展名 .test.a.b.cpp
# LAST_EXT：最后的扩展名 .cpp
# NAME_WE：name without extensions，去掉扩展名之后的文件名 MyMath
# NAME_WLE：去掉最后一个扩展名后的文件名 MyMath.test.a.b
get_filename_component(<var> <FileName> <mode> [CACHE])
```

#### 七、add_subdirectory

语法：

```cmake
# 添加一个子目录到build，子目录中需要CMakeLists.txt和源码
# 如果设置了EXCLUDE_FROM_ALL，subdirectory中的所有target都不会被加到项目中，用户必须自己编译target
add_subdirectory(source_dir [binary_dir] [EXCLUDE_FROM_ALL])
```

#### 八、include_directories

语法

```cmake
# 类似target_include_directories。include_directories将指定的dir1、dir2等目录加到编译器中来查找include头文件
include_directories([AFTER|BEFORE] [SYSTEM] dir1 [dir2 ...])
```

