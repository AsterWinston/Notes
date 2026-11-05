`zap.Logger` 的强类型 API 啰嗦但快，`SugaredLogger` 用起来像 `logrus` 一样顺手，两者可以随时转换。除此之外，Zap 还提供了采样、Hook、动态级别和命名 Logger 这些"进阶旋钮"。这些能力用对了能省很多事，用错了会带来隐蔽的性能和一致性问题。本篇逐个讲透，并给出常见误用的对照。

# 1. SugaredLogger

## 1.1 语法与最小示例

`logger.Sugar()` 返回一个 `*zap.SugaredLogger`，它把字段变成了可变参数：

```go
logger, _ := zap.NewProduction()
defer logger.Sync()

sugar := logger.Sugar()

// 键值对交替：偶数个参数被理解为 key-value
sugar.Infow("order created",
    "order_id", "1001",
    "amount_cent", 9900,
)

// 格式化：注意这里没有字段，只有一整条字符串
sugar.Infof("order %s created with %d cents", "1001", 9900)

// 纯文本：等价于 Info 加一个 msg
sugar.Info("server started")
```

| 方法 | 等价 | 字段 | 性能 |
|------|------|------|------|
| `Infow(msg, k, v, ...)` | `Info` + `zap.Any` | 部分结构化 | 中 |
| `Infof(format, args...)` | 无 | 无，纯字符串 | 低 |
| `Info(args...)` | 无 | 无 | 低 |
| `With(k, v, ...)` | `logger.With` | 预置字段 | 中 |

## 1.2 糖的代价

`Infow` 的 key-value 是 `...interface{}`，意味着：

1. **值会退化成 `interface{}`**，编码时要做类型断言，走不到 `zap.String` 的快速路径。
2. **key 没有编译期校验**，写成 `sugar.Infow("x", "user_id", id, "count", 3)` 时，如果少写一个值，`count` 会被当成值，字段全错位。
3. **`Infof` 完全放弃结构化**，日志变成一整条字符串，无法按字段检索。

```go
// 错误示范：参数少一个，user_id 后面直接跟了 count，字段全部错位
sugar.Infow("order created", "order_id", oid, "user_id", uid, "amount", amount, 3)

// 正确做法：用强类型 API，编译期就能发现参数不匹配
logger.Info("order created",
    zap.String("order_id", oid),
    zap.String("user_id", uid),
    zap.Int64("amount_cent", amount),
)
```

★ **关键认知**：`SugaredLogger` 的定位是**过渡和便利**，不是生产热路径的首选。Uber 官方基准显示它比强类型 API 慢数倍。凡是每秒几万条以上的日志，都应该用 `logger.Info` + `zap.Field`。

## 1.3 什么时候可以用糖

| 场景 | 建议 | 原因 |
|------|------|------|
| 应用启动、配置加载 | 可用 `Infow` | 低频，可读性优先 |
| CLI 工具输出 | 可用 `Infof` | 面向人，不需要结构化 |
| 一次性脚本 | 可用 | 不需要接 ELK |
| HTTP 请求热路径 | 用强类型 | 高频，性能敏感 |
| 需要进 ES 的日志 | 用强类型 | 必须字段结构化 |
| 错误日志 | 用强类型 | 要带堆栈和字段 |

```go
// 可以在需要时来回转换，转换本身很轻
sugar := logger.Sugar()
sugar.Infow("config loaded", "file", path)
logger = sugar.Desugar() // 回到强类型
logger.Info("server listening", zap.String("addr", addr))
```

# 2. 采样

## 2.1 采样器的工作原理

`zapcore.NewSamplerWithOptions` 包装一个 Core，用"每秒计数"限制重复日志：

```text
tick = 1s，first = 100，thereafter = 100

某一秒内同一条日志（同 level + 同 msg）出现：
  第 1   条 → 写
  ...
  第 100 条 → 写
  第 101~199 条 → 丢弃 98 条
  第 200 条 → 写（每 100 条留 1 条）
  下一秒重新计数
```

