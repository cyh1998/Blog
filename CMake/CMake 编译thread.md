## 问 题
Linux环境下，使用C++多线程，即 `std::thread` 时，通过cmake编译报错，`对‘pthread_create’未定义的引用`。

## 原 因
Linux环境下，C++的 `std::thread` 库底层是对pthread的封装

## 解决方法
在 `CMakeLists.txt` 中添加
```
find_package(Threads) //引入外部依赖包
add_executable(Network main.cpp)
target_link_libraries (${PROJECT_NAME} ${CMAKE_THREAD_LIBS_INIT}) //链接 Thread 库
```
或
```
target_link_libraries (${PROJECT_NAME} pthread) 
```
**注：** 使用 `target_link_libraries` 链接库时，需要在 `add_executable` 之后
