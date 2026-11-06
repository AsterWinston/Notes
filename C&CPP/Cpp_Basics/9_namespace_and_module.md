名字冲突是多人协作和引入第三方库时最烦的问题之一：两个库都叫 `Config`，两个头文件都定义 `print`。命名空间就是 C++ 给出的第一层解法，而 C++20 模块则试图从根上解决头文件带来的编译开销和宏污染。本篇讲清楚命名空间的用法、匿名命名空间与 `static` 的关系、`using` 该写到什么粒度，以及模块能做什么、不能做什么。

# 1. 命名空间

## 1.1 为什么需要

```cpp
// liba.h 与 libb.h 各自声明 log，直接放在全局作用域会冲突
void log(const std::string& msg);
```

命名空间给名字加上前缀，把不同来源的符号隔开：

```cpp
namespace liba {
void log(const std::string& msg);
}

namespace libb {
void log(const std::string& msg);
}

liba::log("a");     // 明确调用哪一个
libb::log("b");
```

★ **关键认知**：命名空间是**编译期的名字隔离**，不产生任何运行时开销，也不改变对象布局。它解决的只是"名字查找"问题，不是访问控制，也不是封装。

## 1.2 定义与使用

```cpp
namespace math {
constexpr double pi = 3.14159;

double circleArea(double r) {
    return pi * r * r;
}
}

double a = math::circleArea(2.0);   // 用限定名访问
```

命名空间可以分多次打开：头文件里声明 `namespace math { ... }`，`.cpp` 里再打开同一个命名空间写实现，二者会自动合并。

## 1.3 嵌套与内联

```cpp
namespace company {
namespace project {
namespace detail {
void helper();
}
}
}

// C++17 起可以一次写多层
namespace company::project::detail {
void helper() {}
}
```

内联命名空间让成员**自动出现在外层**，常用于版本管理：

```cpp
namespace api {
inline namespace v2 {
void connect();          // 实际名字是 api::v2::connect
}
}

api::connect();          // 不用写 v2
api::v2::connect();      // 也合法
```

★ **关键认知**：内联命名空间适合"版本升级但想保持源码兼容"的场景。`std::literals` 就是内联命名空间，所以 `using namespace std::literals;` 后能直接用 `"abc"s`。

⚠ **关键认知**：**不要在头文件的全局作用域写 `using namespace std;`**。它会把整个标准库的名字倒进每个包含该头文件的翻译单元，极易与用户代码冲突，而且冲突点离你写 `using` 的地方很远，极难排查。头文件里最多用 `using std::string;` 这种精确引入。

# 2. 匿名命名空间

匿名命名空间里的名字具有**内部链接**，只在当前翻译单元可见：

```cpp
// widget.cpp
namespace {
int counter = 0;              // 只在 widget.cpp 可见

void increment() {            // 只在 widget.cpp 可见
    ++counter;
}
}

void publicApi() {
    increment();              // 同文件内可以直接用
}
```

它等价于 C 风格的 `static`：

```cpp
static int counter = 0;       // 与匿名命名空间效果相同（针对变量/函数）
```

| 写法 | 效果 | 推荐度 |
|------|------|--------|
| 匿名命名空间 | 内部链接，可包含类型、模板 | 推荐 |
| `static` 函数/变量 | 内部链接 | 可用，但只适合单个声明 |

★ **关键认知**：匿名命名空间和 `static` 都能实现"文件内私有"，但**匿名命名空间能包裹类型、模板、常量**，`static` 只能修饰变量和函数。写 C++ 时优先用匿名命名空间，语义更完整。

⚠ **关键认知**：匿名命名空间在**每个包含它的翻译单元里都是不同的命名空间**。不要把匿名命名空间写进头文件——每个包含者都会得到一份独立定义，容易造成 ODR 违规和"明明定义了却链接不到"的困惑。

# 3. using 的边界

`using` 有三种用法，作用范围差别很大：

```cpp
using namespace math;         // 引入整个命名空间（污染最大）
using math::pi;               // 引入单个名字（推荐）
namespace m = math;           // 起别名（推荐）
```

| 写法 | 作用 | 适合位置 |
|------|------|----------|
| `using namespace X;` | 引入全部名字 | 函数内部、`.cpp` 顶部 |
| `using X::name;` | 引入单个名字 | 头文件、类内 |
| `namespace alias = X;` | 起短别名 | 头文件、`.cpp` |

```cpp
void process() {
    using namespace std::string_literals;   // 作用域限定在函数内
    auto s = "hello"s;                      // 只在函数内可直接用
}
```

★ **关键认知**：`using namespace` 的**作用域越小越好**。函数内使用完全可控；`.cpp` 顶部次之；头文件里几乎从不。长度换来的可读性提升，远不如一次名字冲突带来的排查成本。

⚠ **关键认知**：在头文件里 `using namespace` 的破坏是**传染性**的——所有包含它的文件都被污染，而受害者往往不知道污染源在哪。这是代码评审里应该直接拒绝的写法。

# 4. 头文件组织

## 4.1 包含守卫与 pragma once

```cpp
// widget.h
#ifndef PROJECT_WIDGET_H
#define PROJECT_WIDGET_H

class Widget { /* ... */ };

#endif
```

也可以直接写一行 `#pragma once`。

| 方式 | 优点 | 缺点 |
|------|------|------|
| 包含守卫 | 标准、可移植 | 名字要唯一，重复读取有开销 |
| `#pragma once` | 简洁、编译器优化 | 非标准（但主流编译器都支持） |

★ **关键认知**：包含守卫的名字要**全局唯一**，习惯用"项目_目录_文件_H"。只用文件名（如 `WIDGET_H`）在两个同名头文件之间会互相屏蔽，症状是"某个类突然不完整"。