```go
core := zapcore.NewCore(encoder, ws, zap.InfoLevel)

// 每秒首 5 条全留，之后每 100 条留 1 条
sampled := zapcore.NewSamplerWithOptions(
    core,
    time.Second, // tick
    5,           // first：每个 tick 内首次出现的条数
    100,         // thereafter：之后每 N 条留 1 条
)

logger := zap.New(sampled, zap.AddCaller())
```

★ **关键认知**：采样器按 **level + message** 分组，而不是按字段分组。所以 `"order created"` 这个 msg 的所有日志共用一份计数。这意味着：**如果消息里拼了变量，每条 msg 都不同，采样器就完全失效**。这也是 `msg` 必须是固定短语、变量放字段的又一个理由。

## 2.2 计数采样与概率采样

| 方式 | 实现 | 优点 | 缺点 |
|------|------|------|------|
| 计数采样 | `NewSamplerWithOptions` | 保证每秒能看到前 N 条 | 突发流量下仍可能丢首条之后的关键日志 |
| 概率采样 | 自己按随机数决定是否写 | 实现简单、分布均匀 | 低频错误容易整条丢 |
| 按级别 | 调 `LevelEnabler` | 最省 | 无法处理同一级别内的重复 |
| 尾部采样 | 缓冲后按结果决定 | 保留完整错误链路 | 需要缓冲，复杂 |

```go
// 自定义概率采样 Core：只对 Debug 做 10% 采样
type probCore struct {
    zapcore.Core
    rate float64
}

func (c probCore) Check(e zapcore.Entry, ce *zapcore.CheckedEntry) *zapcore.CheckedEntry {
    if e.Level == zap.DebugLevel && rand.Float64() > c.rate {
        return ce // 直接丢弃，不进入 CheckedEntry
    }
    return c.Core.Check(e, ce)
}
```

## 2.3 采样配置示例

```go
func NewSampledLogger() (*zap.Logger, error) {
    cfg := zap.NewProductionConfig()
    cfg.OutputPaths = []string{"stdout"}
    cfg.Level = zap.NewAtomicLevelAt(zap.InfoLevel)
    cfg.Sampling = &zap.SamplingConfig{
        Initial:    100, // 每秒每类日志保留前 100 条
        Thereafter: 100, // 之后每 100 条留 1 条
    }
    return cfg.Build()
}
```

⚠ **关键认知**：不要对 `Error` 及以上采样。Zap 的 `SamplingConfig` 作用于所有级别，如果你的 Error 在某一秒内超过阈值（比如批量任务失败），后续错误会被丢弃。更稳妥的做法是自己实现"Error 不采样"的 Core，或把 Error 单独 `NewTee` 到一个不采样的 Core。

# 3. Hook

## 3.1 Hook 是什么

Hook 是"日志写出时顺带触发一个副作用"的机制。Zap 提供 `zap.Hooks(func(zapcore.Entry) error)`，它在条目生成后、写出前后被调用：

```go
logger := zap.New(core).WithOptions(
    zap.Hooks(func(e zapcore.Entry) error {
        // e 只有级别、时间、消息、调用点，没有字段
        if e.Level >= zap.ErrorLevel {
            metrics.Incr("log_error_total", "msg", e.Message)
        }
        return nil
    }),
)
```

⚠ **关键认知**：Hook 拿到的是 `zapcore.Entry`，**不包含字段**。想在 Hook 里拿到 `order_id` 是做不到的。这是 Zap 为了性能做的取舍：字段在 `Write` 时才编码。所以 Hook 只适合做"按级别计数、按消息路由"这类粗粒度动作。

## 3.2 错误上报 Hook

最常见的用法是把 Error 同步到 Sentry、Prometheus 或告警系统：

```go
func ErrorReportHook(reporter Reporter) zap.Option {
    return zap.Hooks(func(e zapcore.Entry) error {
        if e.Level < zap.ErrorLevel {
            return nil
        }
        // 注意：这里没有字段，只能上报消息和调用点
        // 若需要完整字段，应改用自定义 Core 的 Write 方法
        reporter.Capture(Report{
            Message: e.Message,
            Level:   e.Level.String(),
            Caller:  e.Caller.String(),
            Time:    e.Time,
        })
        return nil
    })
}
```

