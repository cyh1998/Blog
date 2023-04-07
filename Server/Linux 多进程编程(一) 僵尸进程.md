## 前 言
学习完服务器如何处理I/O事件、信号以及定时事件三大事件后，我们开始学习Linux服务器的多进程编程。

## 概 述
有关多进程基础的复制进程 `fork()` 以及替换进程 `exec()` 的相关内容，这里不再赘述。本文主要介绍**僵尸进程**的产生于处理

### 僵尸进程的产生
对于多进程程序，父进程需要跟踪子进程的退出状态，当父进程没有正确处理子进程的返回信息时，子进程都将停留在**僵尸态**，产生僵尸进程的两种情况：
- 子进程运行结束后，父进程依然在运行，还未读取其退出状态，子进程将处于僵尸态
- 父进程结束或异常退出，子进程依然在运行，此时，子进程由init进程接管，在子进程结束之前，其处于僵尸态

无论那种情况，处于僵尸态的子进程都会占据着内核资源。父进程应当避免僵尸进程的产生，或使僵尸态的子进程立即结束

### 僵尸进程的避免和处理
Linux中提供了 `wait()` 以及 `waitpid()` 函数，头文件 `#include <wait.h>`

**1. wait()**  
```
pid_t wait(int *stat_loc);
```
`wait` 函数将阻塞该进程，直到其某个子进程结束运行为止。函数将结束运行的子进程的退出状态存储于stat_loc参数指向的内存，并返回该子进程的PID。

**2. waitpid()**  
对于服务器而言，`wait` 函数的阻塞特性不满足我们的需要。`waitpid` 函数可以解决这个问题，函数原型如下：
```
pid_t waitpid(pid_t pid, int *stat_loc, int options);
```
`waitpid` 函数只等待pid指定的子进程，若pid值为-1，即与 `wait` 函数相同，stat_loc参数含义与 `wait` 函数的stat_loc参数相同。options参数指定  
`waitpid` 函数的行为，最常用的取值为 `WNOHANG`，即 `waitpid` 函数调用是非阻塞的。

**3. waitpid() 和 SIGCHLD 信号配合使用**  
在事件已经发生的情况下执行非阻塞调用才能提高程序的效率，因此，服务器程序应当在某个子进程退出后，在父进程调用 `waitpid` 函数。在之前的学习中，我们知道，当一个进程状态发生改变时，会给其父进程发生 `SIGCHLD` 信号。  
对于多进程服务器，我们在父进程中捕获 `SIGCHLD` 信号，并在信号处理函数中调用 `waitpid` 函数来彻底结束子进程，示例代码如下：
```
void Server::HandleSignal(const string &sigMsg) {
    for (auto &item : sigMsg){ //逐个处理接收的信号
        switch (item) {
            case SIGCHLD: { //SIGCHLD 信号处理
                cout << " SIGCHLD: The child process state changes" << endl;
                pid_t pid;
                int stat;
                while ((pid = waitpid(-1, &stat, WNOHANG)) > 0) {
                    // 对结束的子进程进行处理
                }
            }
            case SIGTERM: {
                cout << " SIGTERM: Kill" << endl;
                m_serverStop = true; //安全的终止服务器主循环
            }
            case SIGINT: {
                cout << " SIGINT: Ctrl+C" << endl;
                m_serverStop = true; //安全的终止服务器主循环
            }
            case SIGALRM: {
                // 用timeout来标记有定时任务
                // 先不处理，因为定时任务优先级不高，优先处理其他事件
                m_timeout = true;
                break;
            }
            //... 其他信号处理
        }
    }
}
```