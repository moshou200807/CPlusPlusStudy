#### 一. 综述
- 宁可拿non-member non-friend函数替换member函数。这样做可以增加封装性、包裹弹性和机能扩充性。  

#### 二. 解释
假设有个class用来表示网页浏览器，提供各种接口。  
	
	class WebBrowser {
	public:
		...
		void clearCache();
		void clearHistory();
		void removeCookies();
		...
	};

许多用户想一次性执行所有这些清除操作，因此也提供这样一个函数：  
	
	class WebBrowser {
	public:
		...
		void clearEverything(); 	// 调用clearCache, clearHistory, 和 removeCookies
		...
	};

另外一种选择是提供non-member non-friend函数执行这些操作：  

	void clearBrowser(WebBrowser& wb)
	{
		wb.clearCache();
		wb.clearHistory();
		wb.removeCookies();
	}

应该选择哪个？  
(应该选择non-member non-friend函数，并把它们置于同一个命名空间。)

##### 2.1 封装性
考虑对象内的数据。越多函数可以直接访问它，那么数据的封装性越低。  
对于private访问权限的数据成员来说，可以直接访问它的函数只有class的member函数加上friend函数。  
所以从封装性的角度讲，non-member non-friend函数有更大的封装性优势。  

##### 2.2 包裹弹性
class不可以跨越源码文件，而多个non-member non-friend函数却可以。  

	// 头文件 "webbrowser.h" – 针对WebBrowser自身及其核心机能
	namespace WebBrowserStuff {
	    classWebBrowser { ... };
	    void clearBrowser(WebBrowser& wb);
	    ...
	}
		
	// 头文件 "webbrowserbookmarks.h"
	namespace WebBrowserStuff {
		... // 书签相关的便利函数
	}
	 
	// 头文件 "webbrowsercookies.h"
	namespace WebBrowserStuff {
		... // cookie相关的便利函数
	}

	...

这样，客户可以自由包含需要的头文件，而无需与不感兴趣的函数发生编译依赖，提高包裹弹性。  
标准库命名空间std内散列各处的vector、map等等，也是类似的组织方式。

##### 2.3 机能扩充性
将所有提供便利性的函数放入多个头文件但隶属于一个namespace中，意味着客户能容易地扩充便利函数的集合，要做的只是在namespace中加入更多的非成员非友元函数。  
例如，如果一个 WebBrowser的客户决定写些关于图像下载的便利函数，他只需要新建一个头文件，把新函数放在WebBrowserStuff namespace内声明。这些新函数现在就像其它便利函数一样可用并集成。这是类不能提供的另一个特性，因为类定义对于客户是不能扩展。当然，客户可以派生新类，但是派生类不能访问基类中被封装的（即private）成员，所以这样的“扩充机能”只拥有次等身份。
