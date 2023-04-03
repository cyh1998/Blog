## 概 述
muduo在实现非阻塞TCP连接时，对socket相关的内容进行了非常详尽的封装，本文梳理下muduo中 `Acceptor InetAddress Socket SocketsOps` 四个类的相关实现。    
由于muduo源码的封装比较复杂，本人在其基础上进行了简化，保留其中核心的代码供于学习，因此示例代码非muduo源码。

## 实 现
**1. InetAddress**  
`InetAddress` 是对网络地址的相关封装，包括初始化网络地址结构，设置/获取网络地址数据等等。源码中区分对IVP6和IVP4的实现，以及对IP地址封装了 `StringArg` 类。  
这里只保留对IVP4的实现，IP地址使用字符串代替，简化代码如下：
```
//InetAddress.h
#include <netinet/in.h> //uint16_t sockaddr_in
#include <string.h> //memset()
#include <string>

class InetAddress {
public:
    explicit InetAddress(uint16_t port = 0, bool ifLoopback = false); //通过端口号构造
    InetAddress(const std::string& ip, uint16_t port); //通过IP地址 和 端口号构造

    const struct sockaddr* getSockAddr() const; //获取网络地址数据
    void setSockAddr(struct sockaddr_in& addr); //设置网络地址

private:
    sockaddr_in m_addr;
};

//InetAddress.cpp
#include <endian.h>
#include <arpa/inet.h> //inet_pton()
#include "InetAddress.h"

InetAddress::InetAddress(uint16_t port, bool ifLoopback) {
    memset(&m_addr, 0, sizeof(m_addr));
    m_addr.sin_family = AF_INET;
    in_addr_t ipType = ifLoopback ? INADDR_LOOPBACK : INADDR_ANY;
    m_addr.sin_addr.s_addr = htobe32(ipType);
    m_addr.sin_port = htobe16(port);
}

InetAddress::InetAddress(const std::string& ip, uint16_t port) {
    memset(&m_addr, 0, sizeof(m_addr));
    m_addr.sin_family = AF_INET;
    inet_pton(AF_INET, ip.c_str(), &m_addr.sin_addr);
    m_addr.sin_port = htobe16(port);
}

const struct sockaddr *InetAddress::getSockAddr() const {
    return (const struct sockaddr*)&m_addr;
    // return reinterpret_cast<const struct sockaddr*>(&m_addr);
}

void InetAddress::setSockAddr(struct sockaddr_in& addr) {
    m_addr = addr;
}
```
**说明：**  
源码对 `htobe32()`、`htobe16()` 等字节序转换函数也进行了封装，在 `\net\Endian.h` 中，这里直接使用系统函数。  
源码将 `getSockAddr()` 等函数的具体实现封装到了 `SocketsOps` 类中，这里因为和网络地址直接相关，在本类中实现。

**2. Socket**  
`Socket` 是一个RAII handle，封装了socket文件描述符的生命周期。简化代码如下：
```
//Socket.h
#include "../Base/Noncopyable.h"

class InetAddress;

class Socket : Noncopyable {
public:
    explicit Socket(int fd);

    int  fd() const { return m_sockfd; }

    void bindAddress(const InetAddress& addr) const; //bind接口
    void listenAddress() const; //listen接口
    int  acceptAddress(InetAddress* peeraddr); //accept接口

    void setAddrReusable(bool on) const; //设置是否地址复用
    void setPortReusable(bool on) const; //设置是否端口复用

private:
    const int m_sockfd;
};

//Socket.cpp
Socket::Socket(int fd) :
    m_sockfd(fd)
{
}

void Socket::bindAddress(const InetAddress& addr) const {
    sockets::bind(m_sockfd, addr.getSockAddr());
}

void Socket::listenAddress() const {
    sockets::listen(m_sockfd);
}

int Socket::acceptAddress(InetAddress* peeraddr) {
    struct sockaddr_in addr;
    memset(&addr, 0, sizeof(addr));
    int connfd = sockets::accept(m_sockfd, &addr);
    if (connfd >= 0) {
        peeraddr->setSockAddr(addr);
    }
    return connfd;
}

void Socket::setAddrReusable(bool on) const {
    int optVal = on ? 1 : 0;
    setsockopt(m_sockfd, SOL_SOCKET, SO_REUSEADDR, &optVal, static_cast<socklen_t>(sizeof optVal));
}

void Socket::setPortReusable(bool on) const {
    int optval = on ? 1 : 0;
    setsockopt(m_sockfd, SOL_SOCKET, SO_REUSEPORT, &optval, static_cast<socklen_t>(sizeof optval));
}

```
`Socket` 类只提供接口，不做任何操作，相关操作封装在 `SocketsOps` 中。

