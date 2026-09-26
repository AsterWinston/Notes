反射（reflection）让程序在运行时检查和操作自身的类型与值。它是很多框架（JSON 序列化、ORM、依赖注入）的基石，但也是性能的"重灾区"。理解反射的底层原理，才能知道何时该用、何时该避。

# 1. 反射的三大定律

## 1.1 反射的核心接口

★ **关键认知**：Go 反射基于 `reflect` 包的两个核心类型——`reflect.Type`（类型信息）和 `reflect.Value`（值信息）：

```go
import "reflect"

var x int = 42
t := reflect.TypeOf(x)     // reflect.Type，描述 int 类型
v := reflect.ValueOf(x)    // reflect.Value，包装值 42

t.Kind()                   // int
t.Name()                   // "int"
v.Int()                    // 42
v.Type()                   // 等价 reflect.TypeOf(x)
```

## 1.2 三大定律

★ **关键认知**：Go 官方文档总结了反射的三大定律：

```
定律 1：从接口值可以反射得到反射对象（TypeOf/ValueOf）
定律 2：从反射对象可以还原出接口值（Interface()）
定律 3：要修改反射对象，其值必须是"可设置"的（CanSet）
```

## 1.3 可设置性（CanSet）

★ **关键认知**：反射修改值有一个关键前提——**可设置性**。只有传入指针、并经过 `Elem()` 解引用后，值才是可设置的：

```go
x := 42
v := reflect.ValueOf(x)
v.CanSet()           // false！值拷贝，不可设置

p := reflect.ValueOf(&x)
e := p.Elem()        // 解引用，得到可设置的值
e.CanSet()           // true
e.SetInt(100)        // 修改 x = 100
```

★ **关键认知**：`ValueOf(x)` 得到的是 x 的**拷贝**，修改它没有意义，所以 CanSet 为 false。必须通过 `ValueOf(&x).Elem()` 才能拿到指向原变量的可设置值。这是反射修改变量值的唯一正确姿势。

# 2. reflect.Type 与 reflect.Value

## 2.1 常用操作

```go
type User struct {
    Name string `json:"name"`
    Age  int    `json:"age"`
}

u := User{Name: "tom", Age: 18}
t := reflect.TypeOf(u)
v := reflect.ValueOf(u)

// 遍历结构体字段
for i := 0; i < t.NumField(); i++ {
    field := t.Field(i)
    fmt.Println(field.Name, field.Type, field.Tag)   // 字段名、类型、标签
    fmt.Println(v.Field(i))                          // 字段值
}

// 调用方法
method := v.MethodByName("String")
result := method.Call(nil)   // 返回 []reflect.Value
```

## 2.2 Kind 与 Type 的区别

★ **关键认知**：`Kind` 和 `Type` 是两个易混淆的概念：

```
Type：具体的完整类型（如 time.Duration、MyInt）
Kind：类型的底层种类（如 int64、struct、ptr）

type MyInt int
var x MyInt
reflect.TypeOf(x).Name()   // "MyInt"（具体类型名）
reflect.TypeOf(x).Kind()   // "int"（底层种类）
```

★ **关键认知**：`Kind` 是一组**有限的枚举**（`reflect.Int`、`reflect.Struct`、`reflect.Slice` 等约 26 种），而 `Type` 是无限的（用户可以定义任意新类型）。判断类型本质时用 Kind，因为新类型的 Kind 仍是其底层种类。

# 3. 反射的底层原理

## 3.1 类型信息存储

★ **关键认知**：反射依赖运行时保存的**类型元数据**。每个类型在编译期都会生成一份 `_type` 结构（含大小、对齐、Kind、方法等），存放在可执行文件的只读段：

```go
// runtime/type.go（简化）
type _type struct {
    size       uintptr   // 类型大小
    ptrdata    uintptr   // 指针数据大小（GC 用）
    hash       uint32    // 类型哈希
    kind       uint8     // 种类
    ...
}

// reflect.Value 包装的就是这个 _type + 值的指针
type Value struct {
    typ  *rtype     // 指向 _type
    ptr  unsafe.Pointer  // 指向实际数据
    flag uintptr    // 标志位（可设置性等）
}
```

★ **关键认知**：`reflect.ValueOf(x)` 做的事，就是**提取 x 的类型元数据 + 数据指针**，包装成 `reflect.Value`。这解释了为什么反射有开销——每次操作都要间接访问元数据、做边界检查。

## 3.2 反射与接口的关系

★ **关键认知**：反射和接口共享同一套底层机制。回忆接口章节——接口值 = 类型 + 数据。反射对象本质上也是"类型元数据 + 数据指针"的包装。`TypeOf(x)` 实际是先把 x 转成接口，再提取其类型信息。

# 4. 反射的性能代价

★ **关键认知**：反射比直接访问慢很多，主要开销来自：

```
1. 类型检查：每次操作都要做类型断言、边界检查
2. 间接访问：通过指针间接取值，无法做编译期优化
3. 装箱：值需要在接口/反射对象间转换
4. 无内联：反射方法调用无法被内联优化
```

★ **关键认知**：反射的性能可比直接操作慢**一到两个数量级**。因此反射的适用场景是"框架层的一次性初始化"（如序列化器启动时解析结构体），而不是"热路径的频繁操作"。高频循环里的反射是性能杀手。

# 5. 反射的典型应用

## 5.1 JSON 序列化的原理

★ **关键认知**：标准库 `encoding/json` 就是反射的典型应用——通过反射遍历结构体字段、读取标签、序列化：

```go
type User struct {
    Name string `json:"name"`
    Age  int    `json:"age,omitempty"`
}

// json.Marshal 内部用反射：
// 1. 获取结构体类型，遍历字段
// 2. 读取 json tag 决定字段名
// 3. 反射取值并序列化
b, _ := json.Marshal(User{Name: "tom", Age: 18})
// {"name":"tom","age":18}
```

## 5.2 其他应用

```
1. ORM（如 gorm）：反射映射结构体字段到数据库列
2. 依赖注入框架：反射构造对象、注入依赖
3. 通用工具函数：DeepEqual、类型转换
```

# 6. 反射的替代方案

★ **关键认知**：能用编译期手段解决的，就不要用反射：

```
1. 泛型（Go 1.18+）：编译期类型安全，替代大量反射场景
2. 接口：多态替代类型断言
3. 代码生成（go:generate）：编译期生成代码，零运行时反射开销
```

★ **关键认知**：很多高性能库（如 jsoniter、easyjson）用**代码生成**替代反射，为每种类型生成专用序列化代码，避免了反射的运行时开销。这是"用编译期成本换运行时性能"的经典实践。

# 7. 小结

| 概念 | 要点 |
|------|------|
| Type/Value | 反射的两大核心对象 |
| 三大定律 | 接口↔反射对象↔可设置性 |
| CanSet | 修改需传指针 + Elem |
| Kind vs Type | 有限枚举 vs 无限具体类型 |
| 底层 | 类型元数据 + 数据指针 |
| 性能 | 慢 1~2 个数量级，勿在热路径用 |
| 替代 | 泛型/接口/代码生成 |

下一篇进入内存布局与对齐，讲清楚 Go 的内存对齐规则、结构体 padding、以及 unsafe.Sizeof/Alignof/Offsetof 的底层含义。
