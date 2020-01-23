> 当 operator new 无法满足某一内存分配需求时，它会抛出异常。在旧版本中，会返回一个 null 指针。旧版本的行为将在最后讨论。

#### 一. new-handler
当 operator new 抛出异常前，它会不断调用一个客户指定的错误处理函数，就是 new-handler，直到找到足够内存。引起反复调用的代码显示于条款 51。  
如果设置了 new-handler，那么 operator new 本身不会抛异常；没有设置，则 operator new 抛出异常。  

	 namespace std {
        typedef void(*new_handler)();
        new_handler set_new_handler(new_handler p) throw();
    }

一个设计良好的 new-handler 函数必须做到以下一个或多个要求：  

- **让更多内存可被使用**：这便造成 operator new 内的下一次内存分配动作可能成功。一个做法是：程序一开始就分配一大块内存，当 new-handler 第一次被调用时将它释放。
- **安装另一个 new-handler**：如果当前 new-handler 无法获得更多可用内存，并且知道另外的 new-handler 可能有此能力，那么它应该可以安装另外的 new-handler。（另外一种变奏是让 new-handler 修改自己的行为，这样当下次被调用时，可以尝试不同的事情。）
- **卸除 new-handler**：就是将 null 指针传给 set\_new\_handler。这样一旦没有安装任何 new-handler， operator new 会在内存分配不成功时抛出异常。
- **抛出 bad\_alloc 的异常**：这里是 new-handler 本身抛出的异常。因为一旦设置了 new-handler， operator new 不会抛出异常，需要  new-handler 自行判断是否抛出异常。
- **不返回**：通常调用 abort 或 exit。

#### 二. class专属new-handler
C\+\+不直接支持 class 专属 new-handlers。不过你可以自行实现出这种行为，只需令每一个 class 提供自己的 set\_new\_handler 和 operator new。  

	class Widget {
    public:
        static std::new_handler set_new_handler(std::new_handler p) throw();
        static void* operator new(std::size_t size) throw(std::bad_alloc);
    private:
        static std::new_handler currentHandler;
    };
    std::new_handler Widget::currentHandler = 0;
    
	std::new_handler Widget::set_new_handler(std::new_handler p) throw()
    {
        std::new_handler oldHandler = currentHandler;
        currentHandler = p;
        reutrn oldHandler;
    }

	void* Widget::operator new(std::size_t size) throw(std::bad_alloc)
	{
		...
	}

- set\_new\_handler 所做的事：保存新指针；返回旧指针。行为如同标准版 std::set\_new\_handler。
- operator new 所做的事：
	- 调用标准 std::set\_new\_handler，将 Widget::currentHandler 安装为 **global** new-handler。
	- 调用 global operator new，执行内存分配。
		- 如果分配失败， global operator new 会调用 Widget 的 new-handler。如果最终无法分配足够内存，会抛出一个 bad\_alloc 异常。在此情况下，Widget 的 operator new 必须恢复原本的 global new-handler，然后再传播该异常。
		- 为了确保原本的 new-handler 总是能够被重新安装回去，我们将使用资源管理对象防止资源泄漏，见 三.
		- 如果分配成功，Widget 的 operator new 返回一个指针。后续 Widget 应将 Widget 的 operator new 被调用前的那个 global new-handler 恢复回来，避免影响其他内存分配时的默认 new-handler。

#### 三. 资源管理对象管理new-handler

	class NewHandlerHolder {
    public:
        explicit NewHandlerHolder(std::new_handler nh)
         : handlere(nh) {}											// 构造函数记录 new-handler
        ~NewHandlerHolder()
        { std::set_new_handler(handler); }							// 析构函数设置 new-handler
    private:
        std::new_handler handler;
        
		NewHandlerHolder(const NewHandlerHolder&);					// 阻止copying
        NewHandlerHolder& operator=(const NewHandlerHolder&);
    };

使用：  

	void* Widget::operator new(std::size_t size) throw(std::bad_alloc)
    {
        NewHandlerHolder
		 	h(std::set_new_handler(currentHandler));		// 安装 Widget 的 new-handler
															// 并且将返回的旧 new-handler 传递给 NewHandlerHolder
															// 函数返回时自动安装 旧 new-handler
        return ::operator new(size);
    }

