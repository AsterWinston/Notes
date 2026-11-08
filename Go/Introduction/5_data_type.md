Go 的类型系统简洁但细节丰富：整型分有符号/无符号，浮点遵循 IEEE-754，还有 C 系少见的复数类型，以及让新手困惑的 `byte` 和 `rune`。本章一次讲透。

# 1. 布尔类型 bool

```go
var ok bool        // false
var yes = true

// 注意：Go 的 bool 就是 bool，不能和整数互相转换
// if 1 { } 是非法的，必须写 if true { }；连 bool(n) 转换也非法
var n int = 1
if n != 0 { }     // 正确做法：把条件显式写成 bool 表达式
```

★ **关键认知**：Go 的 `bool` 与数值**严格分离**，不存在 C 语言里"非 0 即真"的隐式转换。`if 1` 编译报错，必须写 `if true`。这让条件判断更严谨，杜绝了 `if (x = 1)` 这类经典笔误。⚠ 没有 truthy/falsy：判空串写 `s != ""`，判空指针写 `p != nil`。

# 2. 整型

## 2.1 整型家族

```
有符号：int8, int16, int32, int64, int
无符号：uint8, uint16, uint32, uint64, uint, uintptr

int/uint：长度等于机器字长（64 位系统上是 64 位）
uintptr：专门用于存放指针的整数（见 unsafe 章节）
范围：int8 -128~127、int16 -32768~32767、int32 ±2^31、int64 ±2^63、uint8 0~255
```

```go
var idx int   = 3              // 索引、计数：一律用 int
var ts  int64 = 1_700_000_000  // 时间戳、协议字段：用定宽的 int64
// idx = ts                    // 报错！64 位平台上二者同宽仍是不同类型
```

★ **关键认知**：`int` 与 `int64` 在 64 位平台**宽度相同但是不同类型**。选型只看一点：是否需要跨平台定宽。索引、循环用 `int`；协议、文件用 `int64`/`int32`，否则 32 位平台宽度漂移会有兼容性 bug。

## 2.2 字节与 rune 的别名

★ **关键认知**：`byte` 是 `uint8` 的别名，`rune` 是 `int32` 的别名。它们不是新类型，只是让代码语义更清晰：

```go
var b byte = 'A'      // byte 表示单个字节，uint8
var r rune = '我'     // rune 表示 Unicode 码点，int32
```

`byte` 处理原始字节流，`rune` 处理字符——这是理解字符串（下篇）的关键。

```go
s := "Go语言"
for i, r := range s {            // r 是 rune（码点），i 是字节下标（不是字符序号！）
    fmt.Printf("%d:%c ", i, r)   // 0:G 1:o 2:语 5:言 —— i 一步跨了 3 字节
}
// s[2] 返回的是字节（byte），不是字符
```

```
"Go语言" 的底层字节序列（range 下标按字节推进：0→1→2→5）：
┌────┬────┬───────────┬───────────┐
│ G  │ o  │ 语（3 字节）│ 言（3 字节）│
└────┴────┴───────────┴───────────┘
```

★ **关键认知**：string 底层是**只读的 UTF-8 字节序列**，`for range` 解码出一个码点后**按其占用字节数（1–4）推进下标**，所以 `i` 是字节偏移。⚠ `len(s)` 是字节数、`s[i]` 是字节、`[]rune(s)` 要 O(n) 解码，详见下篇 string。

## 2.3 整型的默认类型与字面量

```
默认类型：未显式声明类型的整数字面量，默认是 int
字面量形式：
  123       十进制        0x7B      十六进制
  0o173     八进制        0b1111011 二进制
  1_000_000 下划线分隔，提高可读性
```

```go
const shift = 1 << 40    // 无类型整数常量：编译期任意精度，不会被位宽截断
```

★ **关键认知**：字面量本身没有类型，只有**默认类型**。无类型常量编译期保持**任意精度**，`1 << 40` 不会溢出，赋给具体类型时才做越界检查。

## 2.4 溢出与窄化截断

```go
var a uint8 = 255
a++                     // 0：补码回绕，不 panic、不报错
var n int = 300
u := uint8(n)           // 44：高位被静默丢弃（等价 n % 256）
// u2 := uint8(300)     // 常量转换则在编译期直接报 overflow
```

★ **关键认知**：Go 的整数溢出是**补码回绕**，属于定义行为，但**不主动检查**。两条路径：**常量**越界编译期报错；**变量**窄化转换静默截断。⚠ int→int8/uint8 窄化前要显式边界检查。

## 2.5 uintptr 与 unsafe.Pointer 的边界

```go
// 危险用法：跨语句持有 uintptr，再转回指针
u := uintptr(unsafe.Pointer(&x))   // u 里存的只是一个数值，GC 不追踪、不保命
doSomething()                      // 若此刻 x 所在栈被移动，u 就是悬空数字
p := (*int)(unsafe.Pointer(u))     // 语法合法，语义已不安全 ⚠
```

★ **关键认知**：`uintptr` 是"持有指针数值的整数"，服务于 FFI/syscall 边界，**不是泛型指针**。GC 把它当普通整数——不追踪、不保命；且不搬堆对象但**会搬栈**（goroutine 栈扩容时整段拷贝），所以跨语句持有 uintptr 再转回指针是规范级错误。转换要封闭在单个表达式内（syscall 降格是规范允许的特例），指针运算全程用 `unsafe.Pointer`。

# 3. 浮点型

## 3.1 float32 与 float64

```go
var f32 float32 = 3.14
var f64 float64 = 3.14    // 浮点字面量默认是 float64

0.1 + 0.2 != 0.3           // true！浮点运算有精度误差
```

