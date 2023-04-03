## 使 用
### 一、生成可调试文件
**1. gcc编译**  
使用gcc编程程序时，需要添加 `-g` 表示编译出可调试文件，命令如下：
```
$ gcc -g main.cpp -o main
```
**2. CMake编译**  
如果使用CMake，需要在 `CMakeLists.txt` 文件中添加：
```
SET(CMAKE_BUILD_TYPE "Debug")
SET(CMAKE_CXX_FLAGS_DEBUG "$ENV{CXXFLAGS} -O0 -Wall -g2 -ggdb")
SET(CMAKE_CXX_FLAGS_RELEASE "$ENV{CXXFLAGS} -O3 -Wall")
```
**注：** 如果使用CMake编译多个目录，每个 `CMakeLists.txt` 文件中都需要添加以上的内容，否则调试时无法跳入相应的函数代码中

### 二、启动 gdb
**1. 直接使用gdb启动程序**  
```
// 没有入参的程序
$ gdb main

// 有入参的程序
$ gdb -args main args
// 或者
$ gdb main
(gdb) run args
```
**2. attach方式调试程序**  
直接运行程序，通过进程id调试程序
```
$ gdb
(gdb) attach pid
```
此方式调试程序时，可能会报错没有权限，报错信息如下：  
```
Could not attach to process.  If your uid matches the uid of the target
process, check the setting of /proc/sys/kernel/yama/ptrace_scope, or try
again as the root user.  For more details, see /etc/sysctl.d/10-ptrace.conf
ptrace: 不允许的操作.
```
解决办法：使用 `sudo` 执行gdb或修改 `/etc/sysctl.d/10-ptrace.conf` 中的
```
// kernel.yama.ptrace_scope = 1
kernel.yama.ptrace_scope = 0 //这里改为0
```
**3. 进程id调试程序**  
还可以通过形如 gdb program pid 的方式，通过进程id调试，示例如下：
```
$ gdb main pid
```
### 三、调试
启动gdb后，开始调试，常用的调试命令如下：  
**satrt** ：执行程序，停在main函数第一行语句前面，等待命令  
**run(r)** ：运行程序，停在断点处，等待命令  
**break(b)** ：设置断点，后面可添加行号，函数名，条件等等  
**continue(c)** ：继续执行程序，到下一个断点处或结束程序  
**next(n)** ：单步调试，遇到函数调用时，不进入函数体  
**step(s)** ：单步调试，遇到函数调用时，进入函数体  
**finish** ：运行到当前函数返回后停止，等待命令  
**info(i) locals** ：查看当前堆栈页局部变量的值  
**info(i) break(b)** ：查看断点信息  
**set** ：设定参数的值，后面添加变量名和对应的值  
**print(p)** ：打印值，后面可添加变量名，表达式，函数调用  
**clear** ：清除断点，后面添加行号  
**delete breakpoints** ：清除所有断点  
**quit(q)** ：退出调试  

### 四、其他命令
在调试中，有很多方便调试的命令，如下：  
**list** 可用于查看源码，后面可以添加行号(显示行号为中心的前后5行代码)、函数名(显示所在函数的源代码)、无(继续显示程序的源代码，每次10行)。  
**layout** 可用于分割窗口，一遍查看代码，一遍调试。后面可以添加src(显示源码)、asm(显示反汇编)等等。  
上面两个命令配合使用，效果更佳。  
**注：** 在分割窗口情况下，显示内容会出现覆盖，窜行等现象，可以使用 `Ctrl + L` 来刷新窗口。  
