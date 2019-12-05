#### 一. 问题引入
考虑有理数乘法问题：  

	class Rational{
	public:
	    Rational(int numerator = 0, int denominator = 1);
		const Rational operator*(const Rational& rhs);
	    int numerator() const;
	    int denominator() const;
	private:
	    int numerator;
	    int denominator;
	}

客户可能出现以下几种合理的使用方式：  

	// 相同类型相乘
	Rational oneEight(1,8);
	Rational onehalf(1,2);
	Rational result = oneHalf * oneEight;	// nice
	result = result * oneEight;				// ok
	
	// 混合类型相乘
	result = oneHalf * 2;					// ok 2发生了隐式类型转换。	(1)
	result = 2 * oneHalf;					// wrong ! 					(2)

编译器对operator\*调用的**查找顺序**：  

- 确定**相关类**内是否有参数数目符合的成员函数。
	- 语句(1)，**相关类**是Rational，其中有operator\*。  
	- 语句(2)，**相关类**是int，其中没有operator\*。
- 如果**相关类**内有参数数目符合的成员函数，看是否可以套用。**可能发生隐式转换**：
	- 语句(1)，需要的两个参数类型都是Rational类型，参数2类型不符。编译器尝试将2**隐式转换**为Rational类型。成功通过构造函数转换，于是调用Rational::operator\*。
- 如果**相关类**内没有参数数目符合的成员函数，则在命名空间作用域内或global作用域内查找符合的全局函数。**可能发生隐式转换**：
	- 语句(2)，没有找到任何符合参数数目且可以应用隐式转换的全局函数，语法错误。

特别的，**只有当参数位于参数列内时，这个参数才有可能参与隐式类型转换**。比如上述语句(1)和(2)，相当于：  

	result = oneHalf.operator*(2);
	result = 2.operator*(oneHalf);

分别只能对“2”和“oneHalf”实施隐式类型转换。

#### 二. 改进
声明non-member函数。  

	class Rational {            // 不包含operator*
    	...
	};
	
	const Rational operator*(const Rational& lhs,   // 成为一个non-member函数
	                         const Rational& rhs)
	{   
	    return Rational(lhs.numerator() * rhs.numerator(),
	                    lhs.denominator() * rhs.denominator());
	}

	Rational oneForth(1, 4);
	Rational result;
	result = oneForth * 2;      	// ok
	result = 2 * oneForth;      	// ok

这样，编译器可以在每一个实参上执行隐式类型转换，调用ok。  

特别的，该non-member函数不应该成为Rational的友元函数，因为它需要的所有操作都可以通过public接口完成。而friend会破坏封装性(条款23)。  

本条款的原则更适用于C\+\+的面向对象部分。当跨入C\+\+ 模板部分时，需要考虑新争议、新解法以及一些设计牵连。将在条款46中讨论。  

#### 三. 总结
- 如果你需要为某个函数的所有参数（包括被this指针所指的那个隐喻参数）进行类型转换，那么这个函数必须是个non-member。