如果必须带字段上报，就实现自定义 Core：

```go
type reportCore struct {
    zapcore.Core
    reporter Reporter
}

func (c reportCore) Write(e zapcore.Entry, fields []zapcore.Field) error {
    if e.Level >= zap.ErrorLevel {
        // fields 在这里是完整的，可以序列化后上报
        c.reporter.CaptureWithFields(e, fields)
    }
    return c.Core.Write(e, fields) // 仍然正常落盘
}

func (c reportCore) With(fields []zapcore.Field) zapcore.Core {
    return reportCore{Core: c.Core.With(fields), reporter: c.reporter}
}

func (c reportCore) Check(e zapcore.Entry, ce *zapcore.CheckedEntry) *zapcore.CheckedEntry {
    if c.Enabled(e.Level) {
        return ce.AddCore(e, c)
    }
    return ce
}
```

★ **关键认知**：Hook 抛出的 error 会被 Zap 写到 `ErrorOutputPaths`，但**不会中断日志写出**。所以 Hook 里不要做重操作，更不要在里面再打日志，否则容易递归。

## 3.3 Hook 的边界

| 可以用 Hook | 不要用 Hook |
|-------------|-------------|
| 按级别计数 | 修改日志内容 |
| 粗粒度路由 | 依赖字段做业务判断 |
| 触发异步告警 | 同步 HTTP 上报（阻塞主流程） |
| 打点 | 在 Hook 里再调 logger |

# 4. 动态级别

## 4.1 AtomicLevel

`zap.AtomicLevel` 是并发安全的级别开关，可以挂到 HTTP 上实时调整：

```go
atom := zap.NewAtomicLevelAt(zap.InfoLevel)

cfg := zap.NewProductionConfig()
cfg.Level = atom // 把开关交给配置

logger, _ := cfg.Build()

// 任何时候都可以调
atom.SetLevel(zap.DebugLevel)
fmt.Println(atom.Level()) // debug
```

## 4.2 带鉴权的调级接口

```go
func RegisterLevelHandler(mux *http.ServeMux, atom zap.AtomicLevel, token string) {
    mux.HandleFunc("/admin/loglevel", func(w http.ResponseWriter, r *http.Request) {
        // 简单鉴权：生产应接入内部网关或 mTLS
        if r.Header.Get("X-Admin-Token") != token {
            http.Error(w, "forbidden", http.StatusForbidden)
            return
        }
        if r.Method != http.MethodPost {
            http.Error(w, "method not allowed", http.StatusMethodNotAllowed)
            return
        }

        var req struct {
            Level string `json:"level"`
        }
        if err := json.NewDecoder(r.Body).Decode(&req); err != nil {
            http.Error(w, "bad json", http.StatusBadRequest)
            return
        }

        lvl, err := zapcore.ParseLevel(req.Level)
        if err != nil {
            http.Error(w, "unknown level", http.StatusBadRequest)
            return
        }

        old := atom.Level()
        atom.SetLevel(lvl) // 立即生效，无需重启
        log.Printf("log level changed: %s -> %s", old, lvl)
        w.WriteHeader(http.StatusNoContent)
    })
}
```

⚠ **关键认知**：调级接口是**运维后门**，必须满足三点：只在内网可达、有鉴权、有审计日志。线上被扫到 `/debug` 路径并调到 Debug 导致磁盘写满，是真实发生过的事故。

## 4.3 按模块调级

如果只想让某个模块输出 Debug，可以给不同模块不同的 `AtomicLevel`：

```go
var (
    httpLevel = zap.NewAtomicLevelAt(zap.InfoLevel)
    dbLevel   = zap.NewAtomicLevelAt(zap.WarnLevel)
)

func NewModuleLogger(base *zap.Logger, name string, lvl zap.AtomicLevel) *zap.Logger {
    // 每个模块一个独立 Core，级别互不影响
    return base.Named(name).WithOptions(zap.IncreaseLevel(lvl))
}
```

★ **关键认知**：`zap.IncreaseLevel` 只能**提高**级别（变得更严格），不能降低。想让模块比全局更宽松（比如全局 Info、模块 Debug），要单独 `NewCore`，因为 `IncreaseLevel` 的语义是"至少这么高"。