**3. SocketsOps**  
`SocketsOps` 是对所有socket相关api的封装，简化代码如下：
```
//SocketsOps.h
#include <arpa/inet.h>
#include <unistd.h>

namespace sockets {
    void bind(int sockfd, const struct sockaddr* addr);
    void listen(int sockfd);
    int  accept(int sockfd, struct sockaddr_in* addr);
    void close(int sockfd);
    int  createNonblockingSocket(); //创建非阻塞socket
}

//SocketsOps.cpp
#include "../Log/Log.h"
#include "SocketOps.h"

void sockets::bind(int sockfd, const struct sockaddr* addr) {
    int ret = ::bind(sockfd, addr, sizeof(struct sockaddr_in));
    if (ret < 0) {
        LOG_ERROR("Bind error!")
    }
}

void sockets::listen(int sockfd) {
    int ret = ::listen(sockfd, SOMAXCONN);
    if (ret < 0) {
        LOG_ERROR("Listen error!")
    }
}

int sockets::accept(int sockfd, struct sockaddr_in* addr) {
    socklen_t addrlen = sizeof(*addr);
    int connfd = ::accept4(sockfd, (struct sockaddr*)addr, &addrlen, SOCK_NONBLOCK | SOCK_CLOEXEC);
    if (connfd < 0) {
        //...错误处理
    }
    return connfd;
}

void sockets::close(int sockfd) {
    int ret = ::close(sockfd);
    if (ret < 0) {
        LOG_ERROR("Close error!")
    }
}

int sockets::createNonblockingSocket() {
    int sockfd = socket(AF_INET, SOCK_STREAM | SOCK_NONBLOCK | SOCK_CLOEXEC, IPPROTO_TCP);
    if (sockfd < 0) {
        LOG_ERROR("Create Nonblocking Socket error!")
    }
    return sockfd;
}
```
**说明：**  
对于将文件描述符设置为非阻塞，我们可以使用 `fcntl()` 来设置，如下：
```
// 将文件描述符设置为非阻塞的
int setnonblocking(int fd) {
    int old_option = fcntl(fd, F_GETFL);
    int new_option = old_option | O_NONBLOCK;
    fcntl(fd, F_SETFL, new_option);
    return old_option;
}
```
对于现在的Linux(Linux 2.6.28版本以后，参考文档 [Linux手册](https://man7.org/linux/man-pages/man2/accept4.2.html))，可以一步设置为非阻塞，如上文示例代码，muduo源码中分别提供了两种实现方式，这里简化只保留了更便捷的实现方式。

**4. Acceptor**  
`Acceptor` 类是对连接接收相关接口的封装，用于accept新的TCP连接。其构造函数与成员函数 `Acceptor::listen()` 执行创建TCP服务端的传统步骤，即调用 `socket()`、`bind()`、`listen()`，数据成员 `Channel` 观察此socket上的可读事件，并回调 `Acceptor::handleRead()`，后者调用 `accept()` 来接收连接，并回调用户callback。简化代码如下：
```
//Acceptor.h
#include "../Base/Noncopyable.h"
#include "../Net/EventLoop.h"
#include "../Net/Channel.h"
#include "InetAddress.h"
#include "Socket.h"

class Acceptor : Noncopyable {
public:
    typedef std::function<void (int sockfd, const InetAddress&)> NewConnectionCallback;

    Acceptor(EventLoop* loop, const InetAddress& addr);
    ~Acceptor();

    void setNewConnectionCallback(const NewConnectionCallback& cb) { m_newConnectionCallback = cb; }
    void listen();

private:
    void handleRead(); //可读事件回调

private:
    EventLoop* m_loop;
    Socket m_acceptSocket;
    Channel m_acceptChannel;
    NewConnectionCallback m_newConnectionCallback; //用户回调
};

//Acceptor.cpp
#include "SocketOps.h"
#include "Acceptor.h"

Acceptor::Acceptor(EventLoop *loop, const InetAddress &addr) :
    m_loop(loop),
    m_acceptSocket(sockets::createNonblockingSocket()),
    m_acceptChannel(m_loop, m_acceptSocket.fd())
{
    m_acceptSocket.setAddrReusable(true);
    m_acceptSocket.bindAddress(addr);
    m_acceptChannel.setReadCallback(std::bind(&Acceptor::handleRead, this));
}

Acceptor::~Acceptor() {
    m_acceptChannel.disableAll();
    // m_acceptChannel.remove();
}

void Acceptor::listen() {
    m_loop->assertInLoopThread();
    m_acceptSocket.listenAddress();
    m_acceptChannel.enableReading();
}

void Acceptor::handleRead() {
    m_loop->assertInLoopThread();
    InetAddress peerAddr;
    int connfd = m_acceptSocket.acceptAddress(&peerAddr);
    if (connfd >= 0) {
        if (m_newConnectionCallback) m_newConnectionCallback(connfd, peerAddr);
    } else {
        sockets::close(connfd);
    }
}
```

更多内容，详见github [NetLib](https://github.com/cyh1998/NetLib)  
参考：  
《Linux多线程服务端编程》陈硕 著  
[muduo源码](https://github.com/chenshuo/muduo)  