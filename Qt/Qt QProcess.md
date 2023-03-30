#### 概 述
本文介绍，在Linux环境下，使用Qt中的`QProcess`类执行shell命令并获取输出。
头文件：`#include <QProcess>`

#### 实 现
**一、函数接口**  
`QProcess`类提供了三个函数  
1. `QProcess::execute()` 以堵塞方式的执行shell命令，当命令执行完成后，调用进程才会继续执行。命令输出的任何数据都将转发给调用进程输出(因此无法捕获)。  
2. `QProcess::start()` 以异步方式的执行shell命令，命令输出的数据存储于缓冲区，可以通过`readAllStandardOutput()`捕获  
3. `QProcess::startDetached()` 以分离的方式执行shell命令，调用进程退出，则分离的进程将继续运行，而不受影响。  

**二、执行命令**  
这里主要介绍`execute()`和`start()`：
```
//无参数的shell命令
QProcess p;
p.execute("pwd")  //执行pwd命令

//有参数的shell命令
QProcess p;
p.execute("ifconfig", {-a})  //执行ifconfig -a命令

QStringList  list;  
list << "-a";  
p.execute("ifconfig", list); //以QStringList传递参数
```
`execute()`会将命令输出直接打印到控制台，调用程序无法捕获。
```
//无参数的shell命令
QProcess p;
p.start("pwd")  //执行pwd命令

//有参数的shell命令
QProcess p;
p.start("ifconfig", {-a})  //执行ifconfig -a命令

QStringList  list;  
list << "-a";  
p.start("ifconfig", list); //以QStringList传递参数

//结果捕获
p.waitForFinished(); //等待shell命令执行完成
QString  str = p.readAllStandardOutput(); //捕获输出
qDebug() << str;
```
调用程序可通过`readAllStandardOutput()`捕获shell命令的输出

**三、管 道**  
对于shell命令中的`|`，直接传入参数是不行的。
```
//查看CUP ID命令
sudo dmidecode -t 4 | grep ID |sort -u |awk -F': ' '{print $2}'
```
```
QProcess p;
p.start("dmidecode -t 4 | grep ID |sort -u |awk -F': ' '{print $2}'");
```
以上的方式是无法执行的。  
可以将整个命令作为`sh`的参数传入 或 使用`QProcess::setStandardOutputProcess(QProcess *destination)`即将一个进程的标准输出流传入目标进程的标准输入流
```
//将整个命令作为sh的参数传入
QProcess p;
p.start("sh", QStringList() << "-c" << "dmidecode -t 4 | grep ID |sort -u |awk -F': ' '{print $2}'");
p.waitForFinished();
QString str = p.readAllStandardOutput();
qDebug() << str;

//使用setStandardOutputProcess()
QProcess process1;                                
QProcess process2;                                
QProcess process3;                                
QProcess process4;                                
                                                  
process1.setStandardOutputProcess(&process2);     
process2.setStandardOutputProcess(&process3);     
process3.setStandardOutputProcess(&process4);     
                                                  
process1.start("sudo fdisk -l");                  
process2.start("grep ID");                        
process3.start("sort -u");                        
process4.start("awk", {"-F", ": ", "{print $2}"});
                                                  
process4.waitForFinished(); //等待最后一个命令执行完成               
QString str = process4.readAllStandardOutput();     
qDebug() << str;                                    
```
对于需要sudo权限的命令，需要使用sudo权限打开qtcreator，或者直接在命令前加上sudo(不建议)。  

当然，`QProcess`不仅仅可以执行shell命令，也可以用于执行调用外部程序。