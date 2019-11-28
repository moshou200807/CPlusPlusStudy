> 在构造自己的资源管理类时，需要小心copying行为。

#### 一. 问题引入
假定为互斥锁构造资源管理类：  

	class Lock {
	public:
	    explicit Lock(Mutex *pm)
			: mutexPtr(pm)
	    { lock(mutexPtr); } // 获得资源
	    ~Lock() { unlock(mutexPtr); } // 释放资源
	
	private:
	    Mutex *mutexPtr;
	};

当客户使用Lock对象的复制时，出现问题：  

	Mutex m; // 定义互斥器

	Lock ml1(&m); // 锁定m
	Lock ml2(ml1); // 将ml1复制构造到ml2身上，这会发生什么？ml2获得了一把被锁住的锁的指针！

#### 二. 需要自定义copying行为
应该摒弃编译器默认生成的copying函数。  

##### 2.1 禁止复制
如果RAII对象不应该被复制，则需要禁止它。  

	class Lock: private Uncopyable { // 禁止复制
	public: 
		... // 如前
	};

##### 2.2 对底层资源进行引用计数
保有资源，直至最后一个使用者(RAII对象)被销毁。  
通常可以在RAII class内含一个shared_ptr成员变量来实现引用计数。  

	class Lock {
	public:
	    explicit Lock(Mutex *pm)
			: mutexPtr(pm, unlock) // 以某个mutex初始化shared_ptr并 **以unlock函数为删除器**(shared_ptr默认删除器为delete指针，而互斥量需要unlock)
		{
			lock(mutexPtr.get());
		}
	private:
	    std::tr1::shared_ptr<Mutex> mutexPtr; 	// 使用shared_ptr替换raw pointer
	};

使用编译器**默认copy构造/赋值函数和析构函数**即可：  

- 默认copy构造/赋值函数会调用成员对象mutexPtr的copy构造/赋值函数，会使得mutexPtr的引用计数正常+1；
- 默认析构函数调用成员对象mutexPtr的析构函数，引用计数-1。直至0时，unlcok被调用。  

##### 2.3 复制底部资源
有时需要针对一份资源复制任意数量的副本。这时使用资源管理类的唯一理由就是：当你不再需要某个副本时确保它被释放。  
在此情况下的copying行为，应注意进行深拷贝。  
如string。  

##### 2.4 转移底部资源的拥有权
在某些罕见场合下，你可能希望确保永远只有一个RAII对象指向一个未加工资源，即使RAII对象被复制依然如此。  
这时可以参考auro_ptr(已弃用，应该改用unique_ptr)或转移语义的行为。  

##### 2.5 使用编译器生成的版本
某些情况下你或许想支持copying函数的一般版本，详见条款45。

#### 三. 总结
- 复制RAII对象必须一并复制它所管理的资源，所以**应该由资源的copying行为决定RAII对象的copying行为**。
- 普遍而常见的RAII class copying行为是：抑制copying、施行引用计数法(reference counting)。不过其他行为也都可能被实现。