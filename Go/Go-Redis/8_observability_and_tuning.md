Redis 大部分"性能问题"不是 Redis 慢，而是用法或配置不对：一个 `KEYS *`、一个大 Hash 的 `HGETALL`、一次没有上限的 `SMEMBERS`，就能让整个实例卡住。本篇讲怎么用慢查询、内存指标、连接数、监控面板把问题定位出来，再给出对应的调优手段。

# 1. 先建立性能模型

## 1.1 Redis 为什么快，又为什么会慢

```text
客户端 ──网络──▶ 单线程命令处理 ──▶ 内存操作
          │            │
          │            └── 单条命令本身通常微秒级
          └── RTT 往往才是大头
```

★ **关键认知**：Redis 的核心命令处理是**单线程**的，一条慢命令会阻塞后面所有请求。所以优化的第一目标是"**消灭慢命令**"，第二目标才是"减少往返"和"分摊压力"。加机器、加连接数都救不了单条 O(N) 命令。

## 1.2 四个常见瓶颈

| 瓶颈 | 典型表现 | 方向 |
|------|----------|------|
| 慢命令 | 偶发超时、CPU 飙高 | 慢查询日志、改命令 |
| 内存不足 | 淘汰频繁、写入报错 | 大 key 治理、扩容 |
| 连接/池 | `PoolTimeout`、连接数暴涨 | 池配置、连接泄漏 |
| 网络 | 带宽打满、RTT 高 | Pipeline、就近部署 |

# 2. 慢查询

## 2.1 配置慢日志

```bash
# 超过 10ms 记为慢查询，最多保留 128 条
CONFIG SET slowlog-log-slower-than 10000
CONFIG SET slowlog-max-len 128

SLOWLOG GET 10       # 查看最近 10 条
SLOWLOG LEN          # 条数
SLOWLOG RESET        # 清空
```

注意单位是**微秒**：`10000` 表示 10 毫秒。默认 `slowlog-log-slower-than` 是 10000 微秒。

```go
// Go 里也可以读慢日志
logs, err := rdb.SlowLogGet(ctx, 10).Result()
for _, l := range logs {
    fmt.Printf("耗时=%v 命令=%v\n", l.Duration, l.Args)
}
```

## 2.2 常见慢命令

| 命令 | 复杂度 | 风险 |
|------|--------|------|
| `KEYS pattern` | O(N) | 全库扫描，生产禁用 |
| `HGETALL` | O(N) | 大 Hash 阻塞 |
| `SMEMBERS` | O(N) | 大 Set 阻塞 |
| `LRANGE 0 -1` | O(N) | 大 List 阻塞 |
| `ZRANGE 0 -1` | O(logN + N) | 大 ZSet 阻塞 |
| `DEL bigkey` | O(N) | 删除大 key 阻塞 |
| `FLUSHALL` | O(N) | 全库清空 |

```bash
# 用 SCAN 代替 KEYS，分批遍历
SCAN 0 MATCH user:* COUNT 100

# 用 HSCAN 代替 HGETALL
HSCAN user:1001 0 COUNT 100
```

★ **关键认知**：`SCAN` 是**渐进式遍历**，每次只返回一小批，不会长时间阻塞。但它**不保证完整快照**：遍历期间新增或删除的 key 可能漏掉或重复。用于清理、统计可以，用于强一致遍历不行。

## 2.3 删除大 key

直接 `DEL` 一个百万元素的 Hash 会阻塞。用 `UNLINK` 在后台线程释放内存：

```bash
UNLINK big:hash      # 异步删除，立即返回
```

```go
rdb.Unlink(ctx, "big:hash")
```

# 3. 内存

## 3.1 查看内存

```bash
INFO memory
MEMORY USAGE user:1001     # 单个 key 占多少字节
MEMORY DOCTOR              # 内存诊断建议
```

关注字段：

| 字段 | 含义 |
|------|------|
| `used_memory` | Redis 分配器统计的内存 |
| `used_memory_rss` | 操作系统看到的常驻内存 |
| `used_memory_peak` | 历史峰值 |
| `mem_fragmentation_ratio` | 碎片率，`rss / used`，1.0-1.5 正常 |
| `maxmemory` | 内存上限 |
| `evicted_keys` | 被淘汰的 key 数量 |

⚠ **关键认知**：`used_memory_rss` 远大于 `used_memory`（碎片率 > 1.5）说明内存碎片严重，可以考虑开启 `activedefrag`（Redis 4.0+）或在低峰期重启。碎片率 < 1 说明部分内存被换到 swap，性能会急剧下降。

## 3.2 淘汰策略

当内存达到 `maxmemory` 时，按策略淘汰 key：

