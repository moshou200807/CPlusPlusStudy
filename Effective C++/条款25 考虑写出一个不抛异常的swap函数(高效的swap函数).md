#### 一. 问题引入
对某些类而言，如果使用std::swap，可能有不必要的效率问题。其中最主要的就是“以指针指向一个对象，内含真正数据”那种类型，常见表现形式是所谓"pimpl"手法。  

	class WidgetImpl { 
	public: 
	    ... 
	private: 
	    int a, b, c; 
	    std::vector<double> v; 
	    ... 
	}; 
	
	class Widget { 									// 该类使用pImpl手法
	public: 
	    Widget(const Widget& rhs); 
	    Widget& operator=(const Widget& rhs) 
	    { 
	        ... 
	        *pImpl = *(rhs.pImpl); 					// 对象内容复制，较大的复制成本
			...
	    } 
		...
	private: 
	    WidgetImpl* pImpl; 							// 真正的数据栖身地
	};

std::swap是异常安全性编程（条款29）的脊柱，也是用来处理自我赋值的一种常见机制（条款11）。典型实现如下：  

	namespace std { 
	    template<typename T> 
	    void swap(T& a, T& b) 
	    { 
	        T temp(a); 		// copy构造成本
	        a = b; 			// copy赋值成本
	        b = temp; 		// copy赋值成本
	    } 
	}

上面的典型实现，有三次复制的成本。然而对某些对象的swap来说，这些复制动作无一必要。比如最开始提到的"pimpl"手法类型。

#### 二. 改进
##### 2.1 想法一：提供全特化版本
对Widget对象而言，swap动作需要做的只是交换它的pImpl指针。  

	namespace std {
	    template<>		// 表示它是template<typename T> void swap(T& a, T& b);的一个全特化版本。
	    void swap<Widget>(Widget& a, Widget& b)		// swap<Widget>中的Widget表示该全特化版本针对“T=类Widget”设计。
	    {
	        swap(a.pImpl, b.pImpl); // error！访问 private 成员变量
	    }
	}

该版本无法通过编译：访问了private成员。可以将该全特化函数声明为friend，但我们一般提供一个public的swap成员函数来代替直接访问。  

	class Widget 		// 同上，增加swap函数
	{ 
	public: 
		...
	    void swap(Widget& other) 
	    { 
	        using std::swap; 			// 该声明很有必要，稍后解释
	        swap(pImpl, other.pImpl); 
	    }
		...
	};
	
	namespace std { 
	    template<> 
	    void swap<Widget>(Widget& a, Widget& b) 	// 修改后的std::swap特化版本
	    { 
	        a.swap(b); 		// 调用public成员函数，代替直接访问private成员
	    } 
	}

修改后的做法与STL容器具有一致性：所有STL容器也都提供public swap成员函数和std::swap特化版本。  

##### 2.2 考虑模板类问题
###### 2.2.1 std不接受添加新模板

	template <typename T>
	class WidgetImpl { ... };
	
	template <typename T>
	class Widget { ... };

在Widget内放个swap成员函数如之前一样简单，但是在特化std::swap时遇到问题：  

	namespace std { 
	    template<typename T> 
	    void swap< Widget<T> >(Widget<T>& a, Widget<T>& b) 		// 错误！不合法！ 
	    { 
	        a.swap(b); 
	    } 
	}

企图偏特化一个模板函数template<typename T> void swap(T& a, T& b);。然而，C\+\+不允许对模板参数的偏特化，仅允许全特化。(代码不应该通过编译，虽然有些编译器错误地接受了它)    
一个常见的替换做法是：为需要偏特化的模板函数提供一个重载版本，如下：  

	namespace std { 
	    template<typename T> 
	    void swap(Widget<T>& a, Widget<T>& b) 	// 注意“swap后没有<>” ，因此它仍然是一个普通模板函数，而不是特化或偏特化版本。
	    { 
	        a.swap(b); 
	    } 
	}

