#### 一. inline的优点和缺点
- **优点**：  
将函数调用替换为函数本体，免去函数调用的开销。  
- **缺点**：  
增大目标码，导致额外的换页行为，降低指令高速缓存的命中率，导致效率损失。  

因此，**inline函数的本体很小时，inline才有更高效率。**  

#### 二. inline的声明方式
##### 2.1 隐式声明
将函数定义于 class 定义式内。  

	class Person{ 
	public: 
	    ... 
	    int age() const { return theAge; }		//一个隐喻的inline申请 
	    ... 
	
	private: 
	    int theAge; 
	};

##### 2.2 明确声明
inline关键字。

	template<typename T>
	inline const T& std::max(const T& a, const T& b) 
	{ return a < b ? b: a; }

#### 三. inline注意事项
##### 3.1 尽量将inline定义在头文件中
- 大多数编译环境在编译期进行 inlining 。 inline函数的本体替换需要在编译期确定其实现，因此一般放在头文件中。
- 对链接期 inlining 的编译环境来说，将inline 定义在头文件中有助于保持 inline 的一致性。即使定义在 cpp 中，也应保证同签名、同名函数的实现的一致性，否则可能引发[意想不到的错误](http://blog.bitfoc.us/p/379)。

##### 3.2 inline失败的情形
- 太过复杂的函数：包含递归、循环等。
- 虚函数。虚函数的调用需要在运行期确定调用哪个函数；而 inline 意味着在编译期将调用动作替换为函数本体。
- 是否inline成功，取决于编译环境。大部分编译期在 inline失败时，会给出一个警告信息。  

##### 3.3 取函数地址对inline的影响
- 如果程序要取某个 inline 函数的地址，那么该函数还是会生成一个函数本体。编译器通常不对通过函数指针的调用实施 inline 动作。
- 即使程序中没有显式取函数地址，有些编译器也会使用构造、析构函数的地址，用来在容器内部构造、析构元素。  

		inline void f(){…}  // 假设编译器有意愿 inline “对 f 的调用”
		void (*pf)() = f;	// pf指向f
		f(); 				// 这个调用将被 inlined，因为是一个正常调用
		pf(); 				// 这个调用或许不被 inlined，因为它通过函数指针达成

##### 3.4 构造函数、析构函数一般不应该声明为inline
编译器会对构造、析构函数插入一些隐藏代码用来实现C\+\+的保证：  

- 构造函数：  
	- base class的自动构造。
	- 每一个成员变量的自动构造。
	- 若构造过程中抛出异常，则已构造的部分被自动销毁。  

- 析构函数：
	- base class的自动析构。
	- 每一个成员变量的自动析构。  


			class base { 
			public: 
			    ... 
			private: 
			    std::string bm1, bm2; 
			};
			
			class Derived : public Base { 
			public: 
			    Derived(){}  // Derived 构造函数是空的 是吗? 
			    ... 
			private: 
			    std::string dm1, dm2, dm3; 
			};

相当于：  

	Derived::Derived() 
	{
	   	Base::Base(); 						// 初始化 base class
	   	try { dm1.std::string::string(); } 	// 初始化成员变量 bm1，并处理异常。
	   	catch(...) { 
	        Base::~Base(); 
	        throw; 
	   	} 
	
	   	try { dm2.std::string::string(); } 	// 初始化成员变量 bm2，并处理异常。
		catch(...) { 
	        dm1.std::string::~string(); 
	        Base::~Base(); 
	        throw; 
		} 
	
	    try { dm3.std::string::string(); } 	// 初始化成员变量 bm3，并处理异常。
	    catch(...) { 
	        dm2.std::string::~string(); 
	        dm1.std::string::~string(); 
	        Base::~Base(); 
	        throw; 
	    }
	}

在考虑是否将构造函数声明为 inline 时，也需要考虑这些隐藏代码带来的影响：  
如果 Base 构造函数被 inline，那么它的实现也会复制一份到 Derived 构造函数中；如果 string 构造函数也被 inline，那么 Derived 构造函数将获得五份副本！析构函数也类似。

##### 3.5 inline对程序库的影响
修改程序库内的 inline 函数，会导致使用程序库的所有代码重新编译；如果改为正常函数，则仅需重新链接。  

##### 3.6 inline与调试器
大部分调试器对 inline 函数束手无策，因为 inline 函数实际上不存在函数体。

##### 3.7 inline的使用策略
- 开始时，仅将 inline 声明于那些 “一定成为inline”（条款46） 或 “十分平淡无奇”（例如 2.1的隐式声明） 的函数上。
- 在后续优化过程中，考虑把那些经常被调用的函数声明为 inline。

#### 四. 总结
- 将大多数 inline 限制在**小型、被频繁调用**的函数身上。这可使日后的调试过程和二进制升级更容易，也可以使潜在的代码膨胀问题最小化，使程序的速度提升机会最大化。
- 不要只因为 function templates 出现在头文件，就将它们声明为 inline。模板函数与内联函数并无直接联系。