Redis 几乎是 Go 后端服务的标配：缓存、分布式锁、计数器、排行榜、消息队列都能见到它。但很多人第一次用 go-redis 时，会卡在"连不上""命令超时""连接池耗尽"这些看似简单的问题上。本篇从 Redis 的定位讲起，把连接、Context、连接池和健康检查这条基础链路一次讲透，后面所有高级用法都建立在它之上。

# 1. Redis 的定位与客户端选型

## 1.1 Redis 是什么、解决什么问题

Redis 是一个**基于内存的键值数据库**，数据放在内存里，所以读写延迟通常在微秒到毫秒级。它常被放在数据库前面，承担三类职责：

- **缓存**：挡住大部分读请求，降低数据库压力。
- **共享状态**：分布式锁、限流计数、会话、排行榜，多个服务实例需要看到同一份数据。
- **轻量消息**：Pub/Sub、Stream，用于解耦和异步。

| 能力 | 典型数据结构 | 典型场景 |
|------|--------------|----------|
| 缓存对象 | String、Hash | 用户信息、商品详情 |
| 计数限流 | String（INCR） | 接口调用次数、库存扣减 |
| 排行榜 | ZSet | 热度榜、延时队列 |
| 去重集合 | Set、HyperLogLog | 标签、UV 统计 |
| 消息 | Stream、Pub/Sub | 异步任务、事件通知 |

★ **关键认知**：Redis 快的前提是"数据在内存"。它**不是**关系型数据库的替代品——没有强事务、没有复杂查询、持久化也有丢失窗口。把它当"共享的高速缓存 + 少量原子操作"来用，系统设计会清晰很多。

## 1.2 为什么用 go-redis

Go 生态里能用的 Redis 客户端不止一个，主流的是 `go-redis` 和 `redigo`。选 go-redis 的理由很实际：

| 客户端 | 特点 | 适合场景 |
|--------|------|----------|
| `github.com/redis/go-redis/v9` | API 现代、支持 Context、集群/Sentinel、内置连接池 | 绝大多数新项目 |
| `github.com/gomodule/redigo` | 更底层、手动管理连接 | 需要极致控制、老项目 |
| `github.com/mediocregopher/radix` | 偏向底层协议 | 特殊需求 |

go-redis 的 v9 版本把 Context 作为所有命令的第一个参数，这一点和现代 Go 服务（HTTP、gRPC）的超时与取消体系天然对齐。

★ **关键认知**：本文全部示例基于 `go-redis/v9`。v8 和 v9 的 API 有差异（比如 `Redis` 类型改名、部分选项字段变化），看旧文章时先确认版本。

# 2. 安装与环境准备

## 2.1 启动一个本地 Redis

最快的方式是 Docker：

```bash
docker run -d --name redis-dev -p 6379:6379 redis:7
```

验证是否启动成功：

```bash
docker exec -it redis-dev redis-cli ping
```

返回 `PONG` 就说明服务可用。

## 2.2 安装 go-redis

初始化模块并拉取依赖：

```bash
mkdir go-redis-demo && cd go-redis-demo
go mod init example.com/redis-demo
go get github.com/redis/go-redis/v9
```

国内网络慢可以设置代理：

```bash
go env -w GOPROXY=https://goproxy.cn,direct
```

# 3. 建立连接

## 3.1 最小可运行示例

```go
package main

import (
    "context"
    "fmt"
    "log"

    "github.com/redis/go-redis/v9"
)

func main() {
    ctx := context.Background()

    // 创建客户端；注意 NewClient 不会立即建立网络连接
    rdb := redis.NewClient(&redis.Options{
        Addr: "localhost:6379", // host:port
    })
    defer rdb.Close() // 退出前释放连接池

    // 第一次真正发命令时才会去连接 Redis
    if err := rdb.Ping(ctx).Err(); err != nil {
        log.Fatalf("连接 Redis 失败: %v", err)
    }

    if err := rdb.Set(ctx, "greeting", "hello redis", 0).Err(); err != nil {
        log.Fatalf("写入失败: %v", err)
    }

    val, err := rdb.Get(ctx, "greeting").Result()
    if err != nil {
        log.Fatalf("读取失败: %v", err)
    }
    fmt.Println(val) // hello redis
}
```

★ **关键认知**：`redis.NewClient` 只是构造了一个客户端对象，**不会**发起 TCP 连接。真正的连接发生在第一条命令执行时（懒连接）。所以"NewClient 成功"不代表 Redis 可用，必须用 `Ping` 探活。

## 3.2 Options 详解

生产环境的连接配置通常长这样：

