## 概 述
在多线程服务端程序中，阻塞队列是常用的数据结构。本文介绍使用C++来实现阻塞队列的模板。

## 实 现
所谓 **阻塞队列**，即当队列为空时，弹出操作就会被阻塞，直到队列不再为空时才会被唤醒；当队列已满时，插入操作就会被阻塞，直到队列不再为满时才会被唤醒。  
因此，我们使用一个互斥量 `std::mutex` 和两个条件变量 `std::condition_variable` 来实现阻塞和唤醒。  
内部数据结构使用 `std::deque`，即双端队列。之所以使用 `std::deque`，说明如下：  
- 相比于 `std::vector` ：`deque` 容器的头部操作效率更高，且可以动态分配连续空间并链接，无需像`vector`容器那些，重新分配空间并复制内容。
- 相比于 `std::queue` ：`deque` 容器支持对头部和尾部的插入删除操作，比 `queue` 容器(先进先出)更加灵活，且可以实现类似于优先级的队列，即优先级高的任务，插入队头；优先级低的，插入队尾。

更多有关容器之间的差异和原理，推荐大家看 [STL源码剖析](https://book.douban.com/subject/1110934/)

## 代 码
示例代码如下：
```
#include <mutex>
#include <deque>
#include <condition_variable>

template<typename T>
class BlockQueue {
public:
    explicit BlockQueue (int MaxSize = 1000);
    ~BlockQueue();

    bool Empty();
    bool Full();
    int Size();
    int Capacity();

    T Front();
    T Back();
    void PushFront(const T &item);
    void PushBack(const T &item);
    bool PopFront(T &item);
    bool PopBack(T &item);
    void Flush();
    void Close();

private:
    bool m_isClose;
    int m_maxSize;
    std::deque<T> m_deque;
    std::mutex m_mutex;
    std::condition_variable m_noFullCondVar;
    std::condition_variable m_noEmptyCondVar;
};

template<typename T>
BlockQueue<T>::BlockQueue(int MaxSize) :
    m_isClose(false),
    m_maxSize(MaxSize)
{
}

template<typename T>
BlockQueue<T>::~BlockQueue() {
    Close();
}

template<typename T>
void BlockQueue<T>::Close() {
    {
        std::unique_lock<std::mutex> locker(m_mutex);
        m_deque.clear();
        m_isClose = true;
    }
    m_noFullCondVar.notify_all();
    m_noEmptyCondVar.notify_all();
}

template<typename T>
bool BlockQueue<T>::Empty() {
    std::unique_lock<std::mutex> locker(m_mutex);
    return m_deque.empty();
}

template<typename T>
bool BlockQueue<T>::Full() {
    std::unique_lock<std::mutex> locker(m_mutex);
    return m_deque.empty() >= m_maxSize;
}


template<typename T>
int BlockQueue<T>::Size() {
    std::unique_lock<std::mutex> locker(m_mutex);
    return m_deque.size();
}

template<typename T>
int BlockQueue<T>::Capacity() {
    std::unique_lock<std::mutex> locker(m_mutex);
    return m_maxSize;
}

template<typename T>
T BlockQueue<T>::Front() {
    std::unique_lock<std::mutex> locker(m_mutex);
    return m_deque.front();
}

template<typename T>
T BlockQueue<T>::Back() {
    std::unique_lock<std::mutex> locker(m_mutex);
    return m_deque.back();
}

template<typename T>
void BlockQueue<T>::PushFront(const T &item) {
    std::unique_lock<std::mutex> locker(m_mutex);
    while (m_deque.size() >= m_maxSize) {
        m_noFullCondVar.wait(locker);
    }
    m_deque.emplace_front(item);
    m_noEmptyCondVar.notify_one();
}

template<typename T>
void BlockQueue<T>::PushBack(const T &item) {
    std::unique_lock<std::mutex> locker(m_mutex);
    while (m_deque.size() >= m_maxSize) {
        m_noFullCondVar.wait(locker);
    }
    m_deque.emplace_back(item);
    m_noEmptyCondVar.notify_one();
}

template<typename T>
bool BlockQueue<T>::PopFront(T &item) {
    std::unique_lock<std::mutex> locker(m_mutex);
    while (m_deque.empty()) {
        m_noEmptyCondVar.wait(locker);
        if (m_isClose) return false;
    }

    item = m_deque.front();
    m_deque.pop_front();
    m_noFullCondVar.notify_one();
    return true;
}

template<typename T>
bool BlockQueue<T>::PopBack(T &item) {
    std::unique_lock<std::mutex> locker(m_mutex);
    while (m_deque.empty()) {
        m_noEmptyCondVar.wait(locker);
        if (m_isClose) return false;
    }

    item = m_deque.back();
    m_deque.pop_back();
    m_noFullCondVar.notify_one();
    return true;
}

template<typename T>
void BlockQueue<T>::Flush() {
    std::unique_lock<std::mutex> locker(m_mutex);
    m_noEmptyCondVar.notify_one();
}
```
参考 ：  
《Linux多线程服务端编程》(陈硕 著)  
[C++实现的高性能WEB服务器](https://github.com/markparticle/WebServer/blob/master/code/log/blockqueue.h)