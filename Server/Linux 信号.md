## 概 述
信号是由用户、系统或者进程发送给目标进程的信息。本文介绍如何在服务器中接受信号并处理信息。

## 信 号
使用命令 `kill -l`，可以查看Linux中的信号
```
cyh@cyh-VirtualBox:~$ kill -l
 1) SIGHUP       2) SIGINT       3) SIGQUIT      4) SIGILL       5) SIGTRAP
 6) SIGABRT      7) SIGBUS       8) SIGFPE       9) SIGKILL     10) SIGUSR1
11) SIGSEGV     12) SIGUSR2     13) SIGPIPE     14) SIGALRM     15) SIGTERM
16) SIGSTKFLT   17) SIGCHLD     18) SIGCONT     19) SIGSTOP     20) SIGTSTP
21) SIGTTIN     22) SIGTTOU     23) SIGURG      24) SIGXCPU     25) SIGXFSZ
26) SIGVTALRM   27) SIGPROF     28) SIGWINCH    29) SIGIO       30) SIGPWR
31) SIGSYS      34) SIGRTMIN    35) SIGRTMIN+1  36) SIGRTMIN+2  37) SIGRTMIN+3
38) SIGRTMIN+4  39) SIGRTMIN+5  40) SIGRTMIN+6  41) SIGRTMIN+7  42) SIGRTMIN+8
43) SIGRTMIN+9  44) SIGRTMIN+10 45) SIGRTMIN+11 46) SIGRTMIN+12 47) SIGRTMIN+13
48) SIGRTMIN+14 49) SIGRTMIN+15 50) SIGRTMAX-14 51) SIGRTMAX-13 52) SIGRTMAX-12
53) SIGRTMAX-11 54) SIGRTMAX-10 55) SIGRTMAX-9  56) SIGRTMAX-8  57) SIGRTMAX-7
58) SIGRTMAX-6  59) SIGRTMAX-5  60) SIGRTMAX-4  61) SIGRTMAX-3  62) SIGRTMAX-2
63) SIGRTMAX-1  64) SIGRTMAX
```
各种信号的含义大家可以自行百度，具体内容在 `signum.h` 头文件中

## 信号函数
设置信号处理函数的接口有如下两种：
```
#include <signal.h>
_sighandler_t signal (int sig, _sighandler_t _handler)
int sigsction (int sig, cont struct sigaction* act, struct sigaction* oact)
```
因为 `sigsction()` 函数更加健壮，所有这里主要说明下 `sigsction()` 的参数  
`sig` ：需要捕获的信号类型，即上文**信号**中的展示；  
`act` ：指定新的信号处理方式；  
`oact` ：输出先前的信号处理方式(不为NULL时)。  
函数中的 `act` 和 `oact` 是 `sigaction` 结构体类型指针，`sigaction` 结构体内容如下：
```
/* Structure describing the action to be taken when a signal arrives.  */
struct sigaction
  {
    /* Signal handler.  */
#if defined __USE_POSIX199309 || defined __USE_XOPEN_EXTENDED
    union
      {
	/* Used if SA_SIGINFO is not set.  */
	__sighandler_t sa_handler;
	/* Used if SA_SIGINFO is set.  */
	void (*sa_sigaction) (int, siginfo_t *, void *);
      }
    __sigaction_handler;
# define sa_handler	__sigaction_handler.sa_handler
# define sa_sigaction	__sigaction_handler.sa_sigaction
#else
    __sighandler_t sa_handler;
#endif
    /* Additional set of signals to be blocked.  */
    __sigset_t sa_mask;
    /* Special flags.  */
    int sa_flags;
    /* Restore handler.  */
    void (*sa_restorer) (void);
  };
```
成员说明：  
`sa_handler` ：指定的信号处理回调函数指针  
`sa_mask` ：设置进程的信号掩码  
`sa_flags` ：设置程序收到信号时的行为，可选值如下：  
`sa_restorer` ：已过时，不使用  

