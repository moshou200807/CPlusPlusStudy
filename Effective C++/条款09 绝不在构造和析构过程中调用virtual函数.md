#### 一. 问题引入
假设有一套模拟股票交易的类继承体系，而且每笔交易都要经过记录。  

	class Transaction { // 所有交易的基类
	public:
	     Transaction();
	     virtual void logTransaction() const = 0; // 做出一份因类型不同而不同的日志记录
	     ...
	};
	 
	Transaction::Transaction() // 基类构造函数的实现
	{
	     ...
	     logTransaction(); // 最后动作是志记这笔交易。在基类构造函数中调用了virtual函数
	}
	 
	class BuyTransaction: public Transaction { //derived class
	public:
	      virtual void logTransaction() const;	// 实现派生类特色的日志记录
	      ...
	};
	 
	class SellTransaction: public Transaction {// derived class
	public:
	      virtual void logTransaction() const;	// 实现派生类特色的日志记录
	      ...
	};

	BuyTransaction b;

**base class构造期间virtual函数绝不会下降到derived class阶层**。原因：  

- 构造BuyTransaction对象时，先构造base class对象。base class的构造函数的执行早于derived class构造函数。  
当base class构造函数执行时，derived class的成员变量尚未初始化。如果此期间调用的virtual函数下降至derived class阶层，而derived class的函数几乎必然取用local成员变量(这些变量尚未初始化)，引发不确定行为，C++拒绝执行这样的路线。
- 实际上，在derived class对象的base class构造期间，对象的类型是base class而不是derived class。不仅virtual函数会被编译器解析至base class，如果使用运行期类型信息(dynamic_cast、typeid等)，也会把对象视为base class类型。  

上面的例子很容易被编译器或连接器侦测出来。然而如果采用下面的版本，则通常不会引发任何编译器和连接器的抱怨。  

	class Transaction {
	public:
	     Transaction()
	     { init(); } // 调用non-virtual...
	     virtual void logTransaction() const = 0;	// 如果不存在函数定义，运行时仍会被终止；然而假如基类提供了函数定义，那么将始终调用基类版本而不是派生类版本！！
	     ...
	private:
	    void init() {
	        ...
	        logTransaction(); // 这里调用virtual!
	     }

因此，**不应该在构造和析构函数期间直接或间接地调用virtual函数**。

#### 二. 改进
如果需要确保对象被创建的同时，就有适当版本的日志信息，可以做如下修改：  

- 基类logTransaction 转变为一个 non-virtual函数；
- 派生类构造函数将必要的信息传递给 Transaction 构造函数，而后那个函数就可以安全地调用 non-virtual的 logTransaction。  


		class Transaction {
		public:
		    explicit Transaction(const std::string& logInfo);
		    void logTransaction(conststd::string& logInfo) const;	// 如今是个non-virtual函数。仅有基类版本。参数logInfo被派生类用来传递特殊化信息。
		    ...
		};
		 
		Transaction::Transaction(const std::string& logInfo)
		{
		    ...
		    logTransaction(logInfo); 								//如今是个non-virtual函数
		}
		 
		class BuyTransaction: public Transaction {
		public:
		    BuyTransaction( parameters )
		    : Transaction(createLogString( parameters ))			// 将log信息传递给基类构造函数
		    { ... } 
		    ...
		 
		    private:
		    static std::string createLogString(parameters );		// 注意static
		};

做法相当于派生类将必要的构造信息向上传递至基类构造函数，而不是之前危险的试图向下传递。  
同时**注意static函数createLogString的运用**：时刻注意，构造基类期间，派生类的成员变量处于未定义状态。static约束可以确保该函数不使用派生类的未定义变量。  

#### 三. 总结
- 在构造和析构期间不要调用virtual函数，因为这类调用从不下降至derived class（比起当前执行构造函数和析构函数的那层）。