```go
rdb := redis.NewClient(&redis.Options{
    Addr:         "10.0.0.12:6379", // 地址
    Password:     "secret",         // 密码；无密码留空
    DB:           2,                // 逻辑库编号 0-15
    PoolSize:     50,               // 连接池上限
    MinIdleConns: 10,               // 保持的最小空闲连接
    DialTimeout:  3 * time.Second,  // 建连超时
    ReadTimeout:  2 * time.Second,  // 读超时
    WriteTimeout: 2 * time.Second,  // 写超时
    PoolTimeout:  3 * time.Second,  // 等空闲连接的最长时间
    MaxRetries:   2,                // 命令失败重试次数
})
```

| 字段 | 作用 | 建议 |
|------|------|------|
| `Addr` | 服务地址 | 集群/哨兵模式用 `NewClusterClient` / `NewFailoverClient` |
| `Password` | 认证密码 | 生产必须设置 |
| `DB` | 逻辑库 | 微服务之间尽量不共用 DB，避免键冲突 |
| `PoolSize` | 最大连接数 | 按并发量估算，默认 10 往往偏小 |
| `ReadTimeout` | 单次读超时 | 一般 1-3 秒，避免慢命令拖垮整个请求 |
| `MaxRetries` | 失败重试 | 只对可重试错误生效，写命令要谨慎 |

⚠ **关键认知**：`DB` 只对单机/哨兵模式有效，**Cluster 模式只有 0 号库**。如果代码里写了 `DB: 1` 又切换到集群，配置会被忽略，排查时很容易误判。

## 3.3 用 URL 连接

托管 Redis（如云厂商）通常只给一个连接串：

```go
opt, err := redis.ParseURL("redis://user:pass@host:6379/0?read_timeout=2s")
if err != nil {
    log.Fatal(err)
}
rdb := redis.NewClient(opt)
```

`ParseURL` 会把查询参数解析成对应的 Options 字段，比手写结构体更适合配置中心下发的场景。

# 4. Context 与超时

## 4.1 Context 的三种用法

go-redis 的每个命令都要求传 `context.Context`。它承担三件事：

```go
// 1. 普通调用：用 Background，没有超时
rdb.Get(context.Background(), "k")

// 2. 带超时：请求级超时，最常用
ctx, cancel := context.WithTimeout(context.Background(), 500*time.Millisecond)
defer cancel() // 必须调用，否则泄漏 timer
rdb.Get(ctx, "k")

// 3. 跟随上游取消：HTTP 请求被客户端断开时，Redis 调用也取消
rdb.Get(r.Context(), "k")
```

★ **关键认知**：Context 的取消是**端到端**的。当 HTTP 客户端断开连接，`r.Context()` 会被取消，正在等待的 Redis 命令也会尽快返回。这让"用户已经走了还继续查缓存"这类无效工作自动停止。

## 4.2 客户端级超时配置

Context 超时和客户端超时是两层防线：

| 层级 | 配置 | 作用 |
|------|------|------|
| 单次调用 | `context.WithTimeout` | 控制这一条命令最多等多久 |
| 连接级 | `DialTimeout` | 建立 TCP 连接的最长时间 |
| 读/写级 | `ReadTimeout` / `WriteTimeout` | 单次 socket 读写的最长时间 |
| 取连接 | `PoolTimeout` | 池满时排队等连接的最长时间 |

⚠ **关键认知**：如果 Context 没设超时，而 Redis 网络半断（TCP 没断但不响应），命令会一直挂到客户端的 `ReadTimeout` 才返回。所以**不要依赖单一的 Background Context**，至少给每个业务请求设一个总超时。

## 4.3 为什么每个命令都要 Context

除了超时，Context 还用于链路追踪。go-redis 支持通过 `redis.Hook` 把 trace id 注入，前提就是命令调用链上一直有 Context 传递。写代码时的习惯应该是：**从 handler 一路把 ctx 传到 repository**，而不是在数据层重新造一个 `context.Background()`。

# 5. 连接池

## 5.1 连接池参数

go-redis 内部维护一个连接池，命令执行时"借一条连接，用完还回去"。核心参数：

```go
rdb := redis.NewClient(&redis.Options{
    PoolSize:     50,               // 池中最多 50 条连接
    MinIdleConns: 10,               // 空闲时至少保留 10 条，减少冷启动建连
    MaxIdleConns: 30,               // 最多保留 30 条空闲连接（默认等于 PoolSize）
    ConnMaxIdleTime: 5 * time.Minute, // 空闲超过 5 分钟就关闭
    ConnMaxLifetime: 0,             // 0 表示连接不过期；有负载均衡时可设置
})
```

## 5.2 连接池工作流程

```text
命令执行
   │
   ▼
┌─────────────────────┐
│ 从空闲队列取连接     │
└─────────┬───────────┘
          │ 取到？
     ┌────┴────┐
     ▼         ▼
    是         否
     │         │
     │    ┌────┴─────────────────┐
     │    │ 未达 PoolSize？       │
     │    └────┬──────────┬──────┘
     │         ▼          ▼
     │        是          否
     │         │          │
     │    新建连接   等待 PoolTimeout
     │         │          │
     ▼         ▼          ▼
  执行命令 ←─────── 取到/超时
     │
     ▼
  归还连接（或按策略关闭）
```

