继承让代码复用，多态让同一份代码处理不同类型。两者常被混为一谈：继承是"是什么"的关系，多态是"怎么调用"的机制。真正用起来，坑主要出在虚函数没生效、析构没虚、对象被切片这三处。本篇从内存里的虚表讲起，把这些坑一次讲透。

# 1. 继承基础

## 1.1 三种继承方式

```cpp
class Base {
public:
    int pub = 1;
protected:
    int prot = 2;
private:
    int priv = 3;
};

class D1 : public Base {};      // public 继承：is-a，最常用
class D2 : protected Base {};   // protected 继承：外部看不到继承关系
class D3 : private Base {};     // private 继承：实现复用，不是 is-a
```

访问权限变化：

| 继承方式 | public 成员在派生类 | protected 成员 | private 成员 |
|----------|--------------------|----------------|--------------|
| `public` | public | protected | 不可访问 |
| `protected` | protected | protected | 不可访问 |
| `private` | private | private | 不可访问 |

★ **关键认知**：`private` 成员**无论哪种继承方式，派生类都不能直接访问**。想让派生类用又不想对外暴露，就放 `protected`；但更推荐保持 private，通过 protected 接口暴露必要操作。

⚠ **关键认知**：实践中 99% 的继承都应该是 `public`，表达"派生类是一种基类"。`private` 继承只是实现复用，能用组合就别用继承——组合耦合更低，也避免了后面所有虚函数相关的问题。

## 1.2 构造与析构顺序

```cpp
class Base {
public:
    Base() { std::cout << "Base 构造\n"; }
    ~Base() { std::cout << "Base 析构\n"; }
};

class Member {
public:
    Member() { std::cout << "Member 构造\n"; }
    ~Member() { std::cout << "Member 析构\n"; }
};

class Derived : public Base {
public:
    Derived() { std::cout << "Derived 构造\n"; }
    ~Derived() { std::cout << "Derived 析构\n"; }
private:
    Member m_;
};
```

输出顺序：

```text
Base 构造
Member 构造
Derived 构造
Derived 析构
Member 析构
Base 析构
```

★ **关键认知**：构造是"从地基往上盖"，析构是"从顶往下拆"。这也解释了为什么在基类构造函数里调用虚函数不会派发到派生类——此时派生类部分还没构造出来。

## 1.3 名字隐藏

派生类定义同名函数会**隐藏**基类的所有同名重载，而不是重载它们：

```cpp
class Base {
public:
    void f(int) { std::cout << "Base::f(int)\n"; }
    void f(double) { std::cout << "Base::f(double)\n"; }
};

class Derived : public Base {
public:
    void f(std::string) { std::cout << "Derived::f(string)\n"; }
};

Derived d;
d.f("hi");     // 调用 Derived::f(string)
// d.f(1);     // 错误：Base::f(int) 被隐藏了
```

用 `using` 把基类名字引进来：

```cpp
class Derived : public Base {
public:
    using Base::f;                  // 恢复基类重载
    void f(std::string) { /* ... */ }
};

Derived d;
d.f(1);        // 现在可以了
```

⚠ **关键认知**：名字隐藏和虚函数无关，是**名字查找规则**。只要派生类出现同名成员，基类的所有同名重载在派生类作用域里都不可见，除非用 `using` 声明。这个坑在重写 `operator=`、`size()` 这类常见名字时特别容易踩。

# 2. 虚函数与多态

## 2.1 动态绑定

```cpp
class Shape {
public:
    virtual double area() const { return 0; }
    virtual ~Shape() = default;
};

class Circle : public Shape {
public:
    explicit Circle(double r) : r_(r) {}
    double area() const override { return 3.14159 * r_ * r_; }
private:
    double r_;
};
class Square : public Shape {
public:
    explicit Square(double s) : s_(s) {}
    double area() const override { return s_ * s_; }
private:
    double s_;
};
void printArea(const Shape& s) {
    std::cout << s.area() << '\n';   // 根据实际类型调用
}
```

```cpp
Circle c(2);
Square q(3);
printArea(c);      // 调用 Circle::area
printArea(q);      // 调用 Square::area
```

★ **关键认知**：多态生效需要三个条件同时满足——**通过基类的指针或引用调用**、**函数是虚函数**、**派生类确实重写了该函数**。缺一个就退化成静态绑定，调用的还是基类版本。

```cpp
void bad(Shape s) {       // 按值传参：对象被切片，永远调用 Shape::area
    s.area();
}
```

## 2.2 虚函数表

含虚函数的对象通常多一个隐藏指针 `vptr`，指向该类的**虚函数表**：

```text
Circle 对象
┌──────────────┐
│   vptr       │ ──► Circle 的虚表
├──────────────┤     ┌──────────────────┐
│   r_         │     │ &Circle::area    │
└──────────────┘     │ &Circle::~Circle │
                     └──────────────────┘
```

调用 `s.area()` 的过程：

```text
1. 从对象 s 取出 vptr
2. 通过 vptr 找到虚表
3. 按固定槽位取出 area 的函数地址
4. 间接跳转到实际实现
```

★ **关键认知**：虚函数调用是一次**间接跳转**，比普通函数多一层寻址，且通常无法内联。这就是"多态有运行时代价"的来源。只在确实需要运行期分派时用虚函数，不要为了"看起来面向对象"给所有函数加 `virtual`。

## 2.3 override 与 final

