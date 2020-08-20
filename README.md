# template
### 条款42.了解隐式接口与编译期多态

#### 面向对象中的接口与多态

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

#### 解决方案的实现
假设SmartPtr遵循shared_ptr的接口，存在一个get函数返回原始指针的副本，那我们则可以在构造模板中实现代码约束，具体如下：
```
template <typename T>
class SmartPtr{
public:
    template <typename U>
    SmartPtr(const SmartPtr<U> &other)
        :helder(other.get()) {....}
    T* get() const {return heldptr;}
private:
    T* heldPtr;
}
```

#### 总结
* 可使用member function template生成“可接受所有兼容类型”的函数

### 条款25.若所有参数均需类型转换，请为此采用non-member函数

假设我们建立了一个有理数类，它允许整数隐式转换为有理数：
#### 实例
```
class Rational{
public:
    Rational(int numerator=0,int denominator=1);//刻意不写为explict
    int numerator() const;//成员的访问函数
    int denominator() const;
    ...
}
```

#### 运算符到底是写为member还是non-member呢
首先考虑声明为member函数的情况：
```
const Rational operator* (const Rational& rhs) const;
```
那么很自然地，两个Rational对象可以很自由地相乘。但你会发现这个函数没法做到混合类型计算，比如说,一个int乘以一个ratioanl对象。
```
res = oneHalf * 2;//正确，因为执行了隐式转换
res = 2 * oneHalf;//错误
```
不能编译的原因很简单，int并没有成员函数operator*。解决的方法也很简单，把operator*声明为non-member函数即可
```
const Rational operator* (const Rational& lhs,const Rational& rhs);
```
如此一来，混合类型计算的问题就得到了完美解决，哪怕两个参数都不是Rational，但只要它们存在直接变为Rational对象的隐式转换即可顺利编译与运行。

#### 总结
* 如果某个函数的所有参数都可能需要进行类型转换，那么这个函数必须是个non-member

### 条款47.在需要类型转换时请为模板定义非成员函数

#### 实例
在上一次的实例中，我们以Rational class与operator讨论了在所有实参身上执行隐式转换。本次实例则将它们模板化。
```
template<typename T>
class Rational{
public:
    Rational(const T& numerator = 0,const T& denominator = 1);
    const T numerator() const;
    const T denominator() const;
}
template<typename T>
const rational<T> operator*(const rational<T> &lhs,const rational<T> &rhs) {}

```
我们预期以下代码也会通过编译，因为它们在非模板时确实编译成功了.但是失败了
```
Rational<int> oneHalf(1,2);
Rational<int> result = oneHalf *2;//error 编译失败
```
#### 问题剖析  
operator\*的第一参数被声明为rational<T>,而传递给operator\*的第一实参oneHalf是Rational<int>,那可以肯定T是int.关键在于第二实参接受的应该是一个Rational<int>,但实际传入的却是int。T的推导无法确定。  
    template实参推导过程中并不考虑隐式类型转换。这就是报错的关键。尽管隐式转换在函数调用过程中的确被使用，但在能调用一个函数之前，它必须已经存在（已被推导出并具现化）。  

#### 解决方案
```
template <typename T>
class Rational{
public:
    friend const rational operator*(const Rational& lhs,const Rational& rhs);
};
template<typename T>
const Rational<T> operator*(const Rational<T>&lhs,const Rational<T>&rhs);
```
template class内的friend声明式可以指涉某个特定函数，那意味着class Rational<T>可以声明operator*是它的一个friend函数。class templates并不依赖于template实参推导（实参推导仅发生在function templates身上），所以编译器总是能在class rational<T>具现化时得知T.因此，令Rational<T> class 声明适当的operator*为其friend函数，可以简化整个问题

#### 解决方案剖析
这项技术的关键点在于：我们使用friend并不是因为我们想访问non-public成员。我们为了让类型转换可以用于所有实参，需要一个non-member函数。为了令它自动具现化，我们需要它位于class内部。综合二者，我们得到了friend.
定义在class内部的函数都申请inline，包括friend函数。为了避免可能带来的高成本，一般而言，我们会令它不做任何事情，只是调用class外的一个辅助函数。
```
template<typename T> class Rational;
template<typename T> const Rational<T> doMutiply(const Rational<T> &,......);
template <typename T>
class Rational{
public:
    friend const rational operator*(const Rational& lhs,const Rational& rhs){
        return doMutiply(lhs,rhs);
    }
};
```

#### 总结
* 当我们编写一个class template，而它提供的“与此template相关”函数需要“所有参数支持隐式转换时”，请将那些函数定义为friend函数,并在class template中定义它们。

### 条款48.使用traits classes表现类型信息

#### 实例

STL主要由“容器、迭代器、算法”的templates构成，但也覆盖了若干工具性templates，其中有一个名为advance，用来将迭代器移动若干距离。
```
template<typename IterT,typename Dist>
void advance(IterT& iter,DistT d);
```
既然不同的迭代器具有不同的性质，那我们自然会想到：针对不同的迭代器，advance内部执行不同的操作
```
template<typename IterT,typename Dist>
void advance(IterT &it,Dist d){
    if(iter is random accesss iterator){
        it+=d;
    }
    else{
        while(...) ++it;
    }
}
```
这种做法必须首先判断当前迭代器是否为random access迭代器，也就是说需要取得类型的某些信息。这就是traits的工作：允许你在编译期获取某些类型信息。比如在这个例子中，需要在编译期获取不同类型迭代器是否满足"iter is random accesss iterator"这样的信息。

#### Traits
Traits并非是c++的某个关键词或者一个事先定义好的构件；这是一种技术，也是c++程序员共同遵循的协议。其内置要求之一是：对于内置类型与用户自定义类型的表现必须一样好。  

