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

