#### 一. 问题引入
需要注意 templates 导致的二进制码重复、臃肿问题。  

考虑模拟正方形矩阵问题，并且该矩阵的性质之一是支持逆矩阵运算。  

	template<typename T,
	            std::size_t n>		// 支持 n*n 矩阵，元素类型为 T
	class SquareMatrix {
	public:
		...
	    void invert();				// 求逆矩阵
	};
	
	SquareMatrix<double, 5> sm1;
	...
	sm1.invert();					// 调用 SquareMatrix<double, 5>::invert
	SquareMatrix<double, 10> sm2;
	...
	sm2.invert();					// 调用 SquareMatrix<double, 10>::invert

会导致具象化两份 invert。这两份 invert 除了操作的矩阵大小不同以外，其他部分完全相同。

#### 二. 改进
共性与变性分析（commonality and variability analysis），也就是把共性的部分提取出来，留下变性的部分。

	template<typename T>
	class SquareMatrixBase {					// 与尺寸无关的 base class，用于正方形矩阵
	protected:									// 这些函数只在子类中用到，因此 protected
		SquareMatrixBase(std::size_t n, T *pMem)
	     ： size(n), pData(pMem) { }
	    void setDataPtr(T* ptr) { pData = ptr; }	// 重新赋值给 pData
		...
	    void invert(std::size_t matrixSize);
		...
	private:
		std::size_t size;
		T* pData;								// base class 保存指向 derived classes 数据的指针
	};

子类写法一：

	template<typename T, std::size_t n>
	class SquareMatrix : private SquareMatrixBase<T> {
	private:
	    using SquareMatrixBase<T>::invert;		// 避免遮盖base版本的invert，见条款 33
	public:
	    SquareMatrix()
		 : SquareMatrixBase<T>(n, data) { }		// 送出矩阵大小和数据指针给 base class
		...
	    void invert() { this->invert(n); }		// 制造一个 inline 调用，调用 base class 版本的 invert
												// 使用this是为了避免模板化基类中的函数名称被遮盖
		...
	private:
		T data[n*n];
	}

子类写法二：

	template<typename T, std::size_t n>
	class SquareMatrix : private SquareMatrixBase<T> {
	private:
	    using SquareMatrixBase<T>::invert;
	public:
	    SquareMatrix()
		 : SquareMatrixBase<T>(n, 0),
		pData(new T[n*n])
		{ this->setDataPtr(pData.get()); }
		...
	    void invert() { this->invert(n); }
		...
	private:
		std::shared_ptr<T[]>;
	}

- 对**同样类型**的矩阵（比如 double），上述改进可以实现**忽视矩阵大小共用一份 insert 代码**。
- 本例中，父类的函数仅在子类中用到，所以使用 protected 权限。
- 子类的 inline insert 函数，可以保证额外成本为0。
- 调用父类 insert 时使用 this，是为了避免模板化基类中的函数名称被遮盖，见条款 43。
- 继承关系是 private，表示 base class 只是为了帮助 derived classes 实现，见条款 39。
- 为了 base class 的 insert 可以操作 derived classes 的数据，需要在 base class 体内保存指向数据的指针。
- 子类写法一 不需要动态分配内存，但对象自身可能非常大。
- 子类写法二 需要动态分配内存，但对象仅需要增加一个指针大小。

#### 三. 注意
上述改进有利有弊：  

- 优点：
	- 减少可执行文件大小，降低换页频率，强化指令高速缓冲区内的引用集中化。
- 缺点：
	- 最初造成代码冗余的版本（也就是 一. 问题引入 中，和矩阵大小相关的版本），尺寸是个编译期常量，因此可以藉由编译器的优化，实现常量的广传达到最优化，包括把它们折到被生成指令中成为直接操作数。
- 具体哪一种更有效率，需要实际运行计算。

#### 四. 类型参数导致的膨胀
本条款集中讨论 non-type template parameters（非类型模板参数）带来的膨胀。**其实 type parameters（类型参数）也会导致膨胀**。  
比如，很多平台上 int 和 long 有相同的二进制表述，导致生成的 vector<int> 和 vector<long> 成员函数可能完全相同。某些连接器会进行优化，某些则不会，导致代码膨胀。不过这不是我们能左右的。  
我们**需要特别注意的是涉及到指针的部分** 。大多数平台上所有指针类型都有相同的二进制表述。因此凡 templates 持有指针者（如 list<int\*>， list<const int\*>）往往应该对每一个成员函数使用唯一一份底层实现。这意味着**如果你实现某些成员函数而它们操作强类型指针（strongly typed pointers，即 T\*），你应该令它们调用另一个操作无类型指针（untyped pointers，即 void\*）的函数，由后者统一完成实际工作。**

#### 五. 总结
- templates 生成多个 classes 和多个函数，所以任何 template 代码都不该与某个造成膨胀的 template 参数产生相依关系。
- 因非类型模板参数（non-type template parameters)而造成的代码膨胀，往往可消除，做法是以函数参数或 class 成员变量替换 template 参数。
- 因类型参数（type parameters)而造成的代码膨胀，往往可降低，做法是让带有完全相同二进制表述（binary representations)的具现类型（instantiation types)共享实现码。