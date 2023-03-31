## 概 述
整理一下 shared_ptr 的相关技术以及其使用的注意项。

## 整 理
**1. 用 `make_shared` 创建智能指针**  
shared_ptr 内部包含一个托管对象的原始指针以及一个引用计数，因此直接使用 `new` 来创建一个 shared_ptr 需要两次内存分配：一个用于托管对象，另一个用于引用计数，并且两块内存空间是不连续的。而 `make_shared` 只需要分配一次内存，并在其中创建两个，并且没有暴露指向托管对象的原始指针：
```
// std::shared_ptr<int> p(new int);  改为：
std::shared_ptr<int> p = std::make_shared<int>();
```
因此 `make_shared` 更高效，对 CPU 缓存友好，且避免了内存泄漏的任何可能性。  
*缺点*  
延迟了内存释放的时间：由于 `make_shared` 只分配一次内存，所以将对象的生命周期与控制块(强引用、弱引用的信息)的生命周期捆绑到一起，原本强引用计数为零时就可以释放内存，现在变为了强引用、弱引用都为零时才能释放，延迟了内存释放的时间  

**2. 减少 shared_pt 的拷贝**  
正如上文所说，shared_ptr 的内存占用是原始指针的两倍，因此 shared_ptr 的拷贝开销比拷贝原始指针要高。    
- 使用 `std::move()` 来移动 shared_ptr 的所有权
```
// 当一个 shared_ptr 需要将其所有权转移给另外一个新的 shared_ptr 时
std::shared_ptr<int> p1 = std::make_shared<int>();
// 使用 std::move()，减少拷贝
std::shared_ptr<int> p2 = std::move(p1); //此时 p1 等于 nullptr !
```
- 以 const reference 方式传递  
```
void add(const shared_ptr<number>& pNumber);
void subtract(const shared_ptr<number>& pNumber);

// 传常引用(pass by const reference)，减少拷贝
void func(const int& num) 
{
    auto pNumber = std::make_shared<number>(num);
    add(pNumber);
    subtract(pNumber);
}
```

**3. enable_shared_from_this**  
当我们使用异步回调时，通常会需要传入当前类对象，或者使用类成员函数作为回调。例如：
```
class obj 
{
    // ...
};

void obj::Init()
{
    m_pAsyncTimer->SetCallback(std::bind(&obj::Func, this, std::placeholders::_1));
};
```
如果直接使用 `this` 来传入，无法保证在回调函数中操作的 `this` 对象依然有效。  
对此，我们可以使用 `enable_shared_from_this` 来使得当前对象 `this` 变为 shared_ptr，从而延长对象的生命周期，保证在异步回调时，对象依然有效。
```
class obj : public std::enable_shared_from_this<obj>  
{
    // ...
};

void obj::Init()
{
    m_pAsyncTimer->SetCallback(std::bind(&obj::Func, shared_from_this(), std::placeholders::_1));
};
```

*注：* 使用 `shared_from_this()` 需要对象在堆上创建，并且由 shared_ptr 管理其生命周期。因此 `shared_from_this()` 不能在构造函数中调用，因为在构造对象时，其还没有交给 shared_ptr 接管。   
此外，对于在类的成员函数中需要把当前类对象作为参数传递给其他函数时，如果直接传递 `this`，当前对象就会被多个 shared_ptr 管理，造成二次释放的错误。使用 `enable_shared_from_this` 配合 `shared_from_this()` 也可以很好的解决此问题。

**4. 弱回调**  
上文提到的 `shared_from_this()` 延长了对象的生命周期，如果我们不想额外延长对象的生命周期。例如针对*如果对象还活着，就调用它的回调函数，反之忽略*这样的需求，我们可以使用 weak_ptr 来传入 `std::function` ，这样就不会延长对象的生命周期。在回调时，尝试提升为 shared_ptr，如果成功，说明对象有效，那么执行回调；反之则忽略。  

**5. 自定义析构**  
shared_ptr 的构造函数允许传入自定义删除器 `Deleter`  
```
template< class Y, class Deleter >
shared_ptr( Y* ptr, Deleter d );
```
因此，shared_ptr 支持自定义析构动作。在继承关系中，使用 shared_ptr 管理，虚析构不再是必须的。  
需要注意的是，上文提到的 `make_shared` 构造智能指针不允许自定义删除器，删除器 `Deleter` 支持 `std::function` 对象
```
#include <memory>
#include <functional>

class Sample {
public:
    // ...
};

void deleter(Sample* x) {
    // ...
}

int main() {
    std::shared_ptr<Sample> p1(new Sample, deleter);

    std::shared_ptr<Sample> p2(new Sample, [](Sample* x) {
            // ...
        });

    std::shared_ptr<Sample> p3(new Sample, std::bind(deleter, std::placeholders::_1));

    return 0;
}
```

**6. 析构所在的线程**  
对象的析构是同步的，当指向对象的最后一个 shared_ptr 离开作用域时，对象就会在当前线程析构。这有可能会影响到关键线程的速度，对此可以用一个单独的线程来专门做析构。  
由于 shared_ptr<void> 可以持有任何对象并安全的释放，我们可以通过一个 `BlockingQueue<shared_ptr<void>>` (shared_ptr<void>的阻塞队列)把对象的析构都转移到专用线程，从而解放关键线程。  

**7. 循环引用**  
若A持有B的 shared_ptr，且B也持有A的 shared_ptr，此时会造成循环引用，导致A、B都无法释放。此时，我们可以使用 `weak_ptr`。  
`weak_ptr` 不会增加引用计数，因此可以打破 shared_ptr 的循环引用。通常在继承关系中的做法是父类持有子类的 shared_ptr，子类持有指向父类的 weak_ptr。

参考：  
《Linux多线程服务端编程》陈硕  
https://stackoverflow.com/questions/18301511  
https://stackoverflow.com/questions/712279  
https://zh.cppreference.com/w/cpp/memory/shared_ptr/make_shared  
