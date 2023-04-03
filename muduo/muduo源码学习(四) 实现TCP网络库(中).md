## runInLoop相关
在之前得文章中提到了 `EventLoop::runInLoop()`，该函数用于在 `EventLoop` 的IO线程执行某个用户的任务回调，源码如下：
```
void EventLoop::runInLoop(const Functor& cb)
{
    if (isInLoopThread()) { //判断是否在当前IO线程
        cb(); //同步调用
    } else {
        queueInLoop(cb); //加入队列
    }
}
```
若用户在其他线程调用 `runInLoop()`，则执行 `queueInLoop()`，其实现如下：
```
void EventLoop::queueInLoop(const Functor& cb)
{
    {
    MutexLockGuard lock(mutex_);
    pendingFunctors_.push_back(cb);
    }

    if (!isInLoopThread() || callingPendingFunctors_) {
        wakeup();
    }
}
```
该函数将 `cb` 添加到队列 `pendingFunctors_` 中，并唤醒IO线程。

**1. 队列中的回调是如何触发的**  
在 `EventLoop::loop()` 的事件循环中，通过 `doPendingFunctors()` 来执行队列中的任务回调，实现如下：
```
void EventLoop::loop()
{
    //...
    while (!quit_) {
        //...
        doPendingFunctors();
    }
    //...
}
```
这里可能会有疑问，IO线程会阻塞在事件循环 `EventLoop::loop()` 的 `poll` 调用中，因此通过 `loop()` 来触发用户回调，如果 `EventLoop` 中一直没有事件触发，那 `poll` 会一直阻塞，从而导致用户回调一直无法执行。  
所以为了及时触发用户回调，我们需要去唤醒IO线程。

**2. 唤醒的实现**  
书中提到，传统的做法是使用 `pipe(2)`，让IO线程监视此管道的可读事件。在需要唤醒时，往管道写入一个字节来触发唤醒。  
muduo中使用了 `eventfd(2)` 来实现IO线程的唤醒。其不必管理缓冲区，可以更高效地唤醒。  
在 `EventLoop` 构造时，创建eventfd并注册可读事件，将事件分发至 `EventLoop::handleRead()`。  
和传统做法一样，`wakeup()` 的实现就是对eventfd进行写操作，从而触发可读事件达到唤醒IO线程的目的。
```
void EventLoop::wakeup()
{
    uint64_t one = 1;
    ssize_t n = sockets::write(wakeupFd_, &one, sizeof one);
    //...
}
```
**3. doPendingFunctors的实现**  
上文提到在 `EventLoop::loop()` 事件循环最后，触发队列中的任务回调，其实现如下：
```
void EventLoop::doPendingFunctors()
{
    std::vector<Functor> functors;
    callingPendingFunctors_ = true;

    {
    MutexLockGuard lock(mutex_);
    functors.swap(pendingFunctors_);
    }

    for (const Functor& functor : functors)
    {
        functor();
    }
    callingPendingFunctors_ = false;
}
```
这段代码有两处需要注意的：
- 锁的范围  
`doPendingFunctors()` 没有直接在临界区内依次调用任务回调，而且 `sawp()` 到局部变量中(减小了临界区的长度)。否则，锁会一直等到所有回调函数处理完才释放，阻塞其他线程调用 `queueInLoop()`。

- callingPendingFunctors_   
代码中使用 `callingPendingFunctors_` 来标记是否在执行 `doPendingFunctors()` 过程中，目的是为了 `queueInLoop()` 来判断唤醒的时机。

**4. 唤醒的时机**  
在 `queueInLoop()` 中，是否需要 `weakup()` 进行了这样的判断：
```
if (!isInLoopThread() || callingPendingFunctors_) {
    wakeup();
}
```
即，当调用 `queueInLoop()` 的线程不是IO线程；以及在IO线程调用 `queueInLoop()`，但此时正在执行 `doPendingFunctors()` 过程中，才需要唤醒。  
因为，当 `queueInLoop()` 在IO线程中调用，`doPendingFunctors()` 就会在 `EventLoop::loop()` 事件循环的最后被调用，所以此时无须唤醒。  

## TcpServer
创建 `TcpServer` 对象，在其构造时通过 `Acceptor` 来获得新连接的 `fd`。在 `Acceptor` 的构造函数中调用了 **[1]** `socket()` 和 **[2]** `bind()`。  
启动函数 `TcpServer::start()` 通过 `Acceptor::listen()` (以runInLoop的方式)调用 **[3]** `listen()`，并注册了可读事件。  
当新连接请求时，触发可读事件，在其回调函数 `Acceptor::handleRead()` 中调用了 **[4]** `accept()` 并回调了 `TcpServer::newConnection()`，其实现如下：
```
void TcpServer::newConnection(int sockfd, const InetAddress& peerAddr)
{
    loop_->assertInLoopThread();
    EventLoop* ioLoop = threadPool_->getNextLoop();
    char buf[64];
    snprintf(buf, sizeof buf, "-%s#%d", ipPort_.c_str(), nextConnId_);
    ++nextConnId_;
    string connName = name_ + buf;

    InetAddress localAddr(sockets::getLocalAddr(sockfd));
    TcpConnectionPtr conn(
        new TcpConnection(ioLoop, connName, sockfd, localAddr, peerAddr));
    connections_[connName] = conn;
    conn->setConnectionCallback(connectionCallback_);
    conn->setMessageCallback(messageCallback_);
    conn->setWriteCompleteCallback(writeCompleteCallback_);
    conn->setCloseCallback(std::bind(&TcpServer::removeConnection, this, _1));
    ioLoop->runInLoop(std::bind(&TcpConnection::connectEstablished, conn));
}
```
函数中创建了 `TcpConnection` 对象，把它加入 `ConnectionMap` 中管理并设置了相关回调。  
**注：** 以上的 **[1] [2] [3] [4]** 完成了一次完整的连接。  

## TcpConnection
`TcpConnection` 是muduo中唯一默认使用智能指针管理的类，也是唯一继承 `enable_shared_from_this` 的类。有关 `enable_shared_from_this` 可以参考 [share_ptr相关](../C++/C++%20shared_ptr相关技术.md)。  
`TcpConnection` 作用就是使用 `Channel` 来获得socket上的IO事件，执行各种回调。相关事件的处理，后面的文章再具体介绍。
