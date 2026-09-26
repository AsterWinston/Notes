错误处理是 Go 最具争议也最核心的设计。Go 用"显式返回 error"取代异常机制，配合 Go 1.13 引入的错误包装与 `errors.Is/As`，形成了一套独特的错误处理哲学。理解它，才能写出健壮且可维护的 Go 代码。

# 1. error 接口的本质

## 1.1 error 的定义

★ **关键认知**：`error` 就是 Go 里最简单的接口——只有一个 `Error()` 方法：

```go
// 标准库 builtin/error.go
type error interface {
    Error() string
}
```

任何实现了 `Error() string` 的类型都是 error。这正是上一章"小接口"哲学的典型体现。

## 1.2 创建错误

```go
import "errors"
import "fmt"

errors.New("something failed")              // 基础错误
fmt.Errorf("failed to %s", "open")          // 格式化错误
fmt.Errorf("wrap: %w", err)                 // %w 包装错误（Go 1.13+）
```

★ **关键认知**：`errors.New` 返回的是 `*errorString` 类型（一个简单结构体）。注意 `%w`（包装）和 `%v`（仅格式化）的区别——只有 `%w` 会建立错误的**因果链**，供 `errors.Is/As` 追踪。

# 2. 错误处理惯用法

## 2.1 显式检查

```go
f, err := os.Open("file.txt")
if err != nil {
    return err      // 立即返回错误
}
defer f.Close()
// ... 正常处理
```

★ **关键认知**：Go 的惯用法是**错误就近处理、显式检查**。没有异常抛出的"隐藏控制流"，错误传播路径一目了然。这种"啰嗦"换来的是代码可读性和错误处理的可预测性。

## 2.2 哨兵错误 vs 错误类型

```go
// 哨兵错误（sentinel error）：包级预定义的错误变量
var ErrNotFound = errors.New("not found")
var ErrPermission = errors.New("permission denied")

// 使用：直接比较
if err == ErrNotFound {
    // 特定处理
}

// 错误类型：自定义结构体，携带更多上下文
type PathError struct {
    Op   string
    Path string
    Err  error
}

func (e *PathError) Error() string {
    return e.Op + " " + e.Path + ": " + e.Err.Error()
}
```

★ **关键认知**：哨兵错误用于"可预期的特定错误情况"（用 `==` 比较）；错误类型用于"需要携带额外上下文"的错误。两者各有适用场景，但都要谨慎设计——过度使用哨兵错误会让调用方产生强耦合。

# 3. 错误包装与追踪

## 3.1 包装（Wrapping）

★ **关键认知**：Go 1.13 引入错误包装——用 `%w` 把底层错误包裹进新错误，保留因果链：

```go
func readConfig() error {
    _, err := os.ReadFile("config.json")
    if err != nil {
        return fmt.Errorf("read config: %w", err)   // 包装，保留底层错误
    }
    return nil
}
```

## 3.2 errors.Is —— 沿链查找

★ **关键认知**：`errors.Is` 会**沿着包装链**查找目标错误，判断"链条中是否存在等于 target 的错误"：

```go
err := readConfig()

if errors.Is(err, os.ErrNotExist) {
    // 即使 err 被多层包装，仍能识别出底层的 os.ErrNotExist
}
```

它比 `==` 更强大，因为能穿透包装层。这就是为什么包装错误时要用 `%w` 而不是 `%v`。

## 3.3 errors.As —— 提取错误类型

★ **关键认知**：`errors.As` 沿包装链查找**特定类型的错误**，并提取出来：

```go
var pathErr *os.PathError
if errors.As(err, &pathErr) {
    fmt.Println("failed path:", pathErr.Path)   // 提取出 PathError 的字段
}
```

```
errors.Is  vs  errors.As：
- Is：判断错误链中是否有"值等于"某错误（哨兵错误用）
- As：判断错误链中是否有"类型是"某错误（自定义错误类型用）
```

# 4. panic vs error 的边界

## 4.1 何时用 panic

★ **关键认知**：Go 社区的铁律——**panic 只用于真正的"不可恢复"错误**，即程序 bug 或无法继续执行的状态：

```
应该 panic：
1. 数组越界、nil 解引用（Go 自动 panic）
2. 违反编程契约，如未满足前置条件
3. 无法 recover 的致命错误（如配置严重错误导致程序无法启动）

应该返回 error：
1. 可预期的业务错误（文件不存在、网络超时）
2. 调用方可能希望处理/重试的错误
```

## 4.2 recover 的原则

★ **关键认知**：`recover` 用于在 panic 蔓延时做清理和"兜底"，但**不应**把 recover 当作常规控制流。一个良好设计的 Go 程序，panic/recover 的出现频率应该很低——绝大多数错误通过 error 返回值处理。

```go
// 典型用途：服务器框架捕获单个请求的 panic，避免整个进程崩溃
func (s *Server) handle(w http.ResponseWriter, r *http.Request) {
    defer func() {
        if e := recover(); e != nil {
            log.Printf("panic: %v", e)
            w.WriteHeader(500)
        }
    }()
    // ... 处理请求 ...
}
```

# 5. 错误处理的最佳实践

★ **关键认知**：Go 错误处理的一些经验法则：

```
1. 错误信息应小写开头，不以句号结尾（配合日志串联）
2. 要么处理错误，要么返回错误，不要吞掉错误（至少记录日志）
3. 包装错误用 %w，并添加"做什么失败了"的上下文
4. 不要在错误信息中重复调用方的已知信息（如函数名）
5. 边界处（如 API 层）才打印错误，内部逐层返回
```

# 6. 小结

| 概念 | 要点 |
|------|------|
| error 接口 | 只有一个 Error() string |
| errors.New | 创建基础错误 |
| %w | 包装错误，建立因果链 |
| errors.Is | 沿链比较值（哨兵错误） |
| errors.As | 沿链提取类型（自定义错误） |
| panic | 仅不可恢复错误 |
| 边界 | 业务错误用 error，程序 bug 才 panic |

下一篇进入泛型，讲清楚类型参数、类型约束（type constraint）、类型集合，以及 Go 泛型的编译实现（GCShape 与字典传递）。
