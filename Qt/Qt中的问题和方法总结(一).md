## 1. QString与String的转换
```
//1.QString转换String
QString qstr = "hello";
string str = qstr.toStdString();

//2.String转换QString
string str = "hello";
QString qstr = QString::fromStdString(str);
```

## 2. QString与int的转换
```
//1.QString转int
QString str = "100";
int tmp = str.toInt();

//2.int转QString
int tmp = 100;
QString str = QString::number(tmp);
```

## 3. Qt计时器的使用
头文件 `QTime`
```
#include <QTime>

QTime time;
time.start(); //开始计时

//···代码

int time_Diff = time.elapsed(); //返回从上次start()或restart()开始以来的时间差，单位ms
```

## 4. qDebug()的使用
`qDebug()` 函数，可以将调试信息直接输出到控制台上，有两种方式：  
(1) 将字符串当做参数传给qDebug()函数。  
(2) 使用流输出的方法输出多个字符串。  
```
//头文件
#include <QDebug>

//将字符串作为参数传给qDebug()
int x = 1;
qDebug("x : %d",x);
 
//使用流的方法输出
int y = 2;
qDebug() << "y : " << y;
```

## 5. Qt项目打印信息到控制台
在Qt项目中使用 `cout` 或者 `qDebug` 输出时，看不到输出结果  
需要设置 项目属性 > 链接器 > 系统 > 子系统 的值为 `控制台 (/SUBSYSTEM:CONSOLE)`，再次编译后，会弹出控制台命令框窗口，显示调试信息。 

![设置属性](https://upload-images.jianshu.io/upload_images/22192996-7e7d3628b946505c.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

这里的配置和平台一定要与运行时的一致，否则设置无效

## 6. Qt容器与C++容器的区别
在刚开始写Qt项目时，常遇到一些容器使用的问题，这里简单说下QTL和STL的区别。更多深入的知识可以自行搜索学习。  
C++中有的容器，Qt都有对应的容器。其用法类似，名称和头文件略有不同，例如：
```
//头文件：
//C++
#include <algorithms.h>
#include <vector>

//Qt
//这是用于算法的。
#include <qalgorithms.h>
//这是用于QT QVector容器的
#include <QVector>
```
等等，在C++中的容器、算法前加上 `Q` 即对应Qt中的容器、算法(绝大部分)。所以C++中叫STL,  而QT叫QTL
***
2020.7.10更新
## 7. show与exec的区别
`show()` 和 `exec()` 都用于显示窗口，`show()` 默认显示的是非模态对话框，即此对话框出现后你还可以对其他窗口进行操作。而 `exec()` 出现的只能是模态对话框，即无法对其他窗口进行操作。  
`show()` 显示窗口，可以用 `setModal` 函数进行设置窗口为模态，即无法操作其他窗口，即被阻塞。

## 8. 判断字符串中是否有某个子字符串
在C++中常会使用 `strstr()` 函数来判断字符串中是否有某个子字符串；qt中有内置的函数，即 `contains()`
```
QString Content = ui->consoleEdit->text(); 
if(Content.contains("test")) { 
    ui->answerLabel->setText("yes"); 
} else { 
    ui->answerLabel->setText("no"); 
} 
```

## 9. 如何手动生成moc_xxx.cpp文件
当在VS工程中编写QT和C++程序时，要想不同模块之间通过QT的信号（SIGNALS）和槽（SLOT）的机制进行通信，就需要继承于Q_OBJECT基类，继承于Q_OBJECT基类的类（文件），会相应的生成一个moc文件。  
若没有生成moc文件，则会编译报错。  
**手动生成moc_xxx.cpp文件的方法：**  
右键单击要生成moc文件的.h文件，点击属性->常规，选择自定义生成工具  

![设置自定义生成工具](https://upload-images.jianshu.io/upload_images/22192996-4c10093c1140731a.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

选择下面的自定义生成工具，设置命令行、输出、和依赖项  

![设置生成命令](https://upload-images.jianshu.io/upload_images/22192996-dd57a660787284b5.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

```
//命令行
$(QTDIR)\bin\moc.exe xxx.h -o .\GeneratedFiles\$(ConfigurationName)\moc_%(Filename).cpp
//输出
.\GeneratedFiles\$(ConfigurationName)\moc_%(Filename).cpp
//依赖项
$(QTDIR)\bin\moc.exe %(FullPath)
```
重新编译运行即可。
