## 23.3.9 更 新
单纯安装 `mysql-server` 可能会没有 `mysql.h` 头文件，我们还需要安装下相关的链接库
```
sudo apt-get install libmysqlclient-dev
```
****
## 概 述
Linux下C++连接MySQL

## 实 现
**1. 环境安装**  
首先是MySQL的安装，直接只用 `apt-get` 命令安装(这里推荐使用这种方式安装，因为此方式在后面程序编译时，不需要自己链接相关路径)
```
sudo apt-get install mysql-server
```
本人安装后，相关头文件以及动态库在如下目录中：
```
// 头文件
/usr/include/mysql/ 
// 动态库
/usr/lib/x86_64-linux-gnu/
```
最后，保证MySQL服务在运行状态，相关命令如下：
```
// 启动MySQL
service mysqld start 
// 停止MySQL
service mysqld stop 
// 重启MySQL
service mysqld restart
// 查看MySQL运行状态
systemctl status mysql.service
```

**2. 配置**  
第一次安装MySQL后，是如法直接用root用户登录的，会报错 `Access denied for user 'root'@'localhost'`。此时，我们先查看MySQL配置文件，获取默认用户名和密码：
```
// 命令
sudo cat /etc/mysql/debian.cnf

// 配置文件内容
[client]
host     = localhost
user     = debian-sys-maint
password = ***(这里我隐藏了，是一串英文和数字)
socket   = /var/run/mysqld/mysqld.sock
[mysql_upgrade]
host     = localhost
user     = debian-sys-maint
password = ***(这里我隐藏了，是一串英文和数字)
socket   = /var/run/mysqld/mysqld.sock
```
我们直接使用配置文件中提供的用户名 `debian-sys-maint` 和密码，登录MySQL
```
$ mysql -udebian-sys-maint -p
Enter password: (这里输入配置文件中的密码)
```
进入MySQL后，配置root账户的密码
```
mysql> update user set password=password('密码') where user='root' and host='localhost'
mysql> flush privileges; //这一步不要忘记，让MySQL重新加载权限数据
```
此时，退出MySQL，我们就可以用root账户和设置的密码正常进入MySQL了。

**3. C++操作MySQL**  
我们创建好数据库以及供demo使用的表和字段，具体的命令操作这里就不展示了。    
使用C++操作MySQL是通过MySQL提供的函数接口进行连接操作，示例代码如下：
```
#include <iostream>
#include <mysql/mysql.h> //mysql提供的函数接口头文件

using namespace std;

int main() {
    const char* host = "localhost"; //主机名
    const char* user = "root"; //用户名
    const char* pwd = "cyh123456!"; //密码
    const char* dbName = "cyh"; //数据库名称
    int port = 3306; //端口号

    // 创建mysql对象
    MYSQL *sql = nullptr;
    sql = mysql_init(sql);
    if (!sql) {
        cout << "MySql init error!" << endl;
    }
    
    // 连接mysql
    sql = mysql_real_connect(sql, host, user, pwd, dbName, port, nullptr, 0);
    if (!sql) {
        cout << "MySql Connect error!" << endl;
    }
    
    // 执行命令
    int ret = mysql_query(sql, "INSERT INTO net_addrInfo VALUES('0.0.0.0',8888)");
    if (ret) {
        cout << "error!" << endl;
    } else {
        cout << "success!" << endl;
    }
    
    // 关闭mysql
    mysql_close(sql);
    return 0;
}
```
项目使用CMake编译，CMakeLists.txt内容如下：
```
cmake_minimum_required(VERSION 3.16)
project(sqlDemo)

set(CMAKE_CXX_STANDARD 14)

add_executable(sqlDemo main.cpp)
target_link_libraries(${PROJECT_NAME} libmysqlclient.so) //链接 libmysqlclient.so
```
这里需要说下，上文中提到的建议使用 `apt-get` 命令安装的原因，如果我们自己下载MySQL源码进行编译安装，在编译自己的程序时可能会遇到如下的几种问题：  
- **找不到mysql相关头文件** ：需要在CMakeLists.txt中使用 `include_directories()` 添加MySQL头文件目录
- **找不到mysql动态链接库** ：需要在CMakeLists.txt中使用 `link_directories()` 添加MySQL动态链接库目录

以上相关的目录，在自己编译安装时需要留意，如果你找不到目录在什么地方，可以使用 `locate` 命令去查找。当然，用 `apt-get` 命令安装就可以避免这些问题了。

最后，我们运行程序，发现数据已经插入到数据库中，完成！