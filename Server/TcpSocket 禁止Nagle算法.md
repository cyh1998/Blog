#### 概 述
我们在建立使用Tcp Socket时，会遇到当发送数据较小时，响应超时，或者对端发送频率越慢，相应越慢；发送频率越快，相应越快的情况。这可能时Nagle算法对socket的影响。

#### Nagle算法
Nagle算法在未确认数据发送时会将数据放到缓存中。直到得到明显的数据确认或者直到攒到了一定数量的数据后再发包。该算法是自适应的，即确认到达的越快，数据也就发送的越快；在希望减少微小分组数目的低速广域网上，会发送更少的分组。  
具体有关此算法的原理和实现，这里不再赘述。大家可百度学习。

#### 关闭 Nagle算法
**Linux C++**   
使用`setsockopt()`，函数返回`-1`表示失败，反正成功
```
#include <netinet/tcp.h> //TCP_NODELAY

socket_fd = socket(AF_INET, SOCK_STREAM, 0); //建立的socket套接字

int on = 1;
int result = setsockopt(socket_fd, IPPROTO_TCP, TCP_NODELAY, (char *)&on, sizeof(int));
if (result == -1) {
    cout << "Close Nagle error" << endl;
}
```

**Qt**  
使用`setSocketOption()`
```
//m_socket 为 QTcpSocket
m_socket->setSocketOption(QAbstractSocket::LowDelayOption, 1);
```