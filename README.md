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
  w被声明为Widget，因此w必须支持Widget接口。我们可以通过打开Widget.h观察接口，所以我们称这种接口为显式接口，即此类接口在源码中明确可见。 
由于Widget的部分成员函数是virtual，因此w对那些函数的调用将表现出运行期多态，即于运行期根据w的动态类型决定调用哪个函数。

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
w必须支持哪一种接口，由template function中执行于w身上的操作决定。以实例而言，size，copy构造函数，normalize等似乎是类型T必须支持的一组接口（其实不然）。这一组（必须能够编译）的表达式可以成为类型T必须支持的一组隐式接口。
凡涉及w的任何函数调用，例如operator>等等，都有可能造成template具现化。这样的具现行为发生在编译期，“以不同的template参数具现化function templates”会导致调用不同的函数，此即为编译期多态。
