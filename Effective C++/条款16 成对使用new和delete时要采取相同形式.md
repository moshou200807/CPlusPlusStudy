#### 一. 问题引入
指针所指是单一对象还是对象数组，在内存布局上有根本区别。  

名称 | 布局  
-- | --  
单一对象 | Object  
对象数组 |  n Object Object Object ...  

对象数组所用内存通常包含数组大小的记录。  

当你对着一个指针使用delete，唯一能够让delete知道内存中是否存在一个“数组大小记录”的办法就是：由你使用中括号来告诉它。  
因此，如果用delete删除一个对象数组或用delete[]删除单一对象，都**有可能由于内存错位导致未定义行为**。  

#### 二. 改进
new和delete配套使用；  
new[]和delete[]配套使用。  

特别注意，尽量不要用typedef定义数组形式：  

	typedef std::string AddressLines[4];       // 每个人的地址有4行，每行是一个string
 
	std:string* pal = new AddresLines;        // AddresLines是一个数组。相当于：new string[4];

	delete pal;         // 行为未定义
	delete[] pal;      // 很好

#### 三. 总结
- 如果你在new表达式中使用[]，必须在相应的delete表达式中也使用[]。如果你在new表达式中不使用[]，一定不要在相应的delete表达式中使用[]。