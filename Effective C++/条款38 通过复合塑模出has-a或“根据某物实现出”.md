#### 一. 总结
- 复合（composition）的意义和 public 继承完全不同。
- 在应用域（application domain），复合意味着 has-a（有一个）。在实现域（implementation domain），复合意味 is-implementation-in-terms-of（根据某物实现出）。

#### 二. 解释
##### 2.1 复合
复合是类型之间的一种关系。当某种类型的对象内含其他类型的对象，便是这种关系。  
复合在程序世界还有其他同义词，包括 分层（layering）、内含（containment）、聚合（aggregation）、内嵌（embedding）。

	class Address { ... }; 
	class PhoneNumber { ... }; 
	
	class Person { 
	public: 
	    ... 
	private: 
	    std::string name; 			// 复合
	    Address address; 			// 复合
	    PhoneNumber voiceNumber; 	// 复合
	    PhoneNumber faxNumber; 		// 复合
	};

复合实际上有两个意义。  

##### 2.2 意义一：has-a
**应用域**的意义。  
所谓**应用域**，就是你塑造的世界中的某些事物所在的域，比如人、汽车、飞机等等。  
复合发生于应用域内的对象之间时，表现出 has-a 关系，比如上述 Person **有**一个名称、地址等。  
这个意义与 is-a 很好区分，你总不会认为 人就是名称、地址等。

##### 2.3 意义二：is-implemented-in-term-of
**实现域**的意义。  
所谓**实现域**，就是在实现细节上添加的人工制品。比如缓冲区、互斥器、查找树等等。  
复合发生于实现域内，则表现出 is-implemented-in-term-of 关系，也就是“根据某物实现出”。  
要注意这个意义与 is-a 的区分，有时会造成困扰。  

考虑一个例子：  
STL 中的 set 以红黑树实现，每个元素耗用三个指针，以空间换时间。当你需要实现重视空间复杂度的 template set 时，使用 STL 的 template list 是一个很好的复用方法。  

但是请注意， set is-not-a list。如条款 32 所说，如果 D 是一种 B，对 B 为真的每一件事情对 D 也都应该为真。然而 list 可以包含重复元素， set 却不可以包含重复元素，因此它们不是 is-a 关系，也就不能设计为继承关系。  

	// 错误的声明方法
	template<typename T> 
	class Set : public std::list<T> { ... }; // 将 list 应用于 set。错误做法。

正确的做法是将它们塑造为复合关系中的“根据某物实现出”关系。  

	template<typename T> 
	class Set { 							// 将 list 应用于 set。正确做法。
	public: 
	    void insert(const T& item); 
	    void remove(const T& item); 
	    std::size_t size() const; 
	private: 
	    std::list<T> rep; 					// 用来表述 set 的数据 
	};

是不是很眼熟？实际上 STL 的 queue（根据 deque 或 vector 实现出）、map（根据红黑树实现出）、set（根据红黑树实现出）等都是这种复合关系写法。