★ **关键认知**：连接池不是越大越好。Redis 单线程处理命令，连接过多只会增加上下文切换和服务端压力；真正决定吞吐的是"命令本身有多快"和"网络往返"。一般 `PoolSize` 取 `QPS × 平均耗时(秒)` 的量级，再留一定余量。

## 5.3 池大小怎么定

假设服务峰值 5000 QPS，每条命令平均 1ms（含网络），需要的并发连接约为：

```text
5000 × 0.001 = 5 条
```

实际配置可以取 20-50，留出慢命令和突发流量的余量。如果经常出现 `PoolTimeout`，先查是不是有慢命令占着连接，而不是盲目调大池子。

# 6. 健康检查

## 6.1 Ping 探活

```go
func checkRedis(ctx context.Context, rdb *redis.Client) error {
    ctx, cancel := context.WithTimeout(ctx, time.Second)
    defer cancel()
    return rdb.Ping(ctx).Err() // PONG 返回 nil
}
```

## 6.2 启动探活与就绪探活

- **启动探活**：服务启动时连一次，失败可以选择快速退出（fail fast），避免带病上线。
- **就绪探活**：Kubernetes 的 readiness probe 周期性调用，失败时把实例摘出流量。
- **存活探活**：liveness probe 不建议直接依赖 Redis，否则 Redis 抖动会导致所有实例被重启，形成雪崩。

```go
// HTTP 健康检查接口示例
http.HandleFunc("/healthz", func(w http.ResponseWriter, r *http.Request) {
    if err := checkRedis(r.Context(), rdb); err != nil {
        w.WriteHeader(http.StatusServiceUnavailable) // 503 让 LB 摘流量
        w.Write([]byte("redis unavailable"))
        return
    }
    w.Write([]byte("ok"))
})
```

★ **关键认知**：健康检查要区分"依赖不可用"和"服务自身不可用"。Redis 挂了，服务通常还可以降级读数据库，不应该直接判死。**就绪失败可以，存活失败要慎重**。

# 7. 常见连接错误与排错

| 现象 | 常见原因 | 排查方式 |
|------|----------|----------|
| `connection refused` | Redis 未启动、端口错、容器端口没映射 | `redis-cli ping`、`netstat -ano` |
| `i/o timeout` | 网络不通、防火墙、安全组 | `telnet host 6379` |
| `WRONGPASS` | 密码错误 | 检查配置中心是否下发旧密码 |
| `NOAUTH` | 需要密码但没配置 | 补 `Password` 字段 |
| `PoolTimeout` | 连接池耗尽、慢命令占满连接 | 看慢查询、调大 PoolSize |
| `LOADING` | Redis 正在加载持久化文件 | 等待加载完成 |
| `READONLY` | 连到了从节点，执行了写命令 | 写操作连主节点 |
| `MOVED` | Cluster 模式下键不在当前节点 | 使用 `NewClusterClient` |

⚠ **关键认知**：`MOVED` 错误几乎总是"用单机客户端连了集群"。Cluster 客户端会自动跟随重定向，单机客户端不会。看到这个错误先检查客户端类型，而不是怀疑键写错了。

排错的基本顺序：

```text
1. 服务端活着吗？        redis-cli ping
2. 网络通吗？            telnet host 6379
3. 认证对吗？            redis-cli -a password ping
4. 客户端配置对吗？      打印 Options 核对 Addr/Password/DB
5. 是不是池/超时问题？    看监控里的 PoolTimeout、慢日志
```

# 8. 动手练习

1. 用 Docker 起一个 Redis，写一个程序完成 `Set` → `Get` → `Del`，并打印每一步的耗时。
2. 把连接超时设为 100ms，故意连一个不存在的地址，观察返回的错误类型和耗时。
3. 写一个 `/healthz` 接口，Redis 正常返回 200，停掉 Redis 后返回 503。

# 9. 小结

| 主题 | 要点 |
|------|------|
| 定位 | 内存键值库，做缓存、共享状态、轻量消息 |
| 客户端 | go-redis/v9，Context 贯穿所有命令 |
| 连接 | `NewClient` 懒连接，必须 `Ping` 探活 |
| 超时 | Context 超时 + 客户端超时双层防护 |
| 连接池 | 按并发量估算，池满先查慢命令 |
| 健康检查 | 就绪探活依赖 Redis，存活探活要谨慎 |

下一篇进入数据结构，把 String、Hash、List、Set、ZSet、HyperLogLog、Geo、Stream 的常用命令和 Go 写法逐个过一遍，并给出选型表。