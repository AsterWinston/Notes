异常是 C++ 把"正常流程"和"错误处理"分开的机制，也是唯一能跨多层调用自动传播错误的方式。它和 RAII 是一对搭档：异常负责"发现问题"，RAII 负责"在栈展开时把资源收拾干净"。本篇讲清楚抛出捕获、栈展开、`noexcept`，以及异常安全等级这套工程标准。

# 1. 异常基础

## 1.1 抛出与捕获

```cpp
#include <stdexcept>

double divide(double a, double b) {
    if (b == 0) {
        throw std::invalid_argument("除数不能为 0");   // 抛出异常
    }
    return a / b;
}

int main() {
    try {
        std::cout << divide(1, 0) << '\n';
    } catch (const std::invalid_argument& e) {          // 按 const 引用捕获
        std::cerr << "错误: " << e.what() << '\n';
    }
    return 0;
}
```

`throw` 之后函数立即返回，后面的代码不执行。

★ **关键认知**：异常是**带类型的对象**，`catch` 按类型匹配，不是按值或字符串匹配。`throw std::invalid_argument(...)` 抛出的是一个完整对象，会经历一次拷贝（或移动），所以异常类型要能拷贝。

## 1.2 栈展开

抛出异常后，运行时从 `throw` 处逐层退出函数，销毁沿途所有局部对象，直到找到匹配的 `catch`：

```text
main
 └─ foo()          ← catch 在这里
     └─ bar()
         └─ divide()  ← throw

展开过程：
divide 的局部对象析构
bar 的局部对象析构
foo 的局部对象析构
进入 catch
```

```cpp
void bar() {
    std::string s = "临时对象";
    throw std::runtime_error("boom");   // s 会被析构
}
void foo() {
    try {
        bar();
    } catch (const std::runtime_error& e) {
        // s 已析构，没有泄漏
    }
}
```

★ **关键认知**：栈展开期间，**所有已构造完成的局部对象都会自动析构**。这就是 RAII 能在异常场景下保证资源释放的原因——你不需要写任何 `finally`，析构函数会替你做。

⚠ **关键认知**：如果栈展开过程中又抛出一个异常（例如析构函数抛异常），会直接调用 `std::terminate`，程序立即终止，不给任何补救机会。所以析构函数必须默认 `noexcept`，不要在析构里抛。

## 1.3 标准异常体系

```text
std::exception
 ├── std::logic_error
 │    ├── std::invalid_argument
 │    ├── std::domain_error
 │    ├── std::length_error
 │    └── std::out_of_range
 └── std::runtime_error
      ├── std::range_error
      ├── std::overflow_error
      └── std::underflow_error
```

| 类别 | 含义 | 例子 |
|------|------|------|
| `logic_error` | 程序逻辑错误，理论上可避免 | 参数非法、越界 |
| `runtime_error` | 运行期才可能出现 | 文件打不开、网络断开 |

自定义异常通常继承 `std::runtime_error`：

```cpp
class NetworkError : public std::runtime_error {
public:
    explicit NetworkError(const std::string& msg)
        : std::runtime_error(msg) {}
};
```

# 2. 异常的传递与匹配

## 2.1 按引用捕获

```cpp
catch (const std::exception& e) { ... }   // 正确：按 const 引用
// catch (std::exception e) { ... }       // 错误：对象切片
```

★ **关键认知**：按值捕获会发生**对象切片**——派生类异常被截断成基类对象，`what()` 返回基类信息，派生类附加的字段全丢。按 `const` 引用捕获既避免切片，也避免拷贝。

## 2.2 catch 顺序

`catch` 从上到下匹配，**第一个匹配的胜出**：

```cpp
try {
    throw std::out_of_range("越界");
} catch (const std::exception& e) {       // 太宽，会先吃掉
    std::cout << "通用: " << e.what();
} catch (const std::out_of_range& e) {    // 永远到不了
    std::cout << "越界";
}
```

