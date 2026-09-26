汇编与 unsafe 是 Go 的"底层逃生舱"。通过汇编可以观察编译器到底生成了什么，通过 unsafe 可以绕过类型系统直接操作内存。它们是理解 Go 底层、做极致优化的必备工具，但也是引入 bug 的高危区域。

# 1. 观察汇编的几种方式

## 1.1 go tool objdump

★ **关键认知**：`go tool objdump` 反汇编已编译的二进制，查看机器码：

```bash
go build -o app main.go
go tool objdump -s main.main app    # 反汇编 main 函数
```

## 1.2 编译期输出汇编

```bash
go build -gcflags="-S" main.go      # 编译时打印汇编
go tool compile -S main.go          # 直接对单文件编译并输出汇编
```

★ **关键认知**：`-gcflags="-S"` 最常用，能看到每个 Go 语句对应的汇编实现。结合 `GOSSAFUNC=main go build` 还能生成可视化的 SSA 流程图，是研究编译优化的利器。

# 2. Plan 9 汇编基础

## 2.1 独特的汇编风格

★ **关键认知**：Go 使用 **Plan 9 汇编**（来自贝尔实验室的 Plan 9 操作系统），语法与 x86 AT&T/Intel 风格都不同：

```
Plan 9 汇编特点：
1. 操作数顺序：源操作数在前，目的操作数在后（和 Intel 相反）
2. 寄存器名：用伪寄存器（AX, BX, SP, FP...），跨平台统一
3. 特殊寄存器：FP（帧指针）、PC、SB（静态基址）

示例：
MOVQ $1, AX      // 把 1 移到 AX（注意：源 → 目的）
ADDQ BX, AX      // AX = AX + BX
```

## 2.2 关键伪寄存器

```
SB（Static Base）：静态基址，全局符号的基址
FP（Frame Pointer）：函数参数区
SP（Stack Pointer）：栈指针
PC（Program Counter）：程序计数器
```

★ **关键认知**：理解 SB/FP/SP 是读懂 Go 汇编的前提。`symbol+0(FP)` 表示"FP 偏移 0 处的符号"（函数参数），`symbol<>(SB)` 表示全局符号。这些伪寄存器让汇编代码跨平台可移植。

# 3. 通过汇编看函数调用

## 3.1 一个简单函数

```go
func add(a, b int) int {
    return a + b
}
```

```
对应汇编（amd64，简化）：
TEXT main.add(SB), NOSPLIT, $0-24    // 函数符号、栈大小 0、参数+返回 24 字节
    MOVQ a+0(FP), AX    // 加载参数 a（FP 偏移 0）
    ADDQ b+8(FP), AX    // 加上参数 b（FP 偏移 8）
    MOVQ AX, ret+16(FP) // 结果存到返回值（FP 偏移 16）
    RET
```

★ **关键认知**：从汇编能看到 Go 函数调用的**参数和返回值都通过栈传递**（FP 区），返回地址和栈帧管理由调用约定决定。`$0-24` 表示"栈帧 0 字节，参数+返回值 24 字节"。

# 4. unsafe 包

## 4.1 unsafe.Pointer —— 万能指针

★ **关键认知**：`unsafe.Pointer` 是 Go 的"万能指针"，类似 C 的 `void*`，可以绕过类型系统：

```go
import "unsafe"

var i int = 42
p := unsafe.Pointer(&i)         // *int → unsafe.Pointer
f := (*float64)(p)              // unsafe.Pointer → *float64（危险！）
```

## 4.2 unsafe.Pointer 的合法用途

★ **关键认知**：官方允许的 unsafe.Pointer 转换模式（`unsafe` 包文档明确列出）：

```
1. *T → unsafe.Pointer → *T（往返）
2. unsafe.Pointer → uintptr（指针转整数）
3. uintptr → unsafe.Pointer（整数转指针）
4. 任意指针 → unsafe.Pointer → 任意指针（类型重解释）
```

## 4.3 uintptr 的危险性

