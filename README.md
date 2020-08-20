# template
### 条款42.了解隐式接口与编译期多态

#### OOP中的接口与多态

面向对象编程以显式接口和运行期多态解决问题。举例而言：
```
class Widget{
public:
    Widget();
    virtual ~Widget();
    virtual size_t size() const;
    virtual void normalize();
    void swap(WidgeT& other);
    ...
};
void doSth(Widget& w){
    if(w.size()>10){
        Widget temp(w);
        temp.normalize();
        temp.swap(w);
    }
}
```
对于doSth中的w，我们可以这样分析：  
* w被声明为Widget，因此w必须支持Widget接口。我们可以通过打开Widget.h观察接口，所以我们称这种接口为显式接口，即此类接口在源码中明确可见。 
* 由于Widget的部分成员函数是virtual，因此w对那些函数的调用将表现出运行期多态，即于运行期根据w的动态类型决定调用哪个函数。

#### 隐式接口与编译期多态

Templates与泛型编程的世界，与面向对象有根本性的不同。在此世界中，显式接口和运行期多态仍然存在，但是重要性降低。反倒是隐式接口与编译期多态移到了前面。举例而言：
```
template <typename T>
void doSth(T& w){
    if(w.size()>10){
        Widget temp(w);
        temp.normalize();
        temp.swap(w);
    }
}
```
doSth现在变成了一个函数模板，对于doSth中的w，我们现在可以这样分析：  
* w必须支持哪一种接口，由template function中执行于w身上的操作决定。以实例而言，size，copy构造函数，normalize等似乎是类型T必须支持的一组接口（其实不然）。这一组（必须能够编译）的表达式可以成为类型T必须支持的一组隐式接口。  
* 凡涉及w的任何函数调用，例如operator>等等，都有可能造成template具现化。这样的具现行为发生在编译期，“以不同的template参数具现化function templates”会导致调用不同的函数，此即为编译期多态。  

#### 显式接口与隐式接口的对比
通常情况下显式接口由函数的签名式（函数名称、参数类型、返回类型）构成。举例而言：
```
class Widget{
public:
    Widget();
    virtual ~Widget();
    virtual size_t size() const;
    virtual void normalize();
    void swap(WidgeT& other);
    ...
};
```
隐式接口并不基于函数签名式，而是由有效表达式组成。  
```
if(w.size()>10 && w!=sth);
```
T的隐式接口看起来好像有这些约束：  
* 必须提供一个size函数，该函数返回一个整数值。  
* 必须支持operator!=函数，用来比较两个T对象。
然而，由于操作符重载带来的可能性，这两个约束都不需要满足。

#### 实例剖析
T似乎必须要有一个size函数，但这个函数可能是从base class继承得到的。size也没必要返回一个整数，甚至不需要返回一个数值类型。它只要返回一个类型为X的对象，这个对象配合int内置类型可以调用operator>即可。甚至。。。它都无需获取一个X对象，只要获取一个能够隐式转换为X对象的Y对象即可。  
同理，T也没必要支持operator!=。因为下面的情况也是可能的：operator!=接受一对类型为X与Y的对象，我们只要T可以转为X型，sth可以转为Y型即可。  
更有甚者，也许连operator&&也已经被重载，左右两边已经变成了不知道什么类型，执行什么操作的东西。

#### 真正的约束条件
如果都按照上文的做法分析，似乎约束条件很难界定，但是实际上还是很简单的：  
对于size之类的单个表达式可能约束条件很难界定，但对于整体约束条件却很好判断：
* if里面必须是一个bool.
* 其他隐式接口，诸如copy构造函数之类，必须确保对T型对象有效。

#### 总结
* classes与templates都支持接口与多态
* classes的接口是显式的，以函数签名为中心，多态发生在运行期
* templates的接口是隐式的，奠基于有效表达式。多态的实现则基于templates具现化和函数重载解析，发生于运行期。

### 条款43.了解typename
在template声明式中使用class与typename有什么不同吗？
```
template<class T> class Widget;
template<typename T> class Widget;
```
没有  
但下面的情况一定要用typename
```
template <typename C>
void print2nd (const C& container){
    if(container.size()>=2){
        C::const_iterator iter(container.begin());
        ++iter;
        int value = *iter;
        cout << value;
    }
}
```

#### 嵌套从属名称
iter的类型是C::const_iterator,实际上它是什么取决于template参数C.template内出现的名字如果依赖于某个template参数，则称为从属名称。如果从属名称在class内程嵌套状(::)则称之为嵌套从属名称。  
value是一个int，不依赖于任何template参数，于是它叫做非从属名称.

#### 嵌套从属名称的风险
嵌套从属名称可能会导致解析(parsing)困难。
```
template <typename C>
void print2nd(const C& container){
    C::const_iterator * x;
}
```
窒息的操作...如果C中有一个static成员恰好被命名为const_iterator，又或者x恰好是一个global变量。那这个代码就是执行了一次相乘操作。

#### 解决方案
C++对于这种歧义状态有一种解析规则：如果编译器在template中遭遇了一个嵌套从属名称，它便假设该名称并非类型，除非你主动告诉它这是类型。（这个情况有一个小小的例外）。  
想要程序正确运行，上文中的实例应该改为：
```
typename C::const_iterator iter;
```
任何时候你想在template内指涉一个嵌套从属类型名称，就必须在它之前放上一个typename.另外，typename只被用来验明嵌套从属类型名称，其它名称没必要用它。

#### 小小的例外
typename不可以出现在base classes list内的嵌套从属类型名称之前，也不可以在member initialization list中作为base class修饰符。举例而言：
```
template<typename T>
class Derived:public Base<T>::Nested{// base classes list中禁止出现typename
public:
    explicit Derived(int x)
        :Base<T>::Nested(x){//member initialization list中禁止出现typename
            typename Base<T>::Nested temp;//需要typename
        }
    }
}
```

#### typename与typedef
由于某些声明式实在太长，所以讲typedef与typename相结合可能是一种不错的主意：
```
template<typename IterT>
void workWithIterator(IterT iter){
    typedef typename std::iterator_traits<IterT>::value_type value_type;
}
```
#### 总结
* 声明template参数时，class与typename意义相同。
* 必须用typename标识嵌套从属类型名称，但在base class lists && member initialization list中禁止使用。

