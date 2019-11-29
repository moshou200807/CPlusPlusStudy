设计高效的classes应该像设计语言内置类型般谨慎。以下主题是在设计时应该考虑的：  

- **新type的对象应该如何被创建和销毁？**：  
影响到class的构造函数、析构函数以及内存分配函数和释放函数(operator new，operator new[]，operator delete和operator delete[]。见条款49~52.)的设计。  
- **对象的初始化和对象的赋值该有什么样的差别？**：  
决定你的构造函数和赋值操作符的行为及差异(条款4)。  
- **新type的对象如果被passed by value，意味着什么？**：  
copy构造函数定义一个type的pass-by-value如何实现。  
- **什么是新type的“合法值”？**：  
class的成员变量通常只有某些数值集是有效的。那些数值集决定了class必须维护的约束条件。  
也就是class的成员函数(特别是构造函数、赋值操作符和所谓“setter”函数)必须进行的错误检查工作。  
它也影响函数抛出的异常、(极少被使用的)函数异常明细列(exception specifications)。  
- **你的新type需要配合某个继承图谱吗？**：  
如果继承自某些既有的classes，就会受到那些classes的设计的束缚，特别是受到“它们的函数是virtual或non-virtual”的影响(条款34和条款36)。  
如果允许其他classes继承你的class，那会影响你所声明的函数(尤其是析构函数)是否为virtual(条款7)。  
- **你的新type需要什么样的转换？**：  
需要考虑新type与其他types之间是否应该存在转换行为。  
如果你希望允许T1**隐式转换** 为T2，就必须在T1内写一个类型转换函数(operator T2)或在T2内写一个non-explicit-one-argument构造函数。  
如果你只允许explicit构造函数存在(意味着不允许隐式转换)，就得写出专门负责执行**显式转换**的函数，且不得为类型转换操作符或non-explicit-one-argument构造函数。(条款15有隐式和显式转换函数的范例。)  
- **什么样的操作符和函数对此新type而言是合理的？**：  
将决定你为class声明哪些函数。其中某些该是成员函数，某些则不是(条款23,24,26)。  
- **什么样的标准函数应该驳回？**：  
这些函数必须声明为private(条款6)。  
- **谁该取用新type的成员？**：  
决定哪个成员为public、protected、private；决定哪个classes、functions应该是friends，以及将它们嵌套于另一个之内是否合理。  
- **什么是新type的“未声明接口”(undeclared interface)？**：  
它对效率、异常安全性(条款29)以及资源运用提供何种保证？  
在这方面提供的保证将为你的class实现代码加上相应的约束条件。  
- **你的新type有多么一般化？**：  
或许你其实并非定义一个新type，而是定义一整个types家族。这样，就应该定义一个class template而不是一个新class。  
- **你真的需要一个新type吗？**：  
如果只是定义新的derived class以便为既有的class添加功能，那么说不定单纯定义一或多个non-member函数或templates，更能够达到目标。