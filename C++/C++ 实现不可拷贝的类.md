#### 概 述
在C++中，经常需要实现不可拷贝的类，本文介绍如何实现不可拷贝的类。

#### 实 现
**1. 将拷贝构造函数和赋值函数定义为私有函数**
**Boost库**采用了这样的方式，提供了`noncopyable`基类，来实现不可拷贝的类。源码如下：
```
namespace boost {
//  Private copy constructor and copy assignment ensure classes derived from
//  class noncopyable cannot be copied.
//  Contributed by Dave Abrahams

namespace noncopyable_  // protection from unintended ADL
{
  class noncopyable
  {
   protected:
      noncopyable() {}
      ~noncopyable() {}
   private:  // emphasize the following members are private
      noncopyable( const noncopyable& );
      const noncopyable& operator=( const noncopyable& );
  };
}

typedef noncopyable_::noncopyable noncopyable;
} // namespace boost
```
我们可以引入Boost库，直接继承`noncopyable`基类。当然，也可以参考Boost库，自己实现一个不可拷贝的基类。

**2. 利用 C++ 的 `=delete`**
`=delete`能够禁止编译器生成默认函数，因此，只需要为拷贝构造函数和赋值函数添加`=delete`，示例如下：
```
class Player {
public:
    Player();
    ~Player();

    Player(const Player&) = delete ;
    const Player& operator=(const Player&) = delete ;
}
```
当然，还是建议使用基类的方式，来实现类的不可拷贝。这样更加方便，基类如下：
```
class Noncopyable
{
public:
    Noncopyable(const Noncopyable&) = delete;
    const Noncopyable& operator=(const Noncopyable&) = delete;

protected:
    Noncopyable() = default;
    ~Noncopyable() = default;
};
```