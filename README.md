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

### 条款44.处理模板化基类内的名称

#### 场景
假定我们需要撰写一个程序，它需要传递信息至不同的公司。信息或者是明文，或者是密码。如果我们在编译期可以确定发送至哪家公司，就可以采用基于template的解决方案  
```
class CompanyA{
public:
    ...
    void sendCleartext(const string& msg);
    void sendEncrypted(const string& msg);
    ...
};
class CompanyB{
public:
    ...
    void sendCleartext(const string& msg);
    void sendEncrypted(const string& msg);
    ...
};
class MsgInfo {...};//保存信息
template<typenmae Company>
class MsgSender{
public:
    ...
    void sendClear(const MsgInfo& info){
        string msg;
        //根据info产生信息
        Company c;
        c.sendCleartext(msg);
    }
    void sendSecret(const MsgInfo& info);
};
```
但如果我们希望在每次发送信息时完成记录，我们通过派生一个derived class来完成这种行为:
```
template<typenmae Company>
class LoggingMsgSender:public MsgSender<Company>{
public:
    ...
    void sendClearMsg(const MsgInfo& info){//避免遮蔽名称
        //log sth
        sendClear(info);//试图调用base class函数,无法编译
        //log sth
    }
};
```
编译器会拒绝编译，并给出sendClear不存在这样的错误信息.

#### 问题剖析
报错原因在于，当编译器遭遇class template LoggingMsgSender定义式时，并不了解其基类，因为此时基类尚未被具现化。再具体一些，我们假定有CompanyZ坚持使用加密通讯：
```
class CompanyZ{
public:
    ...//不提供sendClear
    void sendEncrypted(const string& msg);
    ...
};
```
那一般化的MsgSender template对CompanyZ并不合适，因为该template提供了sendClear。因此我们针对CompanyZ做出了一个特化：
```
template<>//全特化
class MsgSender<CompanyZ>{
public:
    ...
    void sendSecret(const MsgInfo& info);
};
```
插播一则模板的全特化，偏特化。  
全特化
```
#include <iostream>
using namespace std;

template<typename T1, typename T2>
class A{
        public:
                void function(T1 value1, T2 value2){
                        cout<<"value1 = "<<value1<<endl;
                        cout<<"value2 = "<<value2<<endl;
                }
};

template<>
class A<int, double>{ // 类型明确化，为全特化类
        public:
                void function(int value1, double value2){
                        cout<<"intValue = "<<value1<<endl;
                        cout<<"doubleValue = "<<value2<<endl;
                }
};

int main(){
        A<int, double> a;
        a.function(12, 12.3);
        return 0;
}                     
```
片特化
```
template<typename T1, typename T2>
class A{
        public:
                void function(T1 value1, T2 value2){
                        cout<<"value1 = "<<value1<<endl;
                        cout<<"value2 = "<<value2<<endl;
                }
};

template<typename T>
class A<T, double>{ // 部分类型明确化，为偏特化类
        public:
                void function(T value1, double value2){
                        cout<<"Value = "<<value1<<endl;
                        cout<<"doubleValue = "<<value2<<endl;
                }
};

int main(){
        A<char, double> a;
        a.function('a', 12.3);
        return 0;
}
```
对主版本模板类、全特化类、偏特化类的调用优先级从高到低进行排序是：  
全特化类>偏特化类>主版本模板类
插播结束，继续考虑derived class LoggingMsgSender
```
template<typenmae Company>
class LoggingMsgSender:public MsgSender<Company>{
public:
    ...
    void sendClearMsg(const MsgInfo& info){
        //log sth
        sendClear(info);//如果参数为CompanyZ，该函数不存在
        //log sth
    }
};
```
如今我们可以清楚地看出编译器拒绝调用的原因：base class template可能被特化，且特化版本可能不具备与一般性版本一致的接口。编译器拒绝在templated base classes内寻找继承而来的名称，面向对象的规则在泛型编程中依然不再适用。