★ **关键认知**：Go 的浮点遵循 IEEE-754 标准，`float32`（4 字节）和 `float64`（8 字节）都有精度问题。`0.1 + 0.2 == 0.3` 为 `false` 不是 Go 的 bug，而是所有遵循 IEEE-754 语言的共性。精确计算（如金额）用 `math/big` 或整数分。

## 3.2 特殊浮点值

```go
import "math"

math.Inf(1)     // 正无穷
math.MaxFloat64 // float64 最大值

n := math.NaN()
n == n            // false！IEEE-754 规定 NaN 与任何值（含自身）比较都为 false
math.IsNaN(n)     // 唯一正确的检查方式
```

# 4. 复数类型

## 4.1 complex64 与 complex128

```go
var c complex128 = complex(1, 2)   // 1 + 2i
var d = 3 + 4i                     // 直接写虚数
real(c)   // 取实部 = 1
imag(c)   // 取虚部 = 2
```

★ **关键认知**：复数（`complex64`=两个 float32、`complex128`=两个 float64）是 Go 少有的内建复数类型，服务于科学计算。日常业务几乎用不到，但有助于理解 `math/cmplx` 标准库。

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

★ **关键认知**：`=`（别名）和空白（新类型定义）语义完全不同。`type byte = uint8` 是别名，byte 能直接当 uint8 用；`type Duration int64`（time 包）是**新类型**，不能直接和 int64 运算——这正是 Go 防止单位混淆的类型安全设计。

## 5.2 字节码与运行时层面的差异

```go
type Celsius = float64       // 别名
type Fahrenheit float64      // 新类型

func (f Fahrenheit) String() string { return fmt.Sprintf("%.1fF", float64(f)) }
// func (c Celsius) String() ...   // 非法！别名不是新类型，等于想给 float64 挂方法
```

★ **关键认知**：别名在字节码里**零痕迹**——编译后即消失，`%T` 打印别名变量只会得到原类型名；新定义类型是类型表中的独立条目（自己的方法集与 `reflect` 名字），布局与底层完全相同，互转不生成指令。分界：**只要语义化名字 → 用 `=`**；**要单位安全、挂方法、做枚举 → 新类型**。

## 5.3 iota 与枚举惯用法

```go
type Weekday int

const (
    _  Weekday = iota   // 0：占掉首行，让有效枚举从 1 起步
    Monday              // 1（继承上一行表达式模板，连同 Weekday 类型）
    Tuesday             // 2
)

fmt.Printf("%T\n", Monday)   // main.Weekday —— 类型隔离生效；赋给 int 必须显式转换
```

★ **关键认知**："**新类型 + iota + const 块**"是 Go 的标准枚举三件套：新类型让 `Monday` 不会被误传进 `int` 参数，iota 免手写取值，还能实现 `Stringer`（别名做不到）。⚠ 0 常表示"未设置"，惯用 `_ = iota` 占首行，让零值区分"没填"和"填了第一项"。

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

这与 C 系语言的隐式提升形成鲜明对比。虽然写起来略繁琐，但避免了大量由隐式转换引起的精度丢失和溢出 bug。而且转换在内存层面**真的要做动作**：int32 → int64 要做符号扩展（补出高 4 字节），int64 → int32 直接丢高 4 字节，float64 → int 抹掉小数部分。

★ **关键认知**：Go 按**内存布局**组织类型——宽度、符号、编码方式任一不同就是不同的表示。与其让编译器悄悄插入有损指令，不如强制写出 `T(v)`，把损失变成**可见的决定**。⚠ `int(3.9)` 得 3（向零截断）；`string(65)` 得 "A"（按码点转换）而不是 "65"，数字转字符串用 `strconv.Itoa`。

# 7. 零值设计

| 类型 | 零值 | 零值能否直接用 |
|------|------|----------------|
| 数值 / bool / string | 0 / false / "" | 可以 |
| slice | nil | 可以：能 append、len 为 0 |
| map | nil | 可读，**写会 panic** ⚠ |
| chan / 指针 / func / 接口 | nil | 需初始化后使用 |
| struct | 所有字段各归零值 | 取决于字段设计 |

```go
type Counter struct{ n int }
var c Counter          // {0}：零值即合法初始态，无需构造函数
var mu sync.Mutex      // 零值 = 未加锁状态
```

★ **关键认知**：零值 = 内存**全零位**，任何分配路径（new、make、栈帧开辟）拿到的都是定义良好的值，从语言层面消灭了"未初始化即 UB"。设计 struct 时应让零值代表一个**有意义的合法状态**（sync.Mutex 零值是未锁、bytes.Buffer 零值是空缓冲），调用方就能省掉初始化。⚠ nil map 能读不能写，nil slice 却可直接 append——两个 nil 行为相反，是日常 panic 头号来源。

# 8. 小结

| 概念 | 要点 |
|------|------|
| bool | 与数值严格分离，不可互转 |
| int/int64 | 64 位平台同宽仍是不同类型；跨平台定宽用后者 |
| byte / rune | uint8 / int32 的别名，string 的基础 |
| 溢出 / 窄化 | 变量静默回绕，常量编译期报错 |
| uintptr | 整数世界的指针数值，转换不可跨语句 |
| float / complex | IEEE-754 有精度误差；内建复数 |
| 别名 vs 定义 | = 别名等价，空白定义新类型 |
| 显式转换 | 布局不同即有损，损失必须可见 |
| 零值 | 全零位即合法；nil map 不能写 ⚠ |
| iota 枚举 | 新类型 + iota + const 三件套 |

下一篇进入字符串，深入 Go 字符串的底层结构（string header）、不可变性、UTF-8 编码，以及 bytes 与 runes 的区别。
