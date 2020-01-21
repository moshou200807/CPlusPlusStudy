#### 一. 总结
- 声明 template 参数时，前缀关键字 class 和 typename 可互换。
- 请使用关键字 typename 标识嵌套从属类型名称；但是不得在 base class lists(基类列) 或 member initialization list(成员初值列)内以它作为 base class 修饰符。

#### 二. 解释
##### 2.1 class和typename可互换的情形

	template<class T> class Widget;			// 使用 class
	template<typename T> class Widget;		// 使用 typename

声明 template 参数时，前缀关键字 class 和 typename 可互换。  
作者喜欢 typename，因为它暗示参数并非一定得是个 class 类型。

##### 2.2 必须使用typename的情形
###### 2.2.1 名词解释
**从属名称（dependent names）**： template内出现的名称如果依赖于某个 template 参数，称之为从属名称。
**嵌套从属名称（nested dependent name）**：如果从属名称在 class 内呈嵌套状，我们称它为嵌套从属名称。特别的，这个名称不一定得是类型，也可能是某个公共变量名。  
**非从属名称（non-dependent names）**：template内出现的、不依赖于 template 参数的名称。  

嵌套从属名称可能导致解析困难：  

	template <typename C> 
	void print2nd(const C& container) 
	{ 
	    C::const_iterator* x; 
	}

在我们知道 C 是什么之前，我们无法事先得知 C::const_iterator 是一个类型还是 C 内的一个 static 变量，依此可以分别解释为某个类型的指针、或相乘动作。  
当编译器开始解析 template print2nd 时，尚未确认 C 是什么东西，因此造成歧义。  
C\+\+规定：**如果解析器在 template 中遭遇一个嵌套从属类型名称，它便假设这名称不是个类型，除非你告诉它是**。  

###### 2.2.2 嵌套从属类型名称必须使用typename
基于 2.2.1 的解释，**如果一个嵌套从属名称是一个类型，那么必须使用关键字 typename 告诉编译器**。  

	template <typename C> 
	void print2nd(const C& container) 
	{ 
	    typename C::const_iterator* x; 
	}

##### 2.3 例外：不允许使用typename的情形
###### 2.3.1 不是嵌套从属类型名称不允许使用typename

	template <typename C> 
	void f(const C& container, 			// 不允许使用 typename 
	       typename C::iterator iter);	// 一定要使用 typename

形参 container 不是嵌套从属类型名称（它并非**嵌套于**任何“取决于 template 参数”的东西内），所以它不允许使用 typename 为前导。

###### 2.3.2 base classes list 和 member initialization list 中不可以出现typename
typename 不可以出现在 **base classes list** 内的嵌套从属类型名称之前，也不可在 **member initialization list**（成员初始化列表）中作为 base class 修饰符。  

	template <typename T> 
	class Derived: public Base<T>::Nested { 		// base class list 中不允许“typename” 
	public: 
	    explicit Derived(int x) 
	        : Base<T>::Nested(x)					// mem.init.list 中不允许“typename” 
	    { 
	       typename Base<T>::Nested temp;			// 嵌套从属类型名称。需加上 typename 
			...
	    }
		...
	};

##### 2.4 使用typedef减少篇幅

	template <typename IterT> 
	void workWithIterator(IterT iter) 
	{ 
	    typename std::iterator_traits<IterT>::value_type temp(*iter); 
		...
	}

减少篇幅：

	template <typename IterT>
	void workWithIterator(IterT iter)
	{
		typedef typename std::iterator_traits<IterT>::value_type value_type;
		value_type temp(*iter);
		...
	}

需要指出的是，typename 相关规则在不同的编译器上有不同的实践。有些编译器表现错误，还有少数编译器甚至不支持 typename。这意味着 typename 和 “嵌套从属类型名称”之间的互动，也许会在移植性方面带给你某种头疼。