# 5. 命名 Logger

## 5.1 Named 的作用

`logger.Named("order")` 会在 logger 上追加一个 `logger` 字段，用于标识模块：

```go
base, _ := zap.NewProduction()
orderLog := base.Named("order")
payLog := base.Named("payment")

orderLog.Info("created")   // {"logger":"order","msg":"created",...}
payLog.Info("captured")    // {"logger":"payment","msg":"captured",...}
```

多次 `Named` 会自动用点连接：

```go
l := base.Named("order").Named("repo")
l.Info("query") // {"logger":"order.repo", ...}
```

## 5.2 模块化实践

```go
type App struct {
    Log *zap.Logger
}

type OrderService struct {
    log  *zap.Logger
    repo *OrderRepo
}

func NewOrderService(base *zap.Logger, repo *OrderRepo) *OrderService {
    return &OrderService{
        log:  base.Named("order.service"), // 模块名体现层级
        repo: repo,
    }
}
```

| 命名方式 | 结果 | 评价 |
|----------|------|------|
| `base.Named("order")` | `logger: order` | 好 |
| `base.Named("OrderService")` | `logger: OrderService` | 大小写混用，难聚合 |
| `base.Named("order").Named("service")` | `logger: order.service` | 推荐，层级清晰 |
| 在 msg 里写 `[order]` | 字段里没有 logger | 差，无法按 logger 过滤 |

★ **关键认知**：`Named` 是**字段级**的模块标识，`msg` 里的 `[order]` 是**字符串级**的。前者可以在 Kibana 里 `logger: "order.service"` 精确过滤，后者只能全文搜索，且容易被误伤。

# 6. 常见误用

| 误用 | 后果 | 正确做法 |
|------|------|----------|
| 热路径全用 `sugar.Infof` | 性能下降、无法检索 | 强类型 `logger.Info` |
| `Infow` 参数个数错位 | 字段全乱 | 用强类型或严格测试 |
| 对 Error 也开采样 | 批量失败时丢日志 | Error 单独走不采样 Core |
| Hook 里做同步 HTTP | 阻塞业务、超时 | 异步队列 + 超时 |
| Hook 里再调 logger | 递归、死循环 | 直接写 reporter |
| 用 `IncreaseLevel` 想降低级别 | 无效，模块仍被过滤 | 单独 `NewCore` |
| 生产暴露无鉴权调级接口 | 被刷爆磁盘 | 内网 + 鉴权 + 审计 |
| `Named` 用中文或空格 | 字段值不规范 | 小写点分层 |

⚠ **关键认知**：`SugaredLogger` 和 `Logger` 共享同一个底层 Core，所以 `Desugar()` 之后的 logger 字段不会丢，级别开关也是同一个。不要以为转换会创建新配置。

# 7. 练习与自检

1. 用 `zapcore.NewSamplerWithOptions` 配置"每秒首 10 条 + 之后每 50 条留 1 条"的采样，并解释 `Initial` 和 `Thereafter` 在突发流量下的行为。
2. 写一个自定义 Core，要求：Error 及以上不采样、Info 按 50% 概率采样、所有日志仍然写文件。说明 `Check` 和 `Write` 各做什么。
3. 你的服务需要临时把 `order.service` 调到 Debug，但不能影响 `payment.service`。请给出实现方案，并说明为什么 `IncreaseLevel` 在这里不适用。

# 8. 小结

| 主题 | 要点 |
|------|------|
| SugaredLogger | 便利但慢，适合低频和 CLI |
| `Infow` | 键值对无编译期校验，易错位 |
| 采样 | 按 level + msg 分组，msg 必须固定 |
| 采样边界 | Error 不应被采样 |
| Hook | 拿不到字段，只适合粗粒度动作 |
| 自定义 Core | 需要字段时重写 `Write` |
| AtomicLevel | 运行时调级，接口必须鉴权 |
| Named | 点分层模块名，可精确过滤 |

下一篇进入日志轮转与归档，讲清按大小/时间轮转、压缩、保留策略、磁盘保护，以及 Lumberjack 的配置细节和它容易踩的坑。
