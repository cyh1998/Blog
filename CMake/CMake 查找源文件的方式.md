### 概 述
CMake中查找源文件的方式。

### 实 现
最简单的方式就是在添加可执行文件时，指定需要的源文件，例如：
```
add_executable(Demo main.cpp)
```
除此以外，还可以使用 `set()` 将源文件设置为变量，通过变量指定，如下：
```
set(DEMO_SRCS
    main.cpp
    )

add_executable(Demo ${DEMO_SRCS})
```
以上两种方式，在源文件比较多的时候，就比较麻烦，需要一个一个的添加。

我们可以使用 `aux_source_directory()` 查找目录中的所有源文件，例如：
```
aux_source_directory(${CMAKE_CURRENT_SOURCE_DIR}/base DEMO_SRCS)

add_executable(demo ${DEMO_SRCS})
```
即将当前base目录下的所有源文件，存储为 `DEMO_SRCS` 并指定添加。

但是，`aux_source_directory()` 无法递归子目录收集源文件。如果我们需要指定某一个目录下(包括子目录)的所有源文件，可以使用 `file(GLOB_RECURSE)`，如下：
```
file(GLOB_RECURSE DEMO_SRC "${CMAKE_CURRENT_SOURCE_DIR}/*.cpp")
file(GLOB_RECURSE DEMO_INC "${CMAKE_CURRENT_SOURCE_DIR}/*.h")

add_executable(demo ${DEMO_INC} ${DEMO_SRC})
```
该 `GLOB_RECURSE` 模式会遍历匹配目录的所有子目录，匹配文件。

**注：** 后两种方式，在添加或删除源文件时，无需对 `CMakeLists.txt` 文件进行更改，因此生成的构建系统无法知道何时要求 `CMake` 重新生成。所以在添加或删除源文件时，我们需要手动去重新生成。
