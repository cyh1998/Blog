## C++ 实现对ARP报文的封装和发送  
因为需要对ARP报文进行封装，所以先了解下ARP报文的格式，并根据报文结构，自定义ARP报文包结构体  

**ARP报文的格式**

![ARP报文格式](https://upload-images.jianshu.io/upload_images/22192996-03e13fad1b837ddf.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

**定义结构体**
```
struct arppacket{
        unsigned char dest_mac[ETH_ALEN];   //以太网目的MAC地址
        unsigned char src_mac[ETH_ALEN];    //以太网源MAC地址
        unsigned short type;                //以太网帧类型 - ARP帧类型值(0x0806)
        unsigned short ar_hrd;              //硬件类型 - 以太网类型值(0x1)
        unsigned short ar_pro;              //上层协议类型 - IP协议(0x0800)
        unsigned char  ar_hln;              //MAC地址长度
        unsigned char  ar_pln;              //IP地址长度
        unsigned short ar_op;               //操作码 - 0x1表示ARP请求包,0x2表示应答包
        unsigned char  ar_sha[ETH_ALEN];    //发送方MAC地址
        unsigned char ar_sip[4];            //发送方IP地址
        unsigned char ar_tha[ETH_ALEN];     //接收方MAC地址
        unsigned char ar_tip[4];            //接收方IP地址
} __attribute__ ((__packed__));
```
注：结构体声明时使用 `_attribute__ ((__packed__))` 关键字，即告诉编译器取消结构在编译过程中的优化对齐，按照实际占用字节数进行对齐。

**ARP报文的封装**
```
#define SRC_MAC  { 0x00,0x0C,0x29,0x34,0x75,0x1a }//源MAC地址(手动配置)
#define DEST_MAC { 0xD4,0x5D,0x64,0xB2,0x3E,0x66 }//目的MAC地址(手动配置)

int main(int argc,char *argv[]){
    struct arppacket arp = {
        DEST_MAC,
        SRC_MAC,
        htons(0x0806),
        htons(0x01),
        htons(0x0800),
        ETH_ALEN,
        4,
        htons(0x01),
        SRC_MAC,
        {0},
        DEST_MAC,
        {0}
    };

    inet_aton("192.168.45.129", &s); //将源IP地址转换为32位网络地址结构体
    memcpy(&arp.ar_sip, &s, sizeof(s));
 
    inet_aton("192.168.125.169", &r); //将目的IP地址转换为32位网络地址结构体
    memcpy(&arp.ar_tip, &r, sizeof(r));

    return 0;
}
```
代码中的源MAC地址、源IP地址、目的MAC地址、目的IP地址根据实际的地址进行配置

**完整代码**
```
#include <stdio.h>
#include <stdlib.h>
#include <sys/socket.h>
#include <sys/types.h>
#include <arpa/inet.h>
#include <net/ethernet.h>
#include <string.h>
#include <iostream>
#include <net/if.h> //ETH_ALEN定义头文件
#include <netpacket/packet.h> //sockaddr_ll结构体头文件
#include <unistd.h>

using namespace std;

#define SRC_MAC  { 0x00,0x0C,0x29,0x34,0x75,0x1a }//源MAC地址(手动配置)
#define DEST_MAC { 0xD4,0x5D,0x64,0xB2,0x3E,0x66 }//目的MAC地址(手动配置)
 
struct arppacket{
        unsigned char dest_mac[ETH_ALEN];//以太网目的MAC地址
        unsigned char src_mac[ETH_ALEN];//以太网源MAC地址
        unsigned short type;//以太网帧类型 - ARP帧类型值(0x0806)
        unsigned short ar_hrd;//硬件类型 - 以太网类型值0x1
        unsigned short ar_pro;//上层协议类型 - IP协议(0x0800)
        unsigned char  ar_hln;//MAC地址长度
        unsigned char  ar_pln;//IP地址长度
        unsigned short ar_op;//操作码 - 0x1表示ARP请求包,0x2表示应答包
        unsigned char  ar_sha[ETH_ALEN];//发送方MAC地址
        unsigned char ar_sip[4];//发送方IP地址
        unsigned char ar_tha[ETH_ALEN];//接收方MAC地址
        unsigned char ar_tip[4];//接收方IP地址
} __attribute__ ((__packed__));

int main(){
    int sock_fd;
    struct in_addr s,r;
    struct sockaddr_ll sl;
    struct arppacket arp = {
        DEST_MAC,
        SRC_MAC,
        htons(0x0806),
        htons(0x01),
        htons(0x0800),
        ETH_ALEN,
        4,
        htons(0x01),
        SRC_MAC,
        {0},
        DEST_MAC,
        {0}
    };
 
    sock_fd = socket(AF_PACKET,SOCK_RAW,htons(ETH_P_ALL));
    if(sock_fd < 0){
        cout << "socket error" << endl;
        return -1;
    }
    memset(&sl, 0, sizeof(sl));
 
    inet_aton("192.168.45.129", &s); //将源IP地址转换为32位网络地址结构体
    memcpy(&arp.ar_sip, &s, sizeof(s));
    inet_aton("192.168.125.169", &r); //将目的IP地址转换为32位网络地址结构体
    memcpy(&arp.ar_tip, &r, sizeof(r));
 
    sl.sll_family = AF_PACKET; //地址族
    sl.sll_ifindex = IFF_BROADCAST; //广播地址有效，即以太网支持广播
	
	while(1){ //循环发送
        if(sendto(fd,&arp, sizeof(arp), 0, (struct sockaddr*)&sl, sizeof(sl)) <= 0)
            cout << "send error" << endl;
        else cout << "send success" << endl;
        sleep(2);
    }

    close(sock_fd);
    return 0;
}
```
**运行结果**

![ARP报文发送](https://upload-images.jianshu.io/upload_images/22192996-5ced0e93a72484de.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

利用Wireshark可以抓到程序发送的ARP报文
