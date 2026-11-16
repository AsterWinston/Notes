Zap 是 Uber 开源的 Go 结构化日志库，主打"零反射、低分配、高性能"。它的 API 看起来简单，但 `Logger`、`Core`、`Encoder`、`WriteSyncer` 四层各管什么、为什么开发环境和生产环境要两套配置，很多人用了半年也说不清。本篇从最小示例出发，把这四层拆开，再落到字段类型和性能实践。

# 1. Zap 的定位与最小示例

## 1.1 为什么选 Zap

Go 生态里的日志库大致分三代：

| 库 | 特点 | 适合 |
|----|------|------|
| 标准库 `log` | 只有一行文本，无级别无字段 | 脚本、极简工具 |
| `logrus` | API 友好，结构化，但反射多、分配多 | 中小流量、追求易用 |
| `zap` | 强类型字段、零反射、可组合 Core | 高并发服务、生产环境 |
| `zerolog` | 链式 API、极致低分配 | 对性能极敏感的场景 |
| `slog`（1.21+） | 标准库结构化日志，生态统一 | 新项目、库作者 |

★ **关键认知**：Zap 快的根本原因是**字段在编译期就确定了类型**（`zap.String`、`zap.Int`），编码时不需要反射；同时它大量使用对象池复用 buffer，避免每条日志都分配内存。代价是 API 偏啰嗦，字段必须显式构造。

## 1.2 最小可运行示例

先确保依赖：

```bash
go get go.uber.org/zap@v1.27.0
```

```go
package main

import (
    "time"

    "go.uber.org/zap"
)

func main() {
    // NewProduction 返回 JSON、写到 stderr、Error 及以上带堆栈
    logger, err := zap.NewProduction()
    if err != nil {
        panic(err) // 只有初始化失败才会返回 error
    }
    defer logger.Sync() // 尽量在退出前刷新缓冲，注意 stderr 上可能返回 EINVAL

    logger.Info("server started",
        zap.String("addr", ":8080"),          // 字符串字段
        zap.Int("pid", 12345),                // 整型字段
        zap.Duration("boot", 12*time.Millisecond), // 自动转成秒的浮点
    )

    logger.Error("db query failed",
        zap.String("sql", "select * from orders where id = ?"),
        zap.Error(err), // error 字段会自动取 Error() 文本
    )
}
```

输出大致是：

```json
{"level":"info","ts":1762411953.12,"caller":"main/main.go:18","msg":"server started","addr":":8080","pid":12345,"boot":0.012}
{"level":"error","ts":1762411953.12,"caller":"main/main.go:23","msg":"db query failed","sql":"select * from orders where id = ?","error":"...","stacktrace":"main.main\n\t..."}
```

★ **关键认知**：`zap.NewProduction()` 默认写到 **stderr** 而不是 stdout，并且时间戳是 Unix 浮点秒（`ts`），不是可读字符串。很多人在容器里看不到日志，就是因为采集端只读了 stdout。

# 2. 四个核心组件

## 2.1 Logger

`*zap.Logger` 是对外暴露的门面，负责：

- 提供 `Debug/Info/Warn/Error/DPanic/Panic/Fatal` 方法；
- 持有若干 `zapcore.Field`；
- 调用 `Core.Check` 判断该级别是否要写；
- 把字段和消息交给 `Core.Write`。

它本身不决定格式、不决定输出位置，这些都在 Core 里。

## 2.2 Core

`zapcore.Core` 是一个接口，定义了三件事：

```go
type Core interface {
    Enabled(Level) bool                          // 这个级别要不要处理
    With([]Field) Core                           // 派生带字段的 Core
    Check(Entry, *CheckedEntry) *CheckedEntry    // 组装一条待写日志
    Write(Entry, []Field) error                  // 真正编码并写出
    Sync() error                                 // 刷新缓冲
}
```

`zapcore.NewCore(encoder, writeSyncer, levelEnabler)` 是最常用的实现，它把"编码器 + 输出目标 + 级别开关"三者绑在一起。

## 2.3 Encoder

Encoder 决定日志长什么样。Zap 自带三种：

| 编码器 | 输出 | 适合 |
|--------|------|------|
| `NewJSONEncoder` | 单行 JSON | 生产环境、ELK 采集 |
| `NewConsoleEncoder` | 键值对文本 | 本地开发、人眼阅读 |
| `NewConsoleEncoder` + `EncodeLevel: CapitalColorLevelEncoder` | 带颜色的文本 | 终端调试 |

