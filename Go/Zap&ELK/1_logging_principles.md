日志谁都会打，但"能打"和"能用来排障"是两件事。线上出问题时，你真正需要的是：在几秒内定位到某个请求、某个用户、某条链路发生了什么，而不是在一堆 `fmt.Println` 里翻找。本篇先把日志的通用原则讲清楚——级别、结构、上下文、采样、脱敏与规范，为后面 Zap 和 ELK 的实战打底。

# 1. 为什么日志需要设计

## 1.1 日志的三类消费者

同一份日志，其实有三拨人在看，诉求完全不同：

| 消费者 | 关心什么 | 对日志的要求 |
|--------|----------|--------------|
| 开发者排障 | 这个请求为什么失败 | 有请求 ID、有错误原因、有上下文 |
| 监控告警 | 错误率、延迟、异常突增 | 级别稳定、错误可统计、字段可聚合 |
| 审计与合规 | 谁在什么时候做了什么 | 不可篡改、保留期、敏感信息不落盘 |

★ **关键认知**：日志的第一目标是**可被机器解析**，第二目标才是**可被人阅读**。因为现代排障流程是"先在 Kibana 里按字段筛出 20 条，再人眼看细节"，如果日志是一整行拼接字符串，第一步就卡住了。

## 1.2 日志不是 printf

初学者最常见的写法是把变量拼进一句话：

```go
// 错误示范：人看得懂，机器解析不了
log.Printf("user %s login failed from %s, reason: %s", userID, ip, reason)
```

改成结构化字段后，同一条信息可以被精确检索：

```go
// 正确做法：字段独立，Kibana 里可以 user_id:"1001" and event:"login_failed"
logger.Warn("login failed",
    zap.String("user_id", userID),   // 业务主键，便于定位单个用户
    zap.String("client_ip", ip),     // 来源 IP，便于风控与封禁
    zap.String("reason", reason),    // 失败原因，便于聚合统计
    zap.String("event", "login_failed"), // 事件名，便于按业务动作筛选
)
```

★ **关键认知**：结构化日志的核心不是"看起来更整齐"，而是**每个变量都成为独立字段**，从而支持过滤、聚合、告警和可视化。一句话里塞三个变量，等于放弃了这三种能力。

# 2. 日志级别

## 2.1 级别的语义

Zap 提供七个级别，日常真正会用到的只有四个：

| 级别 | 数值 | 语义 | 典型场景 |
|------|------|------|----------|
| `Debug` | -1 | 开发期细节 | 变量值、SQL、缓存命中情况 |
| `Info` | 0 | 正常业务事件 | 服务启动、请求完成、订单创建 |
| `Warn` | 1 | 可恢复的异常 | 重试成功、降级、配置缺失用默认值 |
| `Error` | 2 | 需要人介入的失败 | 数据库写失败、第三方接口报错 |
| `DPanic` | 3 | 开发期致命，生产降级为 Error | 不该出现的不变量被打破 |
| `Panic` | 4 | 记录后 panic | 初始化阶段的不可恢复错误 |
| `Fatal` | 5 | 记录后 `os.Exit(1)` | 进程无法继续运行 |

★ **关键认知**：级别表达的是**这条日志需要谁来看、以多快的速度看**，不是"严重程度越高越好"。全部打 `Error` 等于没有级别——告警会被淹没，团队会逐渐无视所有 Error。

## 2.2 级别选择的判断标准

一个简单可执行的判断链：

```text
这条日志出现时，是否需要有人立刻行动？
├─ 是，且进程无法继续        → Fatal
├─ 是，且是本请求的失败      → Error
├─ 否，但说明系统在"带病运行" → Warn
├─ 否，是正常业务事实        → Info
└─ 否，只有开发调试才需要    → Debug
```

几个容易判断错的例子：

| 场景 | 容易打错 | 正确级别 | 原因 |
|------|----------|----------|------|
| 用户密码错误 | `Error` | `Info` 或 `Warn` | 预期内的业务失败，不是系统故障 |
| 第三方接口超时后重试成功 | `Error` | `Warn` | 最终成功，但值得关注频率 |
| 配置项缺失，使用默认值 | `Debug` | `Warn` | 生产环境不该缺配置 |
| 数据库连接失败 | `Warn` | `Error` | 直接影响可用性 |
| 请求参数校验失败 | `Error` | `Info` | 客户端问题，量大且无价值 |

