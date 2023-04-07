## 概 述
本文介绍基于Epoll实现的多人聊天室服务端程序，有关Epoll的相关内容，可以参考博客 [Linux epoll ET模式实现](./Linux%20epoll%20ET模式实现.md)。

## 服务端
服务端利用 `List` 记录所有连接的客户端socket，在收到客户端消息时，广播给所有当前的客户端。  
服务端需要**注意如何处理客户端断开socket连接的逻辑**，当客户端断开连接时，理论上服务端会触发 `EPOLLIN`，`EPOLLRDHUP` 事件，如果我们在服务端只关心 `EPOLLRDHUP` 事件，触发该事件后关闭套接字，这个逻辑是不可行的，有些系统未必会触发 `EPOLLRDHUP`。所以服务端代码采用关心 `EPOLLIN` 事件，然后在 `read()` 时进行处理的方式，分为以下两种情况：  
- read返回0，对方正常调用close关闭连接  
- read返回-1，需要通过errno来判断，如果不是 `EAGAIN` 和 `EINTR`，那么就是对方异常断开连接  

(这里参考了 [知乎 Nov 23](https://www.zhihu.com/question/289965746/answer/490278794) 的回答)

**Sever.h**
```
#ifndef EPOLL_ET_SERVER_H
#define EPOLL_ET_SERVER_H

#include <list> //list
#include <string>

#define MAX_EVENT_NUMBER 5000   //Epoll最大事件数
#define BUFFER_SIZE      0xFFFF //缓存区数据大小

class Server {
public:
    explicit Server();
    bool InitServer(const std::string &Ip, const int &Port);
    void Run();

private:
    int m_socketFd;    //创建的socket文件描述符
    int m_epollFd;     //创建的epoll文件描述符
    std::list<int> m_clientsList;  //已连接的客户端socket列表

private:
    int setnonblocking(int fd); // 将文件描述符设置为非堵塞的
    void addfd(int epoll_fd, int sock_fd, bool epoll_et); // 将文件描述符上的事件注册
};
```
**Sever.cpp**  
```
#include <iostream>
#include <fcntl.h> //fcntl()
#include <sys/epoll.h> //epoll
#include <netinet/in.h> //sockaddr_in
#include <arpa/inet.h> //inet_pton()
#include <string.h> //memset()
#include <unistd.h> //close()
#include "Server.h"

using namespace std;

Server::Server() {
}

bool Server::InitServer(const std::string &Ip, const int &Port) {
    int ret;
    struct sockaddr_in address;
    memset(&address, 0, sizeof(address)); //初始化 address
    address.sin_family = AF_INET;
    inet_pton(AF_INET, Ip.c_str(), &address.sin_addr);
    address.sin_port = htons(Port);

    m_socketFd = socket(AF_INET, SOCK_STREAM, 0); //创建socket
    if (m_socketFd < 0) {
        cout << "Server: socket error! id:" << m_socketFd << endl;
        return false;
    }

    ret = bind(m_socketFd, (struct sockaddr*)&address, sizeof(address)); //bind
    if (ret == -1) {
        cout << "Server: bind error!" << endl;
        return false;
    }

    ret = listen(m_socketFd, 20); //listen
    if (ret == -1) {
        cout << "Server: listen error!" << endl;
        return false;
    }

    m_epollFd = epoll_create(5);
    if (m_epollFd == -1) {
        cout << "Server: create epoll error!" << endl;
        return false;
    }
    addfd(m_epollFd, m_socketFd, true); //注册sock_fd上的事件

    return true;
}

int Server::setnonblocking(int fd) {
    int old_option = fcntl(fd, F_GETFL);
    int new_option = old_option | O_NONBLOCK;
    fcntl(fd, F_SETFL, new_option);
    return old_option;
}

void Server::addfd(int epoll_fd, int sock_fd, bool epoll_et) {
    epoll_event event;
    event.data.fd = sock_fd;
    event.events = EPOLLIN;
    if (epoll_et) event.events |= EPOLLET;
    epoll_ctl(epoll_fd, EPOLL_CTL_ADD, sock_fd, &event);
    setnonblocking(sock_fd);
}

void Server::Run() {
    epoll_event events[MAX_EVENT_NUMBER];

    while (1) {
        int ret = epoll_wait(m_epollFd, events, MAX_EVENT_NUMBER, -1); //epoll_wait
        if (ret < 0) {
            cout << "Server: epoll error" << endl;
            break;
        }

        char buf[BUFFER_SIZE];
        for (int i = 0; i < ret; ++i) {
            int sockfd = events[i].data.fd;
            if (sockfd == m_socketFd) { //新的socket连接
                struct sockaddr_in client_addr;
                socklen_t len = sizeof(client_addr);
                int conn_fd = accept(m_socketFd, (struct sockaddr*)&client_addr, &len);

                addfd(m_epollFd, conn_fd, true); //对 conn_fd 开启ET模式
                m_clientsList.emplace_back(conn_fd);

                cout << "Server: New connect fd:" << conn_fd << " Now client number:" << m_clientsList.size() << endl;
            } else if (events[i].events & EPOLLIN) { //可读事件
                char client_buf[BUFFER_SIZE];
                memset(&client_buf, '\0', BUFFER_SIZE);
                int recvRet = recv(sockfd, client_buf, BUFFER_SIZE - 1, 0);
                if (recvRet == 0) { //对端正常关闭socket
                    close(sockfd);
                    m_clientsList.remove(sockfd);
                    cout << "Server: Client close socket!" << endl;
                    cout << "Server: Now client number:" << m_clientsList.size() << endl;
                } else if (recvRet < 0){
                    if ((errno != EAGAIN) && (errno != EINTR)) { //对端异常断开socket
                        close(sockfd);
                        m_clientsList.remove(sockfd);
                        cout << "Server: Client abnormal close socket!" << endl;
                        cout << "Server: Now client number:" << m_clientsList.size() << endl;
                    } else { //recv error
                        cout << "Server: Recv error!" << endl;
                    }
                } else {
                    cout << "Server: Recv data: " << client_buf << endl;
                    for (auto &i : m_clientsList) {
                        if (i != sockfd) {
                            if (send(i, client_buf, BUFFER_SIZE, 0) < 0) {
                                close(sockfd);
                                m_clientsList.remove(sockfd);
                                cout << "Server: send error! Close client: " << i << endl;
                            }
                        }
                    }
                }
            } else {
                cout << "Server: socket something else happened!" << endl;
            }
        }
    }

    close(m_socketFd);
    close(m_epollFd);
}
```
**main.cpp**  
```
#include "Epoll/Server.h"

using namespace std;

int main() {
    Server server;
    if (server.InitServer("127.0.0.1", 8888)) {
        server.Run();
    }
    return 0;
}
```
## 客户端
因为客户端对高并发的要求不高，并且select模式跨平台性更好，所以客户端代码用select来实现。有关select的原理，这里就不赘述了，大家可以百度学习。  
代码是本人之前学习select时写的，这里拿来使用下，代码如下：
```
#include <iostream>
#include <string.h>
#include <unistd.h>
#include <stdio.h>
#include <stdlib.h>
#include <sys/socket.h>
#include <sys/types.h>
#include <netinet/in.h>

#define MAXSIZE 0xFFFF

using namespace std;

int main() {
    int socket_fd;
    struct sockaddr_in server_addr;
    int len;
    char send_buf[MAXSIZE];
    char get_buff[MAXSIZE];
    int recv_num;
    int fun_res;
    string str;

    fd_set rfds;
    struct timeval tv;
    int max_fd;

    memset(&server_addr,0,sizeof(server_addr));
    server_addr.sin_family = AF_INET;
    server_addr.sin_port = htons(8888);
    server_addr.sin_addr.s_addr = htonl(INADDR_ANY);
    len = sizeof(server_addr);

    socket_fd = socket(AF_INET, SOCK_STREAM, 0);
    if (socket_fd < 0) {
        cout << "socket error!" << endl;
        exit(1);
    }

    fun_res = connect(socket_fd, (struct sockaddr *) &server_addr, len);
    if (fun_res < 0) {
        cout << "connect error!" << endl;
        exit(1);
    }

    while (1) {
        FD_ZERO(&rfds);
        FD_SET(0,&rfds);

        max_fd = 0;
        FD_SET(socket_fd,&rfds);
        if (max_fd < socket_fd) max_fd = socket_fd;

        tv.tv_sec = 5;
        tv.tv_usec = 0;

        fun_res = select(max_fd+1, &rfds, NULL, NULL, &tv);
        if (fun_res < 0) {
            cout << "select error!" << endl;
            exit(1);
        } else if(fun_res == 0) {
            //cout << "no msg!waiting..." << endl;
            continue;
        } else {
            if (FD_ISSET(socket_fd,&rfds)) {
                recv_num = recv(socket_fd, get_buff, sizeof(get_buff), 0);
                if (recv_num < 0) {
                    cout << "recv error!" << endl;
                    exit(1);
                } else {
                    get_buff[recv_num] = 0;
                    cout << "server msg: " << get_buff << endl;
                }
            }

            if (FD_ISSET(0,&rfds)) {
                cin >> send_buf;
                str = send_buf;

                fun_res = send(socket_fd,send_buf,strlen(send_buf),0);
                if (fun_res < 0) {
                    cout << "send error!" << endl;
                    exit(1);
                }
                if (str == "exit") exit(1);
            }
        }
    }

    close(socket_fd);
    return 0;
}
```
## 运行效果

![聊天室](https://upload-images.jianshu.io/upload_images/22192996-3311b8802d8fc651.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

运行服务端后，启动多个客户端连接服务端。客户端发送数据后，服务端打印接受的数据并将数据广播给当前在线的客户端。

**说明：**  
本服务端使用Epoll ET模式，简单的实现了多人聊天室。GitHub地址：[ChatRoomServer](https://github.com/cyh1998/ChatRoomServer)  
代码中没有使用多线程以及使用 `EPOLLONESHOT` 来避免多个线程同时操作一个socket的问题、对通信数据的处理也相对简单。日后工作之余会对代码进行完善更新。
