路由决定"这个请求归谁处理"，上游决定"转发给哪台机器"。这两块配错，网关的其他能力都无从谈起。本篇从匹配字段讲到优先级，再到负载均衡、健康检查、超时与重试，重点放在上线后才暴露的边界上。

# 1. 路由匹配基础

## 1.1 常用字段

| 字段 | 说明 |
|------|------|
| `uri` / `uris` | 单个或多个匹配路径，支持 `*` 和 `:name` |
| `host` / `hosts` | 域名匹配，不区分大小写 |
| `methods` | HTTP 方法列表 |
| `remote_addr` | 单个客户端 IP，不支持 CIDR |
| `vars` | 基于 Nginx 变量的表达式 |
| `filter_func` | 自定义 Lua 函数，最灵活但最慢 |
| `priority` | 优先级，越大越优先，默认 0 |
| `plugins` | 挂载的插件 |
| `upstream` / `upstream_id` | 内联上游或引用上游 |
| `service_id` | 引用 Service |
| `timeout` | 本条路由的 connect/send/read 超时 |
| `status` | 1 启用，0 禁用 |

## 1.2 uri 与 uris

`uri` 默认是**精确匹配**，只有写成通配形式才匹配前缀：

| 写法 | 能匹配 | 不能匹配 |
|------|--------|----------|
| `/api/users` | `/api/users` | `/api/users/1` |
| `/api/users/*` | `/api/users/1`、`/api/users/a/b` | `/api/users` |
| `/api/*/detail` | `/api/1/detail` | `/api/1/2/detail` |
| `/*` | 所有路径 | 无 |

★ **关键认知**：`*` 只能放在路径末尾或作为完整一段，不能当"任意字符"用。匹配"任意多段"写在最后，匹配"单段"用 `:name`。

## 1.3 优先级

多条路由都能匹配时，先看 `priority`，越大越优先；相同优先级下，静态路径优先于参数路径，更长前缀优先。

```json
[
  { "uri": "/api/*", "priority": 0,
    "upstream": { "nodes": { "10.0.0.10:8080": 1 } } },
  { "uri": "/api/orders/*", "priority": 10,
    "upstream": { "nodes": { "10.0.0.20:8080": 1 } } }
]
```

请求 `/api/orders/1` 命中第二条。即使不加 `priority`，更长前缀通常也会赢；但规则一复杂，隐式顺序就不直观。

⚠ **关键认知**：不要依赖"注册得晚"或"JSON 写在后面"。路由匹配是树结构加优先级，注册顺序不参与决策。冲突场景一律用 `priority` 显式定序。

# 2. 参数与高级匹配

## 2.1 `:name` 参数

```json
{
  "uri": "/users/:id/orders/:order_id",
  "upstream": { "nodes": { "10.0.0.11:8080": 1 } }
}
```

- `:id` 只匹配**一段**，不能包含 `/`。
- 参数段不能为空，`/users//orders/1` 不匹配。
- 插件里可通过 `ctx.curr_req_matched._param_id` 读取。

## 2.2 vars 表达式

```json
{
  "uri": "/api/*",
  "vars": [
    ["http_x_client", "==", "web"],
    ["arg_debug", "~=", "1"]
  ]
}
```

多个条件是**逻辑与**。常用变量：`http_xxx`（请求头）、`arg_xxx`（查询参数）、`cookie_xxx`、`remote_addr`、`request_method`。

| 运算符 | 含义 | 示例 |
|--------|------|------|
| `==` / `~=` | 等于 / 不等于 | `["arg_env", "==", "prod"]` |
| `~~` / `~*` | 正则 / 忽略大小写正则 | `["http_host", "~~", "^api\\."]` |
| `in` | 在列表中 | `["arg_env", "in", ["dev", "test"]]` |
| `ipmatch` | IP 或 CIDR 匹配 | `["remote_addr", "ipmatch", ["10.0.0.0/8"]]` |

★ **关键认知**：`remote_addr` 字段只支持单个 IP，网段限制要用 `vars` 的 `ipmatch`，或直接挂 `ip-restriction` 插件。三者不要混用。

## 2.3 filter_func

`filter_func` 写一小段 Lua，例如 `function(vars) return vars['arg_name'] == 'json' end`。它在每个请求上执行，比 `vars` 慢，代码散落在配置里也难测试。能用 `vars` 就不要用它。