| 策略 | 行为 | 适用 |
|------|------|------|
| `noeviction` | 不淘汰，写入报错 | 数据不能丢 |
| `allkeys-lru` | 所有 key 中淘汰最久未用 | 纯缓存，最常用 |
| `allkeys-lfu` | 所有 key 中淘汰访问频率最低 | 热点明显 |
| `allkeys-random` | 随机淘汰 | 访问均匀 |
| `volatile-lru` | 只淘汰设了过期时间的 | 混合存储 |
| `volatile-lfu` | 只淘汰设了过期时间的，按频率 | 混合存储 |
| `volatile-ttl` | 优先淘汰剩余 TTL 短的 | 混合存储 |
| `volatile-random` | 随机淘汰设了过期时间的 | 混合存储 |

```bash
CONFIG SET maxmemory 4gb
CONFIG SET maxmemory-policy allkeys-lru
```

★ **关键认知**：`volatile-*` 系列**只淘汰设了 TTL 的 key**。如果大部分 key 没设过期，内存满了也不会淘汰，最终写入报错。纯缓存场景直接用 `allkeys-lru` 或 `allkeys-lfu`，别用 `volatile-*` 给自己挖坑。

⚠ **关键认知**：LRU 在 Redis 里是**近似 LRU**（采样若干 key 选最久未用），不是精确 LRU。这是为了省内存和 CPU。LFU 需要 `maxmemory-policy` 设为 `allkeys-lfu` 或 `volatile-lfu`，并能反映真实访问频率，适合"少数热点"场景。

# 4. 连接数

## 4.1 查看与限制

```bash
INFO clients
CLIENT LIST
CLIENT KILL ID 12345
```

| 字段 | 含义 |
|------|------|
| `connected_clients` | 当前连接数 |
| `blocked_clients` | 阻塞中的连接（如 `BRPOP`） |
| `maxclients` | 最大连接数，默认 10000 |

```bash
CONFIG SET maxclients 20000
```

★ **关键认知**：连接数暴涨往往不是"业务量大了"，而是**连接泄漏**：客户端创建了新的 `redis.Client` 却没 `Close`，或者每次请求都 `NewClient`。go-redis 的 `Client` 应该是**全局单例**，不是每个请求建一个。

## 4.2 连接池排错

```go
// 定期打印池状态
stats := rdb.PoolStats()
fmt.Printf("总连接=%d 空闲=%d 使用中=%d 等待次数=%d\n",
    stats.TotalConns, stats.IdleConns, stats.Stalls, stats.Timeouts)
```

| 指标 | 含义 | 异常时 |
|------|------|--------|
| `Stalls` | 等待空闲连接的次数 | 持续增长说明池太小或有慢命令 |
| `Timeouts` | 等待超时的次数 | 池满或 Redis 响应慢 |
| `TotalConns` | 当前总连接 | 接近 `PoolSize` 说明吃紧 |
| `IdleConns` | 空闲连接 | 长期为 0 说明连接都被占着 |

⚠ **关键认知**：看到 `PoolTimeout` 的第一反应不该是调大 `PoolSize`。先确认有没有慢命令占着连接不还、有没有 `BRPOP` 这类阻塞命令长期占用连接。阻塞命令应该用独立的客户端实例，不要和普通命令共用池。

# 5. 监控指标

## 5.1 必须盯的指标

| 类别 | 指标 | 告警建议 |
|------|------|----------|
| 可用性 | `redis_up`、`PING` 延迟 | 连续失败告警 |
| 延迟 | `redis_commands_duration_seconds` | P99 > 10ms 关注 |
| 内存 | `used_memory`、`evicted_keys` | 淘汰持续增长告警 |
| 连接 | `connected_clients`、`rejected_connections` | 接近上限告警 |
| 命中率 | `keyspace_hits / (hits + misses)` | 低于 80% 排查 |
| 慢查询 | `slowlog_len` | 持续增长关注 |
| 主从 | `master_link_status`、`repl_lag` | 断连或延迟告警 |
| 持久化 | `rdb_last_bgsave_status`、`aof_last_write_status` | 失败告警 |

```bash
INFO stats
# keyspace_hits、keyspace_misses 用来算命中率
```

```text
命中率 = keyspace_hits / (keyspace_hits + keyspace_misses)
```

★ **关键认知**：命中率低不一定是坏事——如果缓存本来就该频繁失效（比如短 TTL 的验证码），低命中率是正常的。要看**业务语义**，而不是盲目追求 99%。

## 5.2 用 go-redis 暴露指标

```go
func reportMetrics(ctx context.Context, rdb *redis.Client) {
    info, err := rdb.Info(ctx, "stats", "memory", "clients").Result()
    if err != nil {
        return
    }
    // 解析 info 文本，转成 Prometheus 指标或日志
    // 生产建议直接用 redis_exporter，不要自己解析
    _ = info
}
```

