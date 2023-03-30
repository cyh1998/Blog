实现一个简单的客户端，来循环接收IP报文，并打印相关的MAC地址和IP地址信息。先上代码：
```
#include <stdio.h>
#include <stdlib.h>
#include <sys/socket.h>
#include <sys/types.h>
#include <string.h>
#include <iostream>
#include <netinet/ip.h> //结构体iphdr的定义
#include <netinet/if_ether.h> //以太网帧以及协议ID的定义
#include <arpa/inet.h> //提供IP地址转换函数inet_ntoa()

using namespace std;

int main(int argc, char **argv) {
    int sock_fd, recv_size;
    char buffer[2048];
    struct ethhdr *eth; //以太网帧结构体指针
    struct iphdr *iph; //IPv4数据包结构体指针
    struct in_addr addr1, addr2;
    int count = 1; //总接收次数

    //创建接收IP报文的socket连接
    if (0 > (sock_fd = socket(PF_PACKET, SOCK_RAW, htons(ETH_P_IP)))) {
        perror("socket");
        exit(1);
    }

    //循环接收
    while (1) {
        recv_size = recv(sock_fd, buffer, 2048, 0);
        cout << "---------------- count = " << count++ << " ----------------" << endl;
        cout << recv_size << " bytes read" << endl;

        eth = (struct ethhdr*)buffer; //将接收的数据转为以太网帧的格式
        printf("Dest MAC addr : %02x:%02x:%02x:%02x:%02x:%02x\n", eth->h_dest[0], eth->h_dest[1], eth->h_dest[2], eth->h_dest[3], eth->h_dest[4], eth->h_dest[5]);
        printf("Source MAC addr : %02x:%02x:%02x:%02x:%02x:%02x\n", eth->h_source[0], eth->h_source[1], eth->h_source[2], eth->h_source[3], eth->h_source[4], eth->h_source[5]);

        iph = (struct iphdr*)(buffer + sizeof(struct ethhdr)); //将接收的数据转为IPv4数据包的格式
        memcpy(&addr1, &iph->saddr, 4);  //复制源IP地址
        memcpy(&addr2, &iph->daddr, 4);  //复制目的IP地址

        if (iph->version == 4 && iph->ihl == 5) {
            cout << "Source host : " << inet_ntoa(addr1) << endl;
            cout << "Dest host : " << inet_ntoa(addr2) << endl;
        }
    }
}
```
说明：
**1. `socket()` 参数**
第一个参数 `PF_PACKET` 和第二个参数 `SOCK_RAW` 即表明此通信工作在网络层，且进程必须以root用户权限运行。
常见的还有使用 `AF_INET` 与 `SOCK_STREAM` 或者 `SOCK_DGRAM` 配合。`AF_INET` 表明通信工作在传输层，使用Internet协议。`SOCK_STREAM`和`SOCK_DGRAM`则对应TCP和UDP。
第三个参数 `htons(ETH_P_IP)` 为IP协议ID，即 `0x0800`，表示接受IP数据包

**2. `recv()` 函数**
`recv()` 用于接收通信的数据，第一个参数为客户端对应的socket套接字，第二个参数为要保存接收数据的地方，第三个参数是保存区的大小，第四个参数一般设置为0。
成功，则返回接收收据的字节数，失败，则返回错误码。
和 `recv` 相似的还是有 `recvfrom`，`recvfrom`多了两个参数，可以用来接收对方的地址信息。

**3. `in_addr` 结构体和 `inet_ntoa()` 函数**
结构体 `in_addr` 用来表示一个32位的IPv4地址。
函数 `inet_ntoa()` 用来将网络地址转换成`.`点隔的字符串格式
头文件为 `<arpa/inet.h>`

实现结果：
启动客户端，并用主机ping虚拟机，可以看到客户端打印的相关信息。

![IP数据](https://upload-images.jianshu.io/upload_images/22192996-e37d2fba2706dcc6.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
