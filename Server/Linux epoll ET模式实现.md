## 概 述
Epoll是Linux中常用的IO复用技术，Epoll分为两种模式，LT( `Level Triggered`，电平触发)和 ET( `Edge Triggered`，边沿触发)。有关两种模式的原理，大家可以百度学习，这里不再赘述。本文介绍Epoll ET模式的简单实现。

## 主要函数
**1. epoll_create**  
epoll将文件描述符上的事件放在一个内核事件表中，`epoll_create()` 则用于创建这个内核事件表，并返回一个标识该内核的文件描述符，函数原型
```
int epoll_create(int size);
```
参数 `size` 已经不起作用，只是提示内核，该事件表需要多大。

**2. epoll_ctl**  
`epoll_ctl()` 用于操作epoll的内核事件表，成功返回0，失败返回-1，函数原型如下：
```
int epoll_ctl(int epfd, int op, int fd, struct epoll_event *event);
```
参数说明：  
`epfd` ：需要操作的文件描述符，即epoll_create()返回值  
`op` ：操作类型，如下  
- `EPOLL_CTL_ADD`  向事件表中注册fd上的事件  
- `EPOLL_CTL_MOD`  修改fd上注册的事件  
- `EPOLL_CTL_DEL`  删除fd上注册的事件  

`fd` ：需要的监听的socket文件描述符  
`event` ：需要监听的事件，结构如下：  
```
struct epoll_event 
{  
    __uint32_t events;  /* epoll 事件 */  
    epoll_data_t data;  /* 用户数据 */ 
}
```
其中 `events` 成员描述事件类型，epoll支持的事件类型如下：
```
EPOLLIN      //表示数据可读；
EPOLLOUT     //表示数据可写；
EPOLLPRI     //表示高优先级数据可读，比如TCP外带数据；
EPOLLERR     //表示错误；
EPOLLHUP     //表示挂起；
EPOLLET      //表示将epoll设为边缘触发(ET)模式；
EPOLLONESHOT //表示只触发一种事件，且只触发一次。
```
`data` 成员用于存储用户数据，类型 `epoll_data_t` 的定义如下：
```
typedef union epoll_data 
{  
    void        *ptr;  
    int          fd;  
    uint32_t     u32;  
    uint64_t     u64;  
} epoll_data_t;
```

**3. epoll_wait**  
`epoll_wait()` 用于在一段超时时间内等待一组文件描述符上的事件，成功返回就绪的文件描述符个数，失败返回-1，函数原型如下：
```
int epoll_wait(int epfd, struct epoll_event* events, int maxevents, int timeout);
```
参数说明：  
`epfd` ：内核事件表描述符  
`events` ：就绪的事件数组  
`maxevents` ：最大监听的事件数  
`timeout` ：超时时间  

## 实 现
在实现之前，先介绍几个常用的函数  
**1. 将文件描述符设置为非堵塞**  
```
// 将文件描述符设置为非堵塞的
int setnonblocking(int fd)
{
    int old_option = fcntl(fd, F_GETFL);
    int new_option = old_option | O_NONBLOCK;
    fcntl(fd, F_SETFL, new_option);
    return old_option;
}
```

**2. 将文件描述符上的事件注册**  
`epoll_et` 指定是否启用ET模式  
```
void addfd(int epoll_fd, int sock_fd, bool epoll_et)
{
    epoll_event event;
    event.data.fd = sock_fd;
    event.events = EPOLLIN;
    if (epoll_et) event.events |= EPOLLET;
    epoll_ctl(epoll_fd, EPOLL_CTL_ADD, sock_fd, &event);
    setnonblocking(sock_fd);
}
```

**整体代码**  
```
#include <iostream>
#include <netinet/in.h> //sockaddr_in
#include <string.h> //memset()
#include <arpa/inet.h> //inet_pton()
#include <assert.h> //assert()
#include <sys/epoll.h> //epoll
#include <fcntl.h> //fcntl()
#include <unistd.h> //close()

#define MAX_EVENT_NUMBER 1024
#define BUFFER_SIZE 10

using namespace std;

// 将文件描述符设置为非堵塞的
int setnonblocking(int fd)
{
    int old_option = fcntl(fd, F_GETFL);
    int new_option = old_option | O_NONBLOCK;
    fcntl(fd, F_SETFL, new_option);
    return old_option;
}

// 将文件描述符上的事件注册
void addfd(int epoll_fd, int sock_fd, bool epoll_et)
{
    epoll_event event;
    event.data.fd = sock_fd;
    event.events = EPOLLIN;
    if (epoll_et) event.events |= EPOLLET;
    epoll_ctl(epoll_fd, EPOLL_CTL_ADD, sock_fd, &event);
    setnonblocking(sock_fd);
}

int main() {
    string ip = "127.0.0.1"; //IP
    int port = 8888;         //端口号

    int ret;
    struct sockaddr_in address;
    memset(&address, 0, sizeof(address)); //初始化 address
    address.sin_family = AF_INET;
    inet_pton(AF_INET, ip.c_str(), &address.sin_addr);
    address.sin_port = htons(port);

    int sock_fd = socket(AF_INET, SOCK_STREAM, 0); //创建socket
    assert(sock_fd >= 0);

    ret = bind(sock_fd, (struct sockaddr*)&address, sizeof(address)); //bind
    assert(ret != -1);

    ret = listen(sock_fd, 5); //listen
    assert(ret != -1);

    epoll_event events[MAX_EVENT_NUMBER];
    int epoll_fd = epoll_create(5);
    assert(epoll_fd != -1);
    addfd(epoll_fd, sock_fd, true); //注册sock_fd上的事件

    while (1) {
        ret = epoll_wait(epoll_fd, events, MAX_EVENT_NUMBER, -1); //epoll_wait
        if (ret < 0) {
            cout << "epoll error" << endl;
            break;
        }

        char buf[BUFFER_SIZE];
        for (int i = 0; i < ret; ++i) {
            int sockfd = events[i].data.fd;
            if (sockfd == sock_fd) {
                struct sockaddr_in client_addr;
                socklen_t len = sizeof(client_addr);
                int conn_fd = accept(sock_fd, (struct sockaddr*)&client_addr, &len);
                addfd(epoll_fd, conn_fd, true); //对 conn_fd 开启ET模式
            } else if (events[i].events & EPOLLIN) { //可读事件，ET模式下，该事件不会被重复触发，因此我们需要读完全部数据
                cout << "ET: event trigger once" << endl;
                while (1) {
                    memset(&buf, '\0', BUFFER_SIZE);
                    int recvRet = recv(sockfd, buf, BUFFER_SIZE - 1, 0);
                    if (recvRet < 0) {
                        if ((errno == EAGAIN) || (errno == EWOULDBLOCK)) {
                            cout << "ET: read later" << endl;
                            break;
                        }
                        close(sockfd);
                        break;
                    } else if (recvRet == 0) {
                        close(sockfd);
                    } else {
                        cout << "ET: get "<< recvRet << " buf:" << buf << endl;
                    }
                }
            } else {
                cout << "ET: something else happened!" << endl;
            }
        }
    }

    close(sock_fd);
    return 0;
}
```
## 结 果
程序运行后，通过telnet这个服务器程序，并输入数据

![结果](https://upload-images.jianshu.io/upload_images/22192996-f55f01e96d88944b.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

服务器程序触发了一次可读事件，并读取了全部的数据。  

参考：《Linux高性能服务器编程》 游双 著