生产环境更推荐 **redis_exporter + Prometheus + Grafana**，它已经覆盖了绝大多数指标，不用自己造轮子。

# 6. 常见性能问题与调优

## 6.1 延迟高

```text
排查顺序：
1. 慢查询日志 → 有没有 O(N) 命令
2. CPU 使用率 → 是不是单线程打满
3. 内存碎片/swap → rss 是否异常
4. 网络 RTT → 客户端和服务端是否跨机房
5. 持久化 → 是否在 fork / AOF rewrite
```

| 原因 | 解决 |
|------|------|
| 慢命令 | 换 `SCAN`/`HSCAN`，拆分大 key |
| 大 key 删除 | 用 `UNLINK` |
| fork 阻塞 | 控制 RDB/AOF 频率，避免大实例 |
| swap | 增加内存或调低 `maxmemory` |
| 网络远 | 就近部署、同可用区 |

## 6.2 内存持续增长

| 原因 | 排查 |
|------|------|
| key 没有 TTL | `SCAN` 抽样看 `TTL` |
| 大 key | `redis-cli --bigkeys` |
| 客户端缓冲区 | `INFO clients` 看 `client_recent_max_input_buffer` |
| 复制积压缓冲 | `INFO replication` 看 `repl_backlog_size` |
| 碎片 | `mem_fragmentation_ratio` |

## 6.3 吞吐上不去

- 用 Pipeline 减少 RTT。
- 热点读加本地缓存。
- 读请求分流到从节点（可接受旧读时）。
- 拆分热点 key。
- 升级到 Cluster 水平扩展。

★ **关键认知**：调优的顺序应该是"**先定位单点瓶颈，再谈扩容**"。如果瓶颈是单条慢命令，扩容只会让更多节点一起慢；如果瓶颈是单 key 热点，加节点也没用，因为槽位还是落在一个节点。

# 7. 配置检查清单

```bash
# 生产环境建议逐项确认
CONFIG GET maxmemory
CONFIG GET maxmemory-policy
CONFIG GET timeout              # 空闲连接超时，建议 300
CONFIG GET tcp-keepalive        # 建议 300
CONFIG GET slowlog-log-slower-than
CONFIG GET appendonly
CONFIG GET save
```

| 配置 | 建议 |
|------|------|
| `maxmemory` | 显式设置，留 20% 余量 |
| `maxmemory-policy` | 纯缓存用 `allkeys-lru`/`allkeys-lfu` |
| `timeout` | 300，回收空闲连接 |
| `tcp-keepalive` | 300 |
| `slowlog-log-slower-than` | 10000（10ms） |
| `appendonly` | 按持久化要求选择 |
| `save` | 避免过于频繁的全量快照 |

# 8. 常见错误与排错

| 现象 | 原因 | 解决 |
|------|------|------|
| `OOM command not allowed` | 达到 `maxmemory` 且策略为 `noeviction` | 调策略或扩容 |
| 偶发超时但 CPU 不高 | 慢命令、fork、swap | 查慢日志和 `INFO` |
| `PoolTimeout` | 池太小或阻塞命令占用 | 查池状态，分离阻塞客户端 |
| 命中率骤降 | 大量 key 过期、缓存被清 | 查 `evicted_keys`、TTL |
| 内存碎片高 | 频繁增删大 key | `activedefrag` 或重启 |
| 主从延迟大 | 从节点负载高、网络差 | 查 `INFO replication` |
| 连接数只增不减 | 客户端泄漏 | 全局单例 + `Close` |

# 9. 动手练习

1. 用 `CONFIG SET slowlog-log-slower-than 0` 记录所有命令，执行 `KEYS *` 后看慢日志。
2. 造一个 10 万元素的 Hash，对比 `HGETALL` 和 `HSCAN` 的耗时。
3. 用 `MEMORY USAGE` 和 `--bigkeys` 找出一个测试库里最大的 key。

# 10. 小结

| 主题 | 要点 |
|------|------|
| 慢查询 | 单位微秒，O(N) 命令是头号杀手 |
| 大 key | `UNLINK` 异步删，`SCAN` 渐进遍历 |
| 内存 | `maxmemory` + 淘汰策略，纯缓存用 `allkeys-*` |
| 连接 | 客户端全局单例，阻塞命令独立池 |
| 监控 | 延迟、内存、连接、命中率、主从状态 |
| 调优 | 先定位瓶颈，再扩容 |

下一篇是收尾篇，把整个模块串成知识地图，给出高频自检题、生产检查清单，以及本地缓存、Redis、数据库三者的一致性方案。