#### traits的实现
Traits必须可以施加于内置类型，表明我们不应该使用“在自定义类型里附加信息”的技术，因为原始指针无法附加信息。
标准技术是把它放进一个template及一个或者多个特化版本中。针对迭代器的版本名为iterator_traits
```
template<typename IterT> 
struct iterator_traits;//处理迭代器分类信息
```

#### Traits的运作方式
仍以迭代器为例，iterator_traits的运作机理是，针对每一个类型IterT，在struct iterator_traits中必然声明了某个typedef名为iterator_category.这个typedef用来确认IterT的迭代器分类。

#### 实现iterator_traits
iterator_traits以两个部分来实现上述要求。
首先，它要求每个“用户自定义的迭代器类型”必须嵌套一个typedef，名为iterator_category，用以确认当前的卷标结构。举例而言，deque的迭代器可能如下：
```
template<...> 
class deque{
public:
    class iterator{
    public:
        typedef random_access_iterator_tag iterator_category;
    };
};
```
而list的迭代器可双向行进，所以它们应当这样：
```
template<...> 
class list{
public:
    class iterator{
    public:
        typedef bidirectional_iterator_tag iterator_category；
    };
};
```
至于iterators_traits，则是被动响应iterator class的嵌套式typedef：
```
template<typename IterT>
struct iterator_traits{
    typedef typename IterT::iterator_category iterator_category;
}
```
#### 半总结
设计一个traits classes需要以下步骤：
* 确认若干个你希望将来可以取得的信息（例如对迭代器的分类category）
* 为该信息提供名称（如iterator_category）
* 提供一个template与一组特化版本

#### 解决方案
有了iterator_traits后我们似乎可以执行先前的伪代码了
```
template<typename IterT,typename Dist>
void advance(IterT& it,Dist d){
    if(typeid(typename std::iterator_traits<IterT>::iterator_cagetory)==
        typeid(std::random_access_iterator_tag))
    ...
}
```
现在先不去考虑无法编译的问题。正如我们所知，IterT是在编译期获知，所以iterator_traits<IterT>::iterator_category也可以在编译期间确定，关键在于if语句发生在运行期，怎么把它移动到编译期执行呢？  
    
    我们所需要的是一个能在编译期完成if…else操作的条件式，C++恰巧有某种特性支持这种行为：重载。  
    
    当我们重载某个函数f，我们必须详细叙述各个重载件的参数类型。当你调用f，编译期便根据实参选择最恰当的重载件。这简直为我们的问题量身定制。针对advance函数，我们所需要做的就是产生两版重载函数，内含advance的真正操作，但各自接受不同类型的iterator_category对象。具体如下所示：
```
template<typename IterT,typename Dist>
void doadvance(IterT &iter,Dist d,std::random_access_iterator_tag){
    iter+=d;
}
template<typename IterT,typename Dist>
void doadvance(IterT &iter,Dist d,std::bidirectional_iterator_tag){
    if(d>=0) {
        while(d--) ++iter;
    }
    else{
        while(d++) --iter;
    }
}
template<typename IterT,typename Dist>
void doadvance(IterT &iter,Dist d,std::input_iterator_tag){
    if(d>=0) {
        throw std::out_of_range("Negative distance");
    }
    else{
        while(d--) ++iter;
    }
}
```
最终，advance函数的实现如下所示：
```
template<typename IterT,typename Dist>
void advance(IterT &iter,Dist d){
    doadvance(iter,d,typename std::iterator_traits<IterT>::iterator_category());
}
```
#### 总结
* traits class使得“类型相关信息”在编译期可用，它们以templates和偏特化完成实现。
* 整合重载技术后我们可以在编译期执行ifelse测试。

### 条款49.模板元编程
#### 前言
template metaprogramming(TMP 模板元编程)是编写template-based C++程序并执行于编译期的过程。  
所谓TMP，指的是以C++写成、执行于C++编译器内的程序。一旦TMP程序结束执行，其输出，也就是template具现出的若干C++源码，便会一如既往地编译。
#### TMP的优点
* 让某些事情变得容易，如果没有它，则可能无法完成某些任务。
* 由于TMP执行于编译期，所以可以将工作从运行期转移到编译期。这直接导致了运行期的错误可以提前到编译期。另外，程序的每一个方面效率都得到了提高（较小的执行文件、较短的运行期、较少的内存）。然而缺点是编译时间变得很长。
#### 实例
上一节所描述的traits解法就是TMP，它引发编译期发生于类型身上的if…else条件判断。traits-based TMP解法是针对不同类型执行不同代码，每个函数所使用的操作都确保可以实行于其类型。

TMP已被证明是图灵完备的，你可以使用它声明变量、执行循环、编写调用函数等等。但你写出来的东西肯定明显和正常的c++不同，我们之前那一张用TMP写出来的ifelse就是如此（重载），不过那毕竟是汇编语言层级的TMP。

为了大致地描述一下TMP的工作方式，我们首先看看循环操作。TMP没有循环构件，所以循环效果藉由递归完成。TMP主要是一个函数式语言，因此使用递归也十分自然。但是，TMP的递归也并非我们所熟知的递归，因为它并不涉及递归函数调用，而是涉及“递归模板具现化”。
```
template<unsigned n>
struct Factorial{
    enum {value = n*Factorial<n-1>::value};
};
template<>
struct Factorial<0>{
    enum{value =1};//递归基
}
```
首先，每一个Factorial template都是一个struct，value用来保存当前计算所得的阶乘。如果TMP有循环构件，value应该在每一次循环中更新，但实际上由于TMP系以“递归模板具现化”取代循环，每一个具现体有一份自己的value，每一个value有其循环内的适当值。


