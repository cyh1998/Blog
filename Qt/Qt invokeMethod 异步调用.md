#### 概述
程序中，我们经常会调用函数，如果调用的函数耗时较长，同步调用会造成主程序的堵塞。Qt中提供了一个便捷的函数`QMetaObject::invokeMethod`，方便我们异步调用，从而解决这一问题。本文只要讲述`QMetaObject::invokeMethod`的使用方法。

#### 函数原型
```
bool QMetaObject::invokeMethod(QObject *obj, const char *member, 
							Qt::ConnectionType type, 
							QGenericReturnArgument ret,
							QGenericArgument val0 = QGenericArgument(nullptr), 
							QGenericArgument val1 = QGenericArgument(), 
							QGenericArgument val2 = QGenericArgument(), 
						    QGenericArgument val3 = QGenericArgument(), 
							QGenericArgument val4 = QGenericArgument(),
							QGenericArgument val5 = QGenericArgument(),
							QGenericArgument val6 = QGenericArgument(), 
							QGenericArgument val7 = QGenericArgument(),
							QGenericArgument val8 = QGenericArgument(),
							QGenericArgument val9 = QGenericArgument())
```
此函数用于调用对象的成员(信号或插槽)。如果可以调用成员，则返回true。如果没有此类成员或参数不匹配，则返回false。  
`QMetaObject::invokeMethod`除上文这个函数以外还有5个重载函数，这里不再赘述。
**参数说明：**  
`obj`：被调用对象的指针  
`member`：成员方法的名称  
`type`：连接方式，默认值为 Qt::AutoConnection  
- Qt::DirectConnection，则会立即调用该成员。(同步调用)
- Qt::QueuedConnection，则会发送一个QEvent，并在应用程序进入主事件循环后立即调用该成员。(异步调用)
- Qt::BlockingQueuedConnection，则将以与Qt :: QueuedConnection相同的方式调用该方法，除了当前线程将阻塞直到事件被传递。使用此连接类型在同一线程中的对象之间进行通信将导致死锁。(异步调用)
- Qt::AutoConnection，则如果obj与调用者位于同一个线程中，则会同步调用该成员; 否则它将异步调用该成员。

`ret`：接收被调用函数的返回值  
`val0`~`val9`：传入被调用函数的参数，最多十个参数  
注：必须要使用`Q_RETURN_ARG()`宏来封装函数返回值、`Q_ARG()`宏来封装函数参数。

#### 示例
若一个对象obj有一个槽函数func(QString,int),返回值为bool，那么调用方式如下：
```
bool result;
//同步调用
QMetaObject::invokeMethod(obj, "func", Qt::DirectConnection,
                          Q_RETURN_ARG(bool, result),
                          Q_ARG(QString, "test"),
                          Q_ARG(int, 100);
//异步调用
QMetaObject::invokeMethod(obj, "func", Qt::QueuedConnection,
                          Q_ARG(QString, "test"),
                          Q_ARG(int, 100);
```
**注：** 使用`Qt::QueuedConnection`异步调用，将无法获取返回值，因为此连接方式只是负责把事件交给事件队列，然后立刻返回，所以，函数返回值就无法确定了。
但，我们可以使用上文提及的`Qt::BlockingQueuedConnection`连接方式，这个连接方式会阻塞发射信号的线程一直等到队列连接槽返回后，才会恢复阻塞，这样就可以保证我们能得到函数的返回值。使用如下：
```
bool result;
QMetaObject::invokeMethod(obj, "func", Qt::BlockingQueuedConnection,
                          Q_RETURN_ARG(bool, result),
                          Q_ARG(QString, "test"),
                          Q_ARG(int, 100);
```
需要注意的是，qt官方文档中的提醒：**使用此连接类型在同一线程中的对象之间进行通信将导致死锁**

最后，因为连接方式`type`默认值为`Qt::AutoConnection`，所以当被调用的obj与调用者不在同一线程中，可以直接调用：
```
//此Tcpserver为一个独立线程
//在主程序中调用reportImageResult()，因为TcpServer对象与调用者主线程不在同一线程中。
//Qt::AutoConnection此连接方式将会自动以异步的方式调用
void TcpServer::reportImageResult(int imgId, const QImage &image, int result)
{
    QMetaObject::invokeMethod(this, "reportImageResultAsync",
                              Q_ARG(int, imgId),
                              Q_ARG(QImage, image),
                              Q_ARG(int, result);
}
```
#### 进一步讨论
使用`QMetaObject::invokeMethod`来调用函数时，当函数的参数有自定义类型时，程序将会报错，因为调用的类型必须是信号、槽，以及Qt元对象系统能识别的类型。可以使用Qt命名类型所提供的`qRegisterMetaType()`来注册自定义类型。
示例如下：
```
//头文件
#include <QMetaType>

//自定义类型
struct AsynResults {
	int imgId;
	QImage image;
	int result;
};

//在类构造时，注册类型
qRegisterMetaType<AsynResults>("AsynResults");

//QMetaObject::invokeMethod调用
QMetaObject::invokeMethod(this, "reportImageResultAsync",
                              Q_ARG(AsynResults, asynresults);
```