```cpp
class Base {
public:
    virtual void run() const {}
    virtual ~Base() = default;
};

class Child : public Base {
public:
    void run() const override {}    // 让编译器检查是否真的重写了
    // void run() override {}       // 编译错误：签名不匹配，缺 const
};
```

`final` 禁止继续派生或重写：

```cpp
class Leaf final : public Base {};          // 不能被继承

class Child2 : public Base {
public:
    void run() const final {}               // 派生类不能再重写
};
```

★ **关键认知**：**每个重写都写 `override`**。它不会改变行为，但能在签名不匹配时直接编译报错。没有它，你以为在重写，实际只是定义了一个新函数，多态悄悄失效——这是最难查的一类 bug。

# 3. 抽象类

含纯虚函数的类是抽象类，不能实例化：

```cpp
class Shape {
public:
    virtual double area() const = 0;    // 纯虚函数
    virtual ~Shape() = default;
};

// Shape s;          // 错误：抽象类不能实例化
```

纯虚函数也可以提供默认实现，派生类通过 `Base::log()` 显式调用，但派生类仍必须重写才能实例化。

★ **关键认知**：抽象类是**接口契约**，它规定"所有形状都必须能算面积"，但不关心怎么算。设计良好的抽象基类通常只有纯虚函数 + 虚析构，没有数据成员。

# 4. 虚析构

```cpp
class Base {
public:
    ~Base() {}                           // 非虚析构
};

class Derived : public Base {
public:
    ~Derived() { /* 释放资源 */ }
};

Base* p = new Derived;
delete p;                                // 只调用 Base::~Base，资源泄漏
```

改成 `virtual ~Base() = default;` 后，`delete p` 会正确调用 `Derived::~Derived` 再调用 `Base::~Base`。

★ **关键认知**：**只要类可能被继承并通过基类指针删除，析构就必须是虚的**。判断标准不是"现在有没有人这么用"，而是"这个类是不是设计成基类"。作为基类就加 `virtual ~Base() = default;`，成本只是一个虚表指针。

⚠ **关键认知**：虚析构不等于所有函数都该虚。析构需要虚是因为删除动作通过基类指针发生；其他函数按需虚。给不该虚的函数加 `virtual` 会增大对象、妨碍内联。

# 5. 对象切片

```cpp
void takeByValue(Shape s);       // 按值接收基类

Circle c(2);
takeByValue(c);                  // 只拷贝了 Shape 部分，Circle 的 r_ 丢了
```

```text
Circle 对象             传入 Shape 形参后
┌──────────────┐        ┌──────────────┐
│ vptr (Circle)│        │ vptr (Shape) │  ← 变成基类虚表
├──────────────┤        ├──────────────┤
│ r_ = 2       │   ✂    │ （被切掉）    │
└──────────────┘        └──────────────┘
```

⚠ **关键认知**：对象切片发生在**按值传递、按值返回、容器存基类对象**时。避免方式只有一个——**多态对象一律用引用或指针传递**，容器里存 `std::unique_ptr<Shape>` 或 `std::shared_ptr<Shape>`，不要存 `Shape`。

# 6. 常见坑

| 现象 | 原因 | 修复 |
|------|------|------|
| 调用的是基类版本 | 函数没加 `virtual`，或签名不匹配 | 加 `virtual`，重写加 `override` |
| 派生类函数没被调用 | 按值传递导致切片 | 改传引用或指针 |
| `delete` 后派生类资源泄漏 | 基类析构非虚 | 基类析构加 `virtual` |
| 基类同名函数找不到 | 名字隐藏 | 用 `using Base::f;` |
| 构造/析构里调虚函数不派发 | 此时派生类部分未构造/已析构 | 不在构造析构里依赖多态 |
| 多重继承出现二义性 | 两个基类有同名成员 | 显式限定 `A::f()` 或虚继承 |

⚠ **关键认知**：构造和析构函数里调用虚函数，**只会调用当前类层次的版本**，不会派发到派生类。因为构造基类时派生类还不存在，析构基类时派生类已经销毁。需要多态行为，就放到单独的 `init()` 里，在对象完全构造后调用。

# 7. 动手练习

1. 定义抽象基类 `Shape`，派生 `Circle`、`Rectangle`，用 `std::vector<std::unique_ptr<Shape>>` 存放并统一计算总面积。
2. 故意把基类析构写成非虚，用 `new`/`delete` 观察派生类析构是否被调用，再加 `virtual` 对比。
3. 写一个基类和派生类，各定义一个同名但参数不同的函数，先观察名字隐藏，再用 `using` 修复。

# 8. 小结

| 概念 | 要点 |
|------|------|
| public 继承 | 表达 is-a，最常用 |
| 构造析构顺序 | 基类→成员→自身；析构逆序 |
| 名字隐藏 | 派生类同名会隐藏基类全部重载 |
| 虚函数 | 通过基类指针/引用才动态派发 |
| 虚表 | 每个对象一个 vptr，调用多一次间接寻址 |
| `override` | 强制编译器检查签名，必写 |
| 抽象类 | 含纯虚函数，不能实例化 |
| 虚析构 | 作为基类就必须有 |
| 对象切片 | 按值传递多态对象会丢派生部分 |

下一篇进入运算符重载，讲清楚成员与非成员的选择、流运算符为什么必须是非成员、比较运算符怎么成套实现，以及下标和函数调用运算符的边界。
