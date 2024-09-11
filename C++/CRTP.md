#### CRTP 奇特递归模板模式(curiously recurring template pattern)

（1）基本范式：父类是模板，再定义一个类类型继承他。因为类模板不是类，只有实例化之后的类模板才是实际的类类型，所以我们需要实例化它，显式指明模板类型参数，而这个类似参数就是我们定义的类型，也就是子类了。

```c++
template <class Dervied>
class Base {};

class x : public Base<x> {};
```

（2）CRTP父类暴露接口，子类实现接口，以此实现"编译期多态"或者"静态多态"

这里的static_cast是否安全？因为CRTP的设计原则就是假设Derived会继承自 Base\<Derived>

```c++
template <class Derived>
class Base{
public:
	void addwater(int n){
        static_cast<Deried*>(this)->impl(n);
    }    
}

class X : public Base<X>{
public:
    void impl(int n) const{
        std::cout<<"X 设备加了"<<n<<"毫升水\n";
    }
}

X x;
x.addwater(100); 
```

进一步优化，让子类的接口不暴露出来设置友元来让父类得以访问

```c++
class x : public Base<x>{
    friend Base<x>;
    void impl() const{
        std::cout << "X 设备加了 50 毫升水\n";
    }
}
```



#### C++23显式对象形参

（1）定义：将C++23之前隐式的，由编译器自动将this指针传递给成员函数使用的，改成允许用户显式写明了，也就是说：

```c++
struct x{
    // 之前void f() const{}其实就是const this
    void f(this const x& self){};
}

// 例子
struct Ts
{
    void f(){
        this; // 这个是隐式传入的，成员函数与普通函数并无区别
    }
    
    // 不行滴
    static void f(){
        this; // 这个是隐式传入的，成员函数与普通函数并无区别
    }
    
    // 就是显式写明了this
    void f(this Ts& self){
        self;
    }
}
```

```c++
// 也支持模板(可以直接auto而无需再template<typename>)，支持各种修饰
this X self
this X& self
this const X& self
this X&& self
this auto&& self
const auto& self
```

```c++
struct Base {void name(this auto&& self){self.impl();}}
struct D1: Base{void impl() {std::puts("D1::impl()")}};
struct D2: Base{void impl() {std::puts("D2::impl()")}};
```



#### CRTP优势

（1）静态多态，无需使用虚函数，静态绑定，无运行时开销(动态多态需要运行时多一次虚函数指针的索引)

（2）类型安全：通过模板参数传递具体的子类类型确保编译器类型匹配

（3）灵活的接口设计：允许父类定义公共接口并要求子类具体实现操作，使得基类能够提供通用的接口而具体的实现细节留给派生类也就是多态了