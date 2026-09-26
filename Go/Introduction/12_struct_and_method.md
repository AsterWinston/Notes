结构体是 Go 里聚合数据的基本方式，方法则让结构体具备行为。本章重点讲三个底层话题：结构体的**内存对齐**（影响体积和性能）、**方法集**（值接收者 vs 指针接收者的本质区别）、以及**结构体嵌入**（Go 的"继承"替代品）。

# 1. 结构体定义与使用

## 1.1 定义与实例化

```go
type Person struct {
    Name string
    Age  int
    Addr Address       // 嵌套结构体
}

type Address struct {
    City string
    Zip  string
}

// 多种实例化方式
p1 := Person{Name: "tom", Age: 18}
p2 := Person{"jerry", 20}          // 按顺序，少用（易错）
p3 := new(Person)                  // *Person 指针
p4 := &Person{Name: "bob"}         // 指针，最常用
```

## 1.2 访问字段

```go
p := &Person{Name: "tom", Age: 18}
p.Age = 20          // 指针会自动解引用，等价 (*p).Age = 20
```

★ **关键认知**：Go 里通过指针访问字段时**自动解引用**，`p.Age` 和 `(*p).Age` 等价。这种语法糖让指针和值的使用体验几乎一致。

# 2. 结构体的内存对齐

## 2.1 对齐规则

★ **关键认知**：结构体字段在内存中按**对齐规则**排布——每个字段的起始地址必须是其自身大小的整数倍，且整个结构体的大小要能被其最大字段大小整除。这会产生**填充（padding）**：

```go
type Bad struct {
    A byte    // 1 字节
    B int64   // 8 字节
    C byte    // 1 字节
}
// 内存布局（64 位系统）：
// A [1] + 填充[7] + B [8] + C [1] + 填充[7] = 24 字节

type Good struct {
    A byte    // 1 字节
    C byte    // 1 字节
    B int64   // 8 字节
}
// 内存布局：
// A[1] + C[1] + 填充[6] + B[8] = 16 字节
```

## 2.2 用 unsafe 验证

```go
import "unsafe"

type Bad struct {
    A byte
    B int64
    C byte
}

unsafe.Sizeof(Bad{})        // 24
unsafe.Offsetof(Bad{}.B)    // 8（B 的偏移）
```

★ **关键认知**：字段排列顺序直接影响结构体体积。把小的字段排在一起能减少 padding，节省内存。这在大规模数据结构（如缓存、高频对象）中影响显著。完整的对齐机制见内存布局章节。

# 3. 方法与方法集

## 3.1 值接收者 vs 指针接收者

```go
type Counter struct{ N int }

func (c Counter) Add() {     // 值接收者：操作副本
    c.N++
}

func (c *Counter) AddPtr() { // 指针接收者：操作原对象
    c.N++
}

c := Counter{}
c.Add()        // c.N 仍为 0（改了副本）
c.AddPtr()     // c.N 变为 1
```

## 3.2 方法集（Method Set）

★ **关键认知**：方法集是"类型 T 或 *T 能调用哪些方法"的规则，是接口实现判断的基础：

```
类型 T 的方法集：所有值接收者 (T) 的方法
类型 *T 的方法集：所有值接收者 (T) + 所有指针接收者 (*T) 的方法
```

| | 值接收者方法 | 指针接收者方法 |
|---|---|---|
| T 值可调用 | ✅ | ❌ |
| *T 指针可调用 | ✅ | ✅ |

★ **关键认知**：这条规则决定了接口的实现。如果接口 `I` 需要的方法中有指针接收者方法，那么只有 `*T` 能实现该接口，`T` 不行。这是 Go 里"为什么接口经常要求传指针"的底层原因（详见接口章节）。

## 3.3 选择原则

★ **关键认知**：选择值还是指针接收者的经验法则：

```
用指针接收者，当：
1. 方法需要修改接收者本身
2. 接收者是"大"结构体，避免每次调用拷贝
3. 接收者包含 sync.Mutex 等不能拷贝的字段（拷贝锁会破坏语义）

用值接收者，当：
1. 方法不修改接收者，且结构体很小
2. 接收者是 map/slice/chan（本身就是引用语义）
```

# 4. 结构体嵌入（组合优于继承）

## 4.1 嵌入语法

★ **关键认知**：Go 没有继承，用**嵌入（embedding）**实现代码复用——把一个类型"内嵌"进另一个结构体：

```go
type Animal struct {
    Name string
}

func (a Animal) Speak() {
    fmt.Println(a.Name, "speaks")
}

type Dog struct {
    Animal        // 嵌入：Dog 自动获得 Animal 的字段和方法
    Breed string
}

d := Dog{Animal: Animal{Name: "buddy"}, Breed: "golden"}
d.Speak()         // 直接调用嵌入类型的方法（提升）
d.Name = "max"    // 直接访问嵌入类型的字段（提升）
```

## 4.2 字段与方法的提升

★ **关键认知**：嵌入类型的方法和字段会被"提升"到外层结构体，可直接访问。这实现了类似继承的效果，但本质是**组合**——Dog 不是 Animal 的子类，而是"包含"一个 Animal。它没有多态继承的脆弱基类问题，更符合组合优于继承的原则。

## 4.3 命名冲突

```go
type A struct{ X int }
type B struct{ X int }
type C struct {
    A
    B
}
// c.X 会产生歧义，编译错误，必须写 c.A.X 或 c.B.X
```

★ **关键认知**：多个嵌入类型有同名字段/方法时，直接访问会产生**歧义编译错误**，必须显式指定路径。这避免了 C++ 多重继承的"菱形继承"复杂性。

# 5. 结构体标签（Tag）

```go
type User struct {
    Name string `json:"name" validate:"required"`
    Age  int    `json:"age,omitempty"`
}
```

★ **关键认知**：结构体标签是**编译期存储的元数据**，通过反射读取（`reflect.TypeOf(u).Field(0).Tag`）。它本身不影响程序逻辑，但被大量库用于声明序列化、校验等规则（`json`、`gorm`、`validate` 等）。标签格式是反引号包裹的 `key:"value"` 键值对。

# 6. 小结

| 概念 | 要点 |
|------|------|
| 实例化 | &T{}、new(T)、字面量 |
| 内存对齐 | 字段顺序影响体积，小字段排前 |
| 方法集 | *T 包含 T 的方法集，反之不然 |
| 接收者选择 | 改状态/大结构/含锁 → 指针 |
| 嵌入 | 组合代替继承，字段方法提升 |
| Tag | 反射元数据，序列化/校验用 |

下一篇进入接口，这是 Go 最重要的抽象机制。讲清楚隐式实现、接口值的底层结构（iface/eface）、动态派发，以及"nil 接口不等于 nil"等经典陷阱。
