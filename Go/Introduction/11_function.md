函数是 Go 代码的基本单元。Go 的函数设计有几大特色：多返回值（彻底改变了错误处理方式）、命名返回值、闭包、以及贯穿整个语言生命周期的 `defer`/`panic`/`recover` 机制。这一章深入函数本身，defer 的底层实现在下一章展开。

# 1. 函数定义

## 1.1 基本语法

```go
func add(a, b int) int {     // 参数类型可合并声明
    return a + b
}

func swap(a, b int) (int, int) {   // 多返回值
    return b, a
}

func sum(nums ...int) int {   // 可变参数
    total := 0
    for _, n := range nums {
        total += n
    }
    return total
}
```

## 1.2 多返回值

★ **关键认知**：多返回值是 Go 最重要的设计之一。它让函数能同时返回"结果 + 错误"，彻底取代了 C 的 errno、Java 的异常抛出来表达错误的方式：

```go
func div(a, b int) (int, error) {
    if b == 0 {
        return 0, errors.New("division by zero")
    }
    return a / b, nil
}

result, err := div(10, 0)
if err != nil {
    // 处理错误
}
```

这种"显式返回错误值"的模式，让错误处理变得显式、可见、难以忽略（详见错误处理章节）。

## 1.3 命名返回值

```go
func split(sum int) (x, y int) {   // 返回值有名字，零值初始化
    x = sum * 4 / 9
    y = sum - x
    return      // 裸 return，返回命名返回值的当前值
}
```

★ **关键认知**：命名返回值在函数开头就被**零值初始化**。裸 `return` 会返回它们的当前值。命名返回值常用于文档化和配合 `defer` 修改返回值（见下文），但过度使用会降低可读性。

# 2. 函数是一等公民

## 2.1 函数类型与赋值

```go
var f func(int, int) int     // 声明函数类型的变量
f = add
f(1, 2)

// 函数作为参数
func apply(fn func(int) int, x int) int {
    return fn(x)
}

// 函数作为返回值
func adder() func(int) int {
    return func(x int) int { return x + 1 }
}
```

★ **关键认知**：Go 函数是**一等公民**——可以赋给变量、作参数、作返回值。这是闭包、高阶函数、函数式风格的基础。

## 2.2 闭包

★ **关键认知**：闭包 = 函数 + 它捕获的外部变量。Go 的闭包捕获的是**变量的引用**而非值的快照：

```go
func counter() func() int {
    count := 0
    return func() int {
        count++        // 捕获并修改外部 count
        return count
    }
}

c := counter()
c()   // 1
c()   // 2
```

⚠ **关键认知**：闭包捕获变量引用，导致经典陷阱——循环中创建闭包时，所有闭包共享同一个循环变量（Go 1.22 前）：

```go
// Go 1.22 前，以下代码全部打印 3
var funcs []func()
for i := 0; i < 3; i++ {
    funcs = append(funcs, func() { fmt.Println(i) })
}
// 修复：i := i 重新声明，或用参数传递
// Go 1.22 起，for 循环变量每次迭代都是新变量，此坑已消除
```

# 3. 方法的本质

★ **关键认知**：Go 的"方法"其实是**带接收者的函数**，接收者写在 `func` 和函数名之间：

```go
type Point struct{ X, Y int }

func (p Point) Move(dx, dy int) Point {   // 值接收者
    p.X += dx
    p.Y += dy
    return p
}

func (p *Point) MoveInPlace(dx, dy int) { // 指针接收者
    p.X += dx
    p.Y += dy
}
```

值接收者 vs 指针接收者的选择是方法设计的关键决策，完整分析见结构体与方法章节。

# 4. defer / panic / recover

## 4.1 defer —— 延迟执行

★ **关键认知**：`defer` 声明的函数调用会**延迟到当前函数 return 前执行**，且遵循 **LIFO（后进先出）** 顺序：

```go
func f() {
    defer fmt.Println("first")    // 最后执行
    defer fmt.Println("second")   // 先执行
    fmt.Println("body")
}
// 输出：body, second, first
```

`defer` 的典型用途：

```go
func readFile(path string) error {
    f, err := os.Open(path)
    if err != nil {
        return err
    }
    defer f.Close()    // 确保文件一定被关闭，即使中途 return 或 panic
    // ... 使用 f ...
    return nil
}
```

## 4.2 defer 的参数求值时机

★ **关键认知**：defer 的参数在 **defer 语句执行时**立即求值，而不是在真正调用时：

```go
func f() {
    i := 1
    defer fmt.Println(i)   // 参数 i 立即求值 = 1
    i = 2
    // 最终打印 1，不是 2
}
```

若想让 defer 读到最新值，用闭包（引用变量而非传参）。

## 4.3 defer 修改返回值

★ **关键认知**：defer 配合**命名返回值**，可以在函数返回前修改返回值——这是 defer 的高级用法：

```go
func f() (result int) {
    defer func() {
        result++    // 修改命名返回值
    }()
    return 10
}
// f() 返回 11
```

## 4.4 panic 与 recover

★ **关键认知**：`panic` 用于表达"无法继续执行"的严重错误，`recover` 只能在 `defer` 函数中捕获 panic：

```go
func safeCall() {
    defer func() {
        if r := recover(); r != nil {
            fmt.Println("recovered:", r)
        }
    }()
    panic("something went wrong")
}
```

⚠ **关键认知**：`recover` 必须**直接**在 defer 函数中调用才有效，嵌套一层就失效。且 panic 只应在真正的致命错误（如程序 bug）时使用，业务错误应走 error 返回值。Go 社区强烈反对用 panic/recover 做常规错误处理。

# 5. 小结

| 概念 | 要点 |
|------|------|
| 多返回值 | 结果 + 错误的惯用表达 |
| 命名返回值 | 零值初始化，裸 return |
| 一等公民 | 可作参数/返回值 |
| 闭包 | 捕获变量引用，注意循环陷阱 |
| 方法 | 带接收者的函数 |
| defer | LIFO，参数立即求值，可改返回值 |
| panic/recover | 仅致命错误，recover 须在 defer 中 |

下一篇深入 defer 的底层实现（defer 栈 vs open-coded defer、panic 处理链），讲清楚 defer 为什么开销低、以及 defer 在循环中的性能陷阱。
