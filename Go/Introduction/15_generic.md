泛型是 Go 1.18 引入的最重大特性（十年等待），它补上了 Go 类型系统的最后一块拼图。理解泛型的语法、约束机制，以及它的**编译实现方式**（不同于 Java 的类型擦除，也不同于 C++ 的模板实例化），是写出正确高效泛型代码的关键。

# 1. 泛型解决的问题

## 1.1 泛型前的痛点

```go
// 没有泛型时，为不同类型写重复代码
func MaxInt(a, b int) int { ... }
func MaxFloat(a, b float64) float64 { ... }

// 或用空接口，但丢失类型安全 + 运行时开销
func MaxAny(a, b interface{}) interface{} {
    // 需要类型断言，容易出错
}
```

★ **关键认知**：泛型解决的正是"**同一算法适用于多种类型，且保持类型安全**"的问题。它消除了大量重复代码，也避免滥用空接口导致的类型不安全。

# 2. 类型参数与约束

## 2.1 基本语法

```go
// 类型参数声明在方括号中
func Max[T int | float64](a, b T) T {   // T 是类型参数，约束是 int | float64
    if a > b {
        return a
    }
    return b
}

x := Max(3, 5)          // Max[int]
y := Max(3.14, 2.71)    // Max[float64]
```

## 2.2 泛型类型

```go
// 泛型结构体
type Stack[T any] struct {
    items []T
}

func (s *Stack[T]) Push(item T) {
    s.items = append(s.items, item)
}

// 泛型 map 类型
type Set[T comparable] map[T]struct{}

s := Stack[int]{}
s.Push(1)
```

★ **关键认知**：`any` 是 `interface{}` 的别名，作为约束表示"任意类型"；`comparable` 是内建约束，表示"可比较类型"（能作 map 键）。类型参数可以用在函数、结构体、方法上。

## 2.3 类型约束（type constraint）

```go
// 约束用接口定义
type Number interface {
    int | int64 | float64    // 类型集合：只能是这些类型
}

func Sum[T Number](vals []T) T {
    var total T
    for _, v := range vals {
        total += v
    }
    return total
}
```

★ **关键认知**：泛型的"约束"本质是**类型集合（type set）**——约束定义了"哪些类型可以作 T"。`int | int64` 这种并集写法叫"类型项"，表示 T 只能是列出的类型之一。

# 3. 类型集合详解

## 3.1 约束的三种形式

```go
// 1. 任意类型
type C1 = any                       // interface{}，无约束

// 2. 类型并集
type C2 interface {
    int | string | []byte           // 只能是这三种

}

// 3. 方法约束（传统接口，约束"必须有哪些方法"）
type C3 interface {
    String() string                 // 必须有 String 方法
}
```

## 3.2 带底层类型的约束

```go
// ~ 表示"底层类型为 T 的所有类型"
type AnyInt interface {
    ~int | ~int64    // 包括 int 以及所有基于 int 定义的新类型
}

type MyInt int       // MyInt 的底层类型是 int，满足 ~int

func Double[T AnyInt](x T) T { return x * 2 }

Double(MyInt(5))     // 合法，MyInt 满足 ~int
```

★ **关键认知**：`~int` 匹配"底层类型是 int 的所有类型"（包括 `type MyInt int` 这类自定义类型）。没有 `~` 时，`int` 只精确匹配 int。这是泛型约束里最容易被忽视的细节。

# 4. 泛型的编译实现

## 4.1 GCShape 字典传递

★ **关键认知**：Go 泛型的编译实现既**不是** Java 的类型擦除，也**不是** C++ 的完全模板实例化，而是一种**混合方案——GCShape（共享实例化 + 字典传递）**：

```
1. 共享实例化：底层内存布局相同的类型（如所有指针类型，或所有 int 宽度类型）
   共享同一份编译后的机器码，而非每个类型都生成一份（避免代码膨胀）
2. 字典传递：泛型函数内部需要访问类型相关信息（如 T 的大小、方法）时，
   通过额外传入的"字典"参数获取
```

★ **关键认知**：Go 选择 GCShape 是为了平衡两个目标：**不牺牲泛型的运行时性能**（避免接口装箱）和**不产生代码爆炸**（避免 C++ 式每个实例化一份）。代价是泛型代码在某些场景（如需要类型具体信息做优化时）无法内联某些操作。

## 4.2 性能启示

★ **关键认知**：Go 泛型的性能特征：

```
1. 泛型函数调用比接口调用快（无装箱、无动态派发）
2. 泛型代码的实例化发生在编译期，无运行时反射开销
3. 但泛型函数可能比手写的具体类型函数略慢（字典间接访问）
```

实践中，泛型带来的类型安全和代码复用价值远超其微小的性能差异，不必过度纠结。

# 5. 泛型的使用场景与限制

## 5.1 适用场景

```
1. 容器类：Stack[T]、Set[T]、Map/Reduce 工具
2. 算法复用：Max、Sort、Filter 等类型无关逻辑
3. 消除空接口：把 map[string]any 换成类型安全的泛型
```

## 5.2 泛型的限制

★ **关键认知**：Go 泛型有几个刻意的限制：

```
1. 方法不能有自己的类型参数（类型参数只能在类型或函数级别声明）
2. 不能对类型参数做算术运算（除非约束明确允许，如 ~int）
3. 不能访问类型参数的字段（除非约束是结构体接口）
4. 泛型类型的方法不能是另一泛型（无高阶泛型）
```

这些限制让 Go 泛型保持简洁，避免走向 C++ 模板元编程的复杂度深渊。

# 6. 标准库的泛型应用

★ **关键认知**：Go 1.18 起，标准库大量类型被泛型化，最典型的是 `slices` 和 `maps` 包：

```go
import "slices"

slices.Sort([]int{3, 1, 2})          // 泛型排序
slices.Contains([]string{"a"}, "a")  // 泛型包含判断
slices.Max([]int{1, 5, 3})           // 泛型最大值

import "maps"

m := map[string]int{"a": 1}
maps.Keys(m)    // 泛型取键
```

# 7. 小结

| 概念 | 要点 |
|------|------|
| 类型参数 | 方括号声明 T |
| 约束 | 类型集合，any/comparable/接口 |
| ~ | 底层类型匹配 |
| 编译实现 | GCShape 共享实例化 + 字典 |
| 性能 | 优于接口，无代码爆炸 |
| 限制 | 方法不能带类型参数等 |

下一篇进入并发基础，这是 Go 的灵魂。讲清楚 goroutine 的轻量本质、channel 的底层结构（hchan）、以及"通过通信共享内存"的哲学。
