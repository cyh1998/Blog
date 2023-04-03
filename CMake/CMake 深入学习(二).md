## 一、生成动态库和静态库
文件目录如下：
```
├── build
├── lib
├── lib_src
│   ├── testfun.h
│   └── testfun.cpp
└── src
    └── main.cpp
```
在根目录创建 `CMakeLists.txt` 内容如下：
```
cmake_minimum_required(VERSION 2.8)

project(testfun)

#添加 lib_res 子目录
add_subdirectory(lib_src)
```
在 `lib_src` 目录创建 `CMakeLists.txt` 内容如下：
```
aux_source_directory(. LIB_SRCS)

add_library(testfun_shared SHARED ${LIB_SRCS})
add_library(testfun_static STATIC ${LIB_SRCS})

set_target_properties(testfun_shared PROPERTIES OUTPUT_NAME "TestFun")
set_target_properties(testfun_static PROPERTIES OUTPUT_NAME "TestFun")

set(LIBRARY_OUTPUT_PATH ${PROJECT_SOURCE_DIR}/lib)
```
`add_library()` ：用于生成库文件，第1个参数为库的名字；第2个参数为动态还是静态，可不写(默认静态)；第3个参数为生成库的源文件  
`set_target_properties()` ：设置输出的库名称  
`set()` ：用于设置参数的值  
`LIBRARY_OUTPUT_PATH` ：库文件的默认输出路径，这里设置为项目目录下的 `lib` 目录  
***
ps：如果不使用 `set_target_properties()` 来重新定义库文件输出的名字也可以，那么库的名字就是 `add_library()` 里定义的名字。但是当我们连续两次使用 `add_library()` 来指定库文件的名字时，不能重名。所以使用 `set_target_properties()` 把库文件的名字设置为相同，这样相对来说会好看点。

最后进入`build` 目录，执行 `cmake ..` 以及 `make`  
结果如下：

![结果](https://upload-images.jianshu.io/upload_images/22192996-74525553ec8459e8.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

此时可以看到在 `lib` 目录下在生成了以 `.so` 结尾的动态库文件以及以 `.a` 结尾的静态库文件

***
### 补充：动态库和静态库的区别：
静态库：Windows下以 `.lib` 后缀；Linux下以 `.a` 后缀。程序在编译链接时，将库的代码链接到可执行文件中，程序运行时不再需要静态库。缺点(1)，当多次使用到该库文件时，就会被调用多次，浪费内存和磁盘空间；缺点(2)，当库更新时，所有使用改库文件的地方都需要重新下载更新  
动态库：Windows下以 `.dll` 后缀；Linux下以 `.so` 后缀。只有程序在运行时才会去链接动态库的代码，可以多个程序共享动态库的代码。

## 二、链接库
文件目录如下：
```
├── bin
├── build
├── lib
│   ├── libTestFun.a
│   └── libTestFun.so
├── lib_src
│   ├── testfun.h
│   └── testfun.cpp
└── src
    └── main.cpp
```
在根目录中 `CMakeLists.txt` 内容修改为：
```
cmake_minimum_required(VERSION 2.8)

project(testfun)

#添加 lib_res 子目录
add_subdirectory(lib_src)

add_subdirectory(src)
```
`src` 中 `CMakeLists.txt` 内容如下：
```
aux_source_directory(. SRC_LIST)

# 添加头文件目录
include_directories(${PROJECT_SOURCE_DIR}/lib_src)

# 添加需要链接的库文件目录
link_directories(${PROJECT_SOURCE_DIR}/lib)

add_executable(UseFun ${SRC_LIST})

target_link_libraries(UseFun TestFun)

set(EXECUTABLE_OUTPUT_PATH ${PROJECT_SOURCE_DIR}/bin)
```
`include_directories()` ：用于添加头文件目录  
`link_directories()` ：用于添加需要链接的库文件目录    
`LIBRARY_OUTPUT_PATH` ：可执行文件的默认输出路径，这里设置为项目目录下的 `bin` 目录  

进入`build` 目录，执行 `cmake ..` 以及 `make`，进入 `bin` 目录运行  
结果如下：

![结果](https://upload-images.jianshu.io/upload_images/22192996-4565cb2833e6f974.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

ps：在 `lib` 目录下有TestFun的静态库和动态库，`target_link_libraries (UseFun TestFun)` 默认是使用动态库，如果 `lib` 目录下只有静态库，那么这种写法就会去链接静态库。也可以直接指定使用动态库还是静态库，写法是：`target_link_libraries (UseFun libTestFun.so)` 或 
 `target_link_libraries (UseFun libTestFun.a)`  

参考文章：[https://blog.csdn.net/whahu1989/article/details/82078563?depth_1-utm_source=distribute.pc_relevant.none-task&utm_source=distribute.pc_relevant.none-task](https://blog.csdn.net/whahu1989/article/details/82078563?depth_1-utm_source=distribute.pc_relevant.none-task&utm_source=distribute.pc_relevant.none-task)
