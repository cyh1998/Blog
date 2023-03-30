#### 概 述
在使用GCC编译时，一般会使用`-Wall`来开启警告，这可以帮助我们找到代码中有问题的结构。  
`-Wall`包含了很多警告标志，在编译时可以直接使用`-Wall`开启全部，也可以根据自己的需要开启不同的警告级别。本文介绍一些`-Wall`中常用的警告标志。

#### 警告标志
**-Waddress**  
`-Waddress`用于警告地址表达式的可疑用法。包括：  
- 判断函数或声明对象的地址为空  
- 将指针与字符串进行比较  
- 对指针进行加法或减法的结果判断为空  

**-Wparentheses**  
`-Wparentheses`用于警告在某些上下文中省略了括号。例如在`if()`中赋值
```
// Bad
void Foo() {
    if (bar = 0x99) {}
}
```

**-Wreturn-type**  
`-Wreturn-type`用于警告有返回值的函数缺少返回值语句，例如
```
// Bad
int Foo(int bar) 
{ 
    if (bar > 100) { 
        return 10; 
    } 
    else if (bar > 10) { 
        return 1; 
    } 
}
```

**-Wuninitialized**  
`-Wuninitialized`用于警告使用了未初始化的对象，例如
```
// Bad 
void Foo() { 
    int foo; 
    if (Bar()) { 
        foo = 1; 
    }
    Foobar(foo); // foo可能没有初始化 
}
```

#### 使 用  
`-Wall`警告选项可以与`-Werror`一同使用，`-Werror`用于把所有警告都变成错误。

**GCC**  
如果使用gcc编译，直接添加`-Wall`选项或根据需求选择需要的警告标志，例如
```
$ gcc -Wall -Werror demo.c -o demo
$ gcc -Waddress -Wuninitialized -Wreturn-type -Werror demo.c -o demo
```
*注：* 对于gcc编译C++报错未定义的引用，应使用`g++`或`-lstdc++`，例如
```
$ g++ demo.cpp -o demo
$ gcc demo.cpp -o demo -lstdc++
```
**CMake**  
如果使用CMake文件编译，使用`set()`设置编译选项
```
set(CMAKE_C_FLAGS, "-Wall -Werror")  //C
set(CMAKE_CXX_FLAGS, "-Wall -Werror")  //C++
```
****
**2022.11.18 补充**  
```
-Wno-unused-variable：不显示未使用的变量告警
-Wno-unused-parameter：不显示未使用的参数告警
-Wno-unused-function：不显示未使用的函数告警
-Wno-unused-but-set-variable：不显示已赋值但未使用的变量告警
-Wno-unused-private-field：不显示未使用的类私有成员告警
-Wno-unused-label：不显示未使用的跳转标记告警

-Wno-deprecated：不要警告使用已弃用的功能
```

参考：  
[GNU Compiler Collection](https://gcc.gnu.org/onlinedocs/gcc/)  
[腾讯C++安全指南](https://github.com/Tencent/secguide)