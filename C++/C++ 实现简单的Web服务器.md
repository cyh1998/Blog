简单的web服务器就是通过TCP三次握手建立连接后，服务器返回一个HTTP响应给浏览器。浏览器再通过URL去请求服务器的连接，并且通过URL中的路径请求服务器上的资源。

涉及的知识点：  
**1. `sockaddr_in` 结构体**  
`struct sockaddr_in` 用于处理网络通信中的地址，在头文件 `<netinet/in.h>` 中定义
```
struct sockaddr_in{
    sa_family_t       sin_family;    //地址族(Address Family)
    uint16_t          sin_port;      //16位 TCP/UDP端口号
    struct in_addr    sin_addr;      //32位 IP地址
    char              sin_zero[8];   //不使用
};

//in_addr 在arpa/inet.h定义
struct in_addr{
    in_addr_t    s_addr;    //32位IPv4地址
};
```
在使用 `bind()` 的时候，其第二个参数是 `sockaddr` 类型，但是直接向 `sockaddr` 结构体填充地址族、端口号、IP地址等信息不太容易，而填充 `sockaddr_in` 类型结构体则容易得多，故先填充 `sockaddr_in` 再将其转换为 `sockaddr` 类型。  
`sockaddr` 也是用于处理网络通信中地址的结构体，其结构体定义如下
```
struct sockaddr{
    sa_family_t  sin_family; //地址族（Address Family）
    char         sa_data[14]; //地址信息
}
```
**2. `memset()`方法**  
`memset()` 用于将一段内存空间全部设置为某个字符，在socket中常用于将结构体初始化。在头文件 `<string.h>` 中定义。
```
struct sockaddr_in sever_add;

//原型 void *memset(void *str, int c, size_t n)
memset(&sever_add,0,sizeof(sever_add)); //将sever_add初始化为0
```
**3. `assert()`方法**  
`assert()` 在头文件 `<assert.h>` 中定义，其作用是如果它的条件返回错误，则终止程序执行。在Debug模式下可以方便调试，便于定位程序错误的位置。在调试结束后，可以通过在包含 `#include <assert.h>` 的语句之前插入 `#define NDEBUG` 来禁用 `assert` 调用，示例代码如下：
```
#define NDEBUG
#include <assert.h>
```
**4. `sendfile()`方法**  
`sendfile()` 在头文件 `<sys/sendfile.h>` 中定义，其原型为：
```
//in_fd：待读出内容的文件描述符
//out_fd：待写入内容的文件描述符
//offset：指定从读入文件流的哪个位置开始读，如果为空，则使用读入文件流默认的起始位置
//count：指定文件描述符in_fd和out_fd之间传输的字节数
ssize_t senfile(int out_fd,int in_fd,off_t* offset,size_t count);
```
此方法在两个文件描述符之间传递数据（完全在内核中操作），从而避免了内核缓冲区和用户缓冲区之间的数据拷贝，效率很高，被称为**零拷贝**

**5. TCP中常用的 `socket()`、`bind()`、`listen()`、`accept()` 方法**  
TCP中服务端最常用的方法，这里不再赘述。

**整体实现的demo：**  
```
#include <iostream>
#include <netinet/in.h> //sockaddr_in结构体头文件
#include <string.h> //memset()头文件
#include <assert.h> //assert()头文件
#include <sys/types.h> //open()头文件
#include <sys/stat.h>
#include <fcntl.h>
#include <sys/sendfile.h> //sendfile()头文件
#include <unistd.h> //close()头文件

using namespace  std;

int main(){
    int socket_fd;
    int conn_fd;
    int res;
    int len;

    struct sockaddr_in sever_add;
    memset(&sever_add,0,sizeof(sever_add)); //初始化
    sever_add.sin_family = PF_INET;
    sever_add.sin_addr.s_addr = htons(INADDR_ANY);
    sever_add.sin_port = htons(8081);
    len = sizeof(sever_add);

    //socket()
    socket_fd =  socket(AF_INET,SOCK_STREAM,0);
    assert(socket_fd >= 0);

    //bind()
    res = bind(socket_fd,(struct sockaddr*)&sever_add,len);
    assert(res != -1);

    //listen()
    res = listen(socket_fd,1);
    assert(res != -1);

    while(1){
        struct sockaddr_in client;
        int client_len = sizeof(client);
        //accept()
        conn_fd = accept(socket_fd,(struct sockaddr*)&client,(socklen_t *)&client_len);
        if(conn_fd < 0) cout << "error" << endl;
        else{
            char request[1024];
            recv(conn_fd,request,1024,0);
            request[strlen(request) + 1] = '\0';
            char buf[520] = "HTTP/1.1 200 ok\r\nconnection: close\r\n\r\n";//HTTP响应
            int s = send(conn_fd,buf,strlen(buf),0);//发送响应
            int fd = open("WWW/index.html",O_RDONLY);
            sendfile(conn_fd,fd,NULL,2048);//零拷贝发送消息体
            close(fd);
            close(conn_fd);
        }
    }
    return 0;
}
```
运行后，浏览器访问localhost:8081端口，即可浏览WWW目录下的index.html文件。
页面代码：
```
<html>
    <head>
        <meta http-equiv="Content-Type" content="text/html; charset=utf-8">
        <title>MyWeb</title>
    </head>
    <body>
        <h1>C++ 实现Web服务器</h1>
        <hr>
        <p>hello word!</p>
    </body>
</html>
```
结果：

![页面显示](https://upload-images.jianshu.io/upload_images/22192996-27727a566191d1a2.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
