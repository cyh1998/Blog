## 实现用C++读取.csv文件，并存到STL中
.csv文件即将表格数据转换为用分隔字符分隔的值(也可以不是逗号)
### 头文件：
```
#include <fstream>
#include <string>
#include <sstream>
```
### 简单的demo
```
int main(){
    vector<vector<int>> user_arr;
    ifstream fp("xxx/user_data.csv"); //定义声明一个ifstream对象，指定文件路径
    string line;
    getline(fp,line); //跳过列名，第一行不做处理
    while (getline(fp,line)){ //循环读取每行数据
        vector<int> data_line;
        string number;
        istringstream readstr(line); //string数据流化
        //将一行数据按'，'分割
        for(int j = 0;j < 11;j++){ //可根据数据的实际情况取循环获取
            getline(readstr,number,','); //循环读取数据
            data_line.push_back(atoi(number.c_str())); //字符串传int
        }
        user_arr.push_back(data_line); //插入到vector中
    }
    return 0;
}
```
补充：
将字符串类型数据转换成 `int` 类型需要先使用 `.c_str()` 转成 `const char*` 类型，再用 `atoi()` 转成 `int` ，如果转为浮点型则 `atof()` ，`long` 型则 `atol()` 等等。
### 结果  
![.csv文件](https://upload-images.jianshu.io/upload_images/22192996-d0064e2bc504fdf7.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

![输出结果](https://upload-images.jianshu.io/upload_images/22192996-4b0d79d5588742d3.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