正确顺序是**从具体到宽泛**：

```cpp
try {
    throw std::out_of_range("越界");
} catch (const std::out_of_range& e) {
    std::cout << "越界";
} catch (const std::exception& e) {
    std::cout << "其他: " << e.what();
}
```

⚠ **关键认知**：编译器通常不会警告"这个 catch 永远到不了"，因为类型匹配在运行期才确定。把基类 catch 写在最前面，是一个静默 bug。

## 2.3 重新抛出

在 catch 里处理不了，可以原样再抛：

```cpp
try {
    doWork();
} catch (const std::exception& e) {
    logError(e.what());
    throw;                    // 重新抛出当前异常，保持原类型
}
```

★ **关键认知**：用 `throw;` 而不是 `throw e;`。前者重新抛出**原对象**，保留动态类型；后者会拷贝一份，且可能切片。清理现场后继续上抛，一律写裸 `throw;`。

# 3. noexcept

`noexcept` 承诺函数不抛异常：

```cpp
void safe() noexcept;                 // 承诺不抛
void maybe() noexcept(false);         // 明确可能抛
```

如果 `noexcept` 函数真的抛了，程序直接 `std::terminate`，不会栈展开。

```cpp
void f() noexcept {
    throw std::runtime_error("x");    // 不会传播，直接 terminate
}
```

★ **关键认知**：`noexcept` 不只是文档，它影响**优化和容器行为**。`std::vector` 扩容时，只有当元素的移动构造是 `noexcept`，才会用移动；否则为了强异常安全保证，退回复制。所以移动构造、移动赋值、`swap` 都应该标 `noexcept`。

| 场景 | 是否标 noexcept |
|------|----------------|
| 移动构造/移动赋值 | 是 |
| `swap` | 是 |
| 析构函数 | 默认就是，不要改 |
| 简单 getter | 可以 |
| 可能分配内存的函数 | 否 |

⚠ **关键认知**：`noexcept` 是**承诺**，违反了直接终止程序，不是"忽略异常"。不要在没把握的函数上随便标 `noexcept`，尤其是会调用回调、分配内存、做 IO 的函数。

# 4. RAII 与异常

## 4.1 用 RAII 保证清理

```cpp
class Connection {
public:
    explicit Connection(const std::string& url) {
        handle_ = open(url);              // 可能抛异常
        if (!handle_) {
            throw std::runtime_error("连接失败");
        }
    }

    ~Connection() {
        if (handle_) {
            close(handle_);               // 一定会执行
        }
    }

    Connection(const Connection&) = delete;
    Connection& operator=(const Connection&) = delete;

private:
    Handle handle_ = nullptr;
};

void work() {
    Connection c("tcp://localhost");      // 构造成功
    process(c);                           // 抛异常也没关系
}                                         // c 析构，连接关闭
```

## 4.2 构造失败与析构

★ **关键认知**：**构造函数抛异常时，该对象的析构函数不会被调用**，因为对象从未构造完成。但已经构造完成的**成员**会被逆序析构。所以资源获取要放在成员初始化里，或者用 RAII 成员管理，不要指望构造函数的 catch 块。

```cpp
class Widget {
public:
    Widget() : a_(new int(1)), b_(new int(2)) {
        // 如果这里抛异常：b_、a_ 已构造完成，会被析构
        // 但 Widget 自己的析构不会调用
    }
private:
    std::unique_ptr<int> a_;
    std::unique_ptr<int> b_;
};
```

⚠ **关键认知**：构造函数里如果手动 `new` 了裸指针，在 `new` 之后、构造完成之前抛异常，这块内存就泄漏了。解决办法是**用智能指针或容器持有资源**，让成员析构负责清理。

# 5. 异常安全等级

