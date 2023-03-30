#### 概 述
进程间通信(**IPC**)有多种方式，包括管道、命名管道(使用不多)、信号量、消息队列以及共享内存。本文主要介绍共享内存相关的系统调用以及利用共享内存的**POSIX**方式来实现聊天室服务器程序。

#### 函数接口
首先，我们需要了解下，在Linux环境下，IPC有System V以及POXIS两种方式来实现。简单来说，System V和POXIS是Linux环境下的两种标准，拥有不同的函数接口。具体两者的区别，有兴趣的可以看看Linux发展历史。
**一. System V**  
System V方式的共享内存API定义在`sys/shm.h`头文件中，包括以下4个函数
```
#include <sys/shm.h>
int shmget(key_t key, size_t size, int shmflg); //获取共享内存
```
`shmget()`用于获取一块新的或已有的共享内存，成功返回共享内存的标识符，失败返回-1  
参数说明  
`key`：用于标识一段全局唯一的共享内存  
`size`：指定共享内存大小，单位字节  
`shmflg`：函数标志，可选值`IPC_CREAT`和`IPC_EXEC`  
```
#include <sys/shm.h>
void shmat(int shm_id, const void* shm_addr, int shmflg); //映射共享内存
```
`shmat()`用于将共享内存于用户进程映射，成功返回映射后的地址，失败返回`(void *)-1`  
参数说明：  
`shmid`：指定共享内存的标识符，即`shmget()`的返回值  
`shmaddr`：指定共享内存映射到进程的哪块地址空间，若为NULL，则由操作系统选择关联地址  
`shmflg`：函数标志，可选值`SHM_RDONLY`、`SHM_RND`、`SHM_REMAP`、`SHM_EXEC`  
```
#include <sys/shm.h>
int shmdt(const void* shm_addr); //撤销共享内存与进程之间的映射
```
`shmdt()`用于撤销指定共享内存与进程之间的映射，成功返回0，失败返回`(void *)-1`。  
参数`shmaddr`是`shmat()`成功返回的地址  
```
#include <sys/shm.h>
shmctl(int shm_id, int command, struct shmid_ds* buf); //控制共享内存(删除)
```
`shmctl()`用于控制共享内存，成功的返回值由参数`command`决定，失败返回-1  
参数说明：  
`shmid`：指定共享内存的标识符，即`shmget()`的返回值  
`command`：指定要执行的命令，`shmctl()`支持的命令如下表  