![sa_flags 选项](https://upload-images.jianshu.io/upload_images/22192996-4b7c4c14be14b91c.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

## 实 现
整体思路：将信号的处理逻辑放到服务器的主循环中，当触发信号处理函数时，通过管道将信号数据传给主循环，服务器的主循环通过I/O复用来监听管道上的可读事件。这样，将I/O事件和信号事件统一处理，即统一事件源

**1. 信号处理的回调函数**  
```
//Server.h
static void SignalHandler(int sig); //信号处理回调函数

//Server.cpp
void Server::SignalHandler(int sig)
{
    // 保留原有的errno，函数执行后恢复，以保证函数的可重入性
    int old_errno = errno;
    int msg = sig;
    send(s_pipeFd[1], reinterpret_cast<char *>(&msg), 1 , 0); //将信号写入管道，通知主循环
    errno = old_errno;
}
```
这里将 `SignalHandler()` 定义为静态成员函数，因为在类的成员函数中给 `sigaction->sa_handler` 赋值时需要静态成员函数或全局函数的指针。`s_pipeFd` 即定义的管道，用于将信号数据传给主循环。

**2. 设置信号处理回调函数**  
```
//Server.h
void AddSignal(int sig); //设置信号处理回调函数

//Server.cpp
void Server::AddSignal(int sig)
{
    struct sigaction sa;
    memset(&sa, '\0', sizeof(sa));
    sa.sa_handler = SignalHandler; //指定信号处理函数指针
    sa.sa_flags |= SA_RESTART; //设置收到信号时的行为
    sigfillset(&sa.sa_mask); //设置信号掩码
    if (sigaction(sig, &sa, nullptr) == -1) {
        cout << "Server: Add signal error!" << endl;
    }
}
```
**3. 创建管道，设置信号**  
```
bool Server::InitServer(const std::string &Ip, const int &Port)
{
    //...其他流程
    ret = socketpair(PF_UNIX, SOCK_STREAM, 0, s_pipeFd); //使用socketpair创建管道，并注册s_pipeFd[0]上可读写事件
    if (ret == -1) {
        cout << "Server: socketpair error!" << endl;
        return false;
    }
    SetNonblocking(s_pipeFd[1]);
    Addfd(m_epollFd, s_pipeFd[0]);

    AddSignal(SIGHUP); //控制终端挂起
    AddSignal(SIGCHLD); //子进程状态发生变化
    AddSignal(SIGTERM); //终止进程，即kill
    AddSignal(SIGINT); //键盘输入以中断进程 Ctrl+C
    //...其他流程
}
```
**4. 在主循环中，处理信号**  
```
void Server::Run()
{
    cout << "START SERVER!" << endl;
    epoll_event events[MAX_EVENT_NUMBER];
    while (!m_serverStop) {
        int ret = epoll_wait(m_epollFd, events, MAX_EVENT_NUMBER, -1); //epoll_wait
        if (ret < 0 && errno != EINTR) { //见 注
            cout << "Server: epoll error" << endl;
            break;
        }

        for (int i = 0; i < ret; ++i) {
            int sockfd = events[i].data.fd;
            if (sockfd == m_socketFd) { //新的socket连接
                //...
            } else if ((sockfd == s_pipeFd[0]) && (events[i].events & EPOLLIN)) { //处理信号
                char sigBuf[BUFFER_SIZE];
                ret = recv(s_pipeFd[0], sigBuf, sizeof(sigBuf), 0);
                if (ret <= 0) continue;
                else {
                    for (auto &item : sigBuf){ //逐个处理接收的信号
                        switch (item) {
                            case SIGHUP: {
                                cout << " SIGHUP: Terminal Hangup" << endl;
                            }
                            case SIGCHLD: {
                                cout << " SIGCHLD: The child process state changes" << endl;
                            }
                            case SIGTERM: {
                                cout << " SIGTERM: Kill" << endl;
                            }
                            case SIGINT: {
                                cout << " SIGINT: Ctrl+C" << endl;
                                m_serverStop = true; //安全的终止服务器主循环
                            }
                        }
                    }
                }
            } else if (events[i].events & EPOLLIN) { //可读事件
                //...
            } else {
                //...
            }
        }
    }
    //...
}
```
**注：** 当程序在执行处于阻塞状态的系统调用时接收到信号，并且为该信号设置了处理函数时，默认情况下会中断系统调用，并将 `error` 设置为 `EINTR`。所以代码中使用 `errno != EINTR` 来避免错误的判断，保证信号的接收和处理。

## 运行结果

![运行结果](https://upload-images.jianshu.io/upload_images/22192996-8ad083b0b2c00c51.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

通过 `kill -HUP`、`kill`、`Ctrl + C` 等命令，可以看到服务器对应的响应。

参考：《Linux高性能服务器编程》 游双 著  
具体代码，详见GitHub：[ChatRoomServer](https://github.com/cyh1998/ChatRoomServer)