```go
cfg := zap.NewProductionEncoderConfig()
cfg.EncodeTime = zapcore.ISO8601TimeEncoder       // 2026-11-06T10:12:33.123+0800
cfg.EncodeLevel = zapcore.LowercaseLevelEncoder   // info / error
cfg.EncodeDuration = zapcore.MillisDurationEncoder // 统一毫秒

encoder := zapcore.NewJSONEncoder(cfg)
```

## 2.4 WriteSyncer

`zapcore.WriteSyncer` 是 `io.Writer` 加 `Sync()` 的组合。常见构造：

```go
zapcore.AddSync(os.Stdout)        // 标准输出，容器首选
zapcore.AddSync(os.Stderr)        // 标准错误，zap 默认
zapcore.AddSync(file)             // 普通文件，需要自己做轮转
zapcore.Lock(zapcore.AddSync(f))  // 多 goroutine 写同一文件时加锁
zapcore.NewMultiWriteSyncer(a, b) // 同时写多个目标
```

⚠ **关键认知**：`zapcore.AddSync(file)` 不会帮你处理并发写入和轮转。多个 goroutine 共用一个 `*os.File` 时，要么用 `zapcore.Lock` 包一层，要么直接用 Lumberjack（它内部有锁）。否则可能出现日志行交错甚至丢失。

## 2.5 组件如何串起来

```text
logger.Info("order created", zap.String("order_id", "1001"))
        │
        ▼
┌──────────────┐   Check(level)    ┌───────────────────────┐
│   Logger     │ ────────────────► │ Core（LevelEnabler）   │
│ 持有基础字段  │                   │ 判断 info 是否 >= 阈值 │
└──────┬───────┘                   └───────────┬───────────┘
       │ 组装 Entry(msg, time, level)           │ 允许
       ▼                                        ▼
┌──────────────────────────────────────────────────────────┐
│ Core.Write(entry, fields)                                 │
│   ├─ Encoder.EncodeEntry  → 把字段序列化成 []byte          │
│   └─ WriteSyncer.Write    → 写文件 / stdout / 网络         │
└──────────────────────────────────────────────────────────┘
```

★ **关键认知**：`LevelEnabler` 在 `Check` 阶段就把不写的日志拦掉了，所以 `logger.Debug(...)` 即使被调用，只要级别不够，**连字段构造的开销都不该有**——但注意，`zap.String` 这些函数仍会执行。极端热路径可以用 `if logger.Core().Enabled(zap.DebugLevel)` 再包一层。

# 3. 开发配置与生产配置

## 3.1 两套默认配置的差异

| 维度 | `NewDevelopment` | `NewProduction` |
|------|------------------|-----------------|
| 编码 | Console（带颜色） | JSON |
| 输出 | stderr | stderr |
| 最低级别 | Debug | Info |
| 时间格式 | ISO8601 | Unix 浮点秒 |
| 调用点 | 短文件名 | 全路径 |
| 堆栈 | Warn 及以上 | Error 及以上 |
| 采样 | 无 | 每秒首 100 条，之后 1/100 |
| DPanic | 真的 panic | 降级为 Error |

```go
// 开发：可读性优先
dev, _ := zap.NewDevelopment()

// 生产：机器解析优先，并开启采样
prod, _ := zap.NewProduction()
```

## 3.2 自定义生产配置

真实项目通常要改输出位置、时间格式和采样：

```go
func NewLogger() (*zap.Logger, error) {
    cfg := zap.NewProductionConfig()

    cfg.OutputPaths = []string{"stdout"}          // 容器里交给采集端
    cfg.ErrorOutputPaths = []string{"stderr"}     // zap 内部错误单独走 stderr
    cfg.Encoding = "json"
    cfg.Level = zap.NewAtomicLevelAt(zap.InfoLevel) // 可运行时调整

    cfg.EncoderConfig.EncodeTime = zapcore.ISO8601TimeEncoder // 保留可读时间
    cfg.EncoderConfig.EncodeDuration = zapcore.MillisDurationEncoder
    cfg.EncoderConfig.EncodeCaller = zapcore.ShortCallerEncoder

    // 采样：每秒首 100 条，之后每 100 条留 1 条
    cfg.Sampling = &zap.SamplingConfig{
        Initial:    100,
        Thereafter: 100,
    }

    return cfg.Build()
}
```

