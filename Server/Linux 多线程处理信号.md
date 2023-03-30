#### 概 述
利用自己封装的线程类，来实现用线程处理信号

#### 设置信号掩码
在之前的文章[Linux 信号](https://www.jianshu.com/p/10383d4ac963)中，我们利用`sigaction`结构体中的`sa_mask`来设置进程的信号掩码。此外，可以使用`sigprocmask()`来设置或查看进程的信号掩码，函数原型如下：
```
#include <signal.h>
int sigprocmask(int _how, const sigset_t* _set, sigset_t* _oset);
```
而在多线程环境下，我们需要使用pthread版本的`pthread_sigmask()`来设置线程信号掩码，函数原型如下：
```
#include <pthread.h>
#include <signal.h>
int pthread_sigmask(int how, const sigset_t* newmask, sigset_t* oldmask);
```
两个函数的参数意义一致，参数说明如下：  
`oldmask(_oset)`：若不为空，则输出原来的信号掩码  
`newmask(_set)`：指定新的信号掩码  
`how(_how)`：指定设置信号掩码的方式，可选值如下：  

![可选值](https://upload-images.jianshu.io/upload_images/22192996-f8c55ac148546319.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

#### 信号线程
在上一篇文章[Linux pthread封装](https://www.jianshu.com/p/a154747d3f2d)中，我们实现了对pthread的封装，将线程函数定义为`std::function`。但在实际使用中，无法给线程传递参数。所以进行了修改，实现的线程基类如下(若有更好的实现方式，欢迎大佬指点)：
```
//ThreadObject.h
#include <pthread.h>

class ThreadObject {
public:
    explicit ThreadObject();
    virtual ~ThreadObject();

private:
    pthread_t m_pthreadId;
    bool m_isStarted;
    bool m_isJoined;

    static void* entryFunc(void *obj);
    virtual void* run() = 0; //纯虚函数，在子类中实现

public:
    void start();
    void join();
    void cancel();
    bool started() const { return m_isStarted; }
};

//ThreadObject.cpp
#include <iostream>
#include "ThreadObject.h"

ThreadObject::ThreadObject() :
    m_pthreadId(0),
    m_isStarted(false),
    m_isJoined(false)
{

}

ThreadObject::~ThreadObject() {
    if (m_isStarted && !m_isJoined) {
        pthread_detach(m_pthreadId);
    }
}

void ThreadObject::start() {
    m_isStarted = true;
    if (pthread_create(&m_pthreadId, nullptr, entryFunc, static_cast<void*>(this))) {
        m_isStarted = false;
        std::cout << "ThreadObject: Create a new thread failed!" << std::endl;
    }
}

void ThreadObject::join() {
    if (m_isStarted && !m_isJoined) {
        m_isJoined = true;
        pthread_join(m_pthreadId, nullptr);
    }
}

void ThreadObject::cancel() {
    if (!pthread_cancel(m_pthreadId)) {
        m_isStarted = false;
    }
}

void* ThreadObject::entryFunc(void *obj) {
    ThreadObject* ptr = static_cast<ThreadObject*>(obj);
    ptr->run();
    return nullptr;
}
```
创建线程处理线程类，继承于线程基类，重写`run`函数，示例代码如下：
```
//SignalThread.h
#include <signal.h>
#include "ThreadObject.h"

class SignalThread : public ThreadObject {
public:
    explicit SignalThread();
    ~SignalThread() override;

    void setSignalSet(sigset_t *set); //设置信号掩码集

private:
    sigset_t *m_set; //信号掩码集
    void *run() override;
};

//SignalThread.cpp
#include <iostream>
#include "SignalThread.h"

SignalThread::SignalThread() = default;

SignalThread::~SignalThread() = default;

void SignalThread::setSignalSet(sigset_t *set) {
    m_set = set;
}

void *SignalThread::run() {
    int ret, sig;
    for (;;) {
        ret = sigwait(m_set, &sig);
        if (ret == 0) {
            std::cout << "SignalThread: Get signal: " << sig << std::endl;
        }
    }
}
```

#### 实 现
最后，在服务器类中创建信号处理线程并运行，核心代码如下：
```
sigset_t set;
sigemptyset(&set);
sigaddset(&set, SIGQUIT); //添加信号集
sigaddset(&set, SIGUSR1);
ret = pthread_sigmask(SIG_BLOCK, &set, nullptr); //设置信号掩码
if (ret != 0) {
    cout << "Server: pthread_sigmask error!" << endl;
    return false;
}
// m_sigThread为信号处理线程对象
m_sigThread.setSignalSet(&set); //设置信号集
m_sigThread.start(); //开启信号处理线程
```
参考：《Linux高性能服务器编程》 游双 著  
更多内容，详见GitHub：[ChatRoomServer](https://github.com/cyh1998/ChatRoomServer)
****
更新 3.19  
上文描述的无法给`std::function`传递参数的说法是有问题的，这里更正一下  
对于`std::function`对象，可以通过`bind`来将类中的成员函数以及入参转化为`function`并执行，示例代码如下：
```
#include <iostream>
#include <functional> //function bind

using namespace std;

function<void()> fun; //声明一个function类型，无入参，无返回值

class F {
public:
    void func(int a, int b) { //类的成员函数
        cout << a << " " << b << endl;
    }
};

int main()
{
    F f;
    // 类的成员函数需要使用bind，并且需要实例化对象，成员函数要加&
    fun = std::bind(&F::func, f, 1, 2); //传入参数
    fun(); //直接调用函数
    return 0;
}
```
**注：** 函数的返回值需要保持一致。