## 2.3 动态级别的价值

生产环境默认开 `Info`，但排障时往往需要临时看 `Debug`。如果每次都要改配置重启，故障现场早就没了。成熟做法是**运行时动态调级**：

```go
// Zap 提供原子级别的开关，可以在不重启的情况下调整
atom := zap.NewAtomicLevelAt(zap.InfoLevel)

// 暴露一个受保护的 HTTP 接口，排障时临时调到 Debug
http.HandleFunc("/debug/loglevel", func(w http.ResponseWriter, r *http.Request) {
    var req struct {
        Level string `json:"level"`
    }
    if err := json.NewDecoder(r.Body).Decode(&req); err != nil {
        http.Error(w, "bad request", http.StatusBadRequest)
        return
    }
    lvl, err := zapcore.ParseLevel(req.Level) // 支持 "debug"、"info" 等字符串
    if err != nil {
        http.Error(w, "unknown level", http.StatusBadRequest)
        return
    }
    atom.SetLevel(lvl) // 原子写入，正在打日志的 goroutine 立刻看到新值
})
```

⚠ **关键认知**：动态调级接口必须鉴权并限制来源网段。它相当于给外部一个"放大日志量"的开关，被恶意调用会把磁盘和 ES 写爆。

# 3. 结构化日志

## 3.1 字符串拼接的代价

看两条日志在 Kibana 里的差别：

```text
# 拼接式
2026-11-06 10:12:33 WARN user 1001 login failed from 10.0.0.8 reason: bad password

# 结构化
{"level":"warn","ts":1762411953.12,"msg":"login failed","user_id":"1001","client_ip":"10.0.0.8","reason":"bad password","event":"login_failed"}
```

第一条想统计"过去一小时各失败原因的数量"，只能写正则；第二条直接 `reason` 聚合即可。数据量到百万级时，正则方案会慢到不可用。

## 3.2 字段命名约定

字段名一旦定下就很难改，建议团队统一：

| 类别 | 字段名 | 类型 | 说明 |
|------|--------|------|------|
| 时间 | `ts` | number/date | 时间戳，由编码器生成 |
| 级别 | `level` | string | 小写，如 `info` |
| 消息 | `msg` | string | 固定短语，不拼变量 |
| 调用点 | `caller` | string | `file:line` |
| 请求 | `request_id` | string | 全链路唯一 |
| 用户 | `user_id` | string | 不要用自增 ID 的裸数字做检索 |
| 事件 | `event` | string | 稳定的事件名，如 `order_created` |
| 错误 | `error` | string | 错误摘要 |
| 堆栈 | `stacktrace` | string | 仅 Error 及以上 |
| 耗时 | `duration_ms` | number | 统一毫秒，避免单位混乱 |

★ **关键认知**：字段名要像数据库列名一样对待——**同一含义永远用同一个名字**。今天叫 `userId`、明天叫 `user_id`、后天叫 `uid`，最终所有聚合查询都会漏数据。

# 4. 上下文

## 4.1 请求级上下文

一个 HTTP 请求会穿过中间件、Handler、Service、Repository，每一层都想往日志里补字段。最自然的做法是**在入口建一个带字段的 logger，逐层往下传**：

```go
func RequestLogger(base *zap.Logger) gin.HandlerFunc {
    return func(c *gin.Context) {
        requestID := c.GetHeader("X-Request-ID")
        if requestID == "" {
            requestID = uuid.NewString()
        }

        // With 会复制一份 logger，并预置这些字段
        logger := base.With(
            zap.String("request_id", requestID),
            zap.String("method", c.Request.Method),
            zap.String("path", c.Request.URL.Path),
        )

        c.Set("logger", logger) // 交给后续 Handler 取用
        c.Header("X-Request-ID", requestID)
        c.Next()
    }
}
```

