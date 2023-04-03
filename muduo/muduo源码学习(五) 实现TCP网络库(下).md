## 前 言
上一篇文章介绍了连接的创建，引出了 `TcpConnection` 类。其作用就是处理socket上的IO事件，执行各种回调。本文介绍 `TcpConnection` 对断开连接、读取数据、发送数据的处理。

## 断开连接
连接的关闭分为主动断开和被动断开，两者的处理方式基本一致。`muduo` 采用的连接关闭方式：被动断开，其核心函数为 `TcpConnection::handleClose()`。书中提到，如果需要主动断开，添加一个接口调用 `handleClose()` 即可。  

对于远端连接断开的感知：在可读事件处理函数 `handleRead()` 中，当read返回值为0时，即远端断开了连接，调用 
`TcpConnection::handleClose()`。此时处理如下：  
**1.** 取消所有关注的IO事件  
**2.** 调用用户注册回调ConnectionCallback  
**3.** 调用`closeCallback_()`，此回调绑定到`TcpServer::removeConnection()`  

在 `removeConnection()` 中处理如下：  
**4.** 将对应的 `TcpConnection` 对象从 `TcpServer` 中移除  
**5.** 调用 `TcpConnection::connectDestroyed()`，并通过 `std::bind()` 将 `TcpConnection` 对象的生命周期延长到执行完成 `connectDestroyed()`  
**6.** 将连接对应的 `Channel` 从 `EventLoop` 中移除  
**7.** `TcpConnection` 析构，成员 `socket_` 引用计数为0，其析构时会调用 `close()`，关闭连接的 `fd`  

## 读取数据
新连接建立时，通过 `TcpConnection::connectEstablished()` 注册可读事件，当触发可读事件时调用回调，即 `TcpConnection::handleRead()`，其主要内容如下：
```
void TcpConnection::handleRead(Timestamp receiveTime) {
    ssize_t n = inputBuffer_.readFd(channel_->fd(), &savedErrno);
    if (n > 0) {
        messageCallback_(shared_from_this(), &inputBuffer_, receiveTime);
    } else if (n == 0) {
        handleClose();
    } else {
        handleError();
    }
}
```
这里主要进行了两个处理：  
**1.** 读取数据到 `inputBuffer_` 中，其使用 `Buffer::readFd()` 来实现，具体如下。  
**2.** 调用用户回调 `messageCallback_`，此函数是在建立新连接时，将 `TcpServer` 的成员函数 `MessageCallback` 设置为回调，其由用户提供。

`Buffer::readFd()` 实现(非源码)如下：
```
ssize_t Buffer::readFd(int fd, int & savedErrno) {
    // 申请栈上空间
    char extrabuf[65536];
    struct iovec vec[2];
    const size_t writable = writableBytes();

    // 两块iovec分别指向内部buffer的可写空间和栈上空间
    vec[0].iov_base = begin() + m_writerIndex;
    vec[0].iov_len = writable;
    vec[1].iov_base = extrabuf;
    vec[1].iov_len = sizeof extrabuf;

    // 判断内部buffer的可写空间是否足够
    const int iovcnt = (writable < sizeof extrabuf) ? 2 : 1;
    const ssize_t n = sockets::readv(fd, vec, iovcnt);
    if (n < 0) {
        savedErrno = errno;
    } else if (static_cast<size_t>(n) <= writable) {
        // 内部空间足够，直接写入，移动可写索引
        m_writerIndex += n;
    } else {
        // 内部空间不足，先写入栈上空间，再将栈上数据append到内部空间
        m_writerIndex = m_buffer.size();
        append(extrabuf, n - writable);
    }
    return n;
}
```
书中提到，此实现一是使用了 `scatter/gather IO` (分离/聚散IO)，配合内部栈空间使用；二是muduo采用 `level trigger(LT)` 模式，只需要调用一次 `read(2)` 且不会丢失数据。从而兼顾了内存使用量和效率。  

## 发送数据
数据的发送通过 `TcpConnection::send()` 实现，代码如下：
```
void TcpConnection::send(const StringPiece& message)
{
    if (state_ == kConnected) {
        if (loop_->isInLoopThread()) {
            sendInLoop(message);
        } else {
            loop_->runInLoop(std::bind(&TcpConnection::sendInLoop, this, message.as_string()));
        }
    }
}
```
在确保是连接状态的情况下，如果在当前IO线程触发就调用 `TcpConnection::sendInLoop()`，反之则使用 `runInLoop` 将该任务抛给IO线程执行。有关 `runInLoop` 的内容在上一篇已经介绍过，这里不再赘述。  
在 `TcpConnection::sendInLoop()` 中，处理如下：  
**1.** 若 `outputBuffer_` 为空，直接发送数据  
**2.** 若发送数据没有写完，统计剩余的字节数，将剩余数据写入 `outputBuffer_`  
**3.** 注册可写事件  

当socket可写时，调用 `TcpConnection::handleWrite()`，继续发送 `outputBuffer_` 中的数据，一旦发送完成，立刻将可写事件移除。  
此流程需要注意的是可写事件观察的范围，可以看出只有在 `outputBuffer_` 中有数据时，才会注册观察可写事件，因为当 `outputBuffer_` 中没数据时，此时socket一直是处于可写状态的， 这将会导致一直触发 `TcpConnection::handleWrite()`，而我们并没有数据需要发送。所以此触发没有意义，不需要去关注。  

此外，数据发送的流程中：  
当 `outputBuffer_` 中的旧数据字节和剩余数据字节之和大于 `highWaterMark_` 时，会将 `highWaterMarkCallback_` 放入待执行队列中  
当数据发送完毕时，会调用 `writeCompleteCallback_`  
两者配合使用，可以起到限流的作用。  

更多内容，详见github [NetLib](https://github.com/cyh1998/NetLib)  
参考：  
《Linux多线程服务端编程》陈硕 著  
[muduo源码](https://github.com/chenshuo/muduo)  