⚠ **关键认知**：`filter_func` 的字符串会被 `loadstring` 执行，等于把代码执行能力开放给 Admin API 调用方。Admin API 只能对内网开放，否则这是明显的 RCE 入口。

# 3. 上游 Upstream

## 3.1 nodes

```json
{
  "type": "roundrobin",
  "nodes": {
    "10.0.0.11:8080": 1,
    "10.0.0.12:8080": 2
  }
}
```

也可以写成数组：`[{"host":"10.0.0.11","port":8080,"weight":1}]`，适合由程序生成。

权重是相对值，12 会拿到约两倍流量。权重 0 表示不参与负载均衡，但节点仍保留在配置里，常用于临时摘除。

## 3.2 负载均衡算法

| 类型 | 算法 | 适用 | 注意点 |
|------|------|------|--------|
| `roundrobin` | 加权轮询 | 通用 | 默认类型 |
| `chash` | 一致性哈希 | 缓存、会话粘性 | 需要 `hash_on` 和 `key` |
| `ewma` | 指数加权移动平均 | 节点延迟差异大 | 按延迟动态调权 |
| `least_conn` | 最少连接 | 长连接、耗时差异大 | 需要统计连接数 |

## 3.3 hash_on 与会话保持

```json
{
  "type": "chash",
  "hash_on": "cookie",
  "key": "session_id",
  "nodes": { "10.0.0.11:8080": 1, "10.0.0.12:8080": 1 }
}
```

| `hash_on` | `key` 含义 |
|-----------|------------|
| `vars` | Nginx 变量名，如 `remote_addr` |
| `header` | 请求头名，如 `x-user-id` |
| `cookie` | Cookie 名 |
| `consumer` | 不需要 key |
| `vars_combinations` | 多个变量组合 |

★ **关键认知**：一致性哈希只能保证"节点不变时同一个键落到同一台"。扩缩容时仍会有部分键重新映射，这是算法本身决定的。要完全避免，需要客户端分片或固定分片表。

# 4. 健康检查

## 4.1 主动检查

```json
{
  "checks": {
    "active": {
      "type": "http",
      "http_path": "/healthz",
      "host": "api.internal",
      "timeout": 1,
      "healthy": { "interval": 2, "successes": 2, "http_statuses": [200] },
      "unhealthy": {
        "interval": 1,
        "failures": 3,
        "http_statuses": [500, 502, 503, 504],
        "timeouts": 3
      }
    }
  }
}
```

每 2 秒探测一次，连续 2 次 200 标记健康；每 1 秒探测，连续 3 次失败或超时标记不健康。

## 4.2 被动检查

```json
{
  "checks": {
    "passive": {
      "type": "http",
      "healthy": { "http_statuses": [200, 201], "successes": 3 },
      "unhealthy": {
        "http_statuses": [500, 502, 503, 504],
        "failures": 3,
        "timeouts": 3
      }
    }
  }
}
```

被动检查不额外发请求，根据真实流量判断。主动快、被动省，生产通常组合使用。

## 4.3 常见坑

- `host` 不设置时，探针带客户端 Host，可能被上游虚拟主机拒绝。
- 上游只监听 HTTPS 时，`checks.active.type` 要设成 `https`。
- 被动检查依赖流量，低峰期节点故障可能很久才发现。
- 健康状态在共享内存里，重启 APISIX 会清零，需要重新探测。

⚠ **关键认知**：`http_path` 要轻量且能代表业务可用性。用 `/` 只能说明 Nginx 活着，不能说明数据库正常；用重业务接口探测，又可能把探针变成压力。建议单独提供 `/healthz`。

# 5. 超时、重试与连接池

## 5.1 三层超时

```json
{
  "timeout": { "connect": 1, "send": 2, "read": 3 },
  "upstream": {
    "timeout": { "connect": 1, "send": 2, "read": 5 },
    "nodes": { "10.0.0.11:8080": 1 }
  }
}
```

| 字段 | 含义 | 建议 |
|------|------|------|
| `connect` | 建立 TCP 连接的时限 | 内网 1 秒以内 |
| `send` | 发送请求体的时限 | 大文件上传要放大 |
| `read` | 等待上游响应的时限 | 按 P99 设，不要一律 60 秒 |

Route 的 `timeout` 会覆盖 Upstream 的同名配置，适合给单条路由设更严格的预算。

