上线 APISIX 之前，最容易低估的是"部署形态"和"容量"：单机跑通很容易，多实例、etcd 高可用、连接池、文件描述符、滚动升级，每一项都可能在流量上来后变成故障。本篇按"怎么部署 → 怎么调 → 怎么压 → 怎么扩 → 怎么排"的顺序讲。

# 1. 部署模式

| 模式 | 配置存储 | 动态生效 | 依赖 | 适用 |
|------|----------|----------|------|------|
| 传统模式 | etcd | 是 | etcd | 自建机房、通用生产 |
| Standalone | 本地 YAML | 需 reload | 无 | 边缘、配置少 |
| Kubernetes | etcd / CRD | 是 | etcd + K8s | 云原生 |

## 1.1 传统模式

```text
                 ┌──────────────┐
   CI / 运维 ───► │  Admin API   │
                 └──────┬───────┘
                        ▼
                 ┌──────────────┐
                 │  etcd 集群    │
                 └──┬───┬───┬───┘
                    │   │   │ Watch
          ┌─────────┘   │   └─────────┐
          ▼             ▼             ▼
     ┌─────────┐   ┌─────────┐   ┌─────────┐
     │ APISIX  │   │ APISIX  │   │ APISIX  │
     └────┬────┘   └────┬────┘   └────┬────┘
          └─────────────┼─────────────┘
                        ▼
                     上游服务
```

每个网关实例连同一套 etcd，Admin API 写入后所有实例 Watch 生效。最常见，也最灵活。

## 1.2 Standalone

不连 etcd，配置来自本地 `apisix.yaml`：

```yaml
deployment:
  role: traditional
  role_traditional:
    config_provider: yaml
```

```yaml
routes:
  - id: "1"
    uri: /hello
    upstream:
      type: roundrobin
      nodes:
        "127.0.0.1:1980": 1
#END
```

`#END` 必须是最后一行。Standalone 没有跨实例同步，改配置要分发文件并 reload。

## 1.3 Kubernetes

两种用法：用 APISIX Ingress Controller 监听 Ingress/CRD 自动生成路由，或者用 Helm 独立部署 APISIX + etcd、业务自己管路由。前者适合"用 K8s 原生资源描述路由"，后者适合网关团队统一管理。

# 2. 进程与 Worker

```yaml
nginx_config:
  worker_processes: auto          # 等于 CPU 核数
  worker_rlimit_nofile: 65535     # 单进程可打开的文件描述符上限
  event:
    worker_connections: 10240     # 每个 worker 的最大连接数
```

| 配置 | 说明 |
|------|------|
| `worker_processes` | `auto` 按 CPU 核数启动；混部时可设一半 |
| `worker_rlimit_nofile` | 要大于 `worker_connections × 2` |
| `worker_connections` | 每 worker 的连接上限，含客户端和上游 |

最大连接数大致是 `worker_processes × worker_connections`，但要扣掉上游连接、健康检查探针、etcd Watch 和内部管理连接。实际能承载的并发请求大约是一半。

```yaml
nginx_config:
  http:
    enable_reuseport: true
```

`reuseport` 让每个 worker 各自持有监听 socket，内核按连接哈希分发，减少 accept 锁争抢，高并发下能降低 CPU 抖动。

⚠ **关键认知**：`worker_connections` 调大不等于性能提升。瓶颈往往是文件描述符上限、上游连接池、共享内存或 CPU。先把 `ulimit -n` 和 `worker_rlimit_nofile` 对齐，再谈调大连接数。

# 3. 连接与上游优化

## 3.1 keepalive_pool

```json
{
  "keepalive_pool": {
    "size": 320,
    "idle_timeout": 60,
    "requests": 1000
  }
}
```

| 字段 | 作用 | 调优建议 |
|------|------|----------|
| `size` | 每个 worker 对每个上游保留的空闲连接数 | 按并发和上游数量估算 |
| `idle_timeout` | 空闲连接保留秒数 | 小于上游 keepalive 超时 |
| `requests` | 单连接最多复用次数 | 通常 1000 足够 |

上游 `keepalive_timeout` 是 60 秒时，网关 `idle_timeout` 建议 50 秒左右，避免复用到已被上游关闭的连接，造成偶发 502。

## 3.2 DNS 与 TLS

节点写域名时，APISIX 用自己的 DNS 客户端解析：

```yaml
apisix:
  dns_resolver: 10.0.0.2        # 内网 DNS
  dns_resolver_valid: 30        # 解析结果缓存秒数
  resolver_timeout: 5           # 解析超时
```

⚠ **关键认知**：节点写域名但没配 `dns_resolver`，APISIX 可能无法解析，表现为启动报错或请求时 502。容器环境要把 DNS 配成集群 DNS，而不是沿用宿主机的 `127.0.0.1`。

TLS 握手是 CPU 密集型操作。优化方向：只开 TLS 1.2/1.3、启用会话复用、缩短证书链、在网关终止 TLS、内部用 HTTP 或 mTLS。大量短连接场景优先考虑连接复用，而不是频繁握手。

