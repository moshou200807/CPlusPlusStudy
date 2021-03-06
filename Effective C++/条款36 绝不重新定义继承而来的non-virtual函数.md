#### 一. 问题引入
如果子类重新定义了继承自父类的 non-virtual 成员函数 mf，由于静态绑定的关系，当**通过指针或引用访问 mf 时，访问哪一个由指针或引用的类型决定**。  

	class B {
	public:
	    void mf();
		...
	};
	 
	class D : public B {
	public:
	    void mf();
	};

	D x;
	
	B *pB = &x;
	pB->mf();		// 调用B::mf
	
	D *pD = &x;
	pD->mf();		// 调用D::mf

如果该函数是 virtual，则总会调用 D::mf，因为它是动态绑定。  

条款 7 的深层原因就在于此：如果父类**析构函数**为 non-virtual，那么子类就不应该重新定义继承而来的**析构函数**。即使没有声明析构函数也是如此，因为条款 5 表明编译器会自动生成一些代码，包括析构函数。  
就本质而言，条款 7 只不过是本条款的一个特殊案例。  

#### 二. 总结
- 绝对不要重新定义继承而来的 non-virtual 函数。