## 3.3 同时输出到多个目标

一个常见需求：JSON 写文件给 Filebeat，Console 写终端给人看。用 `zapcore.NewTee` 把两个 Core 并联：

```go
func NewTeeLogger() *zap.Logger {
    // 终端用彩色 Console，级别 Debug
    consoleCfg := zap.NewDevelopmentEncoderConfig()
    consoleCfg.EncodeLevel = zapcore.CapitalColorLevelEncoder
    consoleCore := zapcore.NewCore(
        zapcore.NewConsoleEncoder(consoleCfg),
        zapcore.AddSync(os.Stdout),
        zap.DebugLevel,
    )

    // 文件用 JSON，级别 Info
    fileCfg := zap.NewProductionEncoderConfig()
    fileCfg.EncodeTime = zapcore.ISO8601TimeEncoder
    file, _ := os.OpenFile("app.log", os.O_CREATE|os.O_APPEND|os.O_WRONLY, 0o644)
    fileCore := zapcore.NewCore(
        zapcore.NewJSONEncoder(fileCfg),
        zapcore.AddSync(file),
        zap.InfoLevel,
    )

    // Tee：一条日志同时经过两个 Core，各自判断级别
    core := zapcore.NewTee(consoleCore, fileCore)

    return zap.New(core, zap.AddCaller(), zap.AddStacktrace(zap.ErrorLevel))
}
```

★ **关键认知**：`NewTee` 里的每个 Core 独立判断级别。上面的例子中，`Debug` 只出现在终端、不进文件，这正好满足"本地看细节、线上存重点"的需求。

# 4. 字段类型

## 4.1 常用字段构造函数

| 构造函数 | 类型 | 说明 |
|----------|------|------|
| `zap.String(k, v)` | string | 最常用 |
| `zap.Int / Int64 / Uint` | 整数 | 避免用 String 存数字，否则无法范围查询 |
| `zap.Float64(k, v)` | float | 金额建议用整数分，别用 float |
| `zap.Bool(k, v)` | bool | |
| `zap.Duration(k, d)` | duration | 编码成秒或毫秒，取决于 EncoderConfig |
| `zap.Time(k, t)` | time | 编码格式由 EncoderConfig 决定 |
| `zap.Error(err)` | string | 自动取 `err.Error()` |
| `zap.Strings(k, ss)` | []string | |
| `zap.Any(k, v)` | 反射 | 最后手段，性能差且易泄露敏感字段 |
| `zap.Object(k, marshaler)` | 结构 | 自定义序列化，可控且高效 |

## 4.2 错误字段与堆栈

```go
logger.Error("payment failed",
    zap.Error(err),                        // 只记录错误文本
    zap.NamedError("cause", err),          // 字段名叫 cause
    zap.String("order_id", orderID),
)
```

如果错误链很重要，Go 1.20+ 可以用 `errors.Join` 或多重 `%w`，日志里建议把关键包装层显式记录：

```go
logger.Error("create order failed",
    zap.String("order_id", oid),
    zap.NamedError("db_error", dbErr),
    zap.NamedError("biz_error", bizErr),
)
```

⚠ **关键认知**：`zap.Error(nil)` 会输出 `"error": null` 且不会跳过字段。如果你在"可能没有错误"的分支里无条件写 `zap.Error(err)`，日志里会充满 `"error": null`，既浪费空间又干扰检索。用 `if err != nil` 包住。

## 4.3 Object 与自定义编码

`zap.Any` 走反射，字段多时开销明显，而且会把你不想暴露的字段一起打出来。更好的方式是 `zap.Object`：

```go
type Order struct {
    ID     string
    Amount int64
    Secret string // 不希望出现在日志里
}

func (o Order) MarshalLogObject(enc zapcore.ObjectEncoder) error {
    enc.AddString("order_id", o.ID)
    enc.AddInt64("amount_cent", o.Amount) // 金额用分，避免浮点误差
    // Secret 故意不写
    return nil
}

logger.Info("order created", zap.Object("order", Order{ID: "1001", Amount: 9900}))
```

★ **关键认知**：实现了 `zapcore.ObjectMarshaler` 的类型会被编码器直接调用，**不经过反射**，性能接近手写字段，同时天然完成脱敏。凡是要整体打结构体的地方，优先实现这个接口。

# 5. 性能

## 5.1 为什么快

Zap 的性能来自三个设计：