#### 解决方案
* 使用this->
```
template<typenmae Company>
class LoggingMsgSender:public MsgSender<Company>{
public:
    ...
    void sendClearMsg(const MsgInfo& info){
        //log sth
        this->sendClear(info);//假设继承了sendClear，隐式接口
        //log sth
    }
};
```
* 使用using
```
template<typenmae Company>
class LoggingMsgSender:public MsgSender<Company>{
public:
    using MsgSender<Company>::sendClear;//假设存在
    ...
    void sendClearMsg(const MsgInfo& info){
        //log sth
        sendClear(info);
        //log sth
    }
};
```
#### 总结
上述的三种方法其实都是在做同一件事：对编译器承诺“base class template的任何特化版本都将支持其一般版本所提供的接口”。
面对“指涉base class mrmbers”的无效references，编译器的诊断可能在早期（当解析dc template的定义式时），也可能发生在晚期（当template被特定实参具现化时）。c++一般惯于较早诊断，这也就是为什么“当base classes从templates中被具现化时”，它假设它对base classes的内容一无所知的原因。

### 条款45.将参数无关的代码抽离template
Templates是避免代码重复与节约时间的最优解。但有时，template可能会造成代码膨胀：二进制码带着大量重复（或几乎重复）的代码或者数据。其源码可能看起来优雅整洁，但实际上生成的object code完全不是一回事，我们需要对此进行优化。

#### 实例分析
为一个方阵编写template，其性质之一是支持求逆：
```
template<typename T,size_t n>
class SquareMatrix{
public:
    void invert();//求逆矩阵
}
```
假设会这样调用
```
SquareMatrix<double,5> sm1;
sm1.invert();
SquareMatrix<double,10> sm2;
sm2.invert();
```
以上操作直接导致具现化了两个invert，虽然它们并非完全相同，但是相当类似，唯一的区别就是一个作用于5*5矩阵一个作用于10*10矩阵。
... 没看懂 ...

#### 总结
* template生成多个class与多个函数，所以任何template代码都不应该与某个造成膨胀的template参数相互依存
* 因为non-type template parameters造成的代码膨胀往往可以消除，做法是以函数参数或者class成员变量替换template参数。
* 因type parameters造成的代码膨胀往往可以降低，做法是让带有相同二进制表述的具现类型共享实现码。

### 条款46.运用member function template接受所有兼容类型

#### 实例
真实指针有一个很好用的功能：支持隐式转换。举例来说：
* derived class指针可以隐式转为base class指针
* 指向non-const对象的指针可以指向const对象
既然智能指针是行为如同原始指针的对象，那么我们当然希望智能指针也具备上述功能。具体来说，我们希望下述代码能够通过编译：
```
class Top;
class Middle:public Top;
template<typename T>
class Smartptr{//自定义的智能指针
public:
    explicit Smartptr(T* realPtr);
}
Smartptr<Top> pt1 =Smart<Middle>(new Middle);
Smartptr<const Top> pt2 = pt1;
```
但是，同一个template的不同具现化之间并不存在继承关系，所以，smartptr<top>与smartptr<middle>完全无关，为了达到我们期望的smart classes之间的转换能力，我们必须明确地编写它们。  

#### 解决方案
我们需要撰写的是一个构造模板。构造函数模板具体如下：
```
template<typename T>
class SmartPtr{
public:
    template<typename U>
    SmartPtr(const SmartPtr<U>& other);
}
```

#### 方案剖析  
    该构造函数模板声明：对于任何类型T和任何类型U，总可以根据一个SmartPtr<U>对象来构造一个SmartPtr<T>对象。我们把这种构造函数称为泛化copy构造函数。之所以不用explicit声明，是因为原始指针之间的转换就是隐式的，我们需要保证智能指针与原始指针之间的兼容性。  
    声明固然已经完成，但是实现却值得花一番心思。并非任意的U型智能指针都能转为T型智能指针（比如把base转为derived），所以我们必须在某方面对这一member template所创建的成员函数群进行筛选。
