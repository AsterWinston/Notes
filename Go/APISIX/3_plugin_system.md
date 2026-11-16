插件是 APISIX 真正干活的地方：路由只负责"把请求交给谁"，鉴权、限流、改写、日志全部由插件在请求的不同阶段执行。理解挂载层级、执行顺序和上下文，才能写出不互相打架的配置。

# 1. 插件系统概览

## 1.1 插件是什么

一个 APISIX 插件就是一个 Lua 模块，遵循固定约定：

- 导出 `name`、`priority`、`schema`、`version`。
- 实现一个或多个阶段函数，如 `access(conf, ctx)`、`log(conf, ctx)`。
- 通过 `check_schema(conf)` 校验用户传入的配置。

```lua
local _M = {
    version = 0.1,
    priority = 1000,
    name = "demo",
    schema = { type = "object", properties = {} },
}

function _M.access(conf, ctx)
    core.log.info("hello from demo")   -- 插件只拿配置和请求上下文
end

return _M
```

## 1.2 挂载层级

同一个插件可以挂在四个位置：

| 层级 | 作用范围 | 典型用途 |
|------|----------|----------|
| Consumer | 单个调用方 | 单独配额或凭据 |
| Route | 单条路由 | 只给某个接口加 JWT |
| Service | 一组路由 | 整个服务共享限流 |
| Global Rule | 所有请求 | 全站 CORS、Prometheus |

合并优先级是 **Consumer > Route > Service > Global Rule**。同一个插件在多层出现时，更具体的那一层生效。

★ **关键认知**：这个优先级只决定"用哪份配置"，不决定"执行几次"。同一插件在一条请求上通常只执行一次，取合并后的配置；不要同时挂在 Route 和 Global Rule 上再期待它跑两遍。

# 2. 插件分类

| 类别 | 代表插件 | 解决的问题 |
|------|----------|------------|
| 认证鉴权 | `key-auth`、`basic-auth`、`jwt-auth`、`openid-connect` | 判断"你是谁" |
| 访问控制 | `ip-restriction`、`consumer-restriction`、`referer-restriction` | 判断"能不能访问" |
| 限流熔断 | `limit-req`、`limit-count`、`limit-conn`、`api-breaker` | 保护上游 |
| 流量治理 | `traffic-split`、`proxy-rewrite`、`redirect`、`fault-injection` | 灰度、改写、降级 |
| 可观测 | `prometheus`、`http-logger`、`opentelemetry`、`request-id` | 指标、日志、追踪 |
| 协议转换 | `grpc-transcode`、`grpc-web`、`body-transformer` | HTTP 与 gRPC 互转 |
| 缓存压缩 | `proxy-cache`、`gzip` | 降低回源和带宽 |
| Serverless | `serverless-pre-function`、`serverless-post-function` | 临时插入一小段 Lua |

按阶段看，一个插件只实现自己关心的函数：

| 阶段 | 代表插件 |
|------|----------|
| `rewrite` | `proxy-rewrite`、`redirect`、`real-ip` |
| `access` | `key-auth`、`limit-count`、`ip-restriction` |
| `before_proxy` | 自定义插件、部分日志插件 |
| `header_filter` | `cors`、`response-rewrite` |
| `body_filter` | `response-rewrite`、`body-transformer` |
| `log` | `http-logger`、`prometheus` |

# 3. 执行阶段与优先级

## 3.1 阶段顺序

```text
请求进入
  ▼
rewrite ──► access ──► before_proxy ──► 转发上游
                                          ▼
                           header_filter ──► body_filter ──► log
```

同阶段内按 `priority` 从大到小执行：

```text
ip-restriction (3000) → key-auth (2500)
→ consumer-restriction (2400) → proxy-rewrite (1008)
→ api-breaker (1005) → limit-conn (1003)
→ limit-count (1002) → limit-req (1001) → 转发
```

## 3.2 常见优先级

| 插件 | priority | 为什么在这个位置 |
|------|----------|------------------|
| `real-ip` | 23000 | 先还原真实 IP，后面的 IP 限制才有意义 |
| `fault-injection` | 11000 | 故障注入要尽早拦截 |
| `serverless-pre-function` | 10000 | 最靠前的用户 Lua 钩子 |
| `cors` | 4000 | 预检请求要在鉴权前返回 |
| `ip-restriction` | 3000 | 黑名单 IP 不必进入认证 |
| `openid-connect` | 2599 | 需要处理回调和会话 |
| `key-auth` | 2500 | 认证类集中在 2500 附近 |
| `consumer-restriction` | 2400 | 认证后才能拿到 Consumer |
| `proxy-rewrite` | 1008 | 改写要在限流之后 |
| `api-breaker` | 1005 | 先判断上游是否可用 |
| `limit-conn` / `limit-count` / `limit-req` | 1003 / 1002 / 1001 | 最接近转发 |
| `gzip` | 995 | 响应阶段生效 |

