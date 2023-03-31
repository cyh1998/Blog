## 前 言
之前刷Leetcode题：[最接近原点的 K 个点](https://leetcode-cn.com/problems/k-closest-points-to-origin/)，本题直接用`sort()`排序会超时，看到大佬使用了一个叫 `nth_element()` 的函数，因此本文介绍STL中 `nth_element()` 的使用。

## nth_element
首先看下函数原型：
```
template <class RandomAccessIterator>
void nth_element (RandomAccessIterator first, RandomAccessIterator nth,
                    RandomAccessIterator last);
```
函数说明：  
重新排列range[first,last)中的元素，使第nth个位置的元素是按排序顺序在该位置的元素。其他元素没有任何特定的顺序，只是第nth个元素之前的元素都不大于该元素，而第nth个元素后面的元素均小于该元素。
这个很好理解，举个例子
```
//原始数列
[2, 8, 3, 7, 1, 9, 4, 5, 6, 0]

//nth_element nth = 5
[x, x, x, x, x, 5, x, x, x, x]
```
`nth_element()` 将会使得第nth个位置的元素是按排序顺序在该位置的元素，即为5，其他元素没有任何特定的顺序，但保证5之前的元素均比5小，5之后的元素均比5大。(类似于快速排序的原理)

当然，我们可以自定义判断函数，作为 `nth_element()` 的第四个参数
```
template <class RandomAccessIterator, class Compare>
void nth_element (RandomAccessIterator first, RandomAccessIterator nth,
                    RandomAccessIterator last, Compare comp);
```
因此上述Leetcode题的解法：
```
class Solution {
public:
    vector<vector<int>> kClosest(vector<vector<int>>& points, int K) {
        // 法一：排序(超时)

        // 法二：利用multimap
        // multimap<int, vector<int>> data;
        // vector<vector<int>> res;
        // int k = 0;

        // for (int i = 0; i < points.size(); ++i) {
        //     data.insert(pair<int, vector<int>>(pow(points[i][0], 2) + pow(points[i][1], 2), points[i]));
        // }

        // multimap<int, vector<int>>::iterator it = data.begin();
        // while(k < K) {
        //     res.push_back(it->second);
        //     it++;
        //     k++;
        // }

        // return res;

        //法三：优先队列 priority_queue
        // priority_queue<pair<int, vector<int>>> q;
        // int i;

        // for (i = 0; i < K; ++i) {
        //     q.push(pair<int, vector<int>>(pow(points[i][0], 2) + pow(points[i][1], 2), points[i]));
        // }

        // for (; i < points.size(); ++i) {
        //     int dis = pow(points[i][0], 2) + pow(points[i][1], 2);
        //     if (dis < q.top().first) {
        //         q.pop();
        //         q.push(pair<int, vector<int>>(dis, points[i]));
        //     }
        // }

        // vector<vector<int>> res;
        // while (!q.empty()) {
        //     res.push_back(q.top().second);
        //     q.pop();
        // }
        // return res;

        //法四：nth_element
        nth_element(points.begin(), points.begin() + K, points.end(), [](vector<int>& ptA, vector<int>& ptB){
            return ptA[0] * ptA[0] + ptA[1] * ptA[1] < ptB[0] * ptB[0] + ptB[1] * ptB[1];
        });
        return {points.begin(), points.begin()+K};
    }
};
```
也可以学习下其他的解法。

## 进一步讨论
本人在VS 2019(MSVC 2017 编译器)下实测，发现了一个问题  
测试代码：
```
int main()
{
    vector<int> numbers;
    for (int i = 1; i < 15; i++) numbers.push_back(i);
    random_shuffle(numbers.begin(), numbers.end()); //重新排列数组

    for (auto i : numbers) {
        cout << i << " ";
    }
    cout << endl;

    nth_element(numbers.begin(), numbers.begin() + 5, numbers.end());
    for (auto i : numbers) {
        cout << i << " ";
    }
}
```
运行结果：

![结果](https://upload-images.jianshu.io/upload_images/22192996-b97cc6874cb6194d.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

奇怪的事情发生了，为什么 `nth_element()` 把整个数组都排列了？  
查看了下源码，发现了其中的原因：  
```
// FUNCTION TEMPLATE nth_element
template <class _RanIt, class _Pr>
_CONSTEXPR20 void nth_element(_RanIt _First, _RanIt _Nth, _RanIt _Last, _Pr _Pred) {
    // order Nth element, using _Pred
    _Adl_verify_range(_First, _Nth);
    _Adl_verify_range(_Nth, _Last);
    auto _UFirst     = _Get_unwrapped(_First);
    const auto _UNth = _Get_unwrapped(_Nth);
    auto _ULast      = _Get_unwrapped(_Last);
    if (_UNth == _ULast) {
        return; // nothing to do
    }

    while (_ISORT_MAX < _ULast - _UFirst) { // divide and conquer, ordering partition containing Nth
        auto _UMid = _Partition_by_median_guess_unchecked(_UFirst, _ULast, _Pass_fn(_Pred));

        if (_UMid.second <= _UNth) {
            _UFirst = _UMid.second;
        } else if (_UMid.first <= _UNth) {
            return; // Nth inside fat pivot, done
        } else {
            _ULast = _UMid.first;
        }
    }

    _Insertion_sort_unchecked(_UFirst, _ULast, _Pass_fn(_Pred)); // sort any remainder
}
```
源码中的 `_ISORT_MAX` 为32，所以，在VS环境下，当数组长度小于32时，`nth_element()` 会将数组全排序。数组超过32长度时，才会排序第nth位数字。  
使用超过32长度的数组，再试一次，代码都不展示了  

![结果](https://upload-images.jianshu.io/upload_images/22192996-2968be5f2a3a5542.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

得到了预期的结果。

更多我的Leetcode题解，详见 [Leetcode题解](https://github.com/cyh1998/algorithm)