```go
func GetOrder(c *gin.Context) {
    logger := c.MustGet("logger").(*zap.Logger)
    logger.Info("order query start", zap.String("order_id", c.Param("id")))

    order, err := svc.Find(c.Request.Context(), c.Param("id"))
    if err != nil {
        // 这一条自动带上 request_id、method、path
        logger.Error("order query failed", zap.Error(err))
        return
    }
    logger.Info("order query done", zap.Int64("amount_cent", order.Amount))
}
```

★ **关键认知**：`With` 返回的是**新的 logger**，不是修改原对象。这是 Zap 能做并发安全的前提，也意味着你必须把返回值继续往下传，漏传就会丢字段。

## 4.2 用 context 携带 logger

在 Service 层通常只有 `context.Context`，可以把 logger 放进去：

```go
type loggerKey struct{} // 用私有空结构体做 key，避免和其他包冲突

func WithLogger(ctx context.Context, l *zap.Logger) context.Context {
    return context.WithValue(ctx, loggerKey{}, l)
}

func LoggerFrom(ctx context.Context) *zap.Logger {
    if l, ok := ctx.Value(loggerKey{}).(*zap.Logger); ok {
        return l
    }
    return zap.NewNop() // 兜底：没有 logger 也不能 panic
}
```

⚠ **关键认知**：不要把 logger 当成业务参数层层显式传，也不要把"用户 ID、租户 ID"这类**业务字段**塞进 context 后到处取。显式参数更利于测试和阅读，context 只适合传递跨切面的元数据（请求 ID、trace ID、logger）。

# 5. 采样

## 5.1 为什么要采样

假设一个服务每秒处理 5 万请求，每个请求打 3 条 Info，一天就是 130 亿条。即使每条只有 300 字节，也是 3.9 TB/天。绝大多数重复日志对排障没有增量价值，却要付出存储、传输和索引的代价。

## 5.2 常见采样策略

| 策略 | 做法 | 适合 | 风险 |
|------|------|------|------|
| 按级别 | Debug 全关，Info 限量 | 通用 | 可能漏掉低频问题 |
| 计数采样 | 每秒前 N 条全留，之后每 M 条留 1 条 | 高频重复日志 | 突增时仍会丢关键首条 |
| 概率采样 | 按百分比随机保留 | 流量均匀 | 低频错误容易整条丢失 |
| 按键采样 | 同一 request_id 全部保留 | 需要完整链路 | 实现复杂，内存有开销 |
| 尾部采样 | 先缓冲，出错时整条链路保留 | Trace 场景 | 需要缓冲与判定逻辑 |

★ **关键认知**：**采样只应作用于 Debug/Info，绝不能对 Error 采样**。错误量本来就低，每一条都可能是唯一的现场。Zap 的 `Sampler` 默认按"每秒首 N 条 + 之后每 M 条"工作，正是为高频重复日志设计的。

# 6. 敏感信息与合规

## 6.1 常见泄露点

日志泄露往往不是因为有人故意打印密码，而是因为"顺手把整个对象打了出去"：

| 泄露点 | 例子 | 后果 |
|--------|------|------|
| 请求体整体打印 | `zap.Any("body", req)` | 密码、身份证、银行卡全进日志 |
| 错误对象整体打印 | `zap.Error(err)` 里带 SQL 参数 | 用户数据进日志 |
| Header 全量打印 | Authorization、Cookie | Token 被盗用 |
| URL query 打印 | `?token=xxx` | 凭证长期留存 |
| 第三方 SDK 日志 | HTTP client 打印完整请求 | 不可控 |

⚠ **关键认知**：`zap.Any("req", req)` 是最危险的一行代码。它会用反射把结构体所有导出字段序列化，包括你根本没意识到的敏感字段。打印请求体前必须显式挑选字段。

## 6.2 脱敏实现

最简单可靠的做法是**在入口就把敏感字段替换掉**，而不是指望每个打日志的人记得：

```go
// 只记录必要字段，敏感字段显式脱敏
func safeUser(u *User) zap.Field {
    return zap.Object("user", zapcore.ObjectMarshalerFunc(func(enc zapcore.ObjectEncoder) error {
        enc.AddString("user_id", u.ID)
        enc.AddString("phone", maskPhone(u.Phone)) // 138****8000
        // 密码、token、身份证一律不写
        return nil
    }))
}

func maskPhone(p string) string {
    if len(p) < 7 {
        return "***"
    }
    return p[:3] + "****" + p[len(p)-4:]
}
```