★ **关键认知**：插件顺序由 `priority` 硬编码在模块里，**不是配置里谁先写谁先跑**。JSON 对象无序，靠书写顺序推断执行顺序一定会翻车。

## 3.3 冲突案例

- **CORS 与鉴权**：浏览器预检 `OPTIONS` 不带 `Authorization`，鉴权先跑会 401。`cors` 的 4000 高于认证插件，预检才能正常返回。
- **限流与路径改写**：`proxy-rewrite` 先跑，限流可能基于改写后的 URI，同一业务被算成不同配额。限流在 1001 左右、改写 1008，就是为了让限流看到原始请求。
- **IP 限制与认证**：`ip-restriction` 在前，被拉黑的 IP 不会进入认证。若要"先认证再按 IP 判断"，改用 `consumer-restriction`。

# 4. 常用插件速查

| 插件 | 一句话 | 关键字段 |
|------|--------|----------|
| `proxy-rewrite` | 改写 URI、Host、Header | `uri`、`host`、`headers`、`regex_uri` |
| `redirect` | 返回 301/302 | `uri`、`ret_code`、`http_to_https` |
| `traffic-split` | 按权重或规则分流 | `rules`、`weighted_upstreams` |
| `fault-injection` | 注入延迟或错误 | `delay`、`abort` |
| `ip-restriction` | IP 黑白名单 | `whitelist`、`blacklist` |
| `cors` | 跨域响应头 | `allow_origins`、`allow_methods` |
| `request-validation` | 请求体 schema 校验 | `header_schema`、`body_schema` |
| `client-control` | 限制请求体大小 | `max_body_size` |
| `prometheus` | 暴露指标 | `plugin_metadata` 配置 |
| `http-logger` | 日志发到 HTTP 服务 | `uri`、`batch_max_size` |
| `opentelemetry` | OTLP 链路追踪 | `sampler` |
| `grpc-transcode` | HTTP/JSON 转 gRPC | 需要先上传 proto |

# 5. 自定义 Lua 插件

## 5.1 目录与注册

插件文件放在 `apisix/plugins/` 下，文件名就是模块名：

```text
/usr/local/apisix/
├── apisix/plugins/my-plugin.lua
├── conf/config.yaml
└── logs/error.log
```

```yaml
plugins:
  - real-ip
  - cors
  - key-auth
  - limit-count
  - prometheus
  - my-plugin            # 新增的自定义插件
```

⚠ **关键认知**：`plugins` 是**全量列表**，不是增量。写了几项就只加载几项，没列出的内置插件会全部消失。新增插件时要把原列表完整抄下来再追加。

## 5.2 骨架代码

```lua
local core = require("apisix.core")
local plugin_name = "my-plugin"

local schema = {
    type = "object",
    properties = {
        message = { type = "string", default = "hello from apisix" },
        status = { type = "integer", enum = { 200, 403 }, default = 200 },
        add_header = { type = "string" },
    },
    additionalProperties = false,   -- 拒绝未声明字段，避免配置写错却静默忽略
}

local _M = {
    version = 0.1,
    priority = 1500,                -- 位于认证之后、限流之前
    name = plugin_name,
    schema = schema,
}

function _M.check_schema(conf)
    return core.schema.check(schema, conf)
end

function _M.access(conf, ctx)
    if conf.add_header then
        core.request.set_header(ctx, conf.add_header, "1")  -- 给上游留标记
    end
    core.log.warn("my-plugin hit, route_id=", ctx.route_id,
                  ", request_id=", ctx.var.request_id)

    core.response.exit(conf.status, {   -- 直接结束请求，不再转发
        plugin = plugin_name,
        message = conf.message,
        route_id = ctx.route_id,
    })
end

return _M
```

## 5.3 schema 与上下文

`schema` 使用 JSON Schema 子集。常用字段：

| 字段 | 作用 |
|------|------|
| `type` | `object`、`string`、`integer`、`boolean`、`array` |
| `properties` | 各字段的类型与默认值 |
| `required` | 必填字段列表 |
| `enum` | 枚举值 |
| `minimum` / `maximum` | 数值范围 |
| `additionalProperties` | 是否允许未声明字段 |

`ctx` 是请求上下文，常用字段：`ctx.var`（Nginx 变量）、`ctx.route_id`、`ctx.service_id`、`ctx.consumer_name`、`ctx.upstream_id`、`ctx.request_id`。

