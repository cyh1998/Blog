## 前 言
今天刷Leetcode的时候，使用了 `pair<>` 作为 `unordered_map` 的key，编译时报错。

## 解 决
如果没有特殊的需要，可以使用map来代替unordered_map，这样可以通过编译

## 原 因
map是有序的，底层使用的红黑树，map需要对key进行相互比较，从而确定具体插入的位置，所以map的key值需要支持比较函数。而pair重载了相对应的比较操作符，所以使用map没有问题

相反，unordered_map是无序的，是基于哈希表的数据结构，unordered_map需要对key进行hash函数，利用hash出的唯一值确定插入对象的位置，所以unordered_map的key值需要支持hash函数，而pair没有默认的hash函数，所以会编译出错

当然，我们也可以自定义pair类型的hash函数，即重载 `operator()`，返回一个 `size_t` 类型，从而使得 `unordered_map` 支持用 `pair<>` 做为key值
```
struct pair_hash
{
    template<typename T1, typename T2>
    std::size_t operator() (const std::pair<T1, T2>& p) const {
        return std::hash<T1>{}(p.first) ^ std::hash<T2>{}(p.second);
    }
};

struct pair_equal
{
    template<typename T1, typename T2>
    bool operator() (const std::pair<T1, T2>& a, const std::pair<T1, T2>& b) const {
        return a.first == b.first && a.second == b.second;
    }
};

int main()
{
    unordered_map<pair<int, int>, int, pair_hash, pair_equal> m;

    return 0;
}
```
为了更加合理，我们还需要添加一个判断pair是否相等的函数，意在解决冲突。
