#### 概 述
在Linux环境下获取毫秒级时间，我们会想到使用`gettimeofday()`(头文件`<sys/time.h>`)来获取。包括muduo网络库中日志模块打印毫秒级时间时，也使用了`gettimeofday()`。但是此函数已经[弃用](https://man7.org/linux/man-pages/man2/gettimeofday.2.html)，那我们应该如何高效的获取毫秒级时间？

#### 解决方案
在上文的Linux手册中提到，建议改用`clock_gettime(2)`，我们知道`clock_gettime()`精确到纳秒级，因此效率肯定远不及`gettimeofday()`。但是Linux提供了可以降低`clock_gettime()`精度并且提高效率的方式。  
先看下函数原型：
```
extern int clock_gettime (clockid_t __clock_id, struct timespec *__tp) __THROW;
```
`__clock_id`是时钟类型，基本的四种类型这里不再赘述。
为了避免上下文切换到内核，从而加快读取时钟的速度，新增了`CLOCK_MONOTONIC_COARSE`和`CLOCK_REALTIME_COARSE`(两者与`CLOCK_MONOTONIC`和`CLOCK_REALTIME`含义相同)。  
带有`_COARSE`变体的参数，可以使读取速度更快，精度也降为毫秒。这正好符合了我们的需求。

#### 实 现
```
#include <ctime>

timespec tsSturct{};
clock_gettime(CLOCK_REALTIME_COARSE, &tsSturct);
time_t seconds = tsSturct.tv_sec; //秒
int millisecond = tsSturct.tv_nsec / 1000000; //毫秒
```

#### 其 他  
**1. C++通用时间库**  
对于跨平台，C++标准库中提供了`std::chrono`来获取时间相关的函数，比如`std::chrono::system_clock::now()`，此函数一样精确到纳秒级。使用如下：
```
#include <chrono>

const auto p1 = std::chrono::system_clock::now();
std::time_t today_time = std::chrono::system_clock::to_time_t(p1);
```
C++20也进一步扩充了`std::chrono`，有关其深入的内容这里不再展开，大家有兴趣可以自己去学习。

**2. 效率问题**  
`std::chrono::system_clock::now()`的效率是远不如`clock_gettime()`(这里及下文提到的`clock_gettime()`均是配合`_COARSE`使用的)和`gettimeofday()`的。那`clock_gettime()`和`gettimeofday()`呢？
在《Linux多线程服务端编程》一书中，陈硕写到：
> 在x86-64平台上，gettimeofday(2)不是系统调用，而是在用户态实现的，没有上下文切换和陷入内核的开销。

因此本人在x86-64平台上进行测试，`clock_gettime()`比`gettimeofday()`快了75%左右。

参考：
[POSIX Clocks](https://access.redhat.com/documentation/en-us/red_hat_enterprise_linux_for_real_time/7/html/reference_guide/sect-posix_clocks#CLOCK_MONOTONIC_COARSE_and_CLOCK_REALTIME_COARSE)