## 前 言
项目中经常会使用第三方库，因此编译时会需要去链接这些库，这里介绍几种链接第三方库的方式。编译使用 `CMake`，第三方库以 `protobuf 3.20.1` 为例。

## 说 明
*库的安装*  
`protobuf` 可以通过 `apt-get` 来安装也可以通过源码编译安装。

如果使用 `apt-get` 来安装，相关的文件会安装到`/usr/bin` (执行文件)、 `/usr/lib` (库文件)、`/usr/include` (头文件)中。当然，不同的Linux版本安装路径可能会不同，但这些路径默认都会在系统的环境变量中，因此编译时一般直接链接 `protobuf` 即可。

如果自己使用源码编译安装，安装的路径可能不在环境变量中。以下的几种方式均是手动安装且安装路径在 `/usr/loacl/protocbuf` 下的示例。  
*Ps：* 当然也可以将安装路径添加到环境变量里来解决问题，本文主要是介绍不同的链接方式。

## 实 现
**1. 简单方式**  
最简单的方式就是使用 `include_directories` 包含头文件路径以及使用 `link_directories` 指定库搜索路径，如下：
```
# 包含头文件路径
include_directories(/usr/local/protobuf/include)
# 指定库搜索路径
link_directories(/usr/local/protobuf/lib)

add_executable(Demo demo.cpp)

# 链接 protobuf，同时需要链接 pthread
target_link_libraries(Demo protobuf pthread)
```

**2. find_package**  
`find_package` 可以帮助我们找到第三方库的相关依赖，详细内容可以参考官方文档：[cmake find_package](https://cmake.org/cmake/help/latest/command/find_package.html)。  
`find_package` 会在以下路径(优先级由上往下)查找：
```
<package>_DIR
CMAKE_PREFIX_PATH
CMAKE_FRAMEWORK_PATH
CMAKE_APPBUNDLE_PATH
PATH
```
我们可以设置 `<package>_DIR` 值，或者将查找路径添加到 `CMAKE_PREFIX_PATH` 中，实现如下：
```
# 定义查找路径
set(Protobuf_PREFIX_PATH "/usr/local/protobuf")
# 添加到 CMAKE_PREFIX_PATH
list(APPEND CMAKE_PREFIX_PATH "${Protobuf_PREFIX_PATH}")
# 查找 Protobuf
find_package(Protobuf REQUIRED)

# 包含头文件
include_directories(${Protobuf_INCLUDE_DIR})

add_executable(Demo demo.cpp)

# 链接选项
target_link_libraries(Demo ${Protobuf_LIBRARIES} pthread)
```

**3. pkg-config**  
`pkg-config` 是通过库提供的 `.pc` 文件来定位库的各种路径。首先需要安装 `pkg-config`：
```
sudo apt-get install pkg-config
```
接着我们需要让 `pkg-config` 能够找到 `protobuf` 的 `.pc` 文件。  
有两种方式：
- 1.在手动安装 `protobuf` 的路径 `/usr/local/protobuf/lib/pkgconfig` 下可以找到提供的 `.pc` 文件，将其拷贝到 `pkg-config` 默认搜索路径 `/usr/lib/pkgconfig` 中
- 2.将路径 `/usr/local/protobuf/lib/pkgconfig` 添加到环境变量 `PKG_CONFIG_PATH` 中

最后在CMake中使用 `pkg-config` 查找库并链接
```
find_package(PkgConfig)
# pkg_search_module(自定义名  必需项  查找库名)
pkg_search_module(Protobuf REQUIRED protobuf)

include_directories(${Protobuf_INCLUDEDIR})
link_directories(${Protobuf_LIBDIR})

add_executable(Demo demo.cpp)

target_link_libraries(Demo protobuf pthread)
```
使用的变量 `${Protobuf_INCLUDEDIR}` 和 `${Protobuf_LIBDIR}` 是根据我们自定义名以及 `.pc` 中定义的变量而来，查看 `protobuf.pc` 内容如下：
```
prefix=/usr/local/protobuf                                                                                                                                        
exec_prefix=${prefix}
libdir=${exec_prefix}/lib
includedir=${prefix}/include

Name: Protocol Buffers
Description: Google's Data Interchange Format
Version: 3.20.1
Libs: -L${libdir} -lprotobuf
Libs.private: -lz 

Cflags: -I${includedir}
Conflicts: protobuf-lite
```
文中定义了库路径 `libdir` 和头文件路径 `includedir`

## 问 题
在第二种方式中提到，可以设置 `<package>_DIR` 的路径且其优先级最高。根据 `find_package` 的原理，其是寻找路径下的 `<package>Config.cmake` 文件来获取库信息。

手动安装 `protobuf` 情况下，在编译路径中可以找到 `cmake` 文件夹，其中包含了关键文件 `protobuf-config.cmake` 和 `protobuf-config.cmake`(可能带有 .in 后缀)
设置该路径为 `<package>_DIR`，实现如下：
```
# /root/cyh/protobuf/build/protobuf-3.20.1 为我编译的路径
set(protobuf_DIR "/root/cyh/protobuf/build/protobuf-3.20.1/cmake")
find_package(Protobuf REQUIRED)
```
此时，CMake报错：
```
CMake Error at /usr/share/cmake-3.10/Modules/FindPackageHandleStandardArgs.cmake:137 (message):
  Could NOT find Protobuf (missing: Protobuf_INCLUDE_DIR)
Call Stack (most recent call first):
  /usr/share/cmake-3.10/Modules/FindPackageHandleStandardArgs.cmake:378 (_FPHSA_FAILURE_MESSAGE)
  /usr/share/cmake-3.10/Modules/FindProtobuf.cmake:543 (FIND_PACKAGE_HANDLE_STANDARD_ARGS)
  CMakeLists.txt:8 (find_package)
```
使用 `CONFIG` 模式查找
```
find_package(Protobuf REQUIRED CONFIG)
```
CMake报错：
```
CMake Error at CMakeLists.txt:8 (find_package):
Could not find a package configuration file provided by "Protobuf" with any
of the following names:

    ProtobufConfig.cmake
    protobuf-config.cmake
```
奇怪的是，报错提示找不到的 `protobuf-config.cmake` 文件确实在指定目录中。
参考了几个讨论：
- https://stackoverflow.com/questions/50239220/cmake-find-protobuf-compiled-from-source
- https://stackoverflow.com/questions/68274203/could-not-find-a-package-configuration-file-provided-by-protobuf-with-any-of-t
- https://stackoverflow.com/questions/62070189/cmake-cant-find-protobuf-when-compiling-googles-protobuf-example
- https://www.jianshu.com/p/ae5c56845896

还是没能解决问题，于是改用上文设置安装路径到 `CMAKE_PREFIX_PATH` 的方式解决。  
欢迎大佬留言指点！
