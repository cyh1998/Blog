#### 概 述
CMake中经常使用`set()`命令来设置一些CMake变量，本文介绍一些常用变量的含义。

#### 变量含义
**1. CMAKE_CXX_STANDARD**  
设置C++标准
```
set(CMAKE_CXX_STANDARD 11)
# set(CMAKE_CXX_STANDARD 14)
# set(CMAKE_CXX_STANDARD 17)
```

**2. CMAKE_UNITY_BUILD**  
设置开启元编译，于`CMAKE_UNITY_BUILD_BATCH_SIZE`配合使用，用于加速项目编译速度，[参考](https://onqtam.com/programming/2019-12-20-pch-unity-cmake-3-16/)。
```
# 全局设置
set(CMAKE_UNITY_BUILD ON)
# 设置元编译的batch大小
set(CMAKE_UNITY_BUILD_BATCH_SIZE 16) 

# 单个目标
set_target_properties(<target> PROPERTIES UNITY_BUILD ON)
```

**3. CMAKE_BUILD_TYPE**  
设置单个配置生成器上的编译类型，例如`Makefile`等
```
# set(CMAKE_BUILD_TYPE "Debug")
set(CMAKE_BUILD_TYPE "Release")
```

**4. CMAKE_CONFIGURATION_TYPES**  
设置多配置生成器上的可用编译类型，例如` Visual Studio`、`Xcode`等
```
set(CMAKE_CONFIGURATION_TYPES "Release;Debug")
```

**5. CMAKE_<LANG>_FLAGS**  
设置对应语言的编译选项，以C++为例，
- `CMAKE_CXX_FLAGS`：设置C++编译选项
- `CMAKE_CXX_FLAGS_DEBUG`：设置C++ Debug 编译选项
- `CMAKE_CXX_FLAGS_RELEASE`：设置C++ Relese 编译选项

**6. BUILD_USE_64BITS**  
设置使用64位编译
```
set(BUILD_USE_64BITS ON)
```

**7. BUILD_SHARED_LIBS**  
设置是否生成动态库，默认是开启状态，根据`add_library()`生成对应的动态库
```
# set(BUILD_SHARED_LIBS ON)
set(BUILD_SHARED_LIBS OFF)
```

**8. CMAKE_\*_OUTPUT_DIRECTORY**  
设置输出目录
```
# 全局设置
set(CMAKE_ARCHIVE_OUTPUT_DIRECTORY ${CMAKE_BINARY_DIR}/lib)
set(CMAKE_LIBRARY_OUTPUT_DIRECTORY ${CMAKE_BINARY_DIR}/lib)
set(CMAKE_RUNTIME_OUTPUT_DIRECTORY ${CMAKE_BINARY_DIR}/bin)

# 单个目标
set_target_properties( targets...
    PROPERTIES
    ARCHIVE_OUTPUT_DIRECTORY "${CMAKE_BINARY_DIR}/lib"
    LIBRARY_OUTPUT_DIRECTORY "${CMAKE_BINARY_DIR}/lib"
    RUNTIME_OUTPUT_DIRECTORY "${CMAKE_BINARY_DIR}/bin"
)
```

**9. CMAKE_<CONFIG>_POSTFIX**  
设置文件后缀
```
# 全局设置
set(CMAKE_DEBUG_POSTFIX "_debug")
set(CMAKE_RELEASE_POSTFIX "_release")

# 单个目标
set_target_properties( targets...
    PROPERTIES
    DEBUG_POSTFIX "_debug"
    RELEASE_POSTFIX "_release"
)
```
