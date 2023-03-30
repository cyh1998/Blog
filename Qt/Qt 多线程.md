#### 概述
Qt中有多种方式实现多线程，这里主要简单介绍Qt中 `moveToThread` 和 `QtConcurrent::run` 创建线程的方法，以及如何在线程中和Qt界面交互。

Qt中最基础的线程创建方式是使用QThread，即新建一个线程类继承QThread，重写 `run()` 函数并通过 `start()` 函数启动线程。因为Qt官方已经不推荐使用这种方式，所以这里不再阐述。

#### moveToThread
`moveToThread` 是在QThread的用法基础上扩展出来，其继承于QObject类。即通过继承QObject类并使用 `moveToThread` 移到一个QThread的对象中。
示例如下：  
首先，创建一个继承于QObject的类，定义一个槽函数，即在线程中需要执行的函数；定义一个信号，用于线程完成时通知界面。
```
//MyThread.h
#include <QObject>

class MyThread: public QObject {
	Q_OBJECT
public:
	explicit MyThread(QObject *parent = 0);
	~MyThread();
signals:
	void Threadfinish(); //线程完成时的信号
public slots:
	void slot_StartMyThread(); //需要在线程中执行的槽函数
};

//MyThread.cpp
#include "MyThread.h"
#include <QDebug>

MyThread::MyThread(QObject *parent) : QObject(parent) {
}

MyThread::~MyThread(){
}

void MyThread::slot_StartMyThread() {
    //打印线程ID
	qDebug() << "kid " << "threadID : " << QThread::currentThread();

    //...需要执行的内容

	//返回线程完成信号
	emit Threadfinish();
}
```
主界面(因为是公司项目，这里展示主要的代码)：
```
//.h文件
//类中定义：
public:
	void start_thread(); //开启线程函数
signals:
	void sig_startThread(); //通知子线程的信号
private:
	MyThread* m_myThread; //线程指针
	QThread* subthread;
public slots:
	void slot_finishThread(); //线程完成的槽函数

//.cpp文件
void MainFrameUi::start_thread() {
	qDebug() << "main " << "threadID : " << QThread::currentThread();
	subthread->start(); //开启线程
	emit sig_startThread(); //通过信号通知子线程任务
}

void MainFrameUi::slot_finishThread() {
    //处理线程完成的函数    
    //...
}

//构造函数中初始化，也可以在函数中初始化
subthread = new QThread();
m_myThread= new MyThread();
m_myThread->moveToThread(subthread);
connect(this, SIGNAL(sig_startThread()), m_myThread, SLOT(slot_StartMyThread())); 
connect(m_myThread, SIGNAL(Threadfinish()), this, SLOT(slot_finishThread()));
connect(subthread, &QThread::finished, m_myThread, &QObject::deleteLater);
```
初始化后，使用 `start_thread()` 函数开启进程。  
`subthread = new QThread();` 使用QThread创建一个多线程容器  
`m_myThread= new MyThread();` 新建自己的线程类  
`m_myThread->moveToThread(subthread);` 将线程类移动到线程容器中  
`connect(this, SIGNAL(sig_startThread()), m_myThread, SLOT(slot_StartMyThread()));` 开始线程信号绑定到子线程中需要执行的槽函数  
`connect(m_myThread, SIGNAL(Threadfinish()), this, SLOT(slot_finishThread()));` 子线程结束信号绑定到主界面的处理线程完成函数  
`connect(subthread, &QThread::finished, m_myThread, &QObject::deleteLater);` subthread线程结束后，让m_myThread销亡  

**运行结果**
```
main threadID : QThread(0x1393a0920d)
kid threadID : QThread(0x139409cd290)
```

#### QtConcurrent::run
QtConcurrent::run是Qt提供了的高级API接口，能够方便快捷的将任务放到子线程中去执行，无需继承任何类，也不需要重写函数，使用非常简单。  
既可以执行普通函数，也可以执行类成员函数。  
示例如下：  
本人使用VS编写Qt，首先在项目Qt设置中勾选Concurrent模块  

![Qt项目设置](https://upload-images.jianshu.io/upload_images/22192996-52671853c903e8ac.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

引入头文件 `#include <QtConcurrent>`  
使用QtConcurrent::run将任务放入子线程中  
```
//QFuture<T> QtConcurrent::run(Function function, ...)
QString str = "test";
QFuture<void> fun_res1 = QtConcurrent::run(fun1,str)
//QFuture<T> QtConcurrent::run(QThreadPool *pool, Function function, ...)
QFuture<void> fun_res2 = QtConcurrent::run(this, &MainFrameUi::fun2, str)
```
第一种，分别传入函数指针和参数；  
第二种，第一个参数为一个const引用或一个指向该类实例的指针，第二个参数为该类的成员函数，第三往后为函数的参数。  
**注：** 当传入函数的参数大于5个时，则需要将参数定义为结构体，使用结构体传参，否则函数报错。

