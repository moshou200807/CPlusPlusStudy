#### 一. private继承意味着“根据某物实现出”
##### 1.1 private继承不是is-a关系

	class Person { ... };
	class Student: private Person { ... };	// private继承
	void eat(const Person& p);     	 		// 任何人都会吃
	void study(const Student& s);  			// 学生才在校学习
	Person p;
	Student s;
	eat(p);    								// 没问题
	eat(s);    								// 错误！难道学生不是人？

如果 classes 之间的继承关系是 private， 编译器不会自动将一个 derived class 对象转换为一个 base class 对象，这和 public 继承的情况不同。也就意味着 private 继承并非 is-a 关系。  
另外， private 继承而来的所有成员，在 derived classes 内都会变成 private。  

##### 1.2 private继承是is-implemented-in-term-of关系
如果 D 以 private 形式继承 B，意思是 D 对象根据 B 对象**实现**而来，再也没有其他含义了。这也说明，private继承在**软件设计**层面上没有意义，其意义只涉及到**软件实现**层面。private 继承纯粹是一种实现技术。    
借用条款 34 的术语，private 继承意味着只有实现部分被继承，接口部分应略去（也就是 base class 的接口或作用于其上的函数，并不保证可以作用于 derived classes）。  
这里的 is-implemented-in-term-of（根据某物实现出）与条款 38 的复合意义完全相同。**尽可能使用复合，必要时才使用 private 继承**。  

何谓**必要**：  

- 当 protected 成员和/或 virtual 函数牵扯进来的时候。
- 当占用空间方面的利害关系很重要的时候。  

#### 二. 详细说明
##### 2.1 必要一：protected成员和/或 virtual 函数牵扯进来
假设要统计 Widget class 的使用情况，我们需要运行期间周期性地审查信息，这就要用到定时器。  
假设已经有现成的定时器代码：  

	class Timer {
	public:
		explicit Timer(int  tickFrequncy);
	    virtual void onTick() const;   // 定时器每滴答一次，此函数就被自动调用一次
		...
	};

我们可以重新定义 virtual 函数，让其取出 Widget 的当时的状态。这里需要用到继承，但不能是 public 继承（并非 is-a 关系），而应该是 private 继承。  

	class Widget: private Timer {
	private:
		virtual void onTick() const;   // 查看Widget的数据等
		...
	};

由于 private 继承， Timer 的 public 函数 onTick 在 Widget 内变成 private，而我们重新声明时仍应该把它留在那儿，否则会误导客户端让他们以为可以调用它。  

这种需求也可以**改用复合关系**实现：  

	class Widget {
	private:
		class WidgetTimer: public Timer {		// 声明一个 private 内部嵌套类，public 继承 Timer
	    public:
	  		virtual void onTick() const;
			...
		}
		WidgetTimer timer;						// 复合使用内部嵌套类
		...
	};

**复合关系实现的好处**：  

- **防止 Widget 的子类重新定义 onTick**。即使是 private 继承也无法阻止子类重新定义 onTick，但是如果 WidgetTimer 成为父类的 private 成员，那么子类无法继承它或重新定义它的 virtual 函数。
- **降低编译依存性**。继承必须包含父类头文件，而复合可以通过前置声明不包含头文件。如果 WidgetTimer 移出 Widget 外，而 Widget 内含 WidgetTimer 指针，那么可以不包含 WidgetTimer 头文件。

##### 2.2 必要二：占用空间方面的利害关系很重要时
###### 2.2.1 空类
当处理的 classes **没有 non-static 成员变量、没有 virtual 函数，也没有 virtual base classes时（体积上的额外开销见条款 40）**，它不带任何数据，可视为 empty classes。它的对象不使用任何空间，因为没有任何隶属对象的数据需要存储。  
然而由于技术上的理由，C\+\+ 规定凡是**独立对象**都必须有非零大小。  

	class Empty { };  	// 没有数据
	class HoldsAnInt {	
	private:
	    int x;
		Empty e;   		// 应该不需要内存
	}

大多数编译器中， sizeof(Empty) = 1。因为面对“大小为零的独立对象”时，C\+\+要求默默安插一个 char 到空对象内。  
由于对齐要求（见条款 50）， sizeof(HoldsAnInt) = sizeof(int) + sizeof(int) 可能是多数编译器最终的结果。  

**非独立对象**通常作为 base class 成分：  

	class HoldsAnInt: private Empty {	
	private:
	    int x;
	}

这种情况下， sizeof(HoldsAnInt) == sizeof(int) 几乎是可以确定的。这就是所谓的**EBO(empty base optimization，空白基类最优化)**。  
需要注意的是，EBO 一般只在单一继承下才可行。  

###### 2.2.2 现实中的空类
现实中的空类虽然从未拥有 non-static 成员变量，却往往包含 typedefs、enums、static成员变量 或 non-virtual 函数。包含这些内容并不妨碍它们成为空类。    
STL 就有许多技术用途的 empty classes，其中内含有用的成员，通常是 typedefs。  
由于 EBO，这样的类作为单一父类基本上不会增加子类的大小。

###### 2.2.3 选择private继承
当 EBO 生效并且十分在意空间占用时，选择 private 继承而不选择复合关系。

#### 三. 总结
- private继承 意味 is-implemented-in-term-of（根据某物实现出）。它通常比复合的选择优先级低。但是当 derived class 需要访问 protected base class 的成员，或需要重新定义继承而来的 virtual函数时，这么设计是合理的。
- 和复合不同，private继承 可以造成 empty base 最优化。这对于空间要求高的开发而言很重要。