Widget 的客户应该这样使用：  

	void outOfMem();					// 准备处理Widget 的 new-handler
    Widget::set_new_handler(outOfMem);	// 设定 outOfmem 为 Widget 的 new-handling 函数
    Widget* pw1 = new Widget;			// 如果内存分配失败，调用 outOfMem。 operator new 结束后，旧的 new-handler 被恢复
    
	std::string* ps = new std::string;	// 如果内存分配失败，调用 global new-handling（如果有的话）
    
	Widget::set_new_handler(0);			// 设定 Widget 专属 new-handling 为 null
    Widget* pw2 = new Widget;			// 如果内存分配失败，立刻抛出异常

#### 四. mixin风格的NewHandlerHolder
NewHandlerHolder 这种 class 应该适用于任何 class。一种简单的做法是建立起一个 mixin 风格的 base class，允许 derived classes 继承单一特定能力。  

	template<typename T>
    class NewHandlerSupport {		// mixin 风格的 base class，用来支持 class 专属的 set_new_handler
    public:
        static std::new_handler set_new_handler(std::new_handler p) throw();
        static void* operator new(std::size_t size) throw(std::bad_alloc);
        ...
    private:
        static std::new_handler currentHandler;
    };

    template<typename T>
	std::new_handler
    NewHandlerSupport<T>::set_new_handler(std::new_handler p) throw()
    {
        std::new_handler oldHandler = currentHandler;
        currentHandler = p;
        return oldHandler;
    }

    template<typename T>
	void* NewHandlerSupport<T>::operator new(std::size_t size)
    throw(std::bad_alloc)
    {
        NewHandlerHolder h(std::set_new_handler(currentHandler);
        return ::operator new(size);
    }
	
    //将每一个currentHandler初始化为null
    template<typename T>
    std::new_handler NewHandlerSupport<T>::currentHandler = 0;

有了这个 class template，为 Widget 添加 set\_new\_handler 支持能力就轻而易举了：  

    class Widget: public NewHandlerSupport<Widget> {
    	...				// 和先前一样，但不必声明 set_new_handler 或 operator new
    };

NewHandlerSupport template 从未使用其类型参数 T。加这个类型参数的原因是：我们希望继承自 NewHandlerSupport 的每一个 class，都拥有实体互异的 NewHandlerSupport 副本（实际上重要的是其中的 static 成员变量），类型参数 T 只是用来实现这个要求的手段。  
至于 Widget 继承自一个模板化的 base class，而后者又以 Widget 作为类型参数，这是一种有用的技术，叫做“怪异的循环模板模式”（CRTP，curiously recurring template pattern）。

#### 五. 旧式 operator new
C\+\+ 为了兼容旧式 operator new 失败时返回 null 的行为，提供了 “nothrow” 形式的 operator new。  

	class Widget { ... };
    
	Widget* pw1 = new Widget;		// 如果分配失败，抛出bad_alloc
    if (pw1 == null)				// 这个测试一定失败，无效
    
	Widget* pw2 = new (std::nothrow) Widget;	// 如果分配失败，返回null
    if (pw2 == null)							// 测试可能成功

需要注意的是， new (std::nothrow) Widget 仅保证 nothrow 版本的 operator new 被调用。如果分配失败，返回null指针，一切好说；如果分配成功（分配的仅是 Widget 类本身的大小），接下来调用 Widget 构造函数可能会 new 更多内存， 没人可以保证它使用的仍然是 nothrow 版本的 operator new。  
也就是说：**使用 nothrow new 只能保证 operator new 不抛出异常，不保证像 “new (std::nothrow) Widget” 这样的表达式绝不抛出异常。因此其实没有运用 nothrow new 的需要。**

#### 六. 总结
- set\_new\_handler 允许客户指定一个函数，在内存分配无法获得满足时被调用。
- nothrow new 是一个颇为局限的工具，因为它只适用于内存分配；后继的构造函数调用还是可能抛出异常。