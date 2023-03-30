#### 概 述
本文介绍STL常用的辅助函数，可用于一些算法题

#### 1. distance
函数原型：
```
template<class InputIterator>
  typename iterator_traits<InputIterator>::difference_type
    distance (InputIterator first, InputIterator last);
```
`distance()`用于计算迭代器`first`和`last`的距离，即`[first, last)`范围内包含的元素个数。示例：
```
#include <iterator> //头文件

vector<int> array(10);
distance(array.begin(), array.end()); //结果10
```

#### 2. accumulate
函数原型：
```
template <class InputIterator, class T>
   T accumulate (InputIterator first, InputIterator last, T init);
```
`accumulate()`用于计算将`[first, last)`范围内的所有值累加到`init`。示例：
```
#include <numeric> //头文件

vector<int> array(10, 2);
accumulate(array.begin(), array.end(), 5); //结果25
```
`accumulate()`支持自定义累加函数，作为第四个参数
```
template <class InputIterator, class T, class BinaryOperation>
   T accumulate (InputIterator first, InputIterator last, T init,
                 BinaryOperation binary_op);
```
示例：
```
vector<int> array(10, 2);
cout << accumulate(array.begin(), array.end(), 5, [](int x, int y) {
    return x + 5 * y; 
}); //结果105
```

#### 3. lower_bound
函数原型：
```
template <class ForwardIterator, class T>
  ForwardIterator lower_bound (ForwardIterator first, ForwardIterator last,
                               const T& val);
```
`lower_bound`用于在`[first, last)`范围内使用**二分查找**，返回不小于`val`的迭代器。支持自定义规则函数，作为第四个参数。
```
template <class ForwardIterator, class T, class Compare>
  ForwardIterator lower_bound (ForwardIterator first, ForwardIterator last,
                               const T& val, Compare comp);
```
示例：
```
#include <algorithm> //头文件

vector<int> array{ 1, 2, 3, 4, 5, 6, 7, 8, 9 }; //为了方便观察，使用有序的数组
vector<int>::iterator iter = lower_bound(array.begin(), array.end(), 4);
cout << iter - array.begin() << endl; //结果3
```
**注：** 因为使用的是二分查找，所以返回的不一定是数组中第一个不小于`val`的迭代器。

#### 4. upper_bound
函数原型：
```
template <class ForwardIterator, class T>
  ForwardIterator upper_bound (ForwardIterator first, ForwardIterator last,
                               const T& val);
```
`upper_bound`用于在`[first, last)`范围内使用**二分查找**，返回大于`val`的迭代器。支持自定义规则函数，作为第四个参数。
原理与`lower_bound`相似，这里不再赘述。