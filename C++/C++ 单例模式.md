##### 前 言
我们在写程序时，经常需要实现某一个类对象能够全局访问，但又需要保证其唯一的设计。这就需要使用常说的单例模式来实现。

##### 实 现
《设计模式》书中给出了一种实现方式，即定义一个单例类，使用类的私有静态指针变量指向类的唯一实例，并用一个公有的静态方法来获取该实例。
根据书中的实现方式，示例如下：
```
class CSingleton
{
private:
	CSingleton()   //将构造函数设为私有
	static CSingleton *m_pInstance;  //私有静态指针变量，指向唯一实例
public:
	static CSingleton* GetInstance()  //公有静态方法，获取该实例
	{
		if(m_pInstance == nullptr)  //判断是否第一次调用
			m_pInstance = new CSingleton();
		return m_pInstance;
	}
};
```
因为类的构造函数是私有的，所以外部任何创建实例的尝试都将失败，访问实例的唯一方法，即通过公有静态方法`GetInstance()`。`GetInstance()`返回的实例是当这个函数首次被访问时创建的。这就是常说的**懒汉模式**。

以上的实现方式满足了我们的需求，但是存在着诸多问题，比如：该实例如何删除？
我们可以在程序结束时调用GetInstance()并delete掉返回的实例指针。但是这样的操作很容易出错，因为我们很难保证在程序执行的最后删除；也不能保证删除掉实例后，程序不再调用创建实例。

我们知道，系统会在程序结束后释放所有全局变量并析构所有类的静态对象。利用这一个特性，我们可以在类中设计一个静态成员变量，在其析构函数中删除唯一实例。在程序结束时，系统将会调用这个静态成员变量的析构函数，从而帮助我们自动的删除唯一实例，且不会出现人为的意外失误。如下面实例中的CGarbo类：
```
class CSingleton
{
//经典单例(Singleton)设计模式，只创建一个对象，并且自动释放
private:
	CSingleton(void)
	static CSingleton *m_pInstance;
    //其唯一作用就是在析构函数中删除CSingleton实例
	class CGarbo 
	{
	public:
		~CGarbo()
		{
			if(CSingleton::m_pInstance) 
                delete CSingleton::m_pInstance;
		}
	};
	static CGarbo Garbo;  //程序结束时，系统会调用其析构函数
public:
	static CSingleton * GetInstance()
	{
		if(m_pInstance == nullptr)
			m_pInstance = new CSingleton();
		return m_pInstance;
	}
};
```
以上的写法，即满足了全局访问唯一实例，也保证了在程序结束时，系统帮助我们选择正确的释放时机，不必我们关心此实例的释放。但是依旧存在缺陷，因为此方式是线程不安全的。在多线程中，当多个线程同时访问时，会同时判断实例未创建，从而创建出多个实例，很明显违背了我们实例唯一的需求。

不难想出，此时可以使用线程锁来保证线程的安全。如以下示例：
```
static CSingleton * GetInstance()
{
	Lock();  //可以使用临界区CRITICAL_SECTION或者互斥量MUTEX来实现线程锁
	if(m_pInstance == nullptr)
		m_pInstance = new CSingleton();
	UnLock();
	return m_pInstance;
}
```
上面的写法依旧存在缺陷，因为当某个线程要访问时，就立即上锁，这样导致了不必要的锁的消耗。所以我们可以先判断下实例是否存在，再进行是否上锁的操作。这就是所谓的**双检查锁(DCL)**思想，即Double Checked Locking。优化的写法如下实例：
```
static CSingleton * GetInstance()
{  
    if(m_pInstance == nullptr) {  
        Lock();
        if(m_pInstance == nullptr) {  
            m_pInstance = new CSingleton();  
        }  
        UnLock();  
    }  
    return m_pInstance ;  
}
```
此时一个完整的单例模式就实现了，但事实证明，此实现的写法依旧存在着重大的问题，而问题就在于`m_pInstance = new CSingleton;`这一句，具体如下：
```
分析: m_pInstance = new CSingleton()这句话可以分成三个步骤来执行:
1.分配了一个CSingleton类型对象所需要的内存。
2.在分配的内存处构造CSingleton类型的对象。
3.把分配的内存的地址赋给指针m_pInstance。

可能会认为这三个步骤是按顺序执行的,但实际上只能确定步骤 1 是最先执行的,步骤2,3却不一定。
问题就出现在这。假如某个线程A在调用执行m_pInstance = new CSingleton()的时候是按照1, 3, 2的顺序的,
那么刚刚执行完步骤3给singleton类型分配了内存(此时m_ instance就不是nullptr了 )就切换到了线程B,
由于m_pInstance已经不是nullptr了,所以线程B会直接执行return m_ instance得到一个对象,而这个对象并没有真正的被构造! ! 
严重bug就这么发生了。
```
参考：[https://segmentfault.com/a/1190000015950693](https://segmentfault.com/a/1190000015950693)

##### 进一步探讨
著名的《Effective C++》系列书籍的作者 Meyers 提出了C++ 11版本最简洁的跨平台方案，即 `Meyers' Singleton`  
实现如下：
```
class CSingleton
{
private:
    CSingleton(void)
    
public:
    static CSingleton & getInstance()
    {
        static CSingleton m_pInstance;  //局部静态变量
        return m_pInstance;
    }
};
```
这样的写法即简洁又完美！需要注意的是此写法需要支持C++11以上、GCC4.0编译器以上。

以上的所有内容是本人的一些思考和领悟，如有不正确的地方，欢迎大佬指正。