★ **关键认知**：`uintptr` 是**整数**，不是指针，因此**不被 GC 追踪**：

```go
// 危险示例：
p := uintptr(unsafe.Pointer(&x))   // 把指针转成整数
// ... GC 可能在此期间回收 x ...
q := (*int)(unsafe.Pointer(p))     // 用悬垂的整数转回指针 → 悬垂指针！
```

⚠ **关键认知**：`uintptr` 转回指针是 Go 里最危险的 unsafe 操作——GC 不追踪 uintptr，如果对象在转换期间被回收，转回的指针就是悬垂指针。Go 1.19 的 `unsafe.String`/`unsafe.Slice` 和 `checkptr` 机制部分缓解了这个问题，但**指针→uintptr→指针必须保证中间不发生 GC**。

# 5. unsafe 的典型应用

## 5.1 零拷贝 string/[]byte 转换

★ **关键认知**：经典 unsafe 应用——绕过 string 和 []byte 转换的拷贝开销（见字符串章节）：

```go
// 零拷贝 []byte → string（标准库内部也用）
func bytesToString(b []byte) string {
    return *(*string)(unsafe.Pointer(&b))
}

// 零拷贝 string → []byte（危险，结果不可修改）
func stringToBytes(s string) []byte {
    return *(*[]byte)(unsafe.Pointer(&s))
}
```

⚠ **关键认知**：这种转换**极度危险**——string 承诺不可变，但零拷贝转出的 []byte 可以修改，会破坏 string 的不可变契约，导致数据竞争。仅在确认"原数据不会再被修改"时使用，标准库在高性能路径（如 strings.Builder）会谨慎使用。

## 5.2 结构体字段的灵活操作

```go
// 通过偏移访问结构体字段
type S struct {
    a int
    b string
}

p := unsafe.Pointer(&s)
b := (*string)(unsafe.Pointer(uintptr(p) + unsafe.Offsetof(s.b)))
```

# 6. cgo —— 调用 C 代码

## 6.1 cgo 的用法

★ **关键认知**：cgo 让 Go 调用 C 代码（或反之），但代价巨大：

```go
// #include <stdio.h>
// void hello() { printf("hello from C\n"); }
import "C"

func main() {
    C.hello()     // 调用 C 函数
}
```

## 6.2 cgo 的代价

★ **关键认知**：cgo 是"万不得已才用"的手段，因为：

```
1. 性能损失：C 调用需跨越 Go/C 边界，有显著开销
2. 破坏并发：C 调用期间占用 OS 线程，影响调度
3. 失去可移植性：依赖 C 编译器和目标平台
4. 内存管理复杂：Go 和 C 的指针传递有严格限制
```

★ **关键认知**：cgo 会让 Go 失去"纯静态链接、跨平台编译、goroutine 高效调度"等核心优势。多数场景应该用纯 Go 实现，或通过进程间通信（gRPC、管道）替代 cgo。

# 7. 何时使用汇编与 unsafe

★ **关键认知**：汇编和 unsafe 的使用原则——**能不用就不用**：

```
使用时机：
1. 性能关键且编译器优化已达瓶颈（如 crypto、math 库）
2. 需要访问 Go 未暴露的底层能力
3. 跨语言互操作（cgo）

不用时机：
1. 绝大多数业务代码（可读性、安全性优先）
2. 可以用标准库解决（如用 unsafe 库而非自己写 unsafe）
```

# 8. 小结

| 概念 | 要点 |
|------|------|
| 观察汇编 | go tool objdump / -gcflags="-S" |
| Plan 9 汇编 | 源→目的操作数顺序 |
| SB/FP/SP | 静态基址/帧指针/栈指针 |
| unsafe.Pointer | 万能指针，绕过类型系统 |
| uintptr | 整数，不被 GC 追踪 |
| 零拷贝转换 | 危险，破坏不可变契约 |
| cgo | 万不得已才用 |

下一篇进入 context 包，讲清楚 Go 标准库的取消/超时/值传递机制，以及它的底层实现和正确使用姿势。
