#### 一. 问题引入
值传递引发的效率问题以及**[对象切割(slicing)](https://blog.csdn.net/sszgg2006/article/details/7816725)**问题：  

	class Window {
	public:
		string name() const; //  返回窗口名
		virtual void display() const; //  显示窗口及其内容
	};
	
	class WindowWithScrollBars: public Window {
	public:
		virtual void display() const;
	};

	void printNameAndDisplay(Window w)	// 不正确！参数可能被切割。
	{
		std::cout << w.name();
		w.display();
	}

	WindowWithScrollBars wwsb;
	printNameAndDisplay(wwsb);	// wwsb被切割。

- **效率问题**：  
调用类的copy构造函数，其所有数据成员以及基类的copy构造函数都会被调用。  
- **对象切割问题**：  
如上例，派生类对象wwsb被用来copy构造基类Window对象w，对象wwsb的所有派生类特化信息都被切割。printNameAndDisplay函数内调用的总是Window::display。  

#### 二. 改进
以const引用传值。  

	void printNameAndDisplay(const Window& w);

引用往往以指针实现出来。  

特别需要注意的是：  
- **即使是“小型的用户自定义类型”也应该以const reference传值**，基于以下理由：  
	- 效率问题。其成员指针虽小，如果设计了深拷贝构造，那将会拖慢效率；编译器可能会将简单内置类型放入缓存，而不会将同样大小的自定义类型放入。
	- 小型自定义类型有可能在日后维护扩展中变大。
	- 不同的C++编译器也会影响小型自定义类型的大小。  
- 内置类型、STL的迭代器和函数对象 可以以值传递。
	
#### 三. 总结
- 尽量以pass-by-reference-to-const替换pass-by-value，前者通常比较高效，并可避免切割问题。
- 以上规则并不适用于内置类型、STL的迭代器和函数对象。对它们而言，pass-by-value往往比较适当。