# 4. 缓存与共享内存

## 4.1 lua_shared_dict

```yaml
nginx_config:
  http:
    lua_shared_dict:
      my-plugin-cache: 100m
      upstream-healthcheck: 20m
```

共享内存是**单实例内所有 worker 共享**的，不是跨实例。容量不足时 `error.log` 会出现 `no memory`，表现为健康检查失效、限流计数异常、插件报错。

```bash
grep -i "no memory" logs/error.log
```

## 4.2 proxy-cache

```yaml
proxy_cache:
  zones:
    - name: disk_cache_one
      memory_size: 50m
      disk_size: 1G
      disk_path: /tmp/disk_cache_one
      cache_levels: "1:2"
```

```json
{
  "proxy-cache": {
    "cache_zone": "disk_cache_one",
    "cache_key": ["$host", "$request_uri"],
    "cache_method": ["GET", "HEAD"],
    "cache_http_status": [200],
    "cache_ttl": 300,
    "cache_control": false
  }
}
```

| 字段 | 说明 |
|------|------|
| `cache_zone` | 使用哪个缓存区 |
| `cache_key` | 缓存键由哪些变量组成 |
| `cache_method` | 哪些方法参与缓存 |
| `cache_http_status` | 哪些状态码缓存 |
| `cache_ttl` | 默认缓存秒数 |
| `cache_control` | 是否遵循响应里的 `Cache-Control` |

⚠ **关键认知**：缓存键必须包含所有影响响应的维度。只按 URI 缓存，会把 A 用户的数据返回给 B 用户。带认证的接口要么不缓存，要么把 `consumer_name`、租户 ID 加进 `cache_key`。

# 5. 压测方法

| 工具 | 适合 | 命令示例 |
|------|------|----------|
| `wrk` | 高并发、简单接口 | `wrk -t4 -c200 -d60s URL` |
| `wrk2` | 固定速率、精确延迟 | `wrk2 -t4 -c200 -R5000 -d60s URL` |
| `hey` | 快速验证 | `hey -z 60s -c 200 URL` |
| `k6` | 多接口链路、断言 | 脚本化场景 |

压测分三层做：

1. **只压网关**：上游用 mock 或固定响应，测网关自身的转发上限。
2. **网关 + 真实上游**：测端到端容量，找出先扛不住的一层。
3. **故障场景**：上游超时、部分节点挂掉、Redis 延迟升高，观察限流和熔断是否符合预期。

```bash
# 200 并发压 60 秒，观察 RPS、P99 和错误率
wrk -t4 -c200 -d60s -H "apikey: test-key" \
  http://127.0.0.1:9080/api/orders/1
```

压测机不要和被测网关同机，先预热再记录稳定阶段，至少跑三轮。同时观察网关 CPU、内存、FD、连接数、共享内存和上游 CPU。

★ **关键认知**：压测的目标不是"跑出最大 QPS"，而是找到"在目标 P99 和错误率下，单实例能稳定承载多少"。脱离延迟和错误率谈 QPS，数字没有意义。

# 6. 容量规划

```text
单实例安全 QPS = 压测中 CPU ≤ 70%、P99 达标时的 QPS
所需实例数 = ceil(峰值 QPS / 单实例安全 QPS × 1.3)
```

`1.3` 是 30% 余量，用于应对突发流量、单实例故障和滚动替换。如果要求 N+1 高可用，再额外加一台。

带宽也要单独估算：

```text
出口带宽 = 峰值 QPS × 平均响应大小 × 8
```

比如 10000 QPS、平均响应 20KB，出口带宽约 1.6Gbps，很容易先打满网卡。

| 扩容信号 | 阈值建议 | 动作 |
|----------|----------|------|
| CPU 使用率 | 持续 > 70% | 扩容或优化插件 |
| P99 延迟 | 持续 > SLO | 先查上游，再查网关 |
| FD 使用率 | > 70% | 调大 `worker_rlimit_nofile` 或扩容 |
| 连接数 | > 70% 上限 | 扩容，检查连接泄漏 |
| 共享内存 | 出现 `no memory` | 调大对应 dict |
| etcd 同步延迟 | 持续增大 | 检查 etcd 负载和网络 |

网关扩容不代表系统扩容。上游连接数、数据库连接池、Redis 连接数都会随网关实例数增加，每增加一台网关，上游就多承受一份连接。etcd 也要提前规划：3 节点起步，生产 3 或 5 节点，奇数个，用 SSD，定期快照。

# 7. 高可用与升级

```text
        ┌──────────────┐
客户端 ─►│  L4/L7 LB    │
        └──┬───┬───┬───┘
           ▼   ▼   ▼
        APISIX APISIX APISIX
           └───┼───┘
               ▼
             etcd 集群
```

- 至少 2 台 APISIX，最好 3 台，跨可用区部署。
- 前面的 LB 做健康检查，失败自动摘除。
- 健康检查可以用 TCP 9080，或单独建一条 `/healthz` 路由。
- 网关实例无状态，配置都在 etcd，扩容直接加机器。

