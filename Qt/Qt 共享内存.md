#### 概  述
Qt中提供了`QSharedMemory`类来实现共享内存相关的操作，本文介绍Qt中`QSharedMemory`类的常用函数以及具体的实现。  
头文件 `#include <QSharedMemory>`
#### 常用函数
**一、类的创建**  
1. 构造函数`QSharedMemory(const QString &key, QObject *parent = nullptr)`
key为共享内存的关键字，想要操作同一块共享内存需要key一致
```
QSharedMemory* m_shareMemory;
m_shareMemory= new QSharedMemory("SharedMemory_Key");
```
2. 构造函数`QSharedMemory(QObject *parent = nullptr)`
构造实例对象后，调用`setKey()`函数来设置关键字
```
QSharedMemory* m_shareMemory;
m_shareMemory = new QSharedMemory();
m_shareMemory->setKey("SharedMemory_Key");
```
注：只有设置了key，才可以调用`create()`和`attach()`函数。

**二、创建共享内存**  
```
//为传递给构造函数的key创建大小为size的共享内存段，成功返回true，反之false
bool QSharedMemory::create(int size, AccessMode mode = ReadWrite)
```
`size`：创建共享内存的大小
`mode`：内存的访问方式，默认为可读可写。`QSharedMemory`类定义一个枚举类变量`AccessMode`，指定了两种共享内存的访问方式：
- `QSharedMemory::ReadOnly`   只读方式访问共享内存
- `QSharedMemory::ReadWrite`  读写方式访问共享内存

**三、附加/分离共享内存**  

**附加**：将当前进程附加到以关键字为key的共享内存，默认的访问方式为可读可写。只有附加的共享内存才能读取其内存数据。
```
bool QSharedMemory::attach(AccessMode mode = ReadWrite)
```
**分离**：将以关键字为key的共享内存分离进程，分离后，进程将不再能访问共享内存。如果这是附加到共享内存段的最后一个进程，则共享内存段由系统释放，即内容被销毁。
```
bool QSharedMemory::detach()
```

**四、锁定/解锁共享内存**
```
bool QSharedMemory::lock()           //锁定共享内存
bool QSharedMemory::unlock()         //解锁共享内存
```
用法与常见的锁一样，访问共享内存之前锁定，访问结束后解锁。  
由于需要人为手动解锁，这里不建议使用这种方式，可以使用**RAII**机制的`lock_guard`来实现自动的加锁、解锁

**五、其他**
```
QString QSharedMemory::key() const //获取共享内存关键字
void QSharedMemory::setKey(const QString &key) //设置共享内存关键字
bool QSharedMemory::isAttached() const //判断共享内存的附加状态
void * QSharedMemory::data() //获取共享内存的内存地址
```
#### 实  现
**ShareMemA.h**
```
#ifndef SHAREMEMA_H
#define SHAREMEMA_H

#include <QObject>
#include <QSharedMemory>

class ShareMemA : public QObject
{
    Q_OBJECT
public:
    ShareMemA();
    virtual ~ShareMemA();

private:
    QSharedMemory* m_sharedMemmory;
    void sharedMemmory();
};

#endif // SHAREMEMA_H
```
**ShareMemA.cpp**
```
#include "ShareMemA.h"
#include <mutex>
#include <QDebug>
#include <QSharedMemory>

ShareMemA::ShareMemA():
    m_sharedMemmory(new QSharedMemory(this))
{
    m_sharedMemmory->setKey("SharedMemory_Key");
    sharedMemmory();
}

ShareMemA::~ShareMemA()
{

}

void ShareMemA::sharedMemmory()
{
    if (m_sharedMemmory->isAttached()) {
        if(!m_sharedMemmory->detach())
            qDebug() << "detach error";
        return;
    }

    if (m_sharedMemmory->create(1024)) {
        qDebug () <<"create sharedMemmory success";
        std::lock_guard<QSharedMemory> lock(*m_sharedMemmory);
        const char* data = (char*)m_sharedMemmory->data();
        data = "hello";
        memcpy(m_sharedMemmory->data(), data, strlen(data)*sizeof(char));
    }
}
```
**ShareMemB.h**
```
#ifndef SHAREMEMB_H
#define SHAREMEMB_H

#include <QObject>
#include <QSharedMemory>

class ShareMemB : public QObject
{
    Q_OBJECT
public:
    ShareMemB();
    virtual ~ShareMemB();

private:
    QSharedMemory* m_sharedMemmory;
    void sharedMemmory();
};

#endif // SHAREMEMB_H
```
**ShareMemB.h**
```
#include "ShareMemB.h"
#include <mutex>
#include <QDebug>
#include <QSharedMemory>

ShareMemB::ShareMemB():
    m_sharedMemmory(new QSharedMemory(this))
{
    m_sharedMemmory->setKey("SharedMemory_Key");
    sharedMemmory();
}

ShareMemB::~ShareMemB()
{

}

void ShareMemB::sharedMemmory()
{
    std::lock_guard<QSharedMemory> lock(*m_sharedMemmory);
    if (!m_sharedMemmory->attach()) {
        qDebug()<<"attach error!";
        return;
    }

    char *data = (char *)m_sharedMemmory->data();
    qDebug() << "read sharememmory data: " << data;

    m_sharedMemmory->detach();
}
```
#### 运行结果
![共享内存](https://upload-images.jianshu.io/upload_images/22192996-da74f6e58021aa5b.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
