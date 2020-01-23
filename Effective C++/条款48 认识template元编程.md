#### 一. 总结
- template metaprogramming（TMP，模板元编程）可将工作由运行期移到编译期，因而得以实现早期错误侦测和更高的执行效率。
- TMP 可被用来生成“基于政策选择组合”（based on combinations of policy choices）的客户定制代码，也可用来避免生成对某些特殊类型并不适合的代码。

#### 二. 说明
##### 2.1 什么是TMP
TMP 是以 C\+\+ 写成、执行于 C\+\+ 编译器内的程序，它的输出是从 templates 具象出来的若干 C\+\+ 源码，会在稍后继续被编译。

##### 2.2 TMP将某些工作从运行期转到编译期
- 某些错误本来通常在运行期才能侦测到，现在可在编译期找出来。
- 使用 TMP 的 C\+\+ 程序可能在每一方面都更高效：较小的可执行文件、较短的运行期、较少的内存需求。  

考虑 条款 47 的 advance 函数的运行期类型检测实现：  

	template<typename Iter, typename DistT>
    void advance(IteT& iter,DistT d)
    {
        if(typeid(typename std::iterator_traits<IterT>::iterator_category)
        	== typeid(std::random_access_iterator_tag))
            iter += d;					// 编译关键
        else {
            if(d >= 0)
                while(d--) ++iter;
            else 
                while(d++) --iter;
        }
    }

首先，这个解法比 traits 解法效率低。因为类型测试发生在运行期。  
其次，可能导致编译期问题：  

	std::list<int>::iterator iter;
    ...
    advance(iter,10);	// 错误！

编译错误的原因是：编译器必须确保所有源码都有效。advance要求参数 iter 支持执行 += 操作，不论它的实参是否真正会进入到那个分支。然而实参 list 的 iter 显然不支持 += 。  
TMP 解法则根据不同类型设计不同代码，不存在编译问题。

##### 2.3 TMP还能做什么
- 确保度量单位正确。
- 优化矩阵运算。
- 可以生成客户定制之设计模式实现品。  

目前对 TMP 的认识还不充分，以后再回过头来看本条款可能会有更深的理解。