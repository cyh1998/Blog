## 概 述
在之前文章中实现的定时器，都是以固定的频率调用心搏函数 `tick()`，从而处理到期的定时器任务。**时间堆**则采用另一种思路：将所有定时器中到期时间最小的值作为心搏间隔。当调用心搏函数时，此定时器任务必定到期，处理完此定时器任务后，再从所有定时器中到期时间最小的那个，将此到期时间设为下一次的心搏间隔，如此反复，实现定时。

## 实 现
从时间堆定时器的设计思路中不难看出，使用小根堆结构最适合处理这种方案。我们通过对STL中 `priority_queue<>` 的封装来实现时间堆(有关优先级队列的知识这里不做展示了)。书中则是手写了一个基于数组的小根堆来实现时间堆，感兴趣的可以自行学习下，这里也不再展示。示例代码如下：
```
//TimerHeap.h
#include <netinet/in.h>
#include <functional>
#include <vector>
#include <queue>

#define BUFFER_SIZE 0xFFFF  //缓存区数据大小
#define TIMESLOT    30      //定时时间

class heap_timer; //前向声明

// 客户端数据
struct client_data {
    sockaddr_in address;
    int sockfd;
    char buf[BUFFER_SIZE];
    heap_timer *timer;
};

// 定时器类
class heap_timer {
public:
    heap_timer(int delay) {
        expire = time(nullptr) + delay;
    }
public:
    time_t expire; //定时器生效的绝对时间
    std::function<void(client_data *)> callBackFunc; //回调函数
    client_data *user_data; //用户数据
};

struct cmp { //比较函数，实现小根堆
    bool operator () (const heap_timer* a, const heap_timer* b) {
        return a->expire > b->expire;
    }
};

class TimerHeap {
public:
    explicit TimerHeap();
    ~TimerHeap();

public:
    void AddTimer(heap_timer *timer); //添加定时器
    void DelTimer(heap_timer *timer); //删除定时器
    void Tick(); //心搏函数
    heap_timer* Top();

private:
    std::priority_queue<heap_timer *, std::vector<heap_timer *>, cmp> m_timer_pqueue; //时间堆
};

//TimerHeap.cpp
#include "TimerHeap.h"

TimerHeap::TimerHeap() = default;

TimerHeap::~TimerHeap() {
    while (!m_timer_pqueue.empty()) {
        delete m_timer_pqueue.top();
        m_timer_pqueue.pop();
    }
}

void TimerHeap::AddTimer(heap_timer *timer) {
    if (!timer) return;
    m_timer_pqueue.emplace(timer);
}

void TimerHeap::DelTimer(heap_timer *timer) {
    if (!timer) return;
    // 将目标定时器的回调函数设为空，即延时销毁
    // 减少删除定时器的开销，但会扩大优先级队列的大小
    timer->callBackFunc = nullptr;
}

void TimerHeap::Tick() {
    time_t cur = time(nullptr);
    while (!m_timer_pqueue.empty()) {
        heap_timer *timer = m_timer_pqueue.top();
        //堆顶定时器没有到期，退出循环
        if (timer->expire > cur) break;
        //否则执行堆顶定时器中的任务
        if (timer->callBackFunc) timer->callBackFunc(timer->user_data);
        m_timer_pqueue.pop();
    }
}

heap_timer *TimerHeap::Top() {
    if (m_timer_pqueue.empty()) return nullptr;
    else return m_timer_pqueue.top();
}
```

## 运 用
因为时间堆以最小到期时间作为心搏间隔，所以我们在服务器类中定义一个变量来记录当前是否在进行定时任务，从而来判断是否触发 `SIGALRM` 信号，核心代码如下：
```
// SIGALRM信号处理函数
void Server::TimerHandler() {
    m_timerHeap.Tick(); //调用时间堆的心搏函数 Tick() 处理到期的任务
    heap_timer *tmp = nullptr;
    //判断当前是否在进行定时任务，没有则触发 SIGALRM 信号
    if (!s_isAlarm && (tmp = m_timerHeap.Top())) { 
        time_t delay = tmp->expire - time(nullptr);
        if (delay <= 0) delay = 1;
        alarm(delay); //触发 SIGALRM 信号
        s_isAlarm = true; //设置当前已有定时任务在进行
    }
}

// 定时器回调函数
void Server::TimerCallBack(client_data *user_data) {
    epoll_ctl(s_epollFd, EPOLL_CTL_DEL, user_data->sockfd, 0);
    if (user_data) {
        close(user_data->sockfd);
        s_clientsList.remove(user_data->sockfd);
        s_isAlarm = false; //设置当前没有进行定时任务
        cout << "Server: close socket fd : " << user_data->sockfd << endl;
    }
}

void Server::Run() {
    //...
    while (!stop_server) {
        //... epoll_wait        

        for (int i = 0; i < ret; ++i) {
            int sockfd = events[i].data.fd;
            if (sockfd == sock_fd) { //处理新的客户端连接
                //...accept
                
                //创建定时器
                heap_timer *timer = new heap_timer(TIMESLOT);
                timer->user_data = &m_user[conn_fd];
                timer->callBackFunc = TimerCallBack;
                m_timerHeap.AddTimer(timer);
                if (!s_isAlarm) { //当前没有进行定时任务
                    s_isAlarm = true;
                    alarm(TIMESLOT); //触发 SIGALRM 信号
                }
                
            } else if ((sockfd == pipefd[0]) && (events[i].events & EPOLLIN)) { //处理信号
                //...
            } else if (events[i].events & EPOLLIN) { //处理客户端数据
                //...
                heap_timer *timer = m_user[sockfd].timer;
                
                // 更新时间堆定时器
                if (timer) {
                    timer->expire += TIMESLOT;
                    // 更新定时器后，需要设置当前没有进行定时任务
                    // 从而在 TimerHandler() 中重新触发 SIGALRM 信号
                    s_isAlarm = false;
                }

            } else {
                //...
            }
        }
        //...处理定时任务
    }
    //... close
    return 0;
}
```
服务器程序的运行结果与之前实现的定时器类似，这里就不展示了。  
更多内容，详见GitHub：[ChatRoomServer](https://github.com/cyh1998/ChatRoomServer)