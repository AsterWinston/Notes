测试与性能分析是工程能力的分水岭。Go 把测试、基准、竞态检测、模糊测试、性能剖析都内建在工具链里，无需第三方框架。这一章讲透 go test 全家桶和 pprof，让你写出可靠且高性能的代码。

# 1. 单元测试

## 1.1 测试的基本规则

```go
// 文件名：xxx_test.go
// 函数名：TestXxx
// 参数：*testing.T
func TestAdd(t *testing.T) {
    got := add(1, 2)
    want := 3
    if got != want {
        t.Errorf("add(1,2) = %d; want %d", got, want)
    }
}
```

★ **关键认知**：Go 测试遵循固定约定：

```
1. 测试文件和被测文件同目录，命名 xxx_test.go
2. 测试函数命名 TestXxx（Xxx 大写开头）
3. 参数是 *testing.T
4. 用 go test 运行
```

## 1.2 表驱动测试

★ **关键认知**：Go 社区的主流测试风格是**表驱动测试**——用表格列出输入和期望输出，循环测试：

```go
func TestAdd(t *testing.T) {
    tests := []struct {
        name     string
        a, b     int
        expected int
    }{
        {"positive", 1, 2, 3},
        {"negative", -1, -2, -3},
        {"zero", 0, 0, 0},
    }

    for _, tt := range tests {
        t.Run(tt.name, func(t *testing.T) {   // 子测试，可单独运行
            got := add(tt.a, tt.b)
            if got != tt.expected {
                t.Errorf("got %d, want %d", got, tt.expected)
            }
        })
    }
}
```

★ **关键认知**：表驱动测试让测试用例一目了然、易于扩展。`t.Run` 创建子测试，可单独运行（`go test -run TestAdd/positive`）。

## 1.3 测试辅助函数

```go
// 测试辅助：用 t.Helper() 标记辅助函数，报错定位到调用方
func assertEqual(t *testing.T, got, want int) {
    t.Helper()          // 关键：让错误定位到调用 assertEqual 的行
    if got != want {
        t.Fatalf("got %d, want %d", got, want)
    }
}
```

★ **关键认知**：`t.Helper()` 标记辅助函数，使测试失败时报错位置指向**调用辅助函数的行**而非辅助函数内部。这是调试测试的关键技巧。

# 2. 测试覆盖率

```bash
go test -cover                     # 显示覆盖率
go test -coverprofile=cover.out    # 生成覆盖率文件
go tool cover -html=cover.out      # 可视化覆盖率（HTML）
```

★ **关键认知**：覆盖率衡量"多少代码被测试执行到"。高覆盖率不等于高质量测试（还要看测试断言是否有效），但低覆盖率一定意味着大量未测代码。

# 3. 基准测试

## 3.1 基本写法

```go
func BenchmarkAdd(b *testing.B) {
    for i := 0; i < b.N; i++ {     // b.N 由框架自动调整
        add(1, 2)
    }
}

// 运行：go test -bench=. -benchmem
```

★ **关键认知**：基准测试函数命名 `BenchmarkXxx`，参数是 `*testing.B`。`b.N` 由框架自动调整到足够大的值，以获得稳定的测量结果。`-benchmem` 显示内存分配情况。

## 3.2 解读基准结果

```
BenchmarkAdd-8    1000000000    0.30 ns/op    0 B/op    0 allocs/op
    │               │            │            │        │
    函数名         执行次数      每次耗时      每次分配   分配次数
```

★ **关键认知**：`0 B/op` 和 `0 allocs/op` 是性能的关键指标——它们告诉你每次操作是否在堆上分配内存。分配越多，GC 压力越大，性能越差。优化目标之一就是减少 `allocs/op`。

## 3.3 基准对比

```go
// 用 b.ResetTimer 排除准备工作
func BenchmarkSort(b *testing.B) {
    data := generateData()
    b.ResetTimer()          // 重置计时，排除 generateData
    for i := 0; i < b.N; i++ {
        sort.Ints(data)
    }
}
```

# 4. 竞态检测（race detector）

```bash
go test -race          # 运行测试并检测数据竞态
go run -race main.go   # 运行程序并检测竞态
```

★ **关键认知**：`-race` 是 Go 最有价值的工具之一——它用 **ThreadSanitizer** 技术动态检测并发程序的数据竞态：

```
数据竞态：两个 goroutine 并发访问同一变量，且至少一个在写。
结果：行为不确定，可能崩溃或产生错误数据。
```

★ **关键认知**：竞态检测**只能发现实际发生的竞态**（动态检测），所以必须有触发竞态的测试用例。它会让程序变慢约 10 倍、内存翻倍，因此只在测试时开启。生产代码务必在 CI 中跑 `-race`。

# 5. 模糊测试（fuzzing）

## 5.1 基本用法

```go
func FuzzParse(f *testing.F) {
    f.Add("input")                  // 种子输入
    f.Fuzz(func(t *testing.T, s string) {
        parse(s)                    // 被测函数，fuzzer 自动生成随机输入
    })
}

// 运行：go test -fuzz=FuzzParse
```

★ **关键认知**：模糊测试（Go 1.18+）自动生成**随机、畸形**的输入，试图触发崩溃或 panic，用于发现边界情况和安全漏洞。它对解析器、编解码器等"处理不可信输入"的函数特别有价值。

# 6. pprof 性能剖析

## 6.1 采集性能数据

```go
import _ "net/http/pprof"     // 引入 pprof HTTP 接口

go func() {
    http.ListenAndServe(":6060", nil)   // pprof 数据在 /debug/pprof/
}()

// 或基准测试生成：
go test -bench=. -cpuprofile=cpu.out -memprofile=mem.out
```

## 6.2 分析 CPU 热点

```bash
go tool pprof -http=:8080 cpu.out    # 交互式 Web 界面
go tool pprof -top cpu.out           # 命令行查看 top 函数
```

★ **关键认知**：pprof 能精确告诉你**哪个函数消耗了最多 CPU/内存**。核心命令：

```
pprof 分析类型：
- cpu：CPU 时间热点
- heap：堆内存分配
- goroutine：goroutine 数量（排查泄漏）
- block：阻塞情况
- mutex：锁竞争
```

## 6.3 分析内存泄漏

```
1. 采集 heap profile（两个时间点的对比）
2. go tool pprof -inuse_space  查看当前占用
3. go tool pprof -alloc_space  查看累计分配
4. 对比两次采样，找出持续增长的对象
```

★ **关键认知**：排查内存泄漏，通常结合 **goroutine profile** 和 **heap profile**——goroutine 数量持续增长往往意味着泄漏（goroutine 持有引用，导致内存无法回收）。

# 7. 测试的最佳实践

★ **关键认知**：Go 测试的一些经验法则：

```
1. 表驱动测试优于重复的测试函数
2. 用 t.Helper() 标记辅助函数
3. 测试要独立、可重复、无外部依赖（用 mock/接口）
4. CI 中跑 go test -race -cover
5. 关键算法写基准测试和模糊测试
```

# 8. 小结

| 工具 | 用途 |
|------|------|
| go test | 单元测试 |
| -cover | 覆盖率 |
| -bench | 基准测试 |
| -race | 竞态检测 |
| -fuzz | 模糊测试 |
| pprof | 性能剖析 |
| -benchmem | 内存分配统计 |

下一篇（最后一篇）进入构建与交叉编译，讲清楚 build tags、go:embed、go:generate、以及 GOOS/GOARCH 交叉编译的原理与实践。
