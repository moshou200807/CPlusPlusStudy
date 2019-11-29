#### 一. 问题引入
C++编译器对一条表达式内的多个语句的执行顺序有重新排列的自由。也就是说，**同一条表达式内的多条语句的执行顺序不确定**(除非是函数调用及参数、短路等情况)。  

	// API
	int priority();
	void processWidget(std::tr1::shared_ptr<Widget> pw, int priority);

	// 使用情形一
	processWidget(new Widget, priority());		// error，编译错误。shared_ptr可以用指针构造，但是该构造函数是explicit的，无法进行隐式转换。

	// 使用情形二
	processWidget(std::tr1::shared_ptr<Widget>(new Widget), priority());

编译器生成对 processWidget 的调用代码之前，必须首先核算即将被传递的各个实参。也就是在调用 processWidget之前，编译器必须为这三件事情生成代码：  

- 调用 priority。(1)
- 执行 “new Widget”。(2)
- 调用 tr1::shared_ptr 的构造函数。(3)

由于语句(2)是语句(3)的参数，(2)一定在(3)之前执行；但是(1)可能会在任何时候执行：  

- 执行 “new Widget”。(2)
- 调用 priority。(1)
- 调用 tr1::shared_ptr 的构造函数。(3)

如果语句(1)执行异常，那么语句(2)返回的指针将会遗失，无法被我们想要构造的智能指针(3)处理，造成资源泄漏。  

为了防止**在“资源被创建(经由new Widget)”和“资源被转换为资源管理对象(语句3)”两个时间点之间可能发生的异常干扰**，需要使用独立语句将newed对象置入智能指针。

#### 二. 改进

	std::tr1::shared_ptr<Widget> pw(newWidget); // 在单独语句内以智能指针存储newed所得对象
	processWidget(pw, priority());   // 这个调用动作绝不至于造成泄漏

这样，C++编译器必须以顺序执行两个表达式，消除了可能的异常干扰导致的资源泄漏。

#### 三. 总结
- 以独立语句将newed对象存储于(置入)智能指针内。如果不这样做，一旦异常被抛出，有可能导致难以察觉的资源泄漏。