## 5.2 retries 与 retry_timeout

```json
{
  "retries": 2,
  "retry_timeout": 3,
  "upstream": {
    "nodes": { "10.0.0.11:8080": 1, "10.0.0.12:8080": 1 }
  }
}
```

- `retries`：失败后最多再试几次。
- `retry_timeout`：所有重试加起来的时间预算。
- 连接失败、上游返回 502/503/504 等可重试状态时会换节点重试。

★ **关键认知**：重试会**放大流量**。上游已过载时，每次失败再试两次，等于把压力乘以三。只对幂等请求开重试，并配合熔断和限流。

⚠ **关键认知**：POST 默认不幂等。订单创建、支付扣款若开重试，上游又没做幂等键，会产生重复订单。写接口要么关重试，要么用 `Idempotency-Key` 去重。

## 5.3 keepalive_pool

```json
{
  "keepalive_pool": { "size": 320, "idle_timeout": 60, "requests": 1000 }
}
```

| 字段 | 影响 |
|------|------|
| `size` | 每个 worker 对每个上游保留的空闲连接数 |
| `idle_timeout` | 空闲连接保留秒数，要小于上游 keepalive |
| `requests` | 单连接最多复用次数 |

上游 `keepalive_timeout` 是 60 秒时，网关 `idle_timeout` 建议 50 秒左右，避免复用到已被上游关闭的连接。

# 6. Service 与 Upstream 的复用

| 方式 | 做法 | 适合 |
|------|------|------|
| 共享 Upstream | 多条 Route 写同一个 `upstream_id` | 只共享节点和负载均衡 |
| 共享 Service | 多条 Route 写同一个 `service_id` | 共享节点、插件、超时整套配置 |

做法是先用 `PUT /apisix/admin/upstreams/100` 创建上游，再让路由写 `"upstream_id": "100"`。共享 Upstream 只复用节点和负载均衡；共享 Service 还会复用插件、超时等整套配置。

⚠ **关键认知**：Route 同时写 `upstream` 和 `upstream_id` 时，内联 `upstream` 优先，`upstream_id` 被忽略。这种配置不报错，但排查非常困惑，提交前应该用 CI 检查互斥字段。

# 7. 常见错误与排错

| 现象 | 原因 | 解决 |
|------|------|------|
| `/api/users` 能访问，`/api/users/1` 404 | 只写了精确路径 | 改成 `/api/users/*` |
| 参数路由匹配不到 | `:id` 段为空或多一段 | 检查路径段数 |
| 高优先级路由没生效 | `priority` 写反或字段拼错 | Admin API 读回确认 |
| 权重 0 后仍有流量 | 改错了 Upstream 或缓存未更新 | 确认 `upstream_id` |
| 健康检查一直不健康 | 路径非 2xx、Host 不对、超时 | 手动 curl 模拟探针 |
| 偶发 502 | 复用了上游已关闭的连接 | 调小 `idle_timeout` |
| 重试导致重复下单 | POST 非幂等 | 关重试或加幂等键 |
| 504 但上游日志成功 | `read` 小于上游处理时间 | 按 P99 调大 `read` |

# 8. 练习

1. 创建 `/api/*`、`/api/orders/*`、`/api/orders/:id` 三条路由，用不同上游标识，验证请求分别命中哪条。
2. 给一个 Upstream 配主动健康检查，停掉一个节点，观察摘除和恢复时间。
3. 设 `retries: 2`、`retry_timeout: 1`，构造慢上游，观察实际重试是否受总预算限制。

# 9. 小结

| 主题 | 要点 |
|------|------|
| 匹配 | `uri` 精确，`/x/*` 前缀，`:id` 单段，`vars` 组合 |
| 优先级 | `priority` 越大越优先，不依赖注册顺序 |
| 上游 | `nodes` 支持对象和数组，权重 0 表示摘除 |
| 负载均衡 | 轮询、一致性哈希、EWMA、最少连接 |
| 健康检查 | 主动快、被动省，探针要轻量且能代表可用性 |
| 超时 | connect/send/read 分开设，Route 覆盖 Upstream |
| 重试 | 放大流量，只对幂等请求开启 |
| 复用 | Upstream 共享节点，Service 共享整套配置 |

下一篇进入插件系统，讲清插件分类、执行阶段、优先级冲突，以及如何写一个可运行的自定义 Lua 插件并调试它。
