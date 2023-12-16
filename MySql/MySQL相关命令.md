#### 查看表结构
```
desc your_table_name;
```
#### 执行外部sql文件
```
# 绝对路径
source your_filename.sql;
```
#### 查看时间
```
select now();
```
#### 函数和存储过程相关
```
# 查看全部
## 存储过程
show procedure status;
## 函数
show function status; 

# 查看指定数据库
## 存储过程
show procedure status where db = 'your_db_name';
select `name` from mysql.proc where db = 'your_db_name' and `type` = 'PROCEDURE'; 
## 函数
show function status where db = 'your_db_name'; 
select `name` from mysql.proc where db = 'your_db_name' and `type` = 'FUNCTION';  

# 删除
## 存储过程
drop procedure if exists your_procedure_name;
## 函数
drop function if exists your_function_name;

# 查看创建代码
## 存储过程
show create procedure your_procedure_name;
## 函数
show create function your_function_name;
```
#### 导出数据
```
## 导出数据库
mysqldump -u username -p password db_name > mysql.db
## 导出数据表
mysqldump -u username -p password db_name  table_name > mysql.sql
```