## 前 言
之前在学习《Linux高性能服务器编程》时，书中实现的各类定时器都是通过 `alarm()` 发送信号，配合服务端处理 `SIGALRM` 信号来实现定时机制。   
陈硕在《Linux多线程服务端编程》中提到：在多线程程序中，第一原则是不要使用信号。muduo源码中通过Linux提供的 `timerfd` 将时间变成了一个文件描述符，该描述符在定时器超时的那一刻变为可读，可以很方便的融入到select/poll/epoll中，统一事件源。  
本文介绍muduo中定时器的相关实现

## 实 现
**一、计时**  
muduo中使用 `gettimeofday()` 来获取当前时间，支持微秒级计时，有关的系统调用如下：
```
#include <sys/time.h>

struct timeval {
    long tv_sec;   // 秒
    long tv_usec;  // 微秒
};

int gettimeofday(struct timeval* tv, struct timezone* tz);
```
`gettimeofday()` 将当前时间存放在于 `tv`，时区信息存放于 `tz`。成功返回0，失败返回-1

**二、timerfd**  
通过 `timerfd` 系列函数可将时间变成一个文件描述符，并设置超时时间。muduo中使用的系统调用如下：
```
#include <sys/timerfd.h>

struct timespec { //timespec 结构体
    time_t tv_sec;
    long   tv_nsec;
};

struct itimerspec { //itimerspec 结构体
   struct timespec it_interval; //时间间隔
   struct timespec it_value; //超时时间
};

int timerfd_create(int clockid, int flags);
int timerfd_settime(int fd, int flags, 
                    const struct itimerspec *new_value,
                    struct itimerspec *old_value);
```
`timerfd_create()` 创建一个用于定时器的文件描述符，参数说明：  
`clockid` 可选值为：
- `CLOCK_REALTIME` ：系统实时时间。若用户更改系统时间，则其随之而改变
- `CLOCK_MONOTONIC` ：从系统启动那一刻开始计时，不受用户更改系统时间的影响

`flags` 可选值为：
- `TFD_NONBLOCK` ：非阻塞
- `TFD_CLOEXEC` ：调用exec时自动close

`timerfd_settime()` 用于设置timerfd的超时时间，参数说明：  
`fd` ：`timerfd_create()` 返回的文件句柄  
`flags` ：1表示绝对时间；0表示相对时间  
`new_value` ：需要设置的超时时间和间隔  
`old_value` ：旧的超时时间  

**三、定时器**  
muduo中用四个类 `Timestamp`、`Timer`、`TimerId`、`TimerQueue` 来实现定时器。

**1. Timestamp**  
`Timestamp` 是对时间戳的封装，包括获取当前时间、添加时间、是否超时等等；重载了 `<`、`==` 等运算符号；以及上文对 `timerfd` 系统调用的封装。源码比较简单，这里就不展示了。

**2. Timer**  
`Timer` 是定时器的封装，保存回调函数、超时时间 `Timestamp`、定时间隔、是否重复等等，源码如下：
```
class Timer : noncopyable
{
 public:
  Timer(TimerCallback cb, Timestamp when, double interval)
    : callback_(std::move(cb)),
      expiration_(when),
      interval_(interval),
      repeat_(interval > 0.0),
      sequence_(s_numCreated_.incrementAndGet())
  { }

  void run() const //入口函数
  {
    callback_(); //触发回调
  }

  Timestamp expiration() const  { return expiration_; }
  bool repeat() const { return repeat_; }
  int64_t sequence() const { return sequence_; }

  void restart(Timestamp now); //重新计算超时时间

  static int64_t numCreated() { return s_numCreated_.get(); }

 private:
  const TimerCallback callback_; //回调函数
  Timestamp expiration_; //超时时间
  const double interval_; //间隔
  const bool repeat_; //是否重复
  const int64_t sequence_; //序列号，即唯一ID

  static AtomicInt64 s_numCreated_; //原子计数
};
```
当重复定时器触发超时回调后，通过 `Timer::restart()` 来重新计算超时时间：
```
void Timer::restart(Timestamp now)
{
  if (repeat_)
  { //重复定时器，添加间隔到超时时间
    expiration_ = addTime(now, interval_);
  }
  else
  { //反之，超时时间为 0
    expiration_ = Timestamp::invalid();
  }
}
```

**3. TimerId**  
`TimerId` 保存了定时器 `Timer` 和其唯一的ID。ID在向容器中添加定时器时返回，并且可以通过此ID来取消对应的定时器。源码比较简单，这里就不展示了。