滚动升级步骤：

```text
1. 从 LB 摘除一台实例
2. 等待存量连接处理完（apisix quit 优雅退出）
3. 替换二进制或镜像
4. 启动，确认配置加载和健康检查通过
5. 重新加入 LB
6. 对下一台重复
```

```bash
apisix quit        # 优雅停止，处理完存量请求再退出
apisix start       # 启动
apisix reload      # 重载配置（Standalone 常用）
apisix version     # 确认版本
```

⚠ **关键认知**：不要用 `kill -9` 或 `docker rm -f` 做升级。硬杀会中断正在处理的请求，客户端看到 502/499；用 `quit` 让 worker 把存量请求处理完，配合 LB 摘除，才能做到用户无感。

# 8. 常见故障

## 8.1 启动失败

| 现象 | 原因 | 排查 |
|------|------|------|
| 启动即退出 | `config.yaml` 语法错误 | 看 `logs/error.log` 第一段报错 |
| 连不上 etcd | 地址、端口、TLS、网络策略 | 容器内 `curl etcd:2379/health` |
| 端口被占用 | 9080/9180 已被其他进程使用 | `ss -lntp` 查占用 |
| Standalone 无路由 | `apisix.yaml` 缺 `#END` | 补上最后一行 |
| 权限错误 | 日志目录不可写 | 检查目录属主和挂载权限 |

## 8.2 502 与 504

| 状态码 | 常见原因 | 排查 |
|--------|----------|------|
| 502 | 上游拒绝连接、DNS 失败、端口写错、TLS 不匹配 | 看 `$upstream_addr`，手动 curl 上游 |
| 504 | 上游处理慢、`read` 超时太短、连接池等待 | 看 `$upstream_response_time`，对比超时配置 |
| 499 | 客户端提前断开 | 检查客户端超时和网关排队 |
| 503 | 无健康节点、限流、熔断 | 看健康检查、限流日志、`api-breaker` |

## 8.3 文件描述符与内存

```text
worker_connections are not enough
too many open files
```

处理顺序：调大 `worker_rlimit_nofile` 和系统 `ulimit -n`；检查连接泄漏；检查 `TIME_WAIT` 数量；确认 `keepalive_pool` 没有设得过大。

内存问题常见于：共享内存分配不足（`no memory`）、大响应体被缓冲、插件缓存无上限增长、Lua GC 压力大。表现为延迟抖动或 worker 内存持续上涨。

## 8.4 etcd 相关

| 现象 | 原因 | 解决 |
|------|------|------|
| 启动卡在加载配置 | etcd 慢或配置量大 | 检查 etcd 磁盘和网络 |
| 配置不更新 | Watch 断开 | 看 `apisix_etcd_reachable` 和日志 |
| Watch 报 compacted | 断线超过压缩保留期 | 重新全量同步 |
| 写入超时 | etcd 无 leader 或磁盘慢 | 检查 etcd 健康状态 |

★ **关键认知**：etcd 故障时，已运行的 APISIX 仍能用旧配置服务，但**不能改配置，也不能重启**。重启后的实例需要从 etcd 加载配置，起不来。所以 etcd 的高可用和备份不是可选项。

# 9. 练习

1. 用 Docker Compose 起 2 台 APISIX + 3 节点 etcd，停掉一台 APISIX，验证流量是否自动切换。
2. 用 `wrk` 分别压"空插件路由"和"带 JWT + 限流 + 日志插件的路由"，对比 P99 和 RPS，找出插件开销。
3. 把 `worker_connections` 调到很小，制造 `worker_connections are not enough`，再逐步调大，观察连接数和错误率变化。

# 10. 小结

| 主题 | 要点 |
|------|------|
| 部署模式 | 传统 etcd、Standalone、Kubernetes，按动态性和运维能力选 |
| Worker | `worker_processes: auto`，连接数要算上上游 |
| FD | `worker_rlimit_nofile` 要大于连接数的两倍 |
| 连接池 | `keepalive_pool` 减少建连，`idle_timeout` 小于上游 keepalive |
| DNS | 域名节点必须配 `dns_resolver` |
| 缓存 | `proxy-cache` 的 key 要包含所有影响响应的维度 |
| 压测 | 分网关、端到端、故障三层，关注 P99 和错误率 |
| 容量 | 按 CPU 70%、P99 达标估算，留 30% 余量 |
| 高可用 | 多实例 + LB + etcd 奇数节点，滚动升级用 `quit` |
| 故障 | 502 查上游，504 查超时，FD 和共享内存要提前监控 |

到这里 APISIX 模块的八篇就齐了。建议的阅读顺序是：先跑通第 1 篇的最小示例，再用第 2、3 篇把路由和插件配熟，然后按第 4、5 篇补安全和稳定性，最后用第 6、7、8 篇建立可观测、动态配置和部署运维的完整闭环。
