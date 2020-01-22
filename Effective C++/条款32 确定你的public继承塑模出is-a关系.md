#### 一. 问题引入
公开继承意味着 is-a 关系。但是有时直觉可能会误导你，因此在设计时应该谨慎。  
 
鸟可以飞；企鹅是一种鸟，但是企鹅不可以飞。

	class Bird {
	public:
	      virtual void fly();   //鸟可以飞
	};
	 
	class Penguin: public Bird {  //企鹅是一种鸟
	      ...
	};

#### 二. 改进
##### 2.1 细分类
将“飞”从鸟中剔除出来，在编译期直接杜绝错误做法（推荐）：  

	class Bird
	{
		...  //没有声明fly函数
	};
	class flyBird : public Bird
	{
	public:
		virtual void fly();
		...
	};
	class Penguin: public Bird
	{
		...  //没有声明fly函数.
	};

##### 2.2 运行时错误
保留“飞”特性，在试图让企鹅飞时抛出运行时错误。  

	void error(const std::string& msg); //定义于其他某处
	class Penguin:public Bird
	{
	public:
		virtual void fly() { error(“attempt to fly”); }
		...
	};

#### 三. 总结
- is-a 并非唯一存在于 classes 之间的关系。另外两个常见的关系是 has-a 和 is-implemented-in-terms-of(根据某物实现出)，将在条款 38 和条款 39 讨论。
- “public继承” 意味着is-a。适用于 base classes 身上的每一件事情也适用于 derived classes 身上，因为每一个 derived class 对象也都是一个 bass class 对象。