**4. TimerQueue**  
`TimerQueue` 是定时器容器类，用于统一管理所有定时任务。muduo采用 `std::set` (红黑树)来存储，为了处理同一时间会有多个定时任务到来，muduo将超时时间 `Timestamp` 和定时器指针 `Timer*`，组成 `std::pair<Timestamp, Timer*>` 作为 `std::set` 的key。类定义的源码如下：
```
class TimerQueue : noncopyable
{
 public:
  explicit TimerQueue(EventLoop* loop);
  ~TimerQueue();

  // 添加定时器接口，函数返回定时器唯一ID
  TimerId addTimer(TimerCallback cb, //回调函数
                   Timestamp when,   //超时时间
                   double interval); //时间间隔

  // 取消定时器，通过addTimer()返回的定时器唯一ID
  void cancel(TimerId timerId);

 private:
  // 定时器容器的相关数据结构
  typedef std::pair<Timestamp, Timer*> Entry;
  typedef std::set<Entry> TimerList;

  // 用于记录活跃定时器和取消定时器容器的相关数据结构
  typedef std::pair<Timer*, int64_t> ActiveTimer;
  typedef std::set<ActiveTimer> ActiveTimerSet;

  // 由EventLoop::runInLoop()触发添加定时器
  void addTimerInLoop(Timer* timer);
  // 由EventLoop::runInLoop()触发取消定时器
  void cancelInLoop(TimerId timerId);

  // 定时器超时的处理函数
  void handleRead();
  // 从定时器列表中找到所有到期的定时器
  std::vector<Entry> getExpired(Timestamp now);
  // 将重复的超时定时器重新添加到定时器列表timers_中
  void reset(const std::vector<Entry>& expired, Timestamp now);

  // 添加定时器到容器
  bool insert(Timer* timer);

  EventLoop* loop_; //所属的事件循环
  const int timerfd_; //timerfd_create()创建的文件描述符
  Channel timerfdChannel_; //用于监听timerfd的Channel
  TimerList timers_; //定时器列表

  // 用于取消定时器，通过唯一ID找到对应的定时器
  ActiveTimerSet activeTimers_; //活跃定时器记录
  bool callingExpiredTimers_;
  ActiveTimerSet cancelingTimers_; //取消定时器记录
};
```
****
首先，看下 `TimerQueue` 提供的添加定时器的接口函数 `addTimer()` ：
```
TimerId TimerQueue::addTimer(TimerCallback cb,
                             Timestamp when,
                             double interval)
{
  // 创建定时器类
  Timer* timer = new Timer(std::move(cb), when, interval);
  loop_->runInLoop(
      std::bind(&TimerQueue::addTimerInLoop, this, timer));
  // 返回定时器唯一ID
  return TimerId(timer, timer->sequence());
}
```
`addTimer()` 通过所属的事件循环调用 `EventLoop::runInLoop()` 触发添加定时器函数 `TimerQueue::addTimerInLoop()`。关于 `EventLoop::runInLoop()`，参考 [muduo源码学习(四) 实现TCP网络库(中)](./muduo源码学习(四)%20实现TCP网络库(中).md)。  
这里继续看 `addTimerInLoop()` 函数：
```
void TimerQueue::addTimerInLoop(Timer* timer)
{
  loop_->assertInLoopThread();
  bool earliestChanged = insert(timer);

  //判断是否需要更新 timerfd 的超时时间
  if (earliestChanged)
  {
    resetTimerfd(timerfd_, timer->expiration());
  }
}
```
`addTimerInLoop()` 实际就是调用了 `TimerQueue::insert()` 来添加定时器到容器中。如果定时器容器为空或者新添加的定时器位于容器的顶部，即超时时间比当前小，就通过 `resetTimerfd()` ( `timerfd_settime()` 的封装)来更新超时时间。`insert()` 源码如下：
```
bool TimerQueue::insert(Timer* timer)
{
  loop_->assertInLoopThread();
  assert(timers_.size() == activeTimers_.size());
  bool earliestChanged = false;
  Timestamp when = timer->expiration();
  TimerList::iterator it = timers_.begin();
  if (it == timers_.end() || when < it->first) //定时器容器为空 或 超时时间比当前小
  {
    earliestChanged = true;
  }
  {
    // 添加到定时器列表 timers_ 中
    std::pair<TimerList::iterator, bool> result
      = timers_.insert(Entry(when, timer));
    assert(result.second); (void)result;
  }
  {
    // 添加到活跃定时器列表 activeTimers_ 中，用于取消定时器是查找
    std::pair<ActiveTimerSet::iterator, bool> result
      = activeTimers_.insert(ActiveTimer(timer, timer->sequence()));
    assert(result.second); (void)result;
  }

  assert(timers_.size() == activeTimers_.size());
  return earliestChanged;
}
```
定时器添加的流程就是：`addTimer()` => `EventLoop::runInLoop()` => `addTimerInLoop()` => `insert()`。  
但是使用时，并不直接调用 `addTimer()`，`EventLoop` 对其做了更加完善的封装，提供给用户，接口如下：
```
TimerId EventLoop::runAt(Timestamp time, TimerCallback cb) { /... }
TimerId EventLoop::runAfter(double delay, TimerCallback cb) { /... }
TimerId EventLoop::runEvery(double interval, TimerCallback cb) { /... }
```

