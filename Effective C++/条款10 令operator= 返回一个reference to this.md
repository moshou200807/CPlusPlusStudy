#### 一. 问题引入
C和C++的赋值可以是连续赋值，如：  

	int x, y, z;
	x = y = z = 15;

为了实现连续赋值，赋值操作符必须返回一个reference指向操作符的左侧实参。

#### 二. 改进

	class Widget
	{
	public:
	    Widget& operator+=(const Widget& rhs) 			// 返回类型是reference，指向当前对象
	    {                                               // 其他=类的操作符同样适用
	        ...
	        return *this;                //返回左侧对象
	    }
	    Widget& operator=(int rhs)
	    {
	        ...
	        return *this;
	    }
		...
	};

这只是个协议，并无强制性。然而所有内置类型和标准程序库提供的类型如string、vector等都遵守它。

#### 三. 总结
- 令赋值(assignment)操作符返回一个reference to *this。