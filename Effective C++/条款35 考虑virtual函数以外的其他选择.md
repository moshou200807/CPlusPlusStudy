#### 一. 问题引入
在制作游戏时，任务受伤或因其他因素降低健康状态的情况很常见。  
提供一个成员函数 healthValue，返回表示人物健康程度的整数。  
不同的任务可能以不同的方式计算他们的健康指数，那么将 healthValue 声明为 virtual 似乎顺理成章。  

	class GameCharacter {
    public:
        virtual int healthValue() const;	// derived classes 可重新定义
        ...
    };

除了 virtual 设计，还有其他替代方案。  

#### 二. 替代方案
##### 2.1 NVI实现模板方法模式
为了便于讲述，直接在声明时给出定义，并无内联需求。  
NVI（non-virtual interface）手法，是模板方法模式的一种独特表现形式。它的思想是：给客户暴露 non-virtual 函数；子类按需实现各自的 private virtual 函数。  

	class GameCharacter {
    public:
        int healthValue() const				// 暴露 non-virtual 函数，并且子类不重新定义它（见条款 36）
        {
            ...  							// 做事前工作
            int retVal=doHealthValue();		// 真正做实际工作
            ... 							// 做事后工作
            return retVal;
        }
        ...
    private:
        virtual int doHealthValue() const 	// 子类可重新定义
        {
            ...								// 缺省算法
        }
    };

说明：  

- non-virtual 函数可以在调用 virtual 函数前后做额外的动作，包括加锁、写日志等。
- NVI手法下的 virtual 函数不一定必须是 private，也可以是 protected。但是不应该是 public（因为 NVI 意思是非虚函数**接口**）。

##### 2.2 函数指针实现策略模式
该模式下，人物健康指数的计算与人物类型无关，计算函数仅需要获取人物的某些数据。  

	class GameCharacter;								// forward declaration
    int defaultHealthCalc(const GameCharacter& gc);		// 缺省算法
    
	class GameChaaracter {
    public:
        typedef int(*HealthCalcFunc)(const GameCharacter&);
        explicit GameCharacter(HealthCalcFunc hcf = defaultHealthCalc)
            : healthFunc(hcf)
        {}
        int healthValue() const
        { return healthFunc(*this); }
        ...
    private:
        HealthCalcFunc healthFunc;						// 函数指针
    };

与“基于GameCharacter继承体系内的virtual函数”比较，提供了某些弹性：  

- 同一人物类型的不同实体可以有不同的健康计算函数，只要在构造时传递不同的函数指针。
- 某已知人物的健康计算函数可以在运行期变更，只要提供成员函数替换内部的函数指针。
- **特别的，健康计算函数只能访问人物的 public 成分**。如果要访问其他封装权限的成分，那么只能降低人物的封装性（声明为 friend 或提供更多的 public 接口）。

##### 2.3 std::function实现策略模式
改用 std::function 可以进一步降低 2.2 的限制，将函数指针扩展为任何可调用物（包括函数指针、函数对象或成员函数指针）。  

	class GameCharacter;
    int defaultHealthCalc(const GameCharacter& gc);
    
	class GameCharacter {
    public:
        typedef std::function<int (const GameCharacter&)> HealthCalcFunc;		// std::function
        explicit GameCharacter(HealthCalFunc hcf = defaultHealthCalc)
            : healthFunc(hcf)
        {}
        int healthValue() const
        { return healthFunc(*this); }
        ...
    private:
        HealthCalcFunc healthFunc;
    };

	short calcHealth(const GameCharacter&);			// 1. 健康计算函数，返回类型不是int 但是可以隐式转换为int

    struct HealthCalculator {						// 2. 为计算健康设计的函数对象
        int operator()(const GameCharacter&) const
        { ... }
    };

    class GameLevel {
    public:
        float health(const GameCharacter&) const;	// 3. 成员函数计算健康，返回不是int 但是可以隐式转换为int
        ...
    };

    class EvilBadGuy: public GameCharacter {		// 人物一
    public:
		explicit EvilBadGuy(HealthCalFunc hcf = defaultHealthCalc)
            : GameCharacter(hcf)
		...
    };

    class EyeCandyCharacter: public GameCharacter {	// 另一个人物，假设其构造函数和EvilBadGuy相同
        ...
    };

    EvilBadGuy edg1(calcHealth);					// 人物1，使用某个函数计算健康指数
    EyeCandyCharacter ecc1(HealthCalculator());		// 人物2，使用函数对象计算健康指数

    GameLevel currentLevel；
    ...
    EvilBadGuy ebg2(std::bind(&GameLevel::health, currentLevel， _1));	//人物3，使用某个成员函数计算健康指数

##### 2.4 经典策略模式
将健康计算函数做成另一个继承体系中的 virtual 成员函数。  

	class GameCharacter;				// forward declaration
    class HealthCalcFunc {
    public:
        ...
        virtual int calc(const GameCharacter& gc) const
        { ... }
        ...
    };

    HealthCalcFunc defaultHealthCalc;
    
	class GameCharacter {
    public:
        explicit GameCharacter(HealthCalcFunc* phcf = &defaultHealthCalc)
         : pHealthCalc(phcf)
        {}
        int healthValue() const
        { return pHealthClac->calc(*this); }
        ...
    private:
        HealthCalcFunc* pHealthCalc;		// 保存另一个继承体系内的 父类 指针。可以接受 子类 对象。
    };

熟悉标准策略模式的人很容易辨认它。

#### 三. 总结
- virtual函数的替代方案包括 NVI手法 及 策略设计模式的多种形式。 NVI手法 自身是一个特殊形式的模板方法设计模式。
- 将实现从成员函数移到 class外部函数，带来的一个缺点是，非成员函数无法访问 class 的 non-public 成员。
- std::function对象的行为就像一般函数指针。这样的对象可接纳“与给定之目标签名式兼容”的所有可调用物。