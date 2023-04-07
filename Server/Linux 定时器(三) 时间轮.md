## 概 述
上篇文章中实现了基于升序链表的定时器，此定时器存在着当定时器数量变多，效率也会变低的问题。对此，我们使用时间轮在解决。贴一下书中展示的简单时间轮的结构图

![时间轮](https://upload-images.jianshu.io/upload_images/22192996-16acbeeb7c383faf.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

基于升序链表的定时器是将所有的定时器放到一条链表中来管理，而时间轮则是使用哈希表的思想，将定时器散列到不同的链表中，即图中时间轮的槽 `(slot)`。时间轮的每次转动称为一个滴答 `(tick)`，滴答的间隔时间称为槽间隔 `(slot interval)`，即**心搏时间**。

## 实 现
鉴于哈希表的思想，我们使用C++11中的 `unordered_map` 结构来实现时间轮，key对应时间轮中的槽，value使用 `list<>` 来实现槽中的定时器链表。同样的，定时器类中的回调函数，使用 `std::function()` 来实现。简单定时器的实现如下：
```
//TimerWheel.h
#include <netinet/in.h>
#include <list>
#include <functional>
#include <unordered_map>

#define BUFFER_SIZE 0xFFFF  //缓存区数据大小
#define TIMESLOT    1       //定时时间

class tw_timer; //前向声明

//客户端数据
struct client_data {
    sockaddr_in address; //socket地址
    int sockfd; //socket文件描述符
    char buf[BUFFER_SIZE]; //数据缓存区
    tw_timer *timer; //定时器
};

//定时器类
class tw_timer {
public:
    tw_timer(int rot, int ts) : rotation(rot), time_slot(ts){}

public:
    int rotation; //记录定时器在时间轮转多少圈后生效
    int time_slot; //记录定时器属于时间轮上的哪个槽
    std::function<void(client_data *)> callBackFunc; //回调函数
    client_data *user_data; //用户数据
};

class TimerWheel {
public:
    explicit TimerWheel();
    ~TimerWheel();

public:
    tw_timer* AddTimer(int timeout); //根据超时时间 timeout 添加定时器
    void DelTimer(tw_timer *timer); //删除定时器
    void Tick(); //心搏函数

private:
    static const int N = 60; //时间轮上槽的数量
    static const int SI = TIMESLOT; //每 1s 时间轮转动一次，即槽间隔为 1s
    int m_cur_slot; //时间轮的当前槽
    std::unordered_map<int, std::list<tw_timer *>> m_slots; //时间轮的槽，其中每个元素指向一个定时器链表
};

//TimerWheel.cpp
#include "TimerWheel.h"

TimerWheel::TimerWheel() : m_cur_slot(0) {
    for (int i = 0; i < N; ++i) {
        m_slots[i] = std::list<tw_timer *>();
    }
}

TimerWheel::~TimerWheel() {
    for (int i = 0; i < N; ++i) {
        auto iter = m_slots[i].begin();
        while (iter != m_slots[i].end()) {
            delete *iter;
            iter++;
        }
    }
    m_slots.clear();
}

tw_timer *TimerWheel::AddTimer(int timeout) {
    if (timeout < 0) return nullptr;
    int ticks = timeout < SI ? 1 : timeout / SI;
    int rotation = ticks / N; //计算待插入的定时器在时间轮转动多少圈后触发
    int ts = (m_cur_slot + (ticks % N)) % N; //计算待插入的定时器应该插入到时间轮的哪个槽
    tw_timer* timer = new tw_timer(rotation, ts);
    m_slots[ts].push_front(timer);
    return timer;
}

void TimerWheel::DelTimer(tw_timer *timer) {
    if (!timer) return;
    int ts = timer->time_slot;
    m_slots[ts].remove(timer);
    delete timer;
}

void TimerWheel::Tick() {
    auto iter = m_slots[m_cur_slot].begin();
    while (iter != m_slots[m_cur_slot].end()) {
        if ((*iter)->rotation > 0) { //定时器的 ratation 值大于0，则在这一轮中不起作用
            (*iter)->rotation--;
            iter++;
        } else { //定时器已到期，执行定时任务，最后删除该定时器
            (*iter)->callBackFunc((*iter)->user_data);
            delete *iter;
            iter = m_slots[m_cur_slot].erase(iter);
        }
    }
    m_cur_slot = ++m_cur_slot % N;
}
```
## 运 用
时间轮的使用与升序链表定时器的使用相似，整体代码可以参考上一篇文章中 [实现--运用](./Linux%20定时器(二)%20基于升序链表实现.md) 的代码。这里展示核心的内容：
```
int main() {
    //... socket

    //...注册信号处理

    while (!stop_server) {
        //... epoll_wait        

        for (int i = 0; i < ret; ++i) {
            int sockfd = events[i].data.fd;
            if (sockfd == sock_fd) { //处理新的客户端连接
                //...accept
                
                //创建定时器
                tw_timer *timer = m_timerWheel.AddTimer(10); //添加定时器，超时时间10秒
                timer->user_data = &m_user[conn_fd]; //设置用户数据
                timer->callBackFunc = TimerCallBack; //设置回调函数
                
            } else if ((sockfd == pipefd[0]) && (events[i].events & EPOLLIN)) { //处理信号
                //...
            } else if (events[i].events & EPOLLIN) { //处理客户端数据
                //...
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

## 运行结果
为心搏函数添加打印，运行服务端。当有客户端连接后，10秒内未有可读数据，服务端将处理非活动连接，断开与此客户端的socket

![image.png](https://upload-images.jianshu.io/upload_images/22192996-ffee475c89615e13.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

更多内容，详见GitHub：[ChatRoomServer](https://github.com/cyh1998/ChatRoomServer)