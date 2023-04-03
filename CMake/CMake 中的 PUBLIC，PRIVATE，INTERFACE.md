## 概 述
CMake中经常会使用 `target_**()` 相关命令，`target_**()` 命令支持通过 `PUBLIC`，`PRIVATE` 和 `INTERFACE` 关键字来控制传播。本文主要介绍下这三个关键字的区别。

## 解 释
以 `target_link_libraries(A B)` 命令为例，从理解的角度解释：
- `PRIVATE` 依赖项B仅链接到目标A，若有C链接了目标A，C不链接依赖项B
- `INTERFACE` 依赖项B并不链接到目标A，若有C链接了目标A，C会链接依赖项B
- `PUBLIC` 依赖项B链接到目标A，若有C链接了目标A，C也会链接依赖项B

从使用的角度解释，若有C链接了目标A：
- 如果依赖项B仅用于目标A的实现，且不在头文件中提供给C使用，使用 `PRIVATE`
- 如果依赖项B不用于目标A的实现，仅在头文件中作为接口提供给C使用，使用 `INTERFACE`
- 如果依赖项B不仅用于目标A的实现，而且在头文件提供给C使用，使用 `PUBLIC`

### 例 子
举一个简单的例子说明一下
```
add_library(C c.cpp)
add_library(D d.cpp)

add_library(B b.cpp)
target_link_libraries(B PUBLIC C)
target_link_libraries(B PRIVATE D)

add_executable(A a.cpp)
target_link_libraries(A B)
```
因为C是B的PUBLIC依赖项，所以其会被传播到A。  
因为D是B的PRIVATE依赖项，所以其不会传播到A。

## 补 充
这里补充下使用 `target_**()` 相关命令，有无 `target` 的区别。    
以 `target_include_directories()` 命令为例，`include_directories(dir)` 是一个全局设置，其会将 `dir` 添加到当前CMakeLists文件中每个目标的 `INCLUDE_DIRECTORIES` 属性中。即当前CMakeLists文件其下所有的子目录都会添加dir目录。  
**因此，** 建议使用有 `target` 的命令来减少不必要或多余的目录包含和链接。
