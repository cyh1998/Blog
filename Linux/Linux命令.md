**1. 查看进程PID**  
```
ps -ef | grep 进程名称
```
**2. 查看进程启动路径**  
```
#1
ls -l /proc/进程PID

#2
cd /proc/进程PID
grep -r 进程名称
```
**3. 滚动显示文件末尾内容(追踪文件描述符)**  
```
tail -f filename
```
**4. 显示系统时间**  
```
date
```
**5. 修改系统时间**  
```
date -s "2022-08-01 18:00:00"
```
**6. iostat相关**  
cpu属性值说明：
- `%user`：CPU处在用户模式下的时间百分比
- `%nice`：CPU处在带NICE值的用户模式下的时间百分比
- `%system`：CPU处在系统模式下的时间百分比
- `%iowait`：CPU等待输入输出完成时间的百分比
- `%steal`：管理程序维护另一个虚拟处理器时，虚拟CPU的无意识等待时间百分比
- `%idle`：CPU空闲时间百分比

disk属性值说明：
- `tps`：每秒钟发送到的I/O请求数
- `Blk_read/s`：每秒读取的block数
- `Blk_wrtn/s`：每秒写入的block数
- `Blk_read`：读入的block总数
- `Blk_wrtn`：写入的block总数

每隔1秒，显示一次统计信息
```
iostat -d 1
```
**7. top相关**  
参考 [linux top](https://www.csdn.net/tags/OtDaUg1sODA1NC1ibG9n.html)

**8. 设置文件所有者**  
```
chown [-cfhvR] user[:group] file...
```
参数：  
`user` : 新的文件拥有者的使用者 ID  
`group` : 新的文件拥有者的使用者组(group)  
`-c` : 显示更改的部分的信息  
`-f` : 忽略错误信息  
`-h` :修复符号链接  
`-v` : 显示详细的处理信息  
`-R` : 处理指定目录以及其子目录下的所有文件  

实例：
```
# 将目录/var/log/mysql的所有者设置为mysql
chown mysql /var/log/mysql
```
**9. 压缩/解压缩文件**  
```
# 将./test目录压缩为test.tar.gz
tar -czvf test.tar.gz ./test 

# 解压test.tar.gz
tar -xzvf test.tar.gz
```
**10. 待补充**  
