裸指针最大的问题不是"难写"，而是它同时承担了"指向"和"拥有"两种语义，编译器却分不清。智能指针把所有权写进类型，让释放时机由对象生命周期决定；移动语义则让"转移资源"不再需要深拷贝。两者配合，构成了现代 C++ 资源管理的核心。本篇把它们放在一起讲，因为 `unique_ptr` 的转移正是移动语义最典型的应用。

# 1. 为什么需要智能指针

```cpp
void bad() {
    Widget* w = new Widget;
    if (someCondition()) {
        return;              // 忘了 delete，泄漏
    }
    process(w);              // 如果这里抛异常，也泄漏
    delete w;
}
```

裸指针的问题：

| 问题 | 后果 |
|------|------|
| 忘记释放 | 内存泄漏 |
| 释放后继续用 | 悬垂指针，未定义行为 |
| 释放两次 | 双重释放，崩溃 |
| 所有权不明 | 不知道该由谁删 |

★ **关键认知**：智能指针不是"会自动 delete 的指针"，而是**把所有权编码进类型系统的工具**。`unique_ptr` 表达"我独占"，`shared_ptr` 表达"我们共享"，`weak_ptr` 表达"我只观察，不拥有"。看类型就知道谁负责释放。

# 2. unique_ptr

## 2.1 独占所有权

```cpp
#include <memory>
std::unique_ptr<Widget> w = std::make_unique<Widget>();
// 离开作用域自动 delete，无需手动释放
```

`unique_ptr` **不能拷贝**，只能移动：

```cpp
std::unique_ptr<Widget> a = std::make_unique<Widget>();
// std::unique_ptr<Widget> b = a;             // 错误：不能拷贝
std::unique_ptr<Widget> b = std::move(a);     // 正确：所有权转移
// 此时 a == nullptr，b 持有对象
```

★ **关键认知**：禁止拷贝是 `unique_ptr` 的设计核心。同一时刻只有一个所有者，释放时机就完全确定，不需要引用计数，也没有额外开销。**默认首选 `unique_ptr`**，只有确实需要共享时才升级到 `shared_ptr`。

## 2.2 创建与访问

```cpp
auto p = std::make_unique<Widget>(1, 2);   // 推荐：make 系列
// std::unique_ptr<Widget> q(new Widget(1, 2));  // 不推荐：异常安全略差
p->run();              // 像指针一样用 ->
(*p).run();            // 解引用
Widget* raw = p.get(); // 只借用，不转移所有权
if (p) { /* 非空 */ }
p.reset();             // 释放并置空
p.reset(new Widget);   // 释放旧的，接管新的
Widget* released = p.release();  // 放弃所有权，需手动 delete
```

★ **关键认知**：优先用 `std::make_unique`。它在异常安全上更稳（对象构造和指针构造不会交错），写法也更短。`std::make_shared` 同理。

⚠ **关键认知**：`get()` 返回裸指针只是**借用**，不要 `delete` 它，也不要用它再构造另一个智能指针——那会导致双重释放。需要转移所有权时用 `std::move`。

## 2.3 自定义删除器

```cpp
struct FileCloser {
    void operator()(std::FILE* f) const {
        if (f) std::fclose(f);
    }
};
std::unique_ptr<std::FILE, FileCloser> file(
    std::fopen("data.txt", "r"));

// 数组也可以用 unique_ptr<T[]>，它会自动用 delete[]
std::unique_ptr<int[]> arr(new int[10]);
```

| 资源 | 默认删除器 | 自定义 |
|------|-----------|--------|
| `new` 对象 | `delete` | 不需要 |
| `new[]` 数组 | 需 `delete[]` | 用 `std::unique_ptr<T[]>` 或自定义 |
| 文件句柄 | 不能 | `fclose` |
| socket | 不能 | `close` |

★ **关键认知**：`unique_ptr` 的自定义删除器是**类型的一部分**，不同的删除器意味着不同的 `unique_ptr` 类型，这会影响函数签名和容器存储。需要类型擦除时（比如放进同一个容器），才考虑 `shared_ptr`。

# 3. shared_ptr

## 3.1 引用计数

```cpp
auto a = std::make_shared<Widget>();
{
    auto b = a;              // 计数 2
    auto c = a;              // 计数 3
}                            // b、c 析构，计数回到 1
// a 析构，计数 0，对象释放
```

```text
shared_ptr a ──┐
shared_ptr b ──┼──► 控制块（引用计数 = 3）──► Widget 对象
shared_ptr c ──┘
```

