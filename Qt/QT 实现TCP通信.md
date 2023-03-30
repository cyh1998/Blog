#### 概述
使用QT来实现TCP连接并通信

#### 实现
第一步，在.pro文件中添加`network`库
```
QT += network
```
**服务端**  
服务端使用了`QTcpSever`和`QTcpSocket`类，`QTcpSever`用于创建TCP服务，`QTcpSocket`用于控制建立的socket连接。其实现的步骤如下：  
1、创建TCPserver对象
```
m_server = new QTcpServer();
```
2、使用`listen()`监听IP端口
```
m_server->listen(QHostAddress::Any, tcp_port);
```
`QHostAddress::Any`表示任意IP地址，QT中QHostAddress类用于提供一个IP地址，常用的还有：`QHostAddress::LocalHost`(本地主机地址)、`QHostAddress::AnyIPv4`(任意的IPv4地址)等等  
`tcp_port`即端口号

3、当有新的连接时，会发出`newConnection`信号，触发槽函数接受连接并获取连接的套接字QTcpSocket
```
connect(m_server, SIGNAL(newConnection()), SLOT(onNewConnection()));

m_socket = m_server->nextPendingConnection();  //获取连接套接字
```
4、绑定`disconnected`和`readyRead`信号
```
connect(m_socket, SIGNAL(disconnected()), SLOT(onDisconnected()));
connect(m_socket, SIGNAL(readyRead()), SLOT(onReadyRead()));
```
`disconnected`：表示有连接断开
`readyRead`：表示Socket缓存接收到新的数据

5、读取客户端发来的数据
```
m_socket->readAll();
```
服务端整体代码如下：
```
//TcpServer.h
#ifndef TCPSERVER_H
#define TCPSERVER_H

#include <QObject>
#include <QTcpServer>
#include <QTcpSocket>

class TcpServer : public QObject {
    Q_OBJECT

public:
    explicit TcpServer(uint16_t tcp_port);
    ~TcpServer() {};

public:

private slots:
    void onDisconnected();
    void onNewConnection();
    void onReadyRead();

private:
    QTcpServer* m_server;
    QTcpSocket* m_socket;

};

#endif // TCPSERVER_H

//TcpServer.cpp
#include "TcpServer.h"
#include <QTcpServer>
#include <QTcpSocket>

TcpServer::TcpServer(uint16_t tcp_port):
    m_server(new QTcpServer(this))  //创建TCP服务
{
    m_server->listen(QHostAddress::Any, tcp_port);  //监听IP端口
    connect(m_server, SIGNAL(newConnection()), this, SLOT(onNewConnection()));
}

void TcpServer::onNewConnection()
{
    m_socket = m_server->nextPendingConnection();  //获取连接套接字

    connect(m_socket, SIGNAL(disconnected()), SLOT(onDisconnected()));
    connect(m_socket, SIGNAL(readyRead()), SLOT(onReadyRead()));
    qDebug() << "new connection accepted!!";

}

void TcpServer::onReadyRead()
{
    qDebug() << m_socket->readAll();  //读取socket中的数据并打印
}

void TcpServer::onDisconnected()
{
    qDebug() << "client disconnected!!";
}
```

**客户端**  
客户端的实现就相对简单很多，只使用了`QTcpSocket`类，其实现的步骤如下：
1、创建QTcpSocket对象
```
m_clientSocket = new QTcpSocket();
```
2、使用`connectToHost()`连接服务器IP和端口号
```
m_clientSocket->connectToHost(QHostAddress::LocalHost, tcp_port);
```
3、绑定`connected`和`disconnected`信号
```
connect(m_clientSocket, SIGNAL(connected()), SLOT(onConnected()));
connect(m_clientSocket, SIGNAL(disconnected()), SLOT(onDisconnected()));
```
`connected`：表示连接成功

4、发送数据给服务端
```
m_clientSocket->write("hello");
```
客户端整体代码如下：
```
//TcpClient.h
#ifndef TCPCLIENT_H
#define TCPCLIENT_H

#include <QTcpSocket>

class Tcpclient : public QObject
{
    Q_OBJECT
public:
    Tcpclient(uint16_t tcp_port);
    virtual ~Tcpclient() {}

private slots:
    void onConnected();
    void onDisconnected();

private:
    QTcpSocket* m_clientSocket;
};

#endif // TCPCLIENT_H

//TcpClient.cpp
#include "Tcpclient.h"

Tcpclient::Tcpclient(uint16_t tcp_port)
{
    m_clientSocket = new QTcpSocket();
    m_clientSocket->connectToHost(QHostAddress::LocalHost, tcp_port);

    connect(m_clientSocket, SIGNAL(connected()), SLOT(onConnected()));
    connect(m_clientSocket, SIGNAL(disconnected()), SLOT(onDisconnected()));
}

void Tcpclient::onConnected()
{
    qDebug() << "connect ok!";
    m_clientSocket->write("hello");
}

void Tcpclient::onDisconnected()
{
    m_clientSocket->disconnectFromHost();  //断开与服务端的连接
}
```
此外，服务端也可以发送数据给客户端，客户端绑定`readyRead`信号来接收

#### 进一步讨论
我们都知道，在使用原生socket来实现TCP时，为了满足多个客户端的连接，通常会使用select()、epoll()来实现IO复用。在这一点上QT封装的TCP已经帮我们实现了。但是我们如何在槽函数中获取当前发出此信号的socket呢？  
我们可以使用`QObject::sender()`
```
QObject::Sender()返回发送信号的对象的指针，返回类型为QObject *
```
例如在上文服务端的`onDisconnected()`槽函数中，可以使用`sender()`来获取当前发出断开信号`disconnected`的socket对象，并销毁它
```
void TcpServer::onDisconnected()
{
    sender()->deleteLater();  //获取断开连接的socket对象并销毁
    qDebug() << "client disconnected!!";
}
```
因此在槽函数中可以根据`sender()`的不同来进行不同的处理