## 概 述
在之前的文章中提到，在服务器中使用**socket选项**和**I/O复用系统调用的超时参数**来处理定时事件。接下来介绍利用**SIGALRM信号**来实现定时机制。本文介绍一种简单定时器的实现——基于升序链表的定时器，来处理SIGALRM信号。后续文章会介绍高效定时器的实现，即时间轮和时间堆。

## 实 现
**1. 实现基于升序链表的定时器**  
示例代码通过对STL中 `List<>` 的封装，实现了基于升序链表的定时器；定时器类中的回调函数，也是使用C++11中的 `std::function()` 来实现，代码简单易懂。原书中(《Linux高性能服务器编程》 游双 著)则是手写了链表的相关操作来实现升序链表，感兴趣的可以自行学习下，这里不再展示。
```
//Time_list.h
#include <netinet/in.h> //sockaddr_in
#include <list>
#include <functional>

#define BUFFER_SIZE 64

class util_timer; //前向声明

//客户端数据
struct client_data {
    sockaddr_in address; //socket地址
    int sockfd; //socket文件描述符
    char buf[BUFFER_SIZE]; //数据缓存区
    util_timer* timer; //定时器
};

//定时器类
class util_timer {
public:
    time_t expire; //任务超时时间(绝对时间)
    std::function<void(client_data*)> callBackFunc; //回调函数
    client_data* userData; //用户数据
};

class Timer_list {

public:
    explicit Timer_list();
    ~Timer_list();

public:
    void add_timer(util_timer* timer); //添加定时器
    void del_timer(util_timer* timer); //删除定时器
    void adjust_timer(util_timer* timer); //调整定时器
    void tick(); //处理链表上到期的任务

private:
    std::list<util_timer*> m_timer_list; //定时器链表
};

//Timer_list.cpp
#include "Timer_list.h"
#include <time.h>

Timer_list::Timer_list() {

}

Timer_list::~Timer_list() {
    m_timer_list.clear();
}

void Timer_list::add_timer(util_timer* timer) { //将定时器添加到链表
    if (!timer) return;
    else {
        auto item = m_timer_list.begin();
        while (item != m_timer_list.end()) {
            if (timer->expire < (*item)->expire) {
                m_timer_list.insert(item, timer);
                return;
            }
            item++;
        }
        m_timer_list.emplace_back(timer);
    }
}

void Timer_list::del_timer(util_timer* timer) { //将定时器从链表删除
    if (!timer) return;
    else {
        auto item = m_timer_list.begin();
        while (item != m_timer_list.end()) {
            if (timer == *item) {
                m_timer_list.erase(item);
                return;
            }
            item++;
        }
    }
}

void Timer_list::adjust_timer(util_timer *timer) { //调整定时器在链表中的位置
    del_timer(timer);
    add_timer(timer);
}

void Timer_list::tick() { //SIGALRM信号触发，处理链表上到期的任务
    if (m_timer_list.empty()) return;
    time_t cur = time(nullptr);
    
    //检测当前定时器链表中到期的任务。
    while (!m_timer_list.empty()) {
        util_timer* temp = m_timer_list.front();
        if (cur < temp->expire) break;
        temp->callBackFunc(temp->userData);
        m_timer_list.pop_front();
    }
}
```