1. **强类型 Field**：`zap.String` 直接返回一个带类型的结构体，编码时走 switch，不查类型。
2. **Buffer 池**：每次编码从 `sync.Pool` 取 buffer，用完归还，减少 GC 压力。
3. **延迟格式化**：只有在 `Check` 通过后才编码，被级别过滤掉的日志不产生字符串。

## 5.2 使用原则

| 做法 | 影响 |
|------|------|
| 用 `zap.String` 而不是 `fmt.Sprintf` 拼字段 | 少一次分配 |
| 循环外 `logger.With(...)` 预置固定字段 | 少重复编码 |
| 热路径先 `Enabled` 判断 | 避免构造无用 Field |
| 不用 `zap.Any` 打大结构体 | 反射 + 序列化开销大 |
| 不在日志里做 `json.Marshal` | 双重序列化 |

```go
// 好：循环外预置字段
reqLogger := logger.With(zap.String("request_id", rid))
for _, item := range items {
    reqLogger.Debug("item processed", zap.String("sku", item.SKU))
}

// 差：每条都重复构造 request_id
for _, item := range items {
    logger.Debug("item processed", zap.String("request_id", rid), zap.String("sku", item.SKU))
}
```

## 5.3 一个粗略的量级参考

在普通开发机上，Zap 单条 JSON 日志的耗时通常在几百纳秒到一两微秒，`logrus` 往往在几微秒到十几微秒。真实差异取决于字段数量、输出目标（终端 vs 文件 vs 网络）以及是否开启调用点。**不要迷信基准数字，要在自己的机器和真实输出目标上测。**

★ **关键认知**：日志性能的瓶颈通常不在 Zap 的编码，而在**输出目标**。写终端、写网络、写同步文件都会比写内存慢几个数量级。异步输出（如 zap 的 `zapcore.BufferedWriteSyncer`）往往比换日志库更有效。

# 6. 常见错误

| 问题 | 现象 | 正确做法 |
|------|------|----------|
| 用 `zap.NewProduction` 却只看 stdout | 容器里没日志 | 配置 `OutputPaths: ["stdout"]` |
| 忘记 `defer logger.Sync()` | 退出前最后几条丢失 | 退出前 Sync，注意忽略 stderr 的 EINVAL |
| 每条日志都 `zap.Any("req", req)` | 性能差、泄露密码 | 显式字段或 `MarshalLogObject` |
| 多 goroutine 写同一文件 | 行交错、内容损坏 | 用 `zapcore.Lock` 或 Lumberjack |
| 把耗时用 `zap.String` 记录 | 无法做数值聚合 | 用 `zap.Duration` 或 `zap.Int64` |
| 生产环境用 `NewDevelopment` | 彩色文本无法被 Logstash 解析 | 生产用 JSON |
| `zap.Error(nil)` 无条件调用 | 大量 `error: null` | `if err != nil` |

⚠ **关键认知**：`logger.Sync()` 在 stdout/stderr 上可能返回 `invalid argument`（尤其 Windows 和某些终端）。这是正常的，不要把它当成致命错误，通常写成 `_ = logger.Sync()`。

# 7. 练习与自检

1. 用 `zapcore.NewCore` 手写一个 logger：JSON 编码、写 stdout、级别 Info、带调用点和 Error 堆栈，并解释每个参数的作用。
2. 为什么 `NewTee` 能让 Debug 只出现在终端而不进文件？请从 `Check` 和 `LevelEnabler` 的角度说明。
3. 把一个包含 `Password` 字段的 `User` 结构体改造成可以实现 `MarshalLogObject` 的类型，确保密码永远不会被打印。

# 8. 小结

| 主题 | 要点 |
|------|------|
| Logger | 门面，负责级别方法和基础字段 |
| Core | 决定是否写、写到哪、用什么编码 |
| Encoder | JSON 给机器，Console 给人 |
| WriteSyncer | 输出目标，注意并发与 Sync |
| 开发配置 | Console、Debug、带颜色、DPanic 真 panic |
| 生产配置 | JSON、Info、采样、Error 堆栈 |
| 性能 | 强类型字段 + buffer 池 + 延迟编码 |
| 大忌 | `zap.Any` 打整个请求体 |

下一篇进入结构化日志的字段设计，讲清请求 ID 如何贯穿、错误日志该带哪些字段、业务事件如何命名，以及日志如何和 Trace 关联起来。
