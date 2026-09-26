Go 的类型系统简洁但细节丰富：整型分有符号/无符号且长度从 8 到 64 位，浮点遵循 IEEE-754，还有 C 系语言少见的复数类型，以及让新手困惑的 `byte` 和 `rune`。本章把这些一次讲透。

# 1. 布尔类型 bool

```go
var ok bool        // false
var yes = true

// 注意：Go 的 bool 就是 bool，不能和整数互相转换
// if 1 { } 是非法的，必须写 if true { }
```

★ **关键认知**：Go 的 `bool` 与数值**严格分离**，不存在 C 语言里"非 0 即真"的隐式转换。`if 1` 编译报错，必须写 `if true`。这让条件判断更严谨，杜绝了 `if (x = 1)` 这类经典笔误。

# 2. 整型

## 2.1 整型家族

```
有符号：int8, int16, int32, int64, int
无符号：uint8, uint16, uint32, uint64, uint, uintptr

int/uint：长度等于机器字长（64 位系统上是 64 位）
uintptr：专门用于存放指针的整数（见 unsafe 章节）

范围：
int8     -128 ~ 127
int16    -32768 ~ 32767
int32    -2^31 ~ 2^31-1
int64    -2^63 ~ 2^63-1
uint8    0 ~ 255
```

## 2.2 字节与 rune 的别名

★ **关键认知**：`byte` 是 `uint8` 的别名，`rune` 是 `int32` 的别名。它们不是新类型，只是让代码语义更清晰：

```go
var b byte = 'A'      // byte 表示单个字节，uint8
var r rune = '我'     // rune 表示 Unicode 码点，int32
```

`byte` 用于处理原始字节流，`rune` 用于处理字符。这个区别是理解字符串（下一章）的关键。

## 2.3 整型的默认类型与字面量

```
默认类型：未显式声明类型的整数字面量，默认是 int
字面量形式：
  123       十进制
  0x7B      十六进制
  0o173     八进制
  0b1111011 二进制
  1_000_000 下划线分隔，提高可读性
```

# 3. 浮点型

## 3.1 float32 与 float64

```go
var f32 float32 = 3.14
var f64 float64 = 3.14    // 浮点字面量默认是 float64

// 浮点表示不精确：
0.1 + 0.2 != 0.3           // true！浮点运算有精度误差
```

★ **关键认知**：Go 的浮点遵循 IEEE-754 标准，`float32`（4 字节）和 `float64`（8 字节）都存在精度问题。`0.1 + 0.2 == 0.3` 为 `false` 不是 Go 的 bug，而是所有遵循 IEEE-754 语言的共性。需要精确计算（如金额）时应使用 `math/big` 或整数分（int 表示"分"）。

## 3.2 特殊浮点值

```go
import "math"

math.Inf(1)     // 正无穷
math.NaN()      // 非数值
math.MaxFloat64 // float64 最大值
```

# 4. 复数类型

## 4.1 complex64 与 complex128

```go
var c complex128 = complex(1, 2)   // 1 + 2i
var d = 3 + 4i                     // 直接写虚数
real(c)   // 取实部 = 1
imag(c)   // 取虚部 = 2
```

★ **关键认知**：复数（`complex64`/`complex128`）是 Go 相较多数主流语言少有的内建类型，主要服务于科学计算。日常业务开发几乎用不到，但知道它的存在有助于理解 `math/cmplx` 标准库。

# 5. 类型别名与类型定义

## 5.1 两者的区别

```go
type MyInt1 = int    // 类型别名：MyInt1 和 int 完全等价
type MyInt2 int      // 类型定义：MyInt2 是新类型，底层是 int

var a MyInt1 = 10
var b MyInt2 = 10
var x int = a        // 合法，MyInt1 就是 int
var y int = b        // 报错！MyInt2 是不同类型，需显式转换
```

★ **关键认知**：`=`（别名）和空白（新类型定义）语义完全不同。`type byte = uint8` 是别名，所以 byte 能直接当 uint8 用；`type Duration int64`（标准库 time 包）是**新类型**，即使底层是 int64 也不能直接和 int64 运算，必须转换——这正是 Go 类型安全（防止单位混淆）的体现。

# 6. 类型转换（无隐式转换）

★ **关键认知**：Go 是**强类型**语言，且**没有隐式类型转换**。不同数值类型之间运算必须显式转换：

```go
var i int = 42
var f float64 = float64(i)   // 必须显式转换
var u uint = uint(f)

// 这也意味着：
i := 1
var f64 float64
f64 = i          // 报错：cannot use i (type int) as type float64
```

这与 C 系语言的隐式提升形成鲜明对比。虽然写起来略繁琐，但避免了大量由隐式转换引起的精度丢失和溢出 bug。

# 7. 小结

| 类型 | 说明 |
|------|------|
| bool | 与数值严格分离 |
| int/int8~64, uint/uint8~64 | 长度明确，int 随机器字长 |
| byte / rune | uint8 / int32 的别名 |
| float32/float64 | IEEE-754，有精度误差 |
| complex64/128 | 内建复数 |
| 别名 vs 定义 | = 别名等价，空白定义新类型 |

下一篇进入字符串，深入 Go 字符串的底层结构（string header）、不可变性、UTF-8 编码，以及 bytes 与 runes 的区别。
