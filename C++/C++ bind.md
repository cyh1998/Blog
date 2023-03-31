## 概述
`bind` 函数可以看作一个通用的函数适配器，所谓适配器，即使某种事物的行为类似于另外一种事物的一种机制，如容器适配器：`stack(栈)`、`queue(队列)`、`priority_queue(优先级队列)`。  
`bind` 函数接受一个可调用对象，生成一个新的可调用对象来适配原对象。

## 函数原型
```
template <class Fn, class... Args>
  /* unspecified */ bind (Fn&& fn, Args&&... args);
```
`bind` 函数接受一个逗号分割的参数列表 `args`，对应给定函数对象 `fn` 的参数，返回一个新的函数对象。
参数列表 `args` 中：  
- 如果绑定到一个值，则调用返回的函数对象将始终使用该值作为参数
- 如果是一个形如`_n`的占位符，则调用返回的函数对象会转发传递给调用的参数(该参数的顺序号由占位符指定)

## 使用
```
#include <functional> //bind函数 placeholders命名空间

int Plus(int x, int y) {
    return x + y;
}

int PlusOne(int x) {
    auto func = std::bind(Plus, std::placeholders::_1, 1);
    return func(x);
}

int main()
{
    std::cout << PlusOne(9) << std::endl; //结果 10
    return 0;
}
```
**注：** 示例代码中的 `_1` 即为形如 `_n` 的占位符，其定义在命名空间 `placeholders` 中，而此命名空间又定义在命名空间 `std` 中

## 使用场景
根据 `bind` 函数的特征，有以下几个场景时可以使用 `bind`：  
1. 当 `bind` 函数的参数列表绑定到一个值时，则调用返回的函数对象将始终使用该值作为参数。所以 `bind` 函数可以将一个函数的参数特例化，如上文的示例代码
2. 当 `bind` 函数的参数列表是一个占位符时，调用返回的函数对象的参数顺序号由占位符指定，所以 `bind` 函数可以对调用函数对象的参数重新安排顺序，例如：
```
using namespace std;

void output(int a, int b, int c) {
    cout << a << " " << b << " " << c;
}

int main()
{
    auto func = bind(output, placeholders::_2, placeholders::_1, placeholders::_3);
    func(1, 2, 3); //结果 2 1 3
    return 0;
}
```
3. 与 `std::function` 配合，实现回调函数。具体见文章 [C++ std::function](./C++%20function.md)，这里不再赘述。

## 进一步讨论
`bind` 函数中非占位符的参数，将会以值拷贝的方式传递给返回的可调用对象中，所以直接使用 `bind`，无法将参数以引用方式传递，或是绑定的参数类型无法拷贝。  
我们需要使用 `ref` 函数(函数 `ref` 返回一个对象，包含给定的引用，此对象是可以拷贝的)来实现以引用方式传递参数，或将无法拷贝的参数类型可拷贝。  
示例：
```
void add(int &x, int &y) {
    x++;
    y++;
}

int main()
{
    int x = 1, y = 1;
    auto func = bind(add, ref(x), y); //x以引用方式传递，y以值方式传递
    func();
    cout << x << " " << y; //结果 2 1
    return 0;
}
```
相似的，`cref` 函数，用于生成一个保存 `const` 引用的类。函数 `ref` 与 `cref` 也定义在头文件 `functional` 中。

参考 《Primer C++ 第五版》