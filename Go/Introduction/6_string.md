字符串是 Go 中使用最频繁的类型之一，但它的底层实现（string header + 不可变字节序列 + UTF-8）决定了它的很多"反直觉"行为。理解字符串的底层，是理解切片、`[]byte` 转换、以及大量性能优化技巧的前提。

# 1. string 的底层结构

## 1.1 string header

★ **关键认知**：Go 的字符串在内存中是一个 16 字节（64 位系统）的 `reflect.StringHeader` 结构体，由**指向底层字节数组的指针 + 长度**两部分组成：

```go
// runtime/string.go 中的定义（简化）
type stringStruct struct {
    str unsafe.Pointer   // 指向底层字节数组的指针（8 字节）
    len int              // 字节长度（8 字节）
}
```

```
内存布局：
string 变量（16 字节）
┌──────────────┬──────────────┐
│  str 指针    │     len      │
└──────┬───────┴──────────────┘
       │
       ▼
   [h][e][l][l][o]     ← 底层字节数组（不可变）
```

★ **关键认知**：字符串**本身**只是一个 16 字节的"描述符"，真正的内容存放在别处的字节数组中。这解释了为什么字符串作为参数传递是"值拷贝但廉价"——拷贝的只是 16 字节的 header，不是底层内容。

## 1.2 不可变性

```go
s := "hello"
s[0] = 'H'    // 报错：cannot assign to s[0]，字符串不可修改
```

★ **关键认知**：字符串内容**不可变**。任何"修改"操作（如 `s += "x"`）实际上都会**分配新的底层数组**，产生一个新字符串。这个不可变性保证了字符串可以安全地被多个 goroutine 并发读取而无需加锁，也是字符串可被用作 map 键的前提。

# 2. UTF-8 编码与 rune

## 2.1 字符串是字节序列，不是字符序列

```go
s := "Go语言"
len(s)          // 8！因为"语""言"各占 3 个字节，len 返回字节数而非字符数
```

★ **关键认知**：Go 源码和字符串都使用 **UTF-8** 编码。`len(s)` 返回的是**字节数**而非字符数。ASCII 字符占 1 字节，中文等字符占 3 字节，某些 emoji 占 4 字节。这是 Go 新手最常见误解之一。

## 2.2 byte vs rune 的遍历

```go
s := "Go语言"

// 按字节遍历（会拆散中文字符）
for i := 0; i < len(s); i++ {
    fmt.Printf("%c", s[i])   // Goè¯­è¨ —— 乱码！中文被拆成 3 个字节
}

// 按 rune（字符）遍历（推荐）
for i, r := range s {
    fmt.Printf("%d:%c ", i, r)   // 0:G 1:o 2:语 5:言
}
```

★ **关键认知**：`for range` 遍历字符串时，Go 会**自动按 UTF-8 解码成 rune**，索引 `i` 是每个 rune 的**字节起始位置**（所以上例从 2 跳到 5）。要正确处理多字节字符，务必用 `range` 而不是下标索引。

## 2.3 字节与字符的互转

```go
s := "Go语言"

bs := []byte(s)      // 字符串 → 字节切片（深拷贝）
rs := []rune(s)      // 字符串 → rune 切片

string(bs)           // 字节切片 → 字符串
string(rs)           // rune 切片 → 字符串

len([]rune(s))       // 4，真正的字符数
```

★ **关键认知**：`[]byte(s)` 和 `[]rune(s)` 都是**深拷贝**，会分配新内存。频繁做这种转换是性能陷阱。编译器在某些场景（如 `string(bytes)` 且原切片不再使用）会做零拷贝优化，但不能依赖。

# 3. 字符串的常用操作

## 3.1 拼接与比较

```go
s := "hello" + " world"       // + 拼接
strings.Join([]string{"a","b"}, ",")   // 批量拼接更高效
strings.Contains(s, "world")  // 包含判断
strings.HasPrefix(s, "he")    // 前缀
strings.HasSuffix(s, "ld")    // 后缀
strings.Index(s, "world")     // 子串位置
strings.Split(s, " ")         // 分割
```

## 3.2 拼接的性能陷阱

★ **关键认知**：`s += x` 在循环中反复拼接是 **O(n²)** 的性能灾难——每次 `+=` 都分配新数组并拷贝全部旧内容。正确做法是用 `strings.Builder`：

```go
var b strings.Builder
for i := 0; i < 1000; i++ {
    b.WriteString("a")
}
result := b.String()   // 内部预分配，接近 O(n)
```

# 4. 字符串与数字的转换

## 4.1 strconv 包

```go
import "strconv"

strconv.Itoa(123)             // int → string，"123"
strconv.Atoi("123")           // string → int，返回 (int, error)
strconv.FormatFloat(3.14, 'f', 2, 64)   // float → "3.14"
strconv.ParseFloat("3.14", 64)
strconv.FormatBool(true)      // "true"
```

★ **关键认知**：**永远不要用 `string(65)` 把数字转字符串**——`string(65)` 会把 65 当作 Unicode 码点转成字符 `"A"`，而不是 `"65"`。这是 Go 里极高频的坑：数字转字符串用 `strconv.Itoa`，字节转字符串才用 `string([]byte)`。

# 5. 底层进阶：零拷贝转换

## 5.1 string 与 []byte 的零拷贝转换

★ **关键认知**：由于 `string` 和 `[]byte` 的底层结构几乎相同（都是指针 + 长度），可以用 `unsafe` 实现**零拷贝**互转。标准库和大量高性能库都在用这个技巧：

```go
// 原理：绕过类型系统，直接复用底层字节数组
func bytesToString(b []byte) string {
    return *(*string)(unsafe.Pointer(&b))
}
```

⚠ **关键认知**：这个技巧有严格前提——转换后必须保证原底层数组**不再被修改**（因为 string 承诺不可变）。否则会破坏 string 的不可变契约，导致数据竞争。生产代码应优先使用标准库 `strings`/`unsafe` 提供的安全版本，仅在确认安全且性能关键时使用。

# 6. 小结

| 概念 | 要点 |
|------|------|
| 底层结构 | 指针 + 长度（16 字节 header） |
| 不可变 | 修改即分配新数组 |
| len() | 返回字节数，非字符数 |
| UTF-8 | 中文 3 字节，emoji 4 字节 |
| range 遍历 | 按 rune 解码，索引是字节位置 |
| string(65) | 是码点转字符，不是数字转字符串 |

下一篇进入数组与切片，这是 Go 最重要的复合类型。切片底层的"三字段结构 + 扩容机制"是面试和性能优化的核心考点。