![shmctl支持的命令](https://upload-images.jianshu.io/upload_images/22192996-7fc9c8241b5a9bcc.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

**二. POXIS**  
共享内存的POXIS方式基于mmap来实现的，具体函数如下：
```
#include <sys/mman.h>
int shm_open(const char *name, int oflag, mode_t mode);
```
`shm_open()`用于创建/获取一个共享内存，成功返回文件描述符，失败返回-1  
参数说明：  
`name`：指定共享内存名字，必须以`/`开头，并且后面不再有`/`，长度不超过`NAME_MAX`(通常为255)  
`oflag`：指定创建方式，支持以下值  

![创建方式](https://upload-images.jianshu.io/upload_images/22192996-facd996e2799ea99.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

```
#include <unistd.h>
int ftruncate(int fd, off_t length);
```
`ftruncate()`用于修改共享内存(文件)大小，成功返回0，失败返回-1
参数说明：  
`fd`：文件描述符  
`length`：修改的大小  
```
#include <sys/mman.h>
int shm_unlink(const char *name);
```
`shm_unlink()`用于删除`name`指定的共享内存，成功返回0，失败返回-1

利用mmap将共享内存进行映射和卸载
```
#include <sys/mman.h>
void *mmap(void *addr, size_t length, int prot, int flags, int fd, off_t offset);
int munmap(void *addr, size_t length);
```
**注：** 使用POXIS方式的共享内存时，在编译时需要链接`-lrt`，在Cmake中，写法如下：
```
#添加链接库
target_link_libraries(${PROJECT_NAME} -lrt) //在最后添加 -lrt
```
#### 实 现
我们将之前实现的聊天室服务器修改为一个多进程服务器，一个子进程处理一个客户端连接。父子进程之间利用管道相互通信，告知相关信息。同时，将所有客户端的缓冲区设计为一块共享内存。示例代码如下：
```
//MultiProcessServer.h
#include <string> //string
#include <unordered_map> //unordered_map
#include <netinet/in.h> //sockaddr_in

#define MAX_EVENT_NUMBER 5000   //Epoll最大事件数
#define FD_LIMIT         10     //最大客户端数量
#define BUFFER_SIZE      1024   //缓存区数据大小

//客户端数据
struct client_data {
    sockaddr_in address;
    int sockfd;
    pid_t pid;
    int pipefd[2];
};

class MultiProcessServer {
public:
    explicit MultiProcessServer();
    ~MultiProcessServer();
    bool InitServer(const std::string &Ip, const int &Port);
    void Run();

private:
    int m_socketFd = 0; //创建的socket文件描述符
    static int s_epollFd; //创建的epoll文件描述符
    static int s_userCount; //客户端编号
    static int s_pipeFd[2]; //信号通信管道
    client_data* m_user;
    std::unordered_map<int, int> m_subProcess; //子进程Pid和客户端编号的映射表
    int m_shmfd; //共享内存标识符
    std::string m_shmName = "/myshm"; //共享内存名称
    char *m_shareMem; //共享内存首地址
    bool m_serverStop = true;
    bool m_terminate = false;
    static bool s_childStop;

private:
    int SetNonblocking(int fd); //将文件描述符设置为非堵塞的
    void Addfd(int epoll_fd, int sock_fd); //将文件描述符上的事件注册
    static void SignalHandler(int sig); //父进程信号处理回调函数
    static void ChildSignalHandler(int sig); //子进程信号处理回调函数
    static void AddSignal(int sig, void(*handler)(int), bool restart = true); //设置信号处理回调函数
    void HandleSignal(const std::string &sigMsg); //自定义信号处理函数
    void RunChild(int idx, client_data* users, char *share_mem); //子进程函数
};

//MultiProcessServer.cpp
#include <iostream>
#include <fcntl.h> //fcntl()
#include <sys/epoll.h> //epoll
#include <netinet/in.h> //sockaddr_in
#include <arpa/inet.h> //inet_pton()
#include <string.h> //memset()
#include <unistd.h> //close() alarm() ftruncate()
#include <signal.h> //signal
#include <sys/wait.h> //waitpid()
#include <sys/mman.h> //shm_open()
#include "MultiProcessServer.h"

using namespace std;

int MultiProcessServer::s_pipeFd[2];
int MultiProcessServer::s_epollFd = 0;
int MultiProcessServer::s_userCount = 0;
bool MultiProcessServer::s_childStop = false;

MultiProcessServer::MultiProcessServer() = default;

MultiProcessServer::~MultiProcessServer() {
    m_subProcess.clear();
    delete [] m_user;
}

bool MultiProcessServer::InitServer(const std::string &Ip, const int &Port) {
    int ret;
    struct sockaddr_in address;
    memset(&address, 0, sizeof(address)); //初始化 address
    address.sin_family = AF_INET;
    inet_pton(AF_INET, Ip.c_str(), &address.sin_addr);
    address.sin_port = htons(Port);

    m_socketFd = socket(AF_INET, SOCK_STREAM, 0); //创建socket
    if (m_socketFd < 0) {
        cout << "Server: socket error! id:" << m_socketFd << endl;
        return false;
    }

    ret = bind(m_socketFd, (struct sockaddr*)&address, sizeof(address)); //bind
    if (ret == -1) {
        cout << "Server: bind error!" << endl;
        return false;
    }

    ret = listen(m_socketFd, 20); //listen
    if (ret == -1) {
        cout << "Server: listen error!" << endl;
        return false;
    }

    s_epollFd = epoll_create(5);
    if (s_epollFd == -1) {
        cout << "Server: create epoll error!" << endl;
        return false;
    }
    Addfd(s_epollFd, m_socketFd); //注册sock_fd上的事件

    ret = socketpair(PF_UNIX, SOCK_STREAM, 0, s_pipeFd); //使用socketpair创建管道，并注册s_pipeFd[0]上可读写事件
    if (ret == -1) {
        cout << "Server: socketpair error!" << endl;
        return false;
    }
    SetNonblocking(s_pipeFd[1]);
    Addfd(s_epollFd, s_pipeFd[0]);

    AddSignal(SIGCHLD, SignalHandler); //子进程状态发生变化
    AddSignal(SIGTERM, SignalHandler); //终止进程，即kill
    AddSignal(SIGINT, SignalHandler); //键盘输入以中断进程 Ctrl+C
    AddSignal(SIGPIPE, SIG_IGN); //忽略管道引起的信号
    m_serverStop = false;
    m_user = new client_data[FD_LIMIT + 1];

    m_shmfd = shm_open(m_shmName.c_str(), O_CREAT | O_RDWR, 0666);
    if (m_shmfd == -1) {
        cout << "Server: open shared memory error!" << endl;
        return false;
    }
    ret = ftruncate(m_shmfd, FD_LIMIT * BUFFER_SIZE);
    if (ret == -1) {
        cout << "Server: create shared memory error!" << endl;
        return false;
    }
    m_shareMem = reinterpret_cast<char *>(mmap(nullptr, FD_LIMIT * BUFFER_SIZE, PROT_READ | PROT_WRITE, MAP_SHARED, m_shmfd, 0));
    if (m_shareMem == MAP_FAILED) {
        cout << "Server: bind shared memory error!" << endl;
        return false;
    }
    close(m_shmfd);

    cout << "INIT SERVER SUCCESS!" << endl;
    return true;
}

void MultiProcessServer::Run() {
    cout << "START SERVER!" << endl;
    int ret;
    epoll_event events[MAX_EVENT_NUMBER];
    while (!m_serverStop) {
        int number = epoll_wait(s_epollFd, events, MAX_EVENT_NUMBER, -1); //epoll_wait
        if (number < 0 && errno != EINTR) {
            cout << "Server: epoll error" << endl;
            break;
        }

        for (int i = 0; i < number; ++i) {
            int sockfd = events[i].data.fd;
            if (sockfd == m_socketFd) { //新的socket连接
                struct sockaddr_in client_addr;
                socklen_t len = sizeof(client_addr);
                int conn_fd = accept(m_socketFd, (struct sockaddr*)&client_addr, &len);

                if (conn_fd < 0) continue;
                if (s_userCount >= FD_LIMIT) {
                    close(conn_fd);
                    continue;
                }

                m_user[s_userCount].address = client_addr;
                m_user[s_userCount].sockfd = conn_fd;

                ret = socketpair(PF_UNIX, SOCK_STREAM, 0, m_user[s_userCount].pipefd);
                if (ret == -1) break;
                pid_t pid = fork();
                if (pid < 0) { //fork 失败
                    close(conn_fd);
                    continue;
                } else if (pid == 0) { //子进程
                    close(s_epollFd);
                    close(m_socketFd);
                    close(m_user[s_userCount].pipefd[0]);
                    close(s_pipeFd[0]);
                    close(s_pipeFd[1]);

                    RunChild(s_userCount, m_user, m_shareMem);
                    munmap(reinterpret_cast<void *>(m_shareMem), FD_LIMIT * BUFFER_SIZE);
                    exit(0);
                } else { //父进程
                    close(conn_fd);
                    close(m_user[s_userCount].pipefd[1]);
                    Addfd(s_epollFd, m_user[s_userCount].pipefd[0]);

                    m_user[s_userCount].pid = pid;
                    m_subProcess[pid] = s_userCount;
                    s_userCount++;
                    cout << "Server: New connect, Now client number:" << s_userCount << endl;
                }
            } else if ((sockfd == s_pipeFd[0]) && (events[i].events & EPOLLIN)) { //处理信号
                char sigBuf[BUFFER_SIZE];
                ret = recv(s_pipeFd[0], sigBuf, sizeof(sigBuf), 0);
                if (ret <= 0) continue;
                else {
                    string sigMsg(sigBuf);
                    HandleSignal(sigMsg); //自定义处理信号
                }
            } else if (events[i].events & EPOLLIN) { //某个子进程向主进程写入数据
                int child = 0;
                ret = recv(sockfd, reinterpret_cast<char *>(&child), sizeof(child), 0);
                if (ret <= 0) continue;
                else {
                    for (int i = 0; i < s_userCount; ++i) {
                        if (m_user[i].pipefd[0] != sockfd){
                            send(m_user[i].pipefd[0], reinterpret_cast<char *>(&child), sizeof(child), 0);
                        }
                    }
                }
            } else {
                cout << "Server: socket something else happened!" << endl;
            }
        }
    }

    cout << "CLOSE SERVER!" << endl;
    close(m_socketFd);
    close(s_epollFd);
    close(s_pipeFd[0]);
    close(s_pipeFd[1]);
    shm_unlink(m_shmName.c_str());//关闭POSIX共享内存
}

int MultiProcessServer::SetNonblocking(int fd) {
    int old_option = fcntl(fd, F_GETFL);
    int new_option = old_option | O_NONBLOCK;
    fcntl(fd, F_SETFL, new_option);
    return old_option;
}
    
void MultiProcessServer::Addfd(int epoll_fd, int sock_fd) {
    epoll_event event;
    event.data.fd = sock_fd;
    event.events = EPOLLIN | EPOLLET;
    epoll_ctl(epoll_fd, EPOLL_CTL_ADD, sock_fd, &event);
    SetNonblocking(sock_fd);
}

void MultiProcessServer::SignalHandler(int sig) {
    // 保留原有的errno，函数执行后恢复，以保证函数的可重入性
    int old_errno = errno;
    int msg = sig;
    send(s_pipeFd[1], reinterpret_cast<char *>(&msg), 1 , 0); //将信号写入管道，通知主循环
    errno = old_errno;
}

void MultiProcessServer::ChildSignalHandler(int sig) {
    s_childStop = true; //终止子进程主循环
}

void MultiProcessServer::AddSignal(int sig, void(*handler)(int), bool restart) {
    struct sigaction sa;
    memset(&sa, '\0', sizeof(sa));
    sa.sa_handler = handler;
    if (restart) sa.sa_flags |= SA_RESTART;
    sigfillset(&sa.sa_mask);
    if (sigaction(sig, &sa, nullptr) == -1) {
        cout << "Server: Add signal error!" << endl;
    }
}

void MultiProcessServer::HandleSignal(const string &sigMsg) {
    for (auto &item : sigMsg){ //逐个处理接收的信号
        switch (item) {
            case SIGCHLD: { //子进程退出
                cout << " SIGCHLD: The child process state changes" << endl;
                pid_t pid;
                int stat;
                while ((pid = waitpid(-1, &stat, WNOHANG)) > 0) {
                    int del_user = m_subProcess[pid];
                    if (del_user < 0 || del_user > FD_LIMIT) continue;
                    epoll_ctl(s_epollFd, EPOLL_CTL_DEL, m_user[del_user].pipefd[0], 0);
                    close(m_user[del_user].pipefd[0]);
                    s_userCount--;
                }
                if (m_terminate && s_userCount == 0) m_serverStop = true; //安全的终止服务器主循环
                break;
            }
            case SIGTERM: {
                cout << " SIGTERM: Kill" << endl;
                m_serverStop = true; //安全的终止服务器主循环
                break;
            }
            case SIGINT: {
                cout << " SIGINT: Ctrl+C" << endl;
                if (s_userCount == 0) {
                    m_serverStop = true; //安全的终止服务器主循环
                    break;
                }
                for (int i = 0; i < s_userCount; ++i) {
                    int pid = m_user[i].pid;
                    kill(pid, SIGTERM);
                }
                m_terminate = true;
                break;
            }
        }
    }
}

void MultiProcessServer::RunChild(int idx, client_data *users, char *share_mem) {
    epoll_event events[MAX_EVENT_NUMBER];
    int child_epollfd = epoll_create(5);
    if (child_epollfd == -1) {
        cout << "Child Process: create epoll error!" << endl;
        exit(0);
    }
    cout <<  users[idx].sockfd << endl;
    int sockfd = users[idx].sockfd;
    Addfd(child_epollfd, sockfd);
    int pipefd = users[idx].pipefd[1];
    Addfd(child_epollfd, pipefd);
    int ret;
    AddSignal(SIGTERM, ChildSignalHandler, false);

    while (!s_childStop) {
        int number = epoll_wait(child_epollfd, events, MAX_EVENT_NUMBER, -1);
        if (number < 0 && errno != EINTR) {
            cout << "Child Process: epoll error" << endl;
            break;
        }

        for (int i = 0; i < number; ++i) {
            int fd = events[i].data.fd;
            if ((fd == sockfd) && (events[i].events & EPOLLIN)) { //客户端有数据到达
                memset(share_mem + idx * BUFFER_SIZE, '\0', BUFFER_SIZE);
                ret = recv(fd, share_mem + idx * BUFFER_SIZE, BUFFER_SIZE - 1, 0);
                if (ret < 0) {
                    if (errno != EAGAIN) s_childStop = true; //对端异常断开连接socket
                } else if (ret == 0) {  //对端正常关闭socket
                    s_childStop = true;
                } else {
                    send(pipefd, reinterpret_cast<char *>(&idx), sizeof(idx), 0);
                }
            } else if ((fd == pipefd) && (events[i].events & EPOLLIN)) { //主进程通知本进程，将第client个客户端数据发送给本进程负责的客户端
                int client = 0;
                ret = recv(fd, reinterpret_cast<char *>(&client), sizeof(client), 0);
                if (ret < 0) {
                    if (errno != EAGAIN) s_childStop = true; //对端异常断开连接socket
                } else if (ret == 0) {  //对端正常关闭socket
                    s_childStop = true;
                } else {
                    send(sockfd, share_mem + client * BUFFER_SIZE, BUFFER_SIZE, 0);
                }
            } else {
                cout << "Child Process: something else happened" << endl;
                continue;
            }
        }
    }

    close(sockfd);
    close(pipefd);
    close(child_epollfd);
}
```
参考：《Linux高性能服务器编程》 游双 著  
更多内容，详见GitHub：[ChatRoomServer](https://github.com/cyh1998/ChatRoomServer)