## 4.2 前置声明

只需要指针或引用时，用前置声明代替 `#include`：

```cpp
// 不需要包含 widget.h，只需要知道 Widget 是个类
class Widget;

class Manager {
public:
    void setWidget(Widget* w);      // 指针，前置声明够用
private:
    Widget* widget_ = nullptr;
};
```

```cpp
// 需要完整类型时，必须包含
#include "widget.h"
void use() {
    Widget w;                       // 需要完整定义
    w.run();
}
```

★ **关键认知**：前置声明能减少编译依赖、缩短增量编译时间，但**不能用于按值成员、继承、`sizeof`、调用成员函数**——这些都需要完整类型。头文件里能用指针/引用就用前置声明，是很实际的工程习惯。

## 4.3 目录与命名

```text
project/
├── include/project/
│   ├── widget.h
│   └── config.h
├── src/
│   ├── widget.cpp
│   └── config.cpp
└── tests/
```

包含时写 `<project/widget.h>`，配合 `-Iinclude`，好处是**不会和系统头文件重名**，也明确了属于哪个项目。

⚠ **关键认知**：头文件名要和使用它的命名空间/类对应。一个头文件里塞十几个不相关类，会让前置声明和依赖管理都失效；一个类拆五个头文件，又会让包含关系混乱。经验是"一个主要类一个头文件"。

# 5. C++20 模块

## 5.1 模块解决什么问题

传统头文件有三个固有问题：

| 问题 | 表现 |
|------|------|
| 重复解析 | 每个 `.cpp` 都要重新展开所有 `#include` |
| 宏泄漏 | 头文件里的宏污染包含者 |
| 包含顺序敏感 | 先包含谁后包含谁，可能影响结果 |

模块把"文本粘贴"换成"编译一次、导出二进制接口"：

```text
头文件模型：main.cpp ──预处理展开──► 巨大翻译单元 ──编译──► .o
模块模型：  widget.cppm ──编译──► .pcm（二进制接口）
            main.cpp ──导入 .pcm──► 编译 ──► .o
```

★ **关键认知**：模块的核心价值是**接口只编译一次**，并且导入的是结构化信息，不是文本。它从根本上消除了宏泄漏和包含顺序问题，同时能显著加快大型项目的编译速度。

## 5.2 基本语法

```cpp
// math.cppm
export module math;              // 声明模块

export int add(int a, int b) {   // export 才对外可见
    return a + b;
}

int internalHelper() {           // 不 export，模块内私有
    return 42;
}
```

```cpp
// main.cpp
import math;                     // 导入模块
#include <iostream>

int main() {
    std::cout << add(1, 2) << '\n';
}
```

## 5.3 现状与边界

| 维度 | 现状 |
|------|------|
| 标准库模块 | C++23 起 `import std;` 逐步可用 |
| 编译器支持 | GCC 14+、Clang 17+、MSVC 较成熟 |
| 构建系统 | CMake 3.28+ 有实验性支持 |
| 生态 | 大量库仍是头文件形式 |

★ **关键认知**：模块是**长期方向**，但不是"现在就该全面迁移"。它需要编译器、构建系统、依赖库三方同时支持，且和现有头文件库混用时仍有边界问题。新项目可以小范围试点，存量项目不要为了用而用。

⚠ **关键认知**：模块和头文件可以共存，但一个模块单元里 `import` 和 `#include` 混用要小心宏的可见性。过渡期最常见的做法是"自己的代码用模块，第三方库继续 `#include`"。

# 6. 常见坑

| 现象 | 原因 | 修复 |
|------|------|------|
| 头文件里 `using namespace std` 引发冲突 | 污染所有包含者 | 删掉，改用精确 `using` |
| 匿名命名空间里的函数链接不到 | 每个翻译单元独立 | 不要放头文件，改放 `.cpp` |
| 两个同名头文件互相屏蔽 | 包含守卫宏重名 | 用带路径的唯一宏名 |
| 前置声明后调用成员函数报错 | 缺少完整类型 | 在 `.cpp` 里 `#include` |
| 模块导入后找不到符号 | 忘记 `export` | 对外接口加 `export` |
| 命名空间别名在头文件里冲突 | 别名也是名字 | 用更具体的别名，或放 `.cpp` |

⚠ **关键认知**：`using namespace` 引发的问题往往**不在写它的文件里爆发**，而是在某个包含它的 `.cpp` 里报"ambiguous"，报错位置和根源隔着好几层包含。评审时看到头文件里的 `using namespace`，直接要求改掉。

# 7. 动手练习

1. 建两个命名空间 `alpha` 和 `beta`，各定义一个 `print`，在 `main` 里用限定名分别调用。
2. 把一个只在单个 `.cpp` 使用的辅助函数改成匿名命名空间，确认其他文件无法链接到它。
3. 把一个类改成前置声明 + 指针成员，把 `#include` 从头部移到 `.cpp`，比较改动前后的编译依赖。

# 8. 小结

| 概念 | 要点 |
|------|------|
| 命名空间 | 编译期名字隔离，无运行时开销 |
| 嵌套/内联 | C++17 可 `a::b::c`，内联命名空间用于版本 |
| 匿名命名空间 | 内部链接，可包裹类型和模板 |
| `using namespace` | 作用域越小越好，头文件禁用 |
| 包含守卫 | 宏名要全局唯一 |
| 前置声明 | 指针/引用够用时减少依赖 |
| 模块 | 接口编译一次，消除宏泄漏，生态仍在推进 |

下一篇进入智能指针与移动语义，讲清楚 `unique_ptr`/`shared_ptr`/`weak_ptr` 的所有权模型、`std::move` 到底做了什么，以及完美转发如何保持值类别。
