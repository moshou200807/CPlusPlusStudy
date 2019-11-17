> 以编译器替换预处理器。  

#### 一. 用const替换#define

	#define ASPECT_RATIO	1.653

	const double AspectRatio = 1.653

##### 1.1 替换的好处
- 符号表优势：编译器看不到记号名称ASPECT_RATIO。因此，在发生编译错误或进行符号式调试时，无法追踪该符号。
- 目标代码量优势：宏定义只进行简单的替换，会在目标代码内出现多份1.653；而浮点常量不会出现这种问题，目标代码量更小。 
- 安全性优势：const变量有数据类型检查。 

##### 1.2 注意事项
###### 1.2.1 定义常量指针
只读指针指向只读字符串的写法：  

	const char* const authorName = "Scott Meyers";

定义为只读string更佳：  

	const std::string authorName("Scott Meyers");

###### 1.2.2 定义class专属常量

	class GamePlayer{
	private:
	    static const int NumTurns = 5; //常量声明式
	    int scores[NumTurns];          //使用该常量
	};

注意该声明式。**通常C++要求在对使用的任何东西提供一个定义式**，如下：  

	const int GamePlayer::NumTurns;	//NumTurns 的定义

但如果它是一个class专属常量，而且为整数类型(ints,chars,bools)时，只要不取它们的地址，那么**可以不提供上述定义式**。  
如果编译器(不正确地)坚持要看到一个定义式，那么应该在**实现文件**而非头文件中放入该定义式。由于声明时已经获得初值，因此定义时不可以再设初值。  

请注意，我们无法利用#define创建一个class专属常量。**因为#define不重视作用域**。一旦宏被定义，它就在其后的编译过程中有效，除非在某处被#undef。这也意味着#define不能提供任何封装性(没有所谓private #define这种东西)，然而const成员变量是可以被封装的，如NumTurns。

旧式编译器也许不允许static成员在其声明式上获得初值；而且只允许用整数常量在类体内进行初值设定(非static)，那么可以将初值设定迁移到定义式中。  

	class CostEstimate{
	private:
	    static const double FudgeFactor;  //static class 常量声明
										  //位于头文件内
	};
	
	const double							//static class 常量定义
	    CostEstimate::FudgeFactor = 1.35;	//位于实现文件内

#### 二. 用enum替换#define
当class在编译期间需要一个class常量值，但是编译器(错误地)不支持static整数型class变量在class体内的初值设定时，可以改用所谓的"the enum hack"实现替代#define的作用。  

	class GamePlayer{
	private:
	    static const int NumTurns = 5; //常量声明式
	    int scores[NumTurns];          //使用该常量
	};

	//编译器不支持时，改用下面的写法：

	class GamePlayer{
	private:
	    enum { NumTurns = 5 }; 			//"the enum hack"令NumTurns成为5的一个记号名称：属于枚举类型的枚举元素数值可以充当ints被使用
	    int scores[NumTurns];          //使用该常量
	};

- enum hack的行为更像#define而不像const：  
enum和#define不可被**取址**；而const支持取址。  
enum和#define一样绝对**不会导致非必要的内存分配**；而不够优秀的部分编译器却可能为const对象设定另外的储存空间。  
- enum hack是模板元编程(TMP)的基础技术。

#### 三. 用inline替换#define
\#define的另外一种不恰当用法就是用来实现函数功能。  

	#define CALL_WITH_MAX(a, b)   f((a) > (b) ? (a) : (b))	// 以a、b的较大者调用f

虽然省去了函数调用带来的**额外开销**，但是使用这种形式的宏定义需要注意：   
- **为宏内的所有实参加上小括号** ，否则可能导致编译问题。  
- 自增类操作符可能会**多次自增**。  

可以利用**模板内联函数**，同时获得宏定义的效率提升和一般函数的可预料行为和类型安全性：  

	template<typename T>
	inline void callWithMax(const T& a, const T& b)
	// 因为我们不知道 T 的类型是什么，因此我们通过引用传递 const 参数。参见条款20.
	{
	  f(a > b ? a : b);
	}

此外，模板内联函数遵循作用域和访问规则，这也是比宏定义更优秀的地方。  

#### 四. 总结
- **对于单纯常量，最好以const对象或enums替换#defines。**  
- **对于形似函数的宏，最好改用inline函数替换#defines。**