Go 的控制流设计得刻意克制：`for` 是**唯一**的循环关键字（没有 while/do-while），`switch` 有隐式 break 和 fallthrough，`select` 专用于并发。理解这些"精简"设计背后的取舍，能写出更地道的 Go 代码。

# 1. if 语句

## 1.1 基本形式与前置语句

```go
// 基本 if
if x > 0 {
    fmt.Println("positive")
} else if x < 0 {
    fmt.Println("negative")
} else {
    fmt.Println("zero")
}

// if 前置语句（Go 特色）：在 if 前声明变量，作用域仅限于 if 块
if err := doSomething(); err != nil {
    return err
}
// err 在这里不可见
```

★ **关键认知**：`if` 支持**前置语句**（init statement），常与错误处理结合使用。前置语句声明的变量作用域**仅限于整个 if-else 链**，用完即销毁——这让代码更紧凑，也避免了变量泄漏到外层作用域。

## 1.2 条件必须为 bool

★ **关键认知**：`if` 的条件表达式**必须是 bool 类型**，不允许 C 风格的"非 0 即真"：

```go
if 1 { }          // 编译错误：non-boolean condition
if x != 0 { }     // 正确，必须显式比较
```

# 2. for 循环（唯一循环）

## 2.1 for 的四种形态

```go
// 1. 经典三要素
for i := 0; i < 10; i++ {
    fmt.Println(i)
}

// 2. 只有条件（等价 while）
n := 0
for n < 10 {
    n++
}

// 3. 无限循环（等价 for(;;)）
for {
    // 需配合 break 退出
}

// 4. range 遍历（切片/map/字符串/通道）
for i, v := range []int{1, 2, 3} {
    fmt.Println(i, v)
}
```

★ **关键认知**：Go 用 `for` 一种语法统一了 C 系语言的 `for`、`while`、`do-while` 三种循环。省略分号的不同组合就对应不同形态，减少了关键字数量，也让循环语义更统一。

## 2.2 range 的返回值

```go
// 遍历切片/数组：索引 + 值
for i, v := range s { }

// 遍历 map：键 + 值（顺序随机）
for k, v := range m { }

// 遍历字符串：字节索引 + rune
for i, r := range "hello" { }

// 遍历通道：只返回值
for v := range ch { }

// 用 _ 忽略不需要的部分
for _, v := range s { }
for i := range s { }
```

## 2.3 break 与 continue

```go
for i := 0; i < 10; i++ {
    if i == 5 {
        break      // 跳出整个循环
    }
    if i%2 == 0 {
        continue   // 跳过本次，继续下一次
    }
}

// 标签 break：跳出多层循环
outer:
for i := 0; i < 3; i++ {
    for j := 0; j < 3; j++ {
        if i*j > 4 {
            break outer   // 跳出外层循环
        }
    }
}
```

# 3. switch 语句

## 3.1 隐式 break

★ **关键认知**：Go 的 `switch` 每个 case **默认自动 break**，不需要手动写（避免 C 里的"忘记 break 穿透"bug）：

```go
switch day {
case 1:
    fmt.Println("Monday")
    // 自动 break，不会穿透到下一个 case
case 2:
    fmt.Println("Tuesday")
default:
    fmt.Println("other")
}
```

## 3.2 fallthrough 与多值匹配

```go
// 显式穿透（少见，慎用）
switch x {
case 1:
    fallthrough      // 强制执行下一个 case
case 2:
    fmt.Println("x is 1 or 2")
}

// 多值匹配
switch x {
case 1, 3, 5:
    fmt.Println("odd")
}

// 无表达式的 switch（等价于 if-else 链）
switch {
case x > 10:
    fmt.Println("big")
case x > 5:
    fmt.Println("medium")
default:
    fmt.Println("small")
}
```

## 3.3 switch 的类型匹配（type switch）

```go
var i interface{} = 10
switch v := i.(type) {      // type switch，详见接口章节
case int:
    fmt.Println("int:", v)
case string:
    fmt.Println("string:", v)
default:
    fmt.Println("unknown")
}
```

# 4. select 语句

★ **关键认知**：`select` 专用于**channel 的多路复用**，从多个通道中选择就绪的操作执行：

```go
select {
case v := <-ch1:
    fmt.Println("from ch1", v)
case v := <-ch2:
    fmt.Println("from ch2", v)
case ch3 <- 1:
    fmt.Println("sent to ch3")
default:
    fmt.Println("none ready")
}
```

`select` 是 Go 并发模型的核心语法，完整的机制（随机选择、default 非阻塞、超时模式）在并发章节详细展开。

# 5. goto 语句

★ **关键认知**：Go 保留了 `goto` 但强烈建议**少用**。它的主要合法用途是跳出深层嵌套、或实现统一的错误处理出口：

```go
func process() error {
    f, err := openFile()
    if err != nil {
        goto fail
    }
    // ... 处理 ...
    if err2 := write(f); err2 != nil {
        goto fail
    }
    return nil

fail:           // 统一清理出口
    f.Close()
    return err
}
```

Go 对 `goto` 有严格限制：**不能跳转到跳过变量声明的代码块**（防止跳过了变量初始化却使用了它）。

# 6. 小结

| 结构 | 要点 |
|------|------|
| if | 条件必须 bool，支持前置语句 |
| for | 唯一循环，四种形态 |
| range | 遍历 slice/map/string/chan |
| switch | 隐式 break，fallthrough 显式穿透 |
| select | channel 多路复用 |
| goto | 慎用，禁止跳过变量声明 |

下一篇进入函数，讲清楚多返回值、命名返回值、可变参数、闭包，以及 Go 最独特的 defer/panic/recover 错误处理机制。