| 等级 | 保证 | 说明 |
|------|------|------|
| 基本保证 | 不泄漏、不破坏不变式 | 对象仍可用，但状态可能改变 |
| 强保证 | 失败则回滚 | 操作要么完全成功，要么保持原状 |
| 不抛保证 | 绝不抛异常 | `noexcept`，如 `swap`、移动 |

```cpp
// 强保证：先构造新对象，成功后再交换
void push_back(const T& value) {
    auto new_data = std::make_unique<T[]>(capacity_ * 2);  // 可能抛
    std::copy(data_.get(), data_.get() + size_, new_data.get());
    new_data[size_] = value;
    data_ = std::move(new_data);      // 到这里才修改状态，不会抛
    ++size_;
    ++capacity_;
}
```

★ **关键认知**：实现强保证的通用手法是 **copy-and-swap**——先在副本上完成所有可能失败的操作，最后用不抛的 `swap` 一次性替换。这样失败时原对象分毫未动。

⚠ **关键认知**：不是所有操作都能做到强保证。比如返回引用的 `operator[]` 越界，只能在"抛异常"和"未定义行为"之间选。工程上要明确每个接口承诺哪一级，并在文档里写清楚。

# 6. 何时用异常

| 场景 | 推荐 |
|------|------|
| 构造函数失败 | 异常（没有返回码可用） |
| 不可恢复的错误 | 异常 |
| 跨多层调用的错误 | 异常 |
| 高频、可预期的失败 | 返回 `std::optional` / `std::expected` |
| 性能敏感的底层代码 | 错误码 |
| 析构函数 | 绝不抛 |

★ **关键认知**：异常适合"罕见且无法就地处理"的错误，不适合做控制流。用异常处理"文件不存在"这种可预期情况，会让正常路径变慢、代码变难读。C++23 的 `std::expected` 给了一个"返回值 + 错误信息"的现代方案。

# 7. 常见坑

| 现象 | 原因 | 修复 |
|------|------|------|
| 程序直接 terminate | `noexcept` 函数抛了异常 | 去掉 `noexcept` 或内部捕获 |
| 资源泄漏 | 裸指针在构造中抛异常 | 用智能指针/容器 |
| 捕获不到派生异常 | 按值捕获导致切片 | 按 const 引用捕获 |
| 特定 catch 不生效 | 基类 catch 写在前面 | 从具体到宽泛排列 |
| `what()` 信息丢失 | `throw e;` 拷贝切片 | 用 `throw;` |
| 析构抛异常导致崩溃 | 栈展开中二次抛出 | 析构内捕获并吞掉 |

⚠ **关键认知**：调试 `std::terminate` 时，先看有没有 `noexcept` 函数内部抛异常，再看析构函数里有没有可能抛的操作。这两个是"异常没被捕获却直接终止"的头号原因，和 `try/catch` 写没写对无关。

# 8. 动手练习

1. 写一个 `divide` 函数，除数为 0 时抛 `std::invalid_argument`，在 `main` 里捕获并打印 `what()`。
2. 写一个 RAII 的 `FileGuard`，构造打开文件、析构关闭，在打开后立刻抛异常，验证文件句柄被正确释放。
3. 给一个 `push_back` 实现基本保证和强保证两个版本，思考失败时对象状态分别是什么。

# 9. 小结

| 概念 | 要点 |
|------|------|
| 抛出 | `throw` 一个对象，按类型匹配 |
| 栈展开 | 逐层析构局部对象，RAII 生效 |
| 捕获 | 按 `const` 引用，避免切片 |
| catch 顺序 | 从具体到宽泛 |
| 重新抛出 | 用裸 `throw;` |
| `noexcept` | 承诺不抛，违反即 terminate |
| 构造失败 | 自身析构不调用，成员会析构 |
| 安全等级 | 基本、强、不抛 |

下一篇进入命名空间与模块，讲清楚命名空间的作用、匿名命名空间与 `static` 的关系、`using` 的边界、头文件组织，以及 C++20 模块能解决什么问题。
