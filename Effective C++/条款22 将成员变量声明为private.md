#### 一. 总结
- 切记将成员变量声明为private。这可赋予客户访问数据的一致性、可细微划分访问控制、允诺约束条件获得保证，并提供class作者以充分的实现弹性。
- protected并不比public更具有封装性。  

#### 二. 解释
##### 2.1 访问数据一致性
客户不需要一会儿通过函数访问、一会儿又直接访问，造成迷惑。  

##### 2.2 可细微划分访问控制
对private数据，有选择地提供访问权限。  

	class Person{
    public:
        ...
        string getName() const {
            return name_;
        }
        string setAddress(string address) {
            address_ = address;
        }
        void setAge(unsigned char age) {
            age_ = age;
        }
        unsigned char getAge()const {
            return age_;
        }
    private:
        string name_;   		// 提供只读权限
        string address_; 		// 提供只写权限
        unsigned char age_; 	// 提供读写权限
    };

##### 2.3 允诺约束条件获得保证&提供class作者以充分的实现弹性
作者可以修改getter、setter的实现，添加想要的效果。客户只需要重新编译，不需要修改代码。  

	void Person::setAge(unsigned char age) {
		if (age < 160)				// 添加约束条件；提供实现弹性
    		age_ = age;
    }

##### 2.4 protected并不比public更具有封装性
protected成员变量依然可以被派生类直接访问。只要基类的protected成员变量被取消，所有直接使用它的派生类都会被影响，因此protected成员变量就像public成员变量一样缺乏封装性。  
从封装的角度看，其实只有两种访问权限：private(提供封装)和其他(不提供封装)。