★ **关键认知**：校验发生在**写配置时**，不是请求时。schema 写得越严格，坏配置越早被拦住。不要为了"先跑起来"把 `additionalProperties` 设成 `true`，那会让拼写错误变成静默失效。

## 5.4 共享内存

```lua
local dict = ngx.shared["my-plugin-cache"]   -- 需在 config.yaml 声明
dict:set("key", "value", 60)                 -- 60 秒后过期
local v = dict:get("key")
```

```yaml
nginx_config:
  http:
    lua_shared_dict:
      my-plugin-cache: 20m
```

⚠ **关键认知**：`ngx.shared.DICT` 是**单个 APISIX 实例内所有 worker 共享**的，但多台 APISIX 之间不共享。多实例下自己用共享内存做限流，真实配额会变成"配置值 × 实例数"。要跨实例精确计数，用 Redis 或官方 `limit-count` 的 redis 策略。

# 6. 插件调试

## 6.1 日志

APISIX 日志在 `logs/error.log`。把级别调低可以看到更多信息：

```yaml
nginx_config:
  error_log: logs/error.log
  error_log_level: info      # debug / info / warn / error
```

```lua
core.log.warn("conf=", core.json.encode(conf))
core.log.warn("ctx.var=", core.json.encode(ctx.var))
```

`core.json.encode` 遇到循环引用会报错，打印整个 `ctx` 前先挑字段。

## 6.2 Admin API

```bash
# 列出所有已加载插件
curl -s "http://127.0.0.1:9180/apisix/admin/plugins/list" \
  -H "X-API-KEY: $ADMIN_KEY"

# 查看某个插件的 schema
curl -s "http://127.0.0.1:9180/apisix/admin/plugins/my-plugin" \
  -H "X-API-KEY: $ADMIN_KEY"
```

列表里没有你的插件，先检查 `config.yaml` 的 `plugins` 和文件路径，再看启动日志有没有 `failed to load plugin`。

## 6.3 单元测试

APISIX 官方插件测试基于 `Test::Nginx`，放在源码的 `t/plugins/` 下：

```bash
prove -I. t/plugins/my-plugin.t
```

改插件前先跑一遍现有测试，能避免大部分回归。

# 7. 常见错误与排错

| 现象 | 原因 | 解决 |
|------|------|------|
| Admin 报 `unknown plugin` | 插件没在 `plugins` 列表 | 补进 `config.yaml` 并重启 |
| 内置插件全部消失 | 覆盖了 `plugins` 列表 | 保留默认列表再追加 |
| 改了代码不生效 | Lua 代码已缓存 | `/apisix/admin/plugins/reload` 或重启 |
| schema 一直 400 | 字段名拼错、类型不对 | 用 `GET /plugins/{name}` 对照 |
| 插件报 `attempt to index nil` | `ctx` 字段不存在或时机不对 | 换正确阶段，打印 `ctx.var` |
| 自定义插件拖慢请求 | 在插件里做了同步 IO | 用 `ngx.timer` 异步处理 |
| 限流计数偏大 | 单实例共享内存，多实例叠加 | 改用 Redis 策略 |

⚠ **关键认知**：插件运行在 Nginx 的请求处理协程里，任何阻塞操作都会卡住当前 worker 上的其他请求。不要在 `access` 里同步查数据库、调用外部 HTTP 或 `sleep`；需要外部调用时用 `ngx.timer.at` 异步化。

# 8. 练习

1. 写一个插件，在 `access` 检查请求头 `X-Tenant`，缺失时返回 400，存在时写进 `ctx`。
2. 给插件加 `header_filter`，在响应头回写 `X-Tenant`，验证跨阶段能读到值。
3. 把 `config.yaml` 的 `plugins` 改成只留 `my-plugin`，重启后观察插件列表的变化。

# 9. 小结

| 主题 | 要点 |
|------|------|
| 插件本质 | 实现阶段函数的 Lua 模块，通过 schema 校验配置 |
| 挂载层级 | Consumer > Route > Service > Global Rule |
| 执行顺序 | 同阶段按 `priority` 降序，与 JSON 书写顺序无关 |
| 阶段 | rewrite、access、before_proxy、header_filter、body_filter、log |
| 自定义插件 | 放 `apisix/plugins/`，在 `config.yaml` 全量列表里登记 |
| 上下文 | `ctx.var`、`ctx.route_id`、`ctx.consumer_name` |
| 共享内存 | 单实例内共享，多实例要用 Redis |
| 调试 | `error.log` + Admin API schema + Test::Nginx |

下一篇进入认证与安全，讲清 Key Auth、Basic Auth、JWT、OIDC 的配置与边界，以及 IP 黑白名单、CORS、请求体限制和密钥管理。
