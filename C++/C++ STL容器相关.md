#### 一、map转vector
代码：
```
vector<pair<key,value>> vec(map.begin(), map.end());
```
#### 二、map根据value排序
首先map是根据key来排序的，理论上我们无法将map根据value来排序，但是我们可以先将map转为vector，然后利用`sort()`来排序
```
#include <vector>
#include <algorithm>
#include <map>

//按照second降序排序，可自定义排序规则
bool cmp(pair<int, float>a, pair<int, float>b){
    return a.second > b.second;
}

int main(){
    //...
    vector<pair<key,value>> vec(map.begin(), map.end()); //先将map转为vector
    sort(res.begin(),res.end(),cmp); //利用sort排序
}
```
#### 三、将map的value设为数组
1. 是用vector代替数组
2. 利用类
```
//类
class contest{
    public:
        string contest_name;
        int contest_level;
};

map<int,contest> read_contest_list(string path){
    map<int,contest> res_list;
    contest inlist;
    inlist.contest_name = "A比赛";
    inlist.contest_level = 5;
    res_list.insert({1,inlist});
    return res_list;
}
```
当然也可以使用结构体，这里不再赘述
#### 四、vector去重
1. 利用set的特性，即set种的值唯一，将vector转为set，再转回vector
```
set<int> s(vec.begin(), vec.end());
vec.assign(s.begin(), s.end());
```
2. 结合`unique()`和`erase()`函数
`unique()`函数将相邻且重复的元素放到vector的尾部，返回值是第一个重复元素的迭代器，再用`erase()`函数擦除从这个元素到最后元素的所有的元素。记得需要先排序！
```
sort(vec.begin(), vec.end());
vec.erase(unique(vec.begin(), vec.end()), vec.end());
```
#### 五、STL中允许重复key的multimap
我们知道map类型是不允许重复的key，但是multimap允许key重复，其用法与map类似
#### 六、map的相关操作
#### 插入
```
map<int,string> m;

//用insert函數插入pair
m.insert(pair<int, string>(100, "good"));
//或者
m.insert({100, "good"});
 
//用"array"方式插入
m[100] = "good";
```
区别：用`insert()`插入，当key重复时，会插入失败；而用数组插入，将会覆盖重复key的值
#### 查找
```
map<int,string> m;

//使用count()
if(m.count(key) > 0){
    return m[key];
}
return null;

//使用find()
map<int,string>::iterator iter = m.find(key);
if(iter != m.end()){
    return iter->second;
}
return null;
```
#### 遍历
```
map<int,string> m;
 
map<int,string>::iterator it = m.begin();
while(it != m.end()){
    //it->first;
    //it->second;
    it ++;         
}
```
#### 删除
```
map<int,string> m;

//迭代器删除
map<int,string>::iterator iter = m.find(1);
m.erase(iter);
//关键字删除
m.erase(1);

m.erase(m.begin(), m.end()); //清空map
```
#### 七、map和set的区别
map的set底层都是用红黑树实现，都不允许键重复。  
区别：map以键值对的形式存储，set只有值，且值就是键。map可以修改某个key的value，set不可修改