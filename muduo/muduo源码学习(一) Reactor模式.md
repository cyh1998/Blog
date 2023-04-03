## 概 述
最近在学习muduo源码，这里梳理下muduo库中对Reactor模式的实现。  
muduo库中使用三个类： `EventLoop`，`Channel`，`Poller` 来实现Reactor模式的核心，本文基于Epoll来梳理下核心的代码。  
**注：** 文章示例中的代码并非muduo源码，而是本人基于muduo源码的学习实现的一个网络库，github地址 [NetLib](https://github.com/cyh1998/NetLib)。muduo源码地址 [muduo](https://github.com/chenshuo/muduo)。

## 实 现
**1. EventLoop**  
事件循环，其核心函数是 `EventLoop::loop()`。函数中循环执行：调用 `Epoller::poll()`，返回就绪的事件容器 `vector<Channel*>m_activeChannels`，遍历该容器，通过 `Channel::handleEvent()` 分发事件。代码如下：
```
void EventLoop::loop() {
    if (!m_looping) {
        asserInLoopThread();
        m_looping = true;
        m_quit = false;

        while (!m_quit) { //事件循环是否退出
            m_activeChannels.clear();
            m_epoller->poll(kEpollTimeMs, &m_activeChannels); //Epoller::poll()
            for (auto & item : m_activeChannels) {
                item->handleEvent(); //Channel::handleEvent()
            }
        }
    }

    LOG_INFO("EventLoop stop looping");
    m_looping = false;
}
```
**2. Channel**  
IO事件分发，每个Channel只属于一个EventLoop，每一个Channel对象都对应了一个文件描述符fd。其核心成员如下：
```
EventLoop* m_loop; //所属的EventLoop
const int  m_fd; //文件描述符
uint32_t   m_events;  //关心的IO事件
uint32_t   m_revents; //当前活动的IO事件
int        m_index;   //描述符的标识

std::function<void()> m_readCallback;  //可读事件回调
std::function<void()> m_writeCallback; //可写事件回调
std::function<void()> m_errorCallback; //错误事件回调
```
Channel的核心函数是 `Channel::handleEvent()`，函数根据事件的类型触发不同的回调函数，代码如下：
```
void Channel::handleEvent() {
    // 触发错误事件
    if (m_revents & EPOLLERR) {
        if (m_errorCallback) m_errorCallback();
    }
    // 触发可读事件
    if (m_revents & (EPOLLIN | EPOLLPRI | EPOLLRDHUP)) {
        if (m_readCallback) m_readCallback();
    }
    // 触发可写事件
    if (m_revents & EPOLLOUT) {
        if (m_writeCallback) m_writeCallback();
    }
}
```
Channel还提供了注册回调函数、注册文件描述符事件的接口函数，如下：
```
void setReadCallback(const EventCallback& cb) { m_readCallback = cb; } //注册可读事件回调函数
void setWriteCallback(const EventCallback& cb) { m_writeCallback = cb; } //注册可写事件回调函数
void setErrorCallback(const EventCallback& cb) { m_errorCallback = cb; } //注册错误事件回调函数
void enableReading() { m_events |= s_readEvent; update(); } //注册可读事件
void enableWriting() { m_events |= s_writeEvent; update(); } //注册可写事件
void disableAll() { m_events = s_noneEvent; update(); } //取消全部事件
```
其中 `Channel::update()` 通过所属的EventLoop调用 `EventLoop::updateChannel(Channel*)`，再调用 `Epoller::updateChannel(Channel*)`，最后通过 `epoll_ctl()` 来更新注册事件表。相关代码如下：
```
// Channel::update()
void Channel::update() {
    m_loop->updateChannel(this);
}

// EventLoop::updateChannel(Channel*)
void EventLoop::updateChannel(Channel *channel) {
    if (channel->ownerLoop() == this) {
        asserInLoopThread();
        m_epoller->updateChannel(channel);
    }
}

//Epoller::updateChannel(Channel*)
void Epoller::updateChannel(Channel *channel) {
    Epoller::asserInLoopThread();
    const int index = channel->getIndex();
    int fd = channel->fd();
    LOG_INFO("Epoller update: fd = %d events = %d index = %d", fd, channel->getEvents(), index);
    if (index == -1 || index == 2) {
        if (index == -1) {
            m_channelMap[fd] = channel;
        }

        channel->setIndex(1);
        update(EPOLL_CTL_ADD, channel);
    } else {
        if (channel->isNoneEvent()) {
            update(EPOLL_CTL_DEL, channel);
        } else {
            update(EPOLL_CTL_MOD, channel);
        }
    }
}

// epoll_ctl()系统调用
void Epoller::update(int operation, Channel* channel) const {
    epoll_event event;
    event.events = channel->getEvents();
    event.data.ptr = channel;
    int fd = channel->fd();
    epoll_ctl(m_epollFd, operation, fd, &event);
}
```
**3. Epoller(Poller)**  
IO多路复用，muduo中分别封装了poll和epoll，这里我们只以epoll为例。  
Epoller的核心是 `Epoller::poll()`，即调用 `epoll_wait()` 获取当前活动的IO事件，通过私有函数 `fillActiveChannels()` 添加到传入的 `activeChannels` 中并返回。代码如下：
```
void Epoller::poll(int timeoutMs, ChannelList* activeChannels) {
    int numEvents = ::epoll_wait(m_epollFd, m_events.data(), static_cast<int>(m_events.size()), timeoutMs);
    int savedErrno = errno;
    if (numEvents > 0) {
        LOG_INFO("%d events happened", numEvents);
        fillActiveChannels(numEvents, activeChannels);
        if (static_cast<size_t>(numEvents) == m_events.size()) {
            m_events.resize(m_events.size() * 2);
        }
    } else if (numEvents == 0) {
        LOG_INFO("no events happened");
    } else {
        if (savedErrno != EINTR) {
            errno = savedErrno;
            LOG_ERROR("epoll_wait error!");
        }
    }
}

void Epoller::fillActiveChannels(int numEvents, Epoller::ChannelList *activeChannels) const {
    for (int i = 0; i < numEvents; ++i) {
        auto* channel = static_cast<Channel*>(m_events[i].data.ptr);
        channel->setRevents(m_events[i].events);
        activeChannels->push_back(channel);
    }
}
```
以上三个类实现了Reactor的核心，整个流程如下(贴一下书上的时序图)：

![Reactor流程](https://upload-images.jianshu.io/upload_images/22192996-0020d42e01d840ef.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

更多内容，详见github [NetLib](https://github.com/cyh1998/NetLib)  
参考：  
《Linux多线程服务端编程》陈硕 著  
[muduo源码](https://github.com/chenshuo/muduo)  