★ **关键认知**：`shared_ptr` **拷贝是廉价的**（只增加原子计数），但计数操作是**原子操作**，有并发开销。它不是"更好的 unique_ptr"，而是"需要共享所有权时的方案"。能用 `unique_ptr` 就别用 `shared_ptr`。

⚠ **关键认知**：`std::make_shared` 会把对象和控制块**分配在一起**，减少一次分配，但代价是**只要还有一个 `weak_ptr`，整块内存（含对象）就不能释放**。对象很大且 `weak_ptr` 长期存活时，要考虑这个差异。

## 3.2 循环引用

```cpp
struct Node {
    std::shared_ptr<Node> next;
    std::shared_ptr<Node> prev;   // 双向链表，循环引用
};

auto a = std::make_shared<Node>();
auto b = std::make_shared<Node>();
a->next = b;    // b 计数 2
b->prev = a;    // a 计数 2
// a、b 离开作用域，计数各降到 1，永远不会到 0，内存泄漏
```

```text
a ──► Node A ──next──► Node B
      ▲                  │
      └──────prev────────┘
      互相持有，计数永不归零
```

⚠ **关键认知**：循环引用是 `shared_ptr` 最经典的坑。解决办法是把其中一个方向改成 `std::weak_ptr`，通常是"子指向父"或"观察者指向被观察者"的那一侧。

# 4. weak_ptr

`weak_ptr` 不增加引用计数，只观察：

```cpp
struct Node {
    std::shared_ptr<Node> next;
    std::weak_ptr<Node> prev;     // 打破循环
};

auto a = std::make_shared<Node>();
auto b = std::make_shared<Node>();
a->next = b;
b->prev = a;                      // weak，不增加计数
// 使用前必须提升为 shared_ptr
if (auto p = b->prev.lock()) {    // lock() 成功说明对象还活着
    p->doSomething();
} else {
    // 对象已被释放
}
```

| 操作 | 作用 |
|------|------|
| `lock()` | 尝试提升为 `shared_ptr`，失败返回空 |
| `expired()` | 判断对象是否已释放 |
| `use_count()` | 查看共享计数（调试用，别依赖） |

★ **关键认知**：`weak_ptr` 的核心价值是**安全地观察可能已销毁的对象**。缓存、观察者列表、父子反向引用都应该用它。使用前必须 `lock()`，不能直接解引用。

⚠ **关键认知**：`weak_ptr::lock()` 是**原子的**，多线程下"先 `expired()` 再访问"是错的——两步之间对象可能被销毁。要么直接 `lock()` 判断返回值，要么一开始就持有 `shared_ptr`。

# 5. 移动语义

## 5.1 std::move 做了什么

```cpp
std::string a = "hello";
std::string b = std::move(a);     // a 的资源被转移到 b
// 此时 a 处于"有效但未指定"状态，通常为空
```

★ **关键认知**：`std::move` **不移动任何东西**，它只是一个 `static_cast<T&&>`，把左值转成右值引用，告诉编译器"这个对象可以被掏空"。真正干活的是**移动构造函数**。

⚠ **关键认知**：对一个 `const` 对象 `std::move` 得到的是 `const T&&`，无法匹配普通的移动构造，会退回复制。`const std::string a; auto b = std::move(a);` 实际是拷贝，不是移动。

## 5.2 移动构造与移动赋值

```cpp
class Buffer {
public:
    explicit Buffer(std::size_t n) : data_(new int[n]), size_(n) {}
    ~Buffer() { delete[] data_; }
    Buffer(const Buffer& other)                 // 拷贝：深拷贝
        : data_(new int[other.size_]), size_(other.size_) {
        std::copy(other.data_, other.data_ + size_, data_);
    }
    Buffer(Buffer&& other) noexcept             // 移动：偷指针
        : data_(other.data_), size_(other.size_) {
        other.data_ = nullptr;                  // 源对象置空，避免双重释放
        other.size_ = 0;
    }
    Buffer& operator=(Buffer&& other) noexcept {
        if (this != &other) {
            delete[] data_;
            data_ = other.data_;
            size_ = other.size_;
            other.data_ = nullptr;
            other.size_ = 0;
        }
        return *this;
    }
private:
    int* data_;
    std::size_t size_;
};
```

```text
移动前：
other.data_ ──► 堆内存
this.data_  ──► 空

移动后：
other.data_ ──► nullptr
this.data_  ──► 原来的堆内存（所有权转移）
```

## 5.3 移动后的对象状态

