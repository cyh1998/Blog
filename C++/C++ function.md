#### 前言
在之前的文章[C++ std::bind](https://www.jianshu.com/p/82407fb43475)中，我们使用`bind()`来创建可调用对象，示例代码中均使用了`auto`自动类型来接受`bind()`的返回值，而这个返回值则是`std::function`类型。
C++中有多种可调用对象：函数、函数指针、`lambda`表达式、`bind()`创建的对象、重载了函数调用运算符的类(仿函数)。我们可以使用`std::function`将不同类型的可调用对象共享同一种**调用形式**。

#### 函数使用
```
#include <iostream>
#include <functional> //function bind

using namespace std;

function<bool(int, int)> fun; //声明一个function类型，接受两个int，返回bool

bool compare_com(int a, int b) //普通函数
{
    return a > b;
}

auto compare_lambda = [](int a, int b) { return a > b; }; //lambda表达式

class compare_class
{
public:
    bool operator()(int a, int b) //仿函数
    {
        return a > b;
    }
};

class compare
{
public:
    bool compare_member(int a, int b) //类的成员函数
    {
        return a > b;
    }
    static bool compare_static_member(int a, int b) //类的静态成员函数
    {
        return a > b;
    }
};

int main()
{
    fun = compare_com;
    cout << fun(1, 2) << endl; //结果 0

    fun = compare_lambda;
    cout << fun(2, 1) << endl; //结果 1

    fun = compare_class();
    cout << fun(3, 1) << endl; //结果 1

    fun = compare::compare_static_member;
    cout << fun(1, 3) << endl; //结果 0
    
    //类的成员函数需要使用bind，并且需要实例化对象，成员函数要加&
    compare temp;
    fun = std::bind(&compare::compare_member, temp, placeholders::_1, placeholders::_2);
    cout << fun(1, 2) << endl; //结果 0
}
```
上文所示的代码，将不同类型的可调用对象，都通过`std::function`变成了具有相同的调用形式

#### 进一步讨论
在文章[C++/Qt 实现回调函数](https://www.jianshu.com/p/04471ca9601f)中介绍了如何实现回调函数，示例的代码中是通过声明函数指针类型的方式，现在我们可以使用`std::function`来简化流程
```
class Core{
public:
    void RegiseterExitCallback(std::function<void()> cbfun); //注册回调函数
    {
        m_callbackfun = cbfun
    }
private:
    std::function<void()> m_callbackfun; //回调函数模板
}
```
调用模块则可以使用`RegiseterExitCallback`函数，来注册回调函数，可以通过`bind()`、`lambda`等方式来传入`function`对象类型。
