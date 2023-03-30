#### 概 述
C++11提供了`std::thread`类来实现多线程，但是对于Linux上的服务器编程，C++11的`std::thread`使用不便且没有附加值(参考知乎 [陈硕](https://www.zhihu.com/question/36236334/answer/98392431) [徐辰](https://www.zhihu.com/question/36236334/answer/98422670)的回答)。本文介绍对`pthread`的简单封装。

#### 函数接口
下面介绍线程相关的常用API，头文件`pthread.h`  
**1. pthread_create**  
`pthread_create()`用于创建一个线程，成功返回0，失败返回错误码。函数原型：
```
int pthread_create(pthread_t *thread, const pthread_attr_t *attr,
                            void *(*start_routine) (void *), void *arg);
```
参数说明：  
`thread`：新线程的标识符  
`attr`：用于设置线程属性，线程拥有众多属性(这里不再赘述，自行百度学习)  
`start_routine`：指定线程运行的函数  
`arg`：指定运行函数的参数  

**2. pthread_join**  
`pthread_join()`用于阻塞当前线程，等待目标线程结束，成功返回0，失败返回错误码。函数原型：
```
int pthread_join(pthread_t thread, void** retval);
```
参数说明：  
`thread`：目标线程的标识符  
`retval`：目标线程返回的退出信息  

函数可能返回的错误码如下图：

![错误码](https://upload-images.jianshu.io/upload_images/22192996-b2b2876bd87d0cbe.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

**3. pthread_cancel**  
`pthread_cancel()`用于异常终止目标线程，即取消线程，成功返回0，失败返回错误码。函数原型：
```
int pthread_cancel(pthread_t thread);
```
**注：** 接收到取消请求的目标线程`thread`可以决定是否允许被取消以及如何取消

**4. pthread_detach**  
`pthread_detach()`用于让主进程不再阻塞等待目标线程`thread`结束才返回，目标线程进入后台运行，主进程继续执行自身的工作，即分离线程。当主进程结束时，目标线程也会被结束并清理资源。函数原型：
```
int pthread_detach(pthread_t thread);
```

**5. pthread_exit**  
`pthread_exit()`用于结束目标进程`thread`，函数原型：
```
void pthread_exit(void* retval);
```
函数通过`retval`参数向线程的回收者传递其退出信息。它执行完之后不会返回到调用者，而且永远不会失败。

#### 实 现
调用`pthread_create()`时，传入的线程运行函数必须是静态函数，而静态函数在使用类成员变量时非常不方便。所以我们将**this指针**作为入参传递给静态函数，并将运行函数定义为`std::function`，这样在静态函数中就可以方便调用。示例代码如下：
```
//ThreadObject.h
#include <functional>
#include <pthread.h>

class ThreadObject {
public:
    typedef std::function<void ()> ThreadFunc;

    explicit ThreadObject(ThreadFunc);
    ~ThreadObject();

    void start();
    void join();
    void cancel();
    bool started() const { return m_isStarted; }

    static void* run(void *obj);

private:
    pthread_t m_pthreadId;
    bool m_isStarted;
    bool m_isJoined;
    ThreadFunc m_func;
};

//ThreadObject.cpp
#include <iostream>
#include "ThreadObject.h"

ThreadObject::ThreadObject(ThreadFunc func) :
    m_pthreadId(0),
    m_isStarted(false),
    m_isJoined(false),
    m_func(std::move(func))
{

}

ThreadObject::~ThreadObject() {
    if (m_isStarted && !m_isJoined) {
        pthread_detach(m_pthreadId);
    }
}

void ThreadObject::start() {
    m_isStarted = true;
    if (pthread_create(&m_pthreadId, nullptr, run, static_cast<void*>(this))) {
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

void* ThreadObject::run(void *obj) {
    ThreadObject* ptr = static_cast<ThreadObject*>(obj);
    ptr->m_func();
    delete ptr;
    return nullptr;
}
```
参考 [陈硕 muduo 源码](https://github.com/chenshuo/muduo)