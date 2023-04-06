## 简 述
在QT中，我们经常使用 `qDebug()`、`qInfo()` 等来打印调试的信息，但是当打印信息过多时，很不利于查找阅读。所以本文介绍使用 `QtMessageHandler` 类中的 `qInstallMessageHandler()` 来自定义处理调试信息。

## 实 现
一、在主线程中注册调试信息处理回调
```
//注册MessageHandler
qInstallMessageHandler(outputMessage);
```
这里的 `outputMessage` 即为自定义的触发函数，当程序有调试信息时，将会调用此函数

二、实现触发函数
```
void outputMessage(QtMsgType type, const QMessageLogContext &context, const QString &msg)
{
    QString qstrText;
    QString msgType;

    switch (type)
    {
    case QtDebugMsg:
        msgType = "Debug:";
        break;
    case QtInfoMsg:
        msgType = "Info:";
        break;
    case QtWarningMsg:
        msgType = "Warning:";
        break;
    case QtCriticalMsg:
        msgType = "Critical:";
        break;
    case QtFatalMsg:
        msgType = "Fatal:";
        break;
    }

    qstrText = QString(msgType + "%1 [%2:%3] %4").arg(QDateTime::currentDateTime().toString("yyyy-MM-dd hh:mm:ss"))
                                                .arg(context.file) 
                                                .arg(context.line)
                                                .arg(msg);

    QString logFile = QDateTime::currentDateTime().toString("yyyyMMdd") + ".log";

    QFile out(logFile);
    if (out.open(QIODevice::WriteOnly | QIODevice::Append)) {
        QTextStream ts(&out);
        ts << qstrText << endl;
    }

}
```
说明：此函数需要接受三个参数  
- `QtMsgType type` ：表示调试信息类型，包括 `QtDebugMsg` (调试消息)、`QtInfoMsg` (信息消息)、`QtWarningMsg` (警告消息和可恢复的错误)、`QtCriticalMsg` (关键错误和系统错误)、`QtFatalMsg` (致命错误)  
- `const QMessageLogContext &context` ：表示有关日志消息的其他信息，比如文件名 `context.file`、行号 `context.line` 等等。  
- `const QString &msg` ：表示原始的调试信息。 

这样，我们就可以根据调试信息类型，自定义处理调试信息，并打印到日志文件等等。  

## 进一步讨论
但是有时候，我们会有这样的需求，有些类型的信息需要打印到屏幕，而有些类型的信息需要打印到日志。当注册了调试信息处理的回调，如何分类去处理呢？  
查看QT文档中对于 `qInstallMessageHandler()` 的描述，可以知道该函数返回一个指向上一个消息处理程序，可以理解为上一个消息处理函数的指针。因此在使用 `qInstallMessageHandler()` 注册回调时，可以保存函数的返回，从而用之前的处理程序来处理调试信息  

例如：
```
QtMessageHandler s_messageHandler = qInstallMessageHandler(outputMessage);
```
使用 `s_messageHandler` 来保存函数的返回值，即指向了上一个消息处理函数。在 `outputMessage()` 函数中使用 `s_messageHandler`
```
void outputMessage(QtMsgType type, const QMessageLogContext &context, const QString &msg)
{
    QString qstrText;
    QString msgType;

    switch (type)
    {
    case QtDebugMsg:
        s_messageHandler(type, context, msg);
        return;
    case QtInfoMsg:
        msgType = "Info:";
        break;
    case QtWarningMsg:
        msgType = "Warning:";
        break;
    case QtCriticalMsg:
        msgType = "Critical:";
        break;
    case QtFatalMsg:
        msgType = "Fatal:";
        break;
    }

    qstrText = QString(msgType + "%1 [%2:%3] %4").arg(QDateTime::currentDateTime().toString("yyyy-MM-dd hh:mm:ss"))
                                                .arg(context.file)
                                                .arg(context.line)
                                                .arg(msg);

    QString logFile = QDateTime::currentDateTime().toString("yyyyMMdd") + ".log";
    QFile out(logFile);
    if (out.open(QIODevice::WriteOnly | QIODevice::Append)) {
        QTextStream ts(&out);
        ts << qstrText << endl;
    }
}
```
这样就实现了将Info等信息打印到日志，而debug信息打印到屏幕。  
**注：** 以上写入日志文件的写法，并不是线程安全的，需要加锁来保证线程安全，这里就不再赘述。

## 问 题
正常的运行程序，日志内容如下：

![日志](https://upload-images.jianshu.io/upload_images/22192996-fb9a14f6c5c71468.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

实际项目中遇到了编译出的Release版本，日志输出没有文件信息、行数的问题。如下：

![日志](https://upload-images.jianshu.io/upload_images/22192996-e0b31efaad9b1536.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

**解决方法：**  
在.pro文件中添加宏
```
DEFINES += QT_MESSAGELOGCONTEXT
```
一定要先删除掉之前编译的中间文件，重新qmake！这样就可以在Release版本中正确输出日志信息。