对一般的命名空间有效，但是std命名空间有特殊的规则：C\+\+标准委员会决定std的内容。对std内的模板，客户可以全特化，但是**不可以添加新的模板到std内**。  
上面的重载swap将新的模板添加到std内，虽然可能可以编译和执行，但它们可能导致未定义行为。  

###### 2.2.2 解决：non-member swap移到其他命名空间
还是声明一个 non-member swap，但不再是 std::swap 的特化版本或重载版本：  

	namespace WidgetStuff{
		...							// 模板化的WidgetImpl等等
		template<typename T>
		class Widget { ... };		// 同前，内含swap成员函数
		...
		template<typename T> 
		void swap(Widget<T>& a, Widget<T>& b) 		// non-member swap函数；不属于std命名空间
		{ 
		    a.swap(b); 
		} 
	}

现在，C\+\+的名称查找法则保证会优先找到非std命名空间内的、与Widget同命名空间的non-member swap版本。  

即使存在非std命名空间内的swap版本，你仍然需要提供一个std::swap特化版本：  

	std::swap(obj1, obj2);		// 错误的swap调用方式

假如客户以上述错误调用方式调用，编译器则会直接使用std::swap版本。为了防止此类错误，也请提供一个std::swap特化版本。  

###### 2.2.3 客户的正确使用方式
客户应该希望调用非std命名空间的non-member T专属版本，并在该版本不存在的情况下调用std内的一般化版本：  

	template<typename T> 
	void doSomething(T& obj1, T& obj2) 
	{ 
	    using std::swap; 	// 令 std::swap 在此函数内可用。（将std::swap引入） 
	    ... 
	    swap(obj1, obj2); 	// 为 T 型对象调用最佳 swap 版本 
	    ... 
	}

将std::swap引入，可以使编译器在找不到非std版本时，正确使用std::swap版本。  

##### 2.3 总结
至此已经讨论过default swap、member swaps、non-member swaps、std::swap特化版本以及对swap的调用。  

首先，如果swap的缺省实现码对class或class template有可接受的效率，那么不需要做额外的事情；  
其次，如果swap缺省实现版效率不足（通常使用了pimpl手法），试着做以下事情：
  
- 提供一个public swap成员函数，实现高效置换功能，而且绝不应该抛出异常。
- 在class或class template所在的命名空间内提供一个non-member swap，并令它调用上述swap成员函数。
- 如果编写的是一个class而非class template，则为它提供一个全特化版本的std::swap，并令它调用你的swap成员函数。
- 客户调用swap，请确定包含一个using声明式，将std::swap在调用函数内曝光可见，然后不添加任何namespace修饰符，直接调用swap。

##### 2.4 成员版swap绝不可抛出异常
**成员版**swap的一个最好应用是帮助 classes（和class templates）提供强烈的异常安全性保障。条款29对此主题提供了所有细节，但此技术基于一个假设：成员版的 swap 绝不抛出异常。  
对非成员版无此要求。因为swap缺省版本调用的copy函数允许抛出异常。  
当你写下一个自定义版本的swap时，往往提供的不只是高效置换对象值的方法，而且包含不抛出异常。一般而言这两个swap特性是连在一起的，因为高效率的swap几乎总是基于对内置类型的操作（例如pimpl手法的底层指针），而内置类型上的操作绝不会抛出异常。  

#### 三. 总结
- 当std::swap对你的类型效率不高时，提供一个swap成员函数，并确定这个函数不抛出异常。
- 如果你提供一个member swap，也该提供一个non-member swap用来调用前者。对于class（而非 template），也请特化std::swap。
- 调用swap时应针对std::swap使用using声明式，然后调用swap并且不带任何“命名空间资格修饰”。
- 为“用户定义类型”进行std template全特化是好的，但千万不要尝试在std内加入某些对std而言全新的东西。