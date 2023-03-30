#### 概  述
本文介绍如何设置MySQL从而实现查看对数据库执行的所有SQL语句。

#### 实  现
**一、临时设置**  
首先通过 `mysql -u -p` 连接MySQL，设置相关变量
```
set global general_log = on;
set global general_log_file = '/var/log/mysql/mysql.log';
```
- `general_log`： 是否开启log，`on`开启，`off`关闭
- `general_log_file`：log输出路径

*注：*log输出路径必须保证存在且MySQL有权限写入。

查看变量的值
```
show variables like "general_log";
show variables like "general_log_file";
```
此时已经生效了，临时设置不需要重启MySQL

**二、全局设置**  
直接修改MySQL的配置文件，本人MySQL版本为5.7.31，配置文件路径为`/etc/mysql/mysql.conf.d/mysqld.cnf`。不同的版本可能路径略有不同。  
将如下的两行取消注释并配置即可
```
#general_log_file        = /var/log/mysql/mysql.log
#general_log             = 1
```
当然，正如配置中注释写的：
```
#
# The MySQL database server configuration file.
#
# You can copy this to one of:
# - "/etc/mysql/my.cnf" to set global options,
# - "~/.my.cnf" to set user-specific options.
# 
```
我们可以复制此文件到`/etc/mysql/my.cnf`作为全局配置，也可以复制到`~/.my.cnf`作为用户特定设置。  
此方式需要重启MySQL  
```
service mysql restart
```





