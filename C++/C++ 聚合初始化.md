## 问题概述
在工作中遇到一个编译报错，简化代码如下
```
struct Student
{
    int age = 0;
    std::string name;
};
```
定义结构体时，为其中的某个成员添加了默认成员初始化，并在使用时通过 `{}` 来初始化该结构体
```
Student stu = { 
    "xiaoming", 
    20 
};
```
这段代码在Windows vs2019中通过编译，但在Linux服务器上编译时报错，报错信息如下
```
could not convert {...} from <brace-enclosed initializer list> to struct
```

## 问题分析
在 [StackOverFlow](https://stackoverflow.com/questions/37776823/could-not-convert-from-brace-enclosed-initializer-list-to-struct) 上找到了类似的问题，问题就出在默认成员初始化这一行
```
struct Student
{
    int age = 0; // <==
    std::string name;
};
```
文中提到，在C++14之前，默认成员初始化器阻止类成为聚合，因此导致无法通过 `{}` 来聚合初始化  
显然vs2019支持了C++14，而GCC 5才对C++14这一新规则支持  
通过cppreference官网也证实了这一点  

![C++14 core language features](https://upload-images.jianshu.io/upload_images/22192996-706a044894fdb65c.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

最后，查看一下服务器上GCC的版本为4.8，因此并不支持这一新特性

## 总结
根据 [C++语言标准](http://eel.is/c++draft/dcl.init.aggr)，聚合是一个类，必须具有一下特征
- 没有用户声明或继承的构造函数
- 没有私有或受保护的非静态数据成员
- 没有虚函数
- 没有虚、私有或受保护的基类
