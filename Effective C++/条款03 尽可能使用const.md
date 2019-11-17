> const用于指定一个语义约束，表明某对象不该被改动。  

#### 一. const修饰变量
const多才多艺，可以：  
- 在classes外部修饰global或namespace作用域中的常量；  
- 修饰文件、函数、区块作用域中的static对象；  
- 修饰classes内部的static和non-static成员变量；  
- 对于指针，可以分别指定指针本身或所指之物为const类型；  
- 对于迭代器：声明迭代器为const就如同声明指针本身为const。如果需要迭代器所指内容不可被改动，那么需要的是const_iterator。  

		const int test = 1;
		int const test = 1; //对于普通变量，const的位置在类型前后的意义等价

		char greeting[] = "Hello";
		//对于指针，取决于const出现在 * 左边或右边：出现在左边，表示指针所指之物为常量；出现在右边，表示指针本身为常量：
		char* p = greeting;         //non-const pointer,non-const data
		const char* p = greeting;   //non-const pointer,const data
		char* const p = greeting;   //const pointer,non-const data
		const char* const p = greeting; //const pointer,const data

		//对于迭代器，声明迭代器为const类型 如同声明指针本身为const类型
		std::vector<int> vec;
		const std::vector<int>::iterator iter = vec.begin();	// 迭代器本身为const类型。实际上，迭代器是一个类，内部包含指向容器某部位的指针。
		*iter = 10; //ok。	试图修改迭代器对象内部指针所指的值，并没有修改迭代器对象本身的内容，合法。
		++iter;     //error	。 试图修改迭代器对象内部指针本身的值，非法。

		std::vector<int>::const_iterator cIter = vec.begin();	// 迭代器本身非const，迭代器内部的指针为 const T * 类型。
		*cIter = 10; //error。	试图修改迭代器内部指针所指的值。而迭代器内部指针指向的值为const类型。非法。
		++cIter;     //ok。	试图修改迭代器内部指针本身的值。迭代器内部指针本身不是const类型，合法。

#### 二. const修饰函数参数及返回值
##### 2.1 修饰函数参数
防止传入的参数在函数体内被改变，特别是指针或引用类型。  
除非有需要改动参数，否则请将函数参数声明为const。  

##### 2.2 修饰函数返回值
防止返回值被修改，降低因客户错误造成的意外。  

	class Rational{};
	const Rational operator* (const Rational& lhs, const Rational& rhs);

	Rational a, b, c;
	(a * b) = c; // 用const修饰返回值的话，这样的错误就能在编译阶段被立马侦查出来：企图赋值给const类型的变量

#### 三. const修饰成员函数
##### 3.1 const与非const同名函数的重载关系
两个成员函数即使只是常量性不同，也可以被重载。同样的，基于一个**指针或引用形参**是否为指向const，也可以被重载。  

	class TextBlock {
	public:
		const char& operator[](std::size_t position) const	// operator[] for const对象
		{ return text[position]; }
		char& operator[](std::size_t position)				// operator[] for non-const对象
		{ return text[position]; }
	private:
		std::string text;
	};

	TextBlock tb("Hello");
	std::out << tb[0];	// 调用non-const TextBlock::operator[]
	const TextBlock ctb("World");
	std::out << ctb[0];	// 调用const TextBlock::operator[]

##### 3.2 bitwise constness和logical constness
- **bitwise constness**：  
	- C++实际执行的常量性定义。
	- const成员函数**不改变对象内任何non-static成员变量**。(仍然可以改变static成员变量。因为它不是属于某对象的，而是类的对象共有的)。  
	- 对指针成员所指对象的改变，没有严格要求。  
- **logical constness**:
	- const成员函数**可以修改它所处理的对象内的某些bits，但条件是客户端侦测不出来**(客户端无法直接或间接获取这些bits)。  
	- C++编译器不直接支持。但是利用**mutable**释放non-static成员变量的bitwise constness约束。  


			class CTextBlock {  
			public:  
			    ...  
			    std::size_t length() const;  
			private:  
				char* pText;  
				mutable std::size_t textLength; 	// 利用mutable释放bitwise constness约束
				mutable bool lengthIsValid;
			};  
			std::size_t CTextBlock::length() const  
			{  
				if (!lengthIsValid) {  
					textLength = std::strlen(pText);  //现在可以在const成员函数中修改这些non-static成员变量了。
					lengthIsValid = true;
				}                           
				return textLength;
			}

##### 3.3 在const和non-const成员函数中避免重复
当const和non-const函数拥有**实质等价的实现**时，可以利用**常量性消除**避免重复代码。  
注意， **not-const版本调用const版本是安全的，反之不成立** 。  

	class TextBlock {  
	public:  
		...  
		const char& operator[](std::size_t position) const
		{  
		     ...  
	    	 return text[position];  
		}  

		char& operator[](std::size_t position)    //现在只调用const op[]  
		{  
			return  
				const_cast<char&>(          //将op[]返回值的const转除。返回值常量性消除。  
				   static_cast<const TextBlock&>(*this)   //为*this加上const  
	                              [position]              //调用const op[]  
			);  
		}  
		...  
	};

包含两个转型动作：  
- 将*this从TextBlock& 转型为const TextBlock&，使其调用const版本的成员函数。安全转型，使用static\_cast。  
- 移除const版本成员函数返回值的const属性。使用const\_cast。  

#### 四. 总结
- 将某些东西声明为const可利用编译器的帮助侦测出错误用法。const可被施加于任何作用域内的对象、函数参数、函数返回类型、成员函数本体。
- 编译器强制实施bitwise constness，但你编写程序时应该使用"概念上的常量性"（conceptual constness）。  
- 当const和non-const成员函数有着实质等价的实现时，令non-const版本调用const版本可避免代码重复。
