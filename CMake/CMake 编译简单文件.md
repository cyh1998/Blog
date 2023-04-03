## 环境
Ubuntu 18.04

## 1、安装cmake
使用的apt安装cmake
```
sudo apt install cmake
```
查看cmake版本
```
cmake --version
```

## 2、编译简单的单文件
### 创建 `hello.cpp` 文件
```
vi hello.cpp
```
写入代码
```
#include <iostream>

int main(){
    std::cout << "hello world!" << std::endl;
    return 0;
}
```
### 在hello.cpp文件所在目录，创建 `CMakeLists.txt` 文件，并写入（注意大小写，文件名不能错！！！）
```
# CMake 最低版本号要求
cmake_minimum_required (VERSION 2.8)
#设置项目名称
project(hello)
#设置编译
add_executable(Hello hello.cpp)
```
说明：
1. cmake_minimum_required：指定cmake的最低版本，可以不写
2. project：表示项目名称
3. add_executable：将hello.cpp文件编译成名为Hello的可执行文件。

### 编译文件
创建build文件夹，进入，cmake编译  
也可以在当前目录直接编译，但是编译会生成一些文件，使得其与源文件混在一起。为了方便，我们创建build文件夹在其中编译，文件夹的名字可自定义，一般使用build
```
mkdir build
cd build
cmake ..
```
成功后，在build目录执行
```
make
./Hello
```
完成，查看运行结果

![image.png](https://upload-images.jianshu.io/upload_images/22192996-077a5b8e5f20a4ea.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
