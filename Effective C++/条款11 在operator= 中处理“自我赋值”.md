#### 一. 问题引入
不经意的自我赋值：  

	a[i] =a[j]; // 如果i==j
	*px = *py;	// 如果px、py指向同一个东西

不可靠的operator=代码。既不是自我赋值安全的，也不是异常安全的：  

	class Bitmap { ... };
	class Widget {
	    ...
	private:
	    Bitmap *pb; // 指向一个从heap分配而得的对象的指针
	};
	
	Widget& Widget::operator=(constWidget& rhs)
	{
	   delete pb;						// 删除*this.pb。如果是自我赋值，删除的也是rhs.pb
	   pb = new Bitmap(*rhs.pb);		// *this.pb被赋值为失效的rhs.pb
	   return *this;
	}

#### 二. 不充分改进
加入“证同测试”：

	Widget& Widget::operator=(constWidget& rhs)
	{
	   if (this == &rhs) return *this; // 证同测试：如果是自我赋值，就不做任何事
	   delete pb;						// (1)
	   pb = new Bitmap(*rhs.pb);		// (2)
	   return *this;
	}

具备了自我赋值安全，但是不具备异常安全性：  
假如非自我赋值时，语句(1)执行完毕，this->pb被删除；语句(2)执行过程中如果抛出异常，那么Widget最终仍会有一个对象的pb成员指向失效内存。  
然而，让operator= 具备异常安全性，往往自动获得自我赋值安全的回报。因此，把焦点放在实现 异常安全性 即可。  
> 异常安全性 在条款29 中深入探讨。

#### 三. 具备异常安全性的改进

	Widget& Widget::operator=(constWidget& rhs)
	{
	      Bitmap *pOrig = pb; 		// 记住原先的pb
	      pb = new Bitmap(*rhs.pb); // 令pb指向*pb的一个副本
	      delete pOrig; 			// 可能抛出异常的语句执行完毕后再删除原先的pb。拥有异常安全性。
	 	  return*this;
	}

上述代码在具备异常安全性的同时，仍能处理自我赋值：  

- 记住原pb
- 复制一份rhs.pb的拷贝
- 删除原pb  

> 在自我赋值处理时，相比之前的证同操作，多了一步new。如果很关心效率，可以在起始处再加上“证同测试”。然而带来的代码变大、新控制分支、prefetching、caching、pipelining等都会降低效率。
> 相比自我赋值发生的频率来说，加入证同测试可能不太值得。

#### 四. copy and swap技术
另外一种常见的 具备异常安全性的替代方案是copy and swap技术。在条款29中详细说明。  

	class Widget {
	    ...
	    void swap(Widget& rhs); // 交换*this和rhs数据；详见条款29
	    ...
	};
	 
	Widget& Widget::operator=(constWidget& rhs)		// 引用传递。rhs本身不应该被修改
	{
	     Widget temp(rhs); 		// 为rhs数据制作一份副本。如果抛出异常，接下来的swap不会被执行，具备异常安全性
	     swap(temp); 			// 将*this数据和上述复件的数据交换。rhs引用本身的数据不变；temp与this数据交换
	     return *this;
	}

	
	Widget& Widget::operator=(Widget rhs)	// rhs是被传对象的一份副本，注意这里是值传递，rhs可以被修改。
	{
	    swap(rhs);							// 将*this的数据和临时生成的匿名副本数据rhs互换
	    return *this;
	}

这种伶俐而巧妙的修补牺牲了清晰性。然而将“copy操作”从函数体内转移到“函数参数构造阶段”却可令编译器有时生成更高效的代码。

#### 五. 总结
- 确保当对象自我赋值时operator= 有良好行为。其中技术包括比较“来源对象”和“目标对象”的地址(证同测试)、精心周到的语句顺序(三)、以及 copy-and-swap(四)。
- 确定任何函数如果操作一个以上的对象，而其中多个对象是同一个对象时，其行为仍然正确。