★ **关键认知**：脱敏应该发生在**日志编码之前**，也就是字段进入 Zap 的那一刻。等到日志文件写完再去做清洗，数据已经在磁盘和传输链路上留痕了，合规上不算"未落盘"。

# 7. 日志规范

## 7.1 一份可执行的团队规范

规范要短到大家愿意遵守，建议就这几条：

1. **禁止 `fmt.Println` 和标准库 `log` 直接打业务日志**，统一走注入的 logger。
2. **`msg` 用固定短语**，变量一律放字段，例如 `"order created"` 而不是 `"order 123 created"`。
3. **Error 必须带 `error` 字段**，不要只写 `"failed"`。
4. **每个请求必须带 `request_id`**，跨服务透传 `X-Request-ID`。
5. **禁止打印完整请求体、Header、密码、Token**。
6. **Info 只打业务里程碑**，循环体内禁止 Info。
7. **字段名统一小写下划线**，并在团队 wiki 维护字段字典。

## 7.2 成本模型

日志成本不只是磁盘，而是整条链路：

```text
应用写入 → 本地磁盘 → Filebeat 读取 → 网络传输 → Logstash 解析 → ES 索引 → 副本 → 冷热分层 → 备份
    CPU        磁盘          CPU          带宽          CPU         磁盘+CPU    磁盘×2      存储
```

★ **关键认知**：一条日志的成本大约是"它自身大小的 5-10 倍"，因为要算上副本、索引和传输。把一条无用的 Debug 从每秒 10 万条降到 0，省下的往往不是几 GB，而是整条链路的 CPU 和带宽。

# 8. 常见错误与排错

| 现象 | 根因 | 修复 |
|------|------|------|
| Kibana 里搜不到某字段 | 字段被当成 `text` 分词，或根本没结构化 | 改用结构化字段，检查 mapping |
| 日志量大但排障仍困难 | 缺少请求 ID 和关键上下文 | 入口注入 request_id，逐层 `With` |
| 告警天天响但没人处理 | 级别滥用，业务失败打成 Error | 按 2.1 的语义重新分级 |
| 磁盘被日志写满 | 无轮转、无采样、Info 打太多 | 轮转 + 采样 + 降级 |
| 日志里出现密码 | 整体打印请求体 | 显式挑选字段，入口脱敏 |
| 同一请求日志散落多处 | 各层各自 `New` logger | 统一从 context / 参数传递 |

⚠ **关键认知**：日志排错的第一步永远是**确认日志有没有到达 ES**，而不是怀疑查询语法。先看本地文件有没有、Filebeat 有没有报错、Logstash 有没有丢弃，最后才看 Kibana。

# 9. 练习与自检

1. 把下面这条日志改成结构化日志，并说明每个字段的类型和用途：
   `log.Printf("order %s paid %d cents by user %s in %dms", oid, cents, uid, ms)`
2. 你的服务每秒产生 8 万条 `"cache miss"` 的 Info 日志，请给出两种采样方案，并说明各自的取舍。
3. 设计一份"登录接口"的日志字段清单，要求既能排障、又不泄露密码和 Token。

# 10. 小结

| 主题 | 要点 |
|------|------|
| 消费者 | 排障、告警、审计，三者诉求不同 |
| 级别 | 表达"谁来看、多快看"，不是越高越好 |
| 结构化 | 变量独立成字段，才能过滤聚合 |
| 上下文 | 入口建 logger，`With` 逐层传递 |
| 采样 | 只采 Debug/Info，Error 一条不丢 |
| 脱敏 | 编码前处理，永不整体打印请求体 |
| 成本 | 单条日志的真实成本是体积的 5-10 倍 |

下一篇进入 Zap 本体，讲清 `Logger`、`Core`、`Encoder`、`WriteSyncer` 四个组件如何拼成一条日志流水线，以及开发配置和生产配置到底差在哪里。
