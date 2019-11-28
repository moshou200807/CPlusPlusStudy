#### 一. 原因
关键字**多态**、**基类**表明：在删除对象时，用户直接delete的是基类对象指针。  
如果该基类指针指向了派生类对象(这是多态的常见场景)，并且基类析构函数没有声明virtual，那么结果未定义(C++规定)。通常表现是派生类的成分没有被销毁。  

#### 二. 什么时候应该声明virtual析构函数
只有当base class内含至少一个virtual函数，才为它声明virtual析构函数：    
- **基类**：类的设计目的就是被派生。  
- **多态**：类具有其他virtual函数。这样才存在通过delete基类指针完成派生类析构的使用场景。  
  

例：  

- 不存在base class属性的class不应该定义virtual析构函数：  

		class Point {
		public:
			Point(int xCoord, int yCoord);
			~Point();	// 非virtual析构函数，Point对象仅占用2 * sizeof(int)的大小；
						// 如果不必要地声明为virtual，则加入虚函数表指针，导致对象变大。
						// 而且会导致不再和其他语言(如C)内的相同声明有同样的结构，也就不能直接传递给其他语言所写的函数。
		private:
			int x, y;
		};

- 被设计为base classes，但是没有多态用途的class，不应该设计virtual析构函数：如条款6的Uncopyable、input_iterator_tag、实际使用中很多基类等。

- 既不被设计为base classes，又没有多态用途的class不应该被继承。  
如果你曾经企图继承带有non-virtual析构函数的class，请拒绝诱惑。这些class包括STL标准容器、string等。  
另外，为自己编写的class加入final声明也是一个好习惯。  

- 在需要的时候令class带一个pure virtual析构函数：  
如果你希望拥有一个抽象class并且存在多态使用情景，但是手上没有任何pure函数，那么可以尝试将析构函数定义为pure virtual。  
抽象class总是企图被当做一个base class来用；如果存在多态使用情景，那么base class应该有virtual析构函数。  

		class AWOV {
		public:
			virtual ~AWOV() = 0;
		};

		AWOV::~AWOV() {}		// 应该在类体外提供一份实现，否则连接时出错：派生类析构时，会调用基类析构函数。如果找不到基类虚构函数的实现，连接失败。
								// 不影响类体内的pure virtual声明。

#### 三. virtual影响对象大小
声明virtual函数的类的对象，内部包含一个vptr(virtual table pointer,虚表指针)。  
vptr指向一个由函数指针构成的数组，称为vtbl(virtual table,虚表)。  
当对象调用某virtual函数时，实际被调用的函数取决于该对象的vptr所指的那个vtbl--编译器在其中寻找适当的函数指针。  

#### 四. 总结
- polymorphic(带多态性质的) base classes应该声明一个virtual析构函数。如果class带有任何virtual函数，表明它被设计为带多态性质的base classes，它就应该拥有一个virtual析构函数。
- classes的设计目的如果不是作为base classes使用，或不是为了具备多态性(polymorphically)，就不该声明virtual析构函数。