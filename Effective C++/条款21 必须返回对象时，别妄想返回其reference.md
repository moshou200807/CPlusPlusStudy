#### 一. 问题引入
传递引用/指针的确可以提高效率，但是如果过度使用引用传递，则会招致超出预期的后果。  

	class Rational {
	public:
	   Rational(int numerator = 0, int denominator = 1);		// 条款24说明为什么这个构造函数不声明为explicit
	    ...
	private:
	   int n, d; 		// 分子和分母
	   friend const Rational operator* (const Rational& lhs, const Rational& rhs);		// 正确的声明
	};

如果在stack空间创建local对象并返回其引用，招致错误：  

	const Rational& operator*(constRational& lhs, const Rational& rhs)
	{
	   Rational result(lhs.n * rhs.n, lhs.d * rhs.d);	// 一次构造函数的成本
	   return result; 		 //  糟糕的代码！
	}

如上例，local对象result在函数退出前被销毁，返回的reference指向一个现如今不存在的Rational对象，如果使用它可能招致未定义行为。函数返回指针也是一样的效果。  

如果在heap空间创建local对象并返回其引用，也会招致错误：  

	const Rational& operator*(constRational& lhs, const Rational& rhs)
	{
	    Rational *result = new Rational(lhs.n * rhs.n, lhs.d * rhs.d);	// 一次构造函数的成本
	    return *result;		  // 更糟的写法！
	}

如上例，在使用如下写法时，很容易招致内存泄漏：  

	Rational w, x, y, z;
	w = x * y * z; // 与operator*(operator*(x, y), z)相同

new了两次，但是无法进行delete，明显的内存泄漏：**w通过返回的引用拷贝构造而来**；无法通过w取得返回的引用背后隐藏的指针。  

假如非要避免构造函数的成本，考虑static的使用：  

	const Rational& operator*(constRational& lhs, const Rational& rhs)
	{							// 又一堆烂代码！
	    static Rational result; // static对象，此函数返回其reference。 仅在第一次调用时有一次构造函数的成本
	    result= ... ;           // 将lhs乘以rhs，并将结果置于result内。 一次拷贝赋值的成本
		return result;
	}

任何内含local static变量的函数都应该考虑其线程安全性。此外，上例还有其他弱点：  

	bool operator==(const Rational& lhs, const Rational& rhs); 	// 一个针对Rationals写的operator==
	
	Rational a, b, c, d;
	...
	 
	if ((a * b) == (c * d)) {
		当乘积相等时，做适当的相应动作;
	} else {
		当乘积不等时，做适当的相应动作;
	}

表达式((a \* b) == (c \* d))将永远为true！因为它们所引用的是同一个static local对象！  
如果为了解决上述问题，引入local static数组，那绝对是灾难性的想法。且不说数组的大小、数据的映射问题，单单第一次调用的static数据初始化就是一笔很大的开销；此外，每次调用同样存在拷贝赋值的成本开销。而拷贝赋值对很多types而言，相当于调用一个析构函数(销毁旧值)加上一个构造函数(复制新值)，得不偿失。  

#### 二. 改进
一个“必须返回新对象”的函数的正确写法是：就让它返回一个新对象。  

	inline const Rational operator* (const Rational& lhs, const Rational& rhs)
	{
		return Rational(lhs.n * rhs.n, lhs.d * rhs.d);		// 一次构造函数的开销
	}

	Rational c = a * b;			// 一次析构函数(销毁函数内的local对象) + 一次拷贝构造函数(构造变量c) 的开销

实际上，仅函数operator\*而言，可能的额外开销只是一次析构 + 一次拷贝构造函数。  
之所以说“可能”，是因为C\+\+允许编译器实现最优化。在某些情况下，operator\*返回值的构造和析构可被安全地消除。  
所以，当你必须在“返回一个reference和返回一个object”之间抉择时，你的工作就是挑出行为正确的那个。其他的效率问题就让编译器厂商解决去吧。  

#### 三. 总结
- 绝不要返回pointer或reference指向一个local stack对象，或返回reference指向一个heap-allocated对象，或返回pointer或reference指向一个local static对象而有可能同时需要多个这样的对象。