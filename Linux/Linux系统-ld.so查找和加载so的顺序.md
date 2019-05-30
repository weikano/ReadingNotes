### Linux系统-ld.so查找和加载共享库的顺序

> ld-linux.so是专门负责查找库文件的库。以cat为例，首先告诉ld-linux.so它需要libc.so.6这个库文件，ld-linux.so按一定顺序找到库后再给cat调用

#### 那么ld-linux.so又是怎么找到的呢

> ld-linux.so的位置是写死在程序中的，gcc在编译程序时就写死在里面了。gcc写到程序中的ld-linux.so的位置可以通过gcc的spec文件修改

#### 编译时ld-linux.so查找共享库的顺序

1. ld-linux.so由gcc的spec文件指定
2. gcc -print-search-dirs所打印出来的路径，主要是libgcc_s.so等库，可以通过GCC_EXEC_PREFIX来设定
3. LIBRARY_PATH环境变量中设置的路径，或编译的命令行中指定的 -L路径
4. binutils中的ld所设定的default搜索路径，编译binutils时指定。可以通过ld –verbose | grep SEARCH查看
5. 二进制程序的搜索路径顺序为PATH环境变量中设定，一般/usr/local/bin高于/usr/bin
6. 编译时头文件的搜索路径，与library相似。一般/usr/local/include高于/usr/include

#### 运行时ld-linux.so查找共享库的顺序

1. ld-linux.so在可执行的目标文件中被指定，可以用readelf查看
2. 默认在/usr/lib和/lib中搜索。当glibc安装到/usr/local下时，它查找/usr/local/lib
3. LD_LIBRARY_PATH环境变量中设定的路径
4. /etc/ld.so.conf中设置的目录