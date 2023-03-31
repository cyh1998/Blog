## 概 述
我们在对STL容器进行插入操作时，常会使用 `insert` 或 `push_back` 。C++11提出了更高效的插入方法：`emplace`。本文将介绍C++11新特性中 `emplace` 的使用与原理。

## 使 用
首先，介绍下 `emplace` 相对应的函数
```
vector
emplace <--> insert
emplace_back​ <--> ​push_back

set
emplcace <--> insert

map
emplace <--> insert
```
简单的使用，以 `vector` 的 `emplace_back` 为例
```
#include <iostream>
#include <vector>

using namespace std;

struct Student {
    string name;
    int age;

    Student(string&& n, int a)
        :name(std::move(n)), age(a)
    {
    }
};

int main()
{
    //基本数据类型的插入
    vector<int> arr;
    arr.push_back(1);
    arr.emplace_back(1);
  
    //自定义类型的插入
    vector<Student> classes;
    classes.emplace_back("xiaohong", 24); //无需先创建类
    classes.push_back(Student("xiaoming", 23)); //需先创建类
}
```
## 原 理
`push_back()`：先向容器尾部添加一个右值元素(临时对象)，然后调用**构造函数**构造出这个临时对象，最后调用**移动构造函数**将这个临时对象放入容器中并释放这个临时对象。  
**注：** 最后调用的不是拷贝构造函数，而是移动构造函数。因为需要释放临时对象，所以通过 `std::move` 进行移动构造，可以避免不必要的拷贝操作  
`emplace_back()`：在容器尾部添加一个元素，调用**构造函数**原地构造，不需要触发拷贝构造和移动构造。因此比 `push_back()` 更加高效。  
示例代码：
```
#include <iostream>
#include <vector>

using namespace std;

struct Student {
    string name;
    int age;

    Student(string&& n, int a)
        :name(std::move(n)), age(a)
    {
        cout << "构造" << endl;
    }

    Student(const Student& s)
        : name(std::move(s.name)), age(s.age)
    {
        cout << "拷贝构造" << endl;;
    }

    Student(Student&& s)
        :name(std::move(s.name)), age(s.age)
    {
        cout << "移动构造" << endl;
    }

    Student& operator=(const Student& s);
};

int main()
{
    vector<Student> classes_one;
    vector<Student> classes_two;

    cout << "emplace_back:" << endl;
    classes_one.emplace_back("xiaohong", 24);

    cout << "push_back:" << endl;
    classes_two.push_back(Student("xiaoming", 23));
}
```
执行结果：

![结果](https://upload-images.jianshu.io/upload_images/22192996-430159f066547887.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

## 进一步讨论
看了上文的示例代码，可能会有人问，为什么要分成两个容器分开插入来查看结果呢？因为一起插入时，会触发未知的**拷贝构造**操作，以及多次进行 `emplace_back()` 或 `push_back()` 操作时，也会触发**拷贝构造**操作。  
这些拷贝构造操作是什么呢？这和vector的原理有关。当多次插入操作导致vector已满时，就要分配一块更大的内存(比原始大小多50%)，将原始数据复制过来并释放之前的内存。原始数据的复制就是这些**拷贝构造**操作。  
进行多次插入，查看运行结果：
```
int main()
{
    vector<Student> classes;

    classes.emplace_back("xiaohong", 24);
    classes.emplace_back("xiaohong", 24);
    classes.emplace_back("xiaohong", 24);
    classes.emplace_back("xiaohong", 24);
    classes.emplace_back("xiaohong", 24);
    classes.emplace_back("xiaohong", 24);
    classes.emplace_back("xiaohong", 24);
    classes.emplace_back("xiaohong", 24);
}
```
![结果](https://upload-images.jianshu.io/upload_images/22192996-3a9f1e728b0afedf.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

上图打印的“拷贝构造”即vector满时，需要分配更大的内存并复制原始数据而产生的。并且每次分配更大内存时，都比原始大小多50%，即1->2->3->4->6...