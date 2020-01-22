#### 一. 问题引入
矩形可以由左上、右下两个点决定。  

    class Point{
    public:
        Point(int x, int y);
        ...
        void setX(int newVal);
        void setY(int newVal);
        ...
    };

    struct RectData{
        Point ulhc;
        Point lrhc;
    };

    class Rectangle{
    public:
        ...
        Point& upperLeft()const {return pData->ulhc;}
        Point& lowerRight()const {return pData->lrhc;}

    private:
        std::tr1::shared_ptr<RectData> pData;
    };

观察upperLeft()、lowerRight()函数，虽然它们被声明为const，但C\+\+实际执行的const政策(条款3)，仅保证其内部pData不会被改变，pData指向的对象不在该const管辖范围。因此，客户可能通过以下方式非预期地改变矩形Point点：  

    Point coord1(0,0);
    Point coord2(100,100);
    const Rectangle rec(coord1, coord2);
    rec.upperLeft().setX(50); // 现在 rec 变成从 (50,0) 到 (100,100)

两条教训：  
- 成员变量的封装性最多只等于 “返回其 reference” 的函数的访问级别。成员函数也是如此。  
- 如果 const 成员函数传出一个 reference，并且它是一个指针，指向实际数据，那么这个函数的调用者可以修改那笔数据。

#### 二. 改进
加入返回值 const 限定。  

    class Rectangle{
    public:
        ...
        const Point& upperLeft()const {return pData->ulhc;}
        const Point& lowerRight()const {return pData->lrhc;}

    private:
        std::tr1::shared_ptr<RectData> pData;
    };

const 不再是个谎言。至于封装性，这里是蓄意放松封装，有限度的放松：只让渡读取权，涂写权是禁止的。

其他问题：  
**返回 “代表对象内部” 的 handles，有可能导致 dangling handles（空悬的号码牌）。**  
只要handle被传递出去，你就暴露在“handle比其所指对象更长寿”的风险下：客户可能在整个对象被销毁后仍使用该对象的handle，比如迭代器失效！  

#### 三. 总结
- **handle可以是多种形式，包括指针、迭代器或 reference 。**  
- **避免返回 handle（包括 reference、指针、迭代器）指向对象内部。遵守这个条款可增加封装性，帮助 const 成员函数的行为像个 const，并将发生“虚吊号码牌” (dangling handles) 的可能性降至最低。**  
- **这不意味着你绝对不可以让成员函数返回handle，有时候你必须这么做。例如operator[]。**
