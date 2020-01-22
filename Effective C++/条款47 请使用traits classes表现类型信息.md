#### 一. 实例
STL 的 advance 算法，根据迭代器类型采用不同的实现。  

    struct input_iterator_tag {};
    struct output_iterator_tag {};
    struct forward_iterator_tag: public input_iterator_tag {};
    struct bidirectional_iterator_tag: public forward_iterator_tag {};
    struct random_access_iterator_tag: public bidirectional_iterator_tag {};

    template<typename IterT, typename DistT>
    void advance(IterT& iter, DistT d)
    {
      doAdvance(                                            // “工头”转发给合适的“劳工”，
        iter, d,                                            // 在编译期藉由重载函数实现对类型的 if-else 测试
        typename
          std::iterator_traits<IterT>::iterator_category()
      );
    }

    template<typename IterT, typename DistT>
    void doAdvance(IterT& iter, Dist d, std::random_access_iterator_tag)  // “劳工”一
    {
      iter += d;
    }

    template<typename IterT, typename DistT>
    void doAdvance(IterT& iter, Dist d, std::bidirectional_iterator_tag)  // “劳工”二
    {
      if( d>=0 ) { while(d--) ++iter; }
      else { while(d++) --iter; }
    }

    template<typename IterT, typename DistT>
    void doAdvance(IterT& iter, Dist d, std::input_iterator_tag)          // “劳工”三
    {     
      if(d < 0) {
          throw out_of_range("Negative distance");
      }
      while(d--) ++iter;
    }

#### 二. traits class
##### 2.1 名词解释
一种表现类型信息的技术，也是一个C\+\+程序员共同遵守的协议。将类型信息放进一个 template 及其一个或多个特化版本中。针对迭代器的 traits 被命名为 iterator\_traits。

        // iterator_traits 只是用来表现 IterT 类型信息的嵌套式 typedef
        template<typename IterT>
        struct iterator_traits {
            typedef typename IterT::iterator_category iterator_category;
              ...
        };

        // 对指针的偏特化版本
        template<typename Iter>
        struct iterator_traits<IterT*> {
            typedef random_access_iterator_tag iterator_category;
            ...
        };

        // 对 iterator_traits 的类型实参 IterT 的程序员需遵守的约定要求：
        // IterT 内部必须定义了 iterator_category 类型（隐式接口！）
        template<...>
        class deque {
        public:
            class iterator {
            public:
                typedef random_access_iterator_tag iterator_category;
                ...
            };
            ...
        };

        // 使用：
        template<typename IterT, typename DistT>
        void advance(IterT& iter, DistT d)
        {
          doAdvance(                                            // “工头”转发给合适的“劳工”
            iter, d,
            typename
              std::iterator_traits<IterT>::iterator_category()
          );
        }

##### 2.2 设计和实现
设计并实现一个 traits class 的步骤：  

- 确认若干你希望将来可以取得的类型相关信息并编写相关 tag struct。例如对迭代器而言的分类：struct random_access\_iterator\_tag等等。
- 为该信息选择一个名称。例如 iterator\_category。
- 编写 traits class：提供一个 template 和一组特化版本，内含类型相关信息。例如 iterator\_traits 内含 iterator\_category。
- 遵守规定，为每一个需要类型相关信息的类提供类型相关信息。

##### 2.3 使用
- 建立一组重载函数（身份像劳工）或函数模板（例如 doAdvance），彼此间的差异仅在于各自的 traits 参数。
- 建立一个控制函数（身份像工头）或函数模板（例如 advance)，它调用上述那些“劳工函数”并传递 traits class 所提供的信息。

#### 三. 总结
- traits classes 使得“类型相关信息”在编译期可用。它们以 templates 和 “ templates的特化” 完成实现。
- 整合重载技术后，traits classes 有可能在编译期对类型执行 if-else 测试。