★ **关键认知**：被移动后的对象处于"**有效但未指定**"状态——可以析构、可以重新赋值，但不要读它的值。标准库容器通常把它变成空，但这不是语言保证，不能依赖具体内容。

⚠ **关键认知**：移动构造和移动赋值必须标 `noexcept`。否则 `std::vector` 扩容时不敢用移动（怕 halfway 失败无法回滚），只能退回复制，性能悄悄下降一个数量级。这是"写了移动却感觉没变快"的常见原因。

# 6. 完美转发

```cpp
template <typename T>
void wrapper(T&& arg) {                 // 转发引用（不是右值引用）
    target(std::forward<T>(arg));       // 保持原始值类别
}
std::string s = "hi";
wrapper(s);                             // T = std::string&，转发左值
wrapper(std::string("hi"));             // T = std::string，转发右值
```

| 概念 | 写法 | 作用 |
|------|------|------|
| 转发引用 | `T&&` 且 T 需推导 | 左右值都能接 |
| `std::move` | 无条件转右值 | 明确要移动 |
| `std::forward<T>` | 有条件转发 | 保持原值类别 |

★ **关键认知**：`T&&` 只有在**类型推导**语境下才是"转发引用"，能同时绑定左值和右值。如果 `T` 是具体类型（如 `std::string&&`），那就是普通右值引用，只能绑右值。这是区分两者的唯一标准。

⚠ **关键认知**：`std::move` 和 `std::forward` 都只是类型转换，不会自动"优化"。对左值用 `std::move` 会强制移动，可能掏空你还想用的对象；对转发引用一律用 `std::forward<T>`，不要图省事写 `std::move`。

# 7. 选型与常见坑

| 场景 | 选择 |
|------|------|
| 默认持有堆对象 | `std::unique_ptr` |
| 需要共享所有权 | `std::shared_ptr` |
| 观察但不拥有 | `std::weak_ptr`（或裸指针/引用） |
| 数组 | `std::vector` 优先，其次 `unique_ptr<T[]>` |
| 工厂函数返回 | `std::unique_ptr<Base>` |
| 函数参数只读 | `const T&` |
| 函数参数要接管 | `std::unique_ptr<T>` 按值 |

| 坑 | 后果 | 修复 |
|----|------|------|
| `shared_ptr` 循环引用 | 内存泄漏 | 一侧改 `weak_ptr` |
| 用 `get()` 再构造智能指针 | 双重释放 | 用 `std::move` 转移 |
| 移动构造没标 `noexcept` | 容器退回复制 | 加 `noexcept` |
| 移动后继续读源对象 | 值未指定 | 移动后只析构或重新赋值 |
| `const` 对象 `std::move` | 退化成拷贝 | 去掉 const 或直接拷贝 |
| `shared_ptr` 跨线程频繁拷贝 | 原子计数开销 | 传引用或 `const&` |

⚠ **关键认知**：函数参数用 `std::unique_ptr<T>` 按值传递，语义是"我要接管所有权"，调用方必须 `std::move` 进来。如果只是使用对象，用 `T&` 或 `T*`，不要为了"看起来现代"把参数写成智能指针。

# 8. 动手练习

1. 用 `std::unique_ptr` 管理一个 `Widget`，把它传入一个函数并在函数内 `std::move` 到局部变量，验证调用方指针变为空。
2. 构造一个双向链表节点，用 `shared_ptr` 实现 `next`、`weak_ptr` 实现 `prev`，用 `use_count()` 验证没有循环引用。
3. 写一个带移动构造的 `Buffer` 类，用 `std::vector<Buffer>` 反复 `push_back`，分别在移动构造标 `noexcept` 和不标的情况下观察拷贝次数。

# 9. 小结

| 概念 | 要点 |
|------|------|
| `unique_ptr` | 独占所有权，不可拷贝，默认首选 |
| `shared_ptr` | 共享所有权，引用计数，有原子开销 |
| `weak_ptr` | 不增加计数，打破循环引用 |
| `make_unique` | 异常安全，写法简洁 |
| `std::move` | 只是类型转换，真正移动靠移动构造 |
| 移动后状态 | 有效但未指定，不要读 |
| `noexcept` | 移动操作必须标，否则容器退回复制 |
| 完美转发 | `T&&` + `std::forward<T>` 保持值类别 |

下一篇进入 STL 概览，讲清楚 `vector`、`string`、`map`、`unordered_map` 的底层结构与选型、迭代器的分类、常用算法和 lambda 的配合方式。
