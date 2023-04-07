## 概 述
在服务器中，我们需要处理I/O事件、信号以及定时事件。所谓定时事件，即在预期的时间点触发响应的逻辑，比如定期检测客户端连接的状态。Linux中提供了三种方式来实现定时机制：  
**1. socket选项SO_RCVTIMEO和SO_SNDTIMEO**  
**2. SIGALRM信号**  
**3. I/O复用系统调用的超时参数**  

## 实 现
### 一、socket选项SO_RCVTIMEO和SO_SNDTIMEO
我们可以通过 `setsockopt()` 函数来设置socket接受数据与发送数据的超时时间，`SO_RCVTIMEO` 和 `SO_SNDTIMEO` 仅对系统提供的socket API(包括send、sendmsg、recv、recvmsg、accept和connect)有效。其系统调用函数与选项的对应关系和行为如下表：

![系统调用与选项对应表](https://upload-images.jianshu.io/upload_images/22192996-96c2d4145b98065c.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

以 `connect()` 和 `SO_SNDTIMEO` 为例，示例代码如下：
```
int main() {
    //...其他流程

    int ret;
    int socket_fd = socket(PF_INET, SOCK_STREAM, 0); //建立socket

    struct timeval timeout;
    timeout.tv_sec = 10; //设置的超时时间值
    timeout.tv_usec = 0;
    socklen_t len = sizeof(timeout);
    //使用setsockopt() SO_SNDTIMEO选项 设置connect超时时间，时间类型是timeval
    ret = setsockopt(socket_fd, SOL_SOCKET, SO_SNDTIMEO, &timeout, len);

    ret = connect(socket_fd, (struct sockaddr *)&server_addr, len);
    if (ret == -1) {
        if (errno == EINPROGRESS) { //错误码为EINPROGRESS时，表示超时
            cout << "connecting timeout" << endl;
        } else { //正常的connect失败
            cout << "connecting error" << endl;
        }
    }
 
    //...其他流程
}
```

### 二、SIGALRM信号
SIGALRM信号的处理比较复杂，且有多种实现方式，会在后续的文章中介绍

### 三、I/O复用系统调用的超时参数
我们可以使用I/O复用的系统调用来设置超时参数，并统一处理信号和I/O事件。以I/O复用的Epoll为例，示例代码如下：
```
#define TIMEOUT 5000 //自定义超时时间

int timeout = TIMEOUT;
time_t start = time(nullptr);
time_t end = time(nullptr);

while (1) {
    start = time(nullptr);
    //通过epoll_wait()系统调用，设置超时时间为 timeout
    int number = epoll_wait(epoll_fd, events, MAX_EVENT_NUMBER, timeout);
    if (ret < 0 && errno != EINTR) {
        cout << "epoll error " << errno << endl;
        break;
    }

    if (number == 0) { //epoll_wait成功，返回0，表示超时
        //... 处理超时任务
        timeout = TIMEOUT; //重置定时时间
        continue;
    }

    end = time(nullptr);
    timeout -= (end - start) * 1000; //更新超时参数
    // 若更新后的超时时间小于等于0，说明不仅有文件描述符就绪，并且超时时间也到了
    if (timeout <= 0) {
        //... 处理超时任务
        timeout = TIMEOUT; //重置定时时间
    }

    //...处理文件描述符事件
}
```