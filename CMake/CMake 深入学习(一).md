## 一：同一目录下编译多个文件
文件目录如下：
```
├── build
├── main.cpp
├── testfun.h
└── testfun.cpp
```
`testfun.h` 文件内容如下;
```
#ifndef _TESTFUN_H_
#define _TESTFUN_H_

void test_fun(int);

#endif
```
`testfun.cpp` 文件内容如下：
```
#include <iostream>
#include "testfun.h"

using namespace std;

void test_fun(int data){
    cout << "data is:" << data << endl;
}
```
`main.cpp` 文件内容如下：
```
#include <iostream>
#include "testfun.h"

using namespace std;

int main(){

    cout << "use testfun" << endl;
    test_fun(1);
    
    return 0;
}
```
`CMakeLists.txt` 文件内容如下：
```
cmake_minimum_required(VERSION 2.8)

project(testfun)

add_executable(TestFun main.cpp testfun.cpp)
```

即将需要编译的文件加到 `add_executable()` 参数中   
进入build目录，cmake，make编译，运行  
结果如下：

![同一目录下编译多个文件.png](https://upload-images.jianshu.io/upload_images/22192996-89e15aae7faf9497.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

如果有多个源文件，一个一个添加到 `add_executable()` 中会非常麻烦。cmake可以使用 `aux_source_directory()` 指定某个目录下所有的源文件存储在一个变量中，然后直接编译变量中的源文件。其语法如下：
```
aux_source_directory(<dir> <variable>)
```
`CMakeLists.txt` 文件内容修改为：
```
cmake_minimum_required(VERSION 2.8)

project(testfun)

#查找当前目录下的所有源文件
#将其名称保存为 DIR_SRCS 变量
aux_source_directory(. DIR_SRCS)

add_executable(TestFun ${DIR_SRCS})
```
这样可以化简编译多个源文件

####二：多个目录下编译多个文件
文件目录如下：
```
├── build
├── src
│   ├── testfun.h
│   └── testfun.cpp
└── main.cpp
```
需要在根目录以及src目录中各编写一个 `CMakeLists.txt` 文件
根目录 `CMakeLists.txt` 内容如下：
```
cmake_minimum_required(VERSION 2.8)

project(testfun)

#查找当前目录下的所有源文件
#将其名称保存为 DIR_SRCS 变量
aux_source_directory(. DIR_SRCS)

#添加 src 子目录
add_subdirectory(src)

add_executable(TestFun ${DIR_SRCS})

#添加链接库
target_link_libraries(TestFun UseFun)
``` 
`add_subdirectory()` 用于添加子目录，cmake会去处理此目录中的 `CMakeLists.txt` 文件  
`target_link_libraries()` 用于添加链接库，指明可执行文件 TestFun 需要连接一个名为 UseFun 的链接库  

子目录src中的 `CMakeLists.txt` 内容如下：
```
aux_source_directory(. USE_DIR_SRCS)

#生成链接库
add_library(UseFun ${USE_DIR_SRCS})
```
`add_library()` 用于生成链接库  
进入build目录，cmake，make编译，运行  
结果如下：  

![多个目录下编译多个文件.png](https://upload-images.jianshu.io/upload_images/22192996-8dcf804f08520d00.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