**2. 运 用**  
我们可以使用上述的升序定时器链表来处理一段时间内非活动的客户端连接。服务器利用 `alarm()` 周期性的触发 `SIGALRM` 信号，主循环收到该信号后，执行定时器链表上的定时任务，即关闭非活动的客户端连接。  
关键代码如下：
```
//main.cpp
#include <iostream>
#include <netinet/in.h> //sockaddr_in
#include <string.h> //memset()
#include <arpa/inet.h> //inet_pton()
#include <assert.h> //assert()
#include <sys/epoll.h> //epoll
#include <fcntl.h> //fcntl()
#include <unistd.h> //close() alarm()
#include <signal.h>
#include "Timer_list.h" //利用升序定时器链表

#define FD_LIMIT 65535 //最大客户端数量
#define MAX_EVENT_NUMBER 1024
#define TIMESLOT 10 //定时时间

using namespace std;

static int pipefd[2]; //信号通讯管道
static Timer_list timer_list; //创建升序定时器链表类
static int epoll_fd = 0;

int setnonblocking(int fd) {
    //...
}

void addfd(int epoll_fd, int sock_fd) {
    //...
}

void sig_handler(int sig) {
    //...
}

void addsig(int sig) {
    //...
}

//SIGALRM 信号的处理函数
void timer_handler() {
    timer_list.tick(); //调用升序定时器链表类的tick() 处理链表上到期的任务
    alarm(TIMESLOT); //再次发出 SIGALRM 信号
}

//定时器回调函数 删除socket上注册的事件并关闭此socket
void timer_callback(client_data* user_data) {
    epoll_ctl(epoll_fd, EPOLL_CTL_DEL, user_data->sockfd, 0);
    if (user_data) {
        close(user_data->sockfd);
        cout << "close fd : " << user_data->sockfd << endl;
    }
}

int main() {
    //... socket

    //注册信号处理
    addsig(SIGTERM); //终止进程，即kill
    addsig(SIGALRM); //计时器到期
    bool stop_server = false;

    client_data* users = new client_data[FD_LIMIT]; //客户端数据数组
    bool timeout = false;
    alarm(TIMESLOT); //开始定时

    while (!stop_server) {
        //... epoll_wait        

        for (int i = 0; i < ret; ++i) {
            int sockfd = events[i].data.fd;
            if (sockfd == sock_fd) { //处理新的客户端连接
                struct sockaddr_in client_addr;
                socklen_t len = sizeof(client_addr);
                int conn_fd = accept(sock_fd, (struct sockaddr*)&client_addr, &len);
                addfd(epoll_fd, conn_fd);

                users[conn_fd].address = client_addr;
                users[conn_fd].sockfd = conn_fd;
                
                //创建定时器
                util_timer* timer = new util_timer;
                timer->userData = &users[conn_fd];      //设置用户数据
                timer->callBackFunc = timer_callback;   //设置回调函数
                time_t cur = time(nullptr);
                timer->expire = cur + 3 * TIMESLOT;     //设置客户端活动时间
                users[conn_fd].timer = timer;           //绑定定时器
                timer_list.add_timer(timer);    //将定时器添加到升序链表中

            } else if ((sockfd == pipefd[0]) && (events[i].events & EPOLLIN)) { //处理信号
                ret = recv(pipefd[0], buf, sizeof(buf), 0);
                if (ret <= 0) continue;
                else {
                    for (auto &item : buf){
                        switch (item) {
                            case SIGTERM: {
                                cout << " SIGTERM: Kill" << endl;
                                stop_server = true;
                            }
                            case SIGALRM: {
                                //用timeout来标记有定时任务
                                //先不处理，因为定时任务优先级不高，优先处理其他事件
                                timeout = true;
                                break;
                            }
                        }
                    }
                }
            } else if (events[i].events & EPOLLIN) { //处理客户端数据
                while (1) {
                    //... recv 客户端数据

                    util_timer* timer = users[sockfd].timer; //获取对应定时器
                    if (ret < 0) {
                        if (errno != EAGAIN) { //读错误
                            timer_callback(&users[sockfd]);
                            if (timer) timer_list.del_timer(timer);
                        }
                        break;
                    } else if (ret == 0) { //对端关闭socket连接
                        timer_callback(&users[sockfd]);
                        if (timer) timer_list.del_timer(timer);
                    } else {
                        if (timer) { //客户端有数据可读，调整对应定时器的时间
                            time_t cur = time(nullptr);
                            timer->expire = cur + 3 * TIMESLOT;
                            timer_list.adjust_timer(timer);
                        }
                    }
                }
            } else {
                //...
            }
        }
        
        //最后处理定时任务
        if (timeout) {
            timer_handler();
            timeout = false;
        }
    }
    //... close
    return 0;
}
```
**注：** 示例代码中，因为I/O事件优先级更高，所以服务器优先处理I/O事件，最后处理定时事件，这样会导致定时任务不能精确地按照预期时间执行。

## 运行结果
启动服务器程序后，服务器每隔一端时间就会触发 `tick()` 函数，执行定时器链表上的定时任务。通过两个客户端连接到服务器，服务器在收到客户端数据时，会调整对应的定时器并断开一段时间内未活动的客户端。

![运行结果](https://upload-images.jianshu.io/upload_images/22192996-2d3a93fd41b2f453.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

更多内容，详见GitHub：[ChatRoomServer](https://github.com/cyh1998/ChatRoomServer)