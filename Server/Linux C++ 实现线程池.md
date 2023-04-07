## 概 述
在服务端程序中，并发的处理多客户任务时，为每一个客户任务创建一个线程(或进程)，这是不高效。因此引入了线程池(或进程池)来实现高效的并发处理。本文主要介绍线程池的实现。

## 实 现
线程池实现的核心是对任务队列的处理，线程池需要提供接口，供主线程往队列中添加任务，线程池中的线程则从队列中取任务并执行处理。因为涉及到线程安全，我们使用C++ 11提供的 `std::mutex` 和 `std::unique_lock` 来保证线程安全，以及 `std::condition_variable` 来通知等待的线程(有关锁和条件变量的使用，这里不再赘述)。实现方法如下：

**1. 模板类线程池**  
```
#include <iostream>
#include <vector>
#include <queue>
#include <pthread.h>
#include <mutex>
#include <condition_variable>

template<typename T>
class ThreadPool {
public:
    explicit ThreadPool(int threadNumber = 5, int maxRequests = 10000);
    ~ThreadPool();

    bool append(T *request); //添加任务接口

private:
    static void *entryFunc(void *arg);
    void run();

private:
    int m_threadNumber; //线程数
    int m_maxRequests; //最大任务数
    std::queue<T *> m_workQueue; //任务队列
    std::mutex m_Mutex; //互斥量
    std::condition_variable m_CondVar; //条件变量
    bool m_stop; //线程池是否执行
};

template<typename T>
ThreadPool<T>::ThreadPool(int threadNumber, int maxRequests) :
    m_threadNumber(threadNumber),
    m_maxRequests(maxRequests),
    m_stop(false)
{
    if (m_threadNumber <= 0 || m_threadNumber > m_maxRequests) throw std::exception();

    for (int i = 0; i < m_threadNumber; ++i) {
        pthread_t pid;
        if (pthread_create(&pid, nullptr, entryFunc, static_cast<void*>(this)) == 0) { //创建线程
            std::cout << "ThreadPool: Create " << i + 1 << " thread" << std::endl;
            pthread_detach(pid); //分离线程
        }
    }
}

template<typename T>
ThreadPool<T>::~ThreadPool(){
    {
        std::unique_lock<std::mutex> lock(m_Mutex);
        m_stop = true;
    }
    m_CondVar.notify_all(); //通知所有线程停止
};

template<typename T>
bool ThreadPool<T>::append(T *request) {
    if (m_workQueue.size() > m_maxRequests) {
        std::cout << "ThreadPool: Work queue is full" << std::endl;
        return false;
    }
    {
        std::unique_lock<std::mutex> lock(m_Mutex);
        m_workQueue.emplace(request);
    }
    m_CondVar.notify_one(); //通知线程
    return true;
}

template<typename T>
void *ThreadPool<T>::entryFunc(void *arg) {
    ThreadPool *ptr = static_cast<ThreadPool *>(arg);
    ptr->run();
    return nullptr;
}

template<typename T>
void ThreadPool<T>::run() {
    std::unique_lock<std::mutex> lock(m_Mutex);
    while (!m_stop) {
        m_CondVar.wait(lock);
        if (!m_workQueue.empty()) {
            T *request = m_workQueue.front();
            m_workQueue.pop();
            if (request) request->process();
        }
    }
}
```
此线程池中的模板类是任务类，需要有执行函数 `process()`。此线程池适用于执行统一的任务。

**2. 函数对象线程池**  
```
//FuncThreadPool.h
#include <iostream>
#include <functional>
#include <vector>
#include <queue>
#include <thread>
#include <mutex>
#include <condition_variable>


class FuncThreadPool {
public:
    typedef std::function<void(void)> Task;

    explicit FuncThreadPool(int threadNumber = 5, int maxRequests = 10000);
    ~FuncThreadPool();

    bool append(Task task); //添加任务接口

private:
    static void *entryFunc(void *arg);
    void run();

private:
    int m_threadNumber; //线程数
    int m_maxRequests; //最大任务数
    std::queue<Task> m_workQueue; //任务队列
    std::mutex m_Mutex; //互斥量
    std::condition_variable m_CondVar; //条件变量
    bool m_stop; //线程池是否执行
};

//FuncThreadPool.cpp
#include "FuncThreadPool.h"

FuncThreadPool::FuncThreadPool(int threadNumber, int maxRequests) :
    m_threadNumber(threadNumber),
    m_maxRequests(maxRequests),
    m_stop(false)
{
    if (m_threadNumber <= 0 || m_threadNumber > m_maxRequests) throw std::exception();

    for (int i = 0; i < m_threadNumber; ++i) {
        std::cout << "FuncThreadPool: Create " << i + 1 << " thread" << std::endl;
        std::thread([this] { FuncThreadPool::entryFunc(this); }).detach(); //创建并分离线程
    }
}

FuncThreadPool::~FuncThreadPool() {
    {
        std::unique_lock<std::mutex> lock(m_Mutex);
        m_stop = true;
    }
    m_CondVar.notify_all(); //通知所有线程停止
}

bool FuncThreadPool::append(FuncThreadPool::Task task) {
    if (m_workQueue.size() > m_maxRequests) {
        std::cout << "FuncThreadPool: Work queue is full" << std::endl;
        return false;
    }
    {
        std::unique_lock<std::mutex> lock(m_Mutex);
        m_workQueue.emplace(task);
    }
    m_CondVar.notify_one(); //通知线程
    return true;
}

void *FuncThreadPool::entryFunc(void *arg) {
    FuncThreadPool *ptr = static_cast<FuncThreadPool *>(arg);
    ptr->run();
    return nullptr;
}

void FuncThreadPool::run() {
    std::unique_lock<std::mutex> lock(m_Mutex);
    while (!m_stop) {
        m_CondVar.wait(lock);
        if (!m_workQueue.empty()) {
            Task task = m_workQueue.front();
            m_workQueue.pop();
            task();
        }
    }
}
```
此线程池中的任务是任意可调用的函数对象，适用于执行不同的任务。