****
当 `timerfd` 超时，调用定时器超时的处理函数 `TimerQueue::handleRead()`，此函数在容器构造时通过 `Channel` 注册，源码如下：
```
TimerQueue::TimerQueue(EventLoop* loop)
  : loop_(loop),
    timerfd_(createTimerfd()),
    timerfdChannel_(loop, timerfd_),
    timers_(),
    callingExpiredTimers_(false)
{
  timerfdChannel_.setReadCallback(
      std::bind(&TimerQueue::handleRead, this));
  timerfdChannel_.enableReading();
}
```
`handleRead()` 通过 `TimerQueue::getExpired()` 从定时器列表中找到所有到期的定时器，然后依次执行其回调函数，并将重复的超时定时器重新添加到定时器列表 `timers_` 中，源码如下：
```
void TimerQueue::handleRead()
{
  loop_->assertInLoopThread();
  Timestamp now(Timestamp::now());
  readTimerfd(timerfd_, now);
  
  // 从定时器列表中找到所有到期的定时器
  std::vector<Entry> expired = getExpired(now);

  callingExpiredTimers_ = true;
  cancelingTimers_.clear();
  for (const Entry& it : expired)
  {
    it.second->run(); //依次执行其回调函数
  }
  callingExpiredTimers_ = false;
  
  // 将重复的超时定时器重新添加到定时器列表 timers_ 中
  reset(expired, now);
}

std::vector<TimerQueue::Entry> TimerQueue::getExpired(Timestamp now)
{
  assert(timers_.size() == activeTimers_.size());
  std::vector<Entry> expired;
  Entry sentry(now, reinterpret_cast<Timer*>(UINTPTR_MAX));
  // 利用 lower_bound() 找到第一个大于等于参数的位置，返回迭代器
  TimerList::iterator end = timers_.lower_bound(sentry);
  assert(end == timers_.end() || now < end->first);
  // 构造vector<Entry>
  std::copy(timers_.begin(), end, back_inserter(expired));
  timers_.erase(timers_.begin(), end);

  for (const Entry& it : expired)
  {
    ActiveTimer timer(it.second, it.second->sequence());
    size_t n = activeTimers_.erase(timer);
    assert(n == 1); (void)n;
  }

  assert(timers_.size() == activeTimers_.size());
  return expired;
}

void TimerQueue::reset(const std::vector<Entry>& expired, Timestamp now)
{
  Timestamp nextExpire;

  for (const Entry& it : expired)
  {
    ActiveTimer timer(it.second, it.second->sequence());
    if (it.second->repeat() //判断是否时重复定时器
        && cancelingTimers_.find(timer) == cancelingTimers_.end()) //判断用户是否取消了这个定时任务
    {
      it.second->restart(now); //重新计算超时时间
      insert(it.second); //重新添加到容器中
    }
    else
    {
      delete it.second;
    }
  }

  // 计算下次timerfd被激活的时间
  if (!timers_.empty())
  {
    nextExpire = timers_.begin()->second->expiration();
  }

  // 设置超时时间
  if (nextExpire.valid())
  {
    resetTimerfd(timerfd_, nextExpire);
  }
}
```
****
最后，梳理下muduo中定时器工作的整个流程：  
1. 利用 `timerfd` 将定时器变成文件描述符进行监听
2. 通过 `EventLoop` 封装的接口，即调用 `addTimer()` 添加定时器
3. 当超时时间到达，`timerfd` 变为可读，对应的 `Channel` 调用超时函数 `handleRead()`
4. 超时函数将找出容器中所有超时的定时器，通过入口函数 `run()` 依次调用用户注册的回调函数
5. 对于重复的定时器，重新添加到容器中

## 补 充
陈硕在书中提到，目前 `TimerQueue` 的实现有一个不理想的地方，即 `Timer` 是用裸指针来管理的，必要的时候需要手动delete。他还提到用 `shared_pte` 小题大做，可以使用C++11提供的 `unique_ptr` 来实现。  
于是，我在实践中将 `Timer*` 改为了 `unique_ptr<Timer>` 来管理，即
```
typedef std::pair<Timestamp, std::unique_ptr<Timer>> Entry;
```
然而，在函数 `TimerQueue::getExpired()` 中，当拷贝超时定时器到 `vector` 中时，出现了问题，代码如下：
```
// 使用默认的方式
std::copy(m_timers.begin(), end, back_inserter(expired));

// 直接构造
std::vector<Entry> expired(m_timers.begin(), end);

// 使用assign
std::vector<Entry> expired;
expired.assign(m_timers.begin(), s.end());
```
以上的方式，都无法通过编译，原因如下：  
`std::unique_ptr` 是不能 cpoy 的，只能 move，因此包含 `std::unique_ptr` 的 `std::pair<>` 也是如此。但由于 `std::set` 的 key 是不可修改的，只能 copy。于是当执行上面的语句时，编译器都会默认使用 copy pair 的方式去执行，最终失败了。   
本人最后还是使用 `shared_pte` 来改造了 `Timer*`，由于本人能力有限，不知道是否有更好的实现方式。  

本人参考muduo实现的网络库，有兴趣，详见github [NetLib](https://github.com/cyh1998/NetLib)  
参考：  
《Linux多线程服务端编程》陈硕 著  
[muduo源码](https://github.com/chenshuo/muduo)  