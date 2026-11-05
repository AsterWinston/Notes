APISIX 是 Apache 顶级项目中的云原生 API 网关，底层是 OpenResty（Nginx + LuaJIT），配置存在 etcd，改一条路由不需要 reload。本篇先讲清它和 Nginx 反代的差别，再拆架构、核心对象和请求阶段，最后用 Docker 跑通第一条路由。

# 1. 定位

## 1.1 反向代理与网关的差别

反向代理解决"请求转发到哪里"：监听端口、匹配路径、转发上游、终止 TLS。网关仍然是反向代理，但额外承担了所有服务都要用的横切能力：

| 能力 | 纯 Nginx | APISIX |
|------|----------|--------|
| 鉴权 | 自己写配置或 Lua | 插件，按 Consumer 管理凭据 |
| 限流 | `limit_req` | 多维度、可接 Redis 的插件 |
| 灰度 | `split_clients` | `traffic-split` 按 Header 或权重 |
| 动态配置 | 改文件后 reload | Watch etcd，毫秒级生效 |
| 观测 | 访问日志、stub_status | 指标、追踪、结构化日志插件 |

★ **关键认知**：网关的价值不是"转发更快"，而是**把鉴权、限流、灰度、观测从业务服务里抽出来，变成可动态配置的插件**。转发只是底座。

## 1.2 适用边界

适合：微服务统一入口、多团队共用网关、需要按 Consumer 做配额、Kubernetes 集群入口。

不适合：只做静态文件或极简反代（Nginx 更轻）、业务逻辑很重（网关不该连数据库）、团队完全不想维护 etcd（可用 Standalone 模式或 Nginx）。

# 2. 架构

## 2.1 控制面与数据面

```text
┌──────────────────┐   Admin API :9180    ┌──────────────────┐
│ Dashboard / CI   │ ───────────────────► │      etcd        │
│ 运维 / 发布系统   │                      │  配置的唯一真相   │
└──────────────────┘                      └────────┬─────────┘
                                                   │ Watch
                                          ┌────────▼─────────┐
   客户端 ────── HTTP/HTTPS ─────────────► │  APISIX Worker   │ ─────► 上游服务
              :9080 / :9443                │  路由 + 插件      │
                                          └──────────────────┘
```

- **控制面**：Admin API 校验并写入 etcd；Dashboard 是可视化前端；CI 也可以直接调 API。
- **数据面**：Nginx worker 只做两件事——Watch etcd 拿配置、处理真实流量。
- **etcd**：唯一真相来源，数据面本地有缓存。

★ **关键认知**：Admin API 只是 etcd 的"带校验的写入口"。Admin API 挂了，已生效的配置仍能工作；etcd 挂了，新实例起不来，也改不了配置。

## 2.2 端口

| 组件 | 默认端口 | 作用 | 对外 |
|------|----------|------|------|
| HTTP | 9080 | 业务流量 | 是 |
| HTTPS | 9443 | TLS 流量 | 是 |
| Admin API | 9180 | 管理配置 | 否 |
| Control API | 9090 | 本机健康检查、schema | 否 |
| Prometheus | 9091 | 指标抓取 | 否 |
| etcd | 2379 | 配置存储 | 否 |
| Dashboard | 9000 | 可视化管理 | 否 |

⚠ **关键认知**：Admin API 默认密钥是公开的 `edd1c9f034335f136f87ad84b625c8f1`。生产必须替换，并用网络策略把 9180 限制在内网；把 Admin API 暴露公网等于把配置权交给所有人。

## 2.3 请求路径

```text
客户端
  ▼
rewrite：改写 URI、Header，还原真实 IP
  ▼
access：匹配 Route，执行鉴权、限流、CORS
  ▼
before_proxy：选定上游后的最后修改
  ▼
proxy：连接上游，发送请求，读取响应
  ▼
header_filter → body_filter → log
```

路由匹配发生在 access 阶段，所以"先鉴权还是先匹配路由"本质是插件优先级问题，第 5 节展开。

# 3. 与 Nginx、Kong 的对比

| 维度 | Nginx | Kong | APISIX |
|------|-------|------|--------|
| 配置存储 | 本地文件 | PostgreSQL / 声明式文件 | etcd |
| 动态生效 | reload | DB 模式可动态 | Watch etcd，毫秒级 |
| 扩展语言 | C / Lua | Lua | Lua |
| 多租户 | 自己设计 | 原生 | 原生 |
| 灰度 | 手写 split_clients | 插件/企业版 | traffic-split |
| 运维依赖 | 无 | 数据库 | etcd |
| 典型场景 | 静态资源、通用反代 | 国外生态 | 云原生、K8s 入口 |

选型建议：只做反代选 Nginx；已深度使用 Kong 且插件满足需求，迁移收益不大；需要强动态、K8s 集成、按 Consumer 配额，选 APISIX。

# 4. 核心概念

| 对象 | 作用 |
|------|------|
| Route | 匹配规则 + 插件 + 上游 |
| Upstream | 节点集合 + 负载均衡 + 健康检查 |
| Service | 可复用的插件与上游组合 |
| Consumer | 调用方身份与凭据 |
| Plugin | 在请求阶段执行的 Lua 模块 |
| Global Rule | 对所有请求生效的插件 |
| Plugin Config | 可复用的插件集合 |
| SSL | 证书与 SNI |

```text
Client ──► Route ──► Upstream ──► Node
             │           │
             │           └── 健康检查 / 负载均衡 / 连接池
             ├── Service（共享插件与上游）
             ├── Plugin Config（共享插件）
             └── Consumer（认证身份）
```

## 4.1 优先级

同一个配置项出现在多个层级时，**Route 内联 > Service > Global Rule**。Route 上写了 `upstream` 就不会用 Service 的；Route 上写了同名插件，会覆盖 Service 上的配置。

```json
{
  "id": "1",
  "uri": "/api/users/*",
  "methods": ["GET"],
  "plugins": { "key-auth": {} },
  "upstream": {
    "type": "roundrobin",
    "nodes": { "10.0.0.11:8080": 1, "10.0.0.12:8080": 1 }
  }
}
```

# 5. 请求阶段

| APISIX 阶段 | 能做什么 | 代表插件 |
|-------------|----------|----------|
| `rewrite` | 改 URI、改头、重定向 | `proxy-rewrite` |
| `access` | 路由匹配、鉴权、限流 | `key-auth`、`limit-count` |
| `before_proxy` | 转发前最后修改 | 自定义插件 |
| `header_filter` | 改响应头 | `cors` |
| `body_filter` | 改响应体 | `response-rewrite` |
| `log` | 日志、指标 | `prometheus` |

同阶段内按 `priority` 从大到小执行：

```text
real-ip (23000) → cors (4000) → ip-restriction (3000)
→ key-auth (2500) → proxy-rewrite (1008)
→ api-breaker (1005) → limit-req (1001) → 转发
```

★ **关键认知**：插件顺序由 `priority` 硬编码决定，**不是 JSON 里写的先后顺序**。JSON 对象无序，靠书写顺序推断执行顺序一定会出错。

# 6. 最小可运行示例

## 6.1 启动 etcd 与 APISIX

`docker-compose.yaml`：

```yaml
services:
  etcd:
    image: bitnami/etcd:3.5
    environment:
      ALLOW_NONE_AUTHENTICATION: "yes"          # 仅本地学习，生产必须开认证
      ETCD_ADVERTISE_CLIENT_URLS: "http://etcd:2379"
    ports:
      - "2379:2379"

  apisix:
    image: apache/apisix:3.9.0-debian
    depends_on:
      - etcd
    ports:
      - "9080:9080"
      - "9180:9180"
      - "9443:9443"
    volumes:
      - ./apisix_conf/config.yaml:/usr/local/apisix/conf/config.yaml:ro
```

`apisix_conf/config.yaml`：

```yaml
deployment:
  admin:
    admin_key:
      - name: admin
        key: "edd1c9f034335f136f87ad84b625c8f1"  # 本地示例，生产必须替换
        role: admin
  etcd:
    host:
      - "http://etcd:2379"
    prefix: "/apisix"
```

```bash
docker compose up -d
docker compose logs -f apisix       # 看到 start worker process 即可
```

## 6.2 创建路由

```bash
curl -i "http://127.0.0.1:9180/apisix/admin/routes/1" \
  -H "X-API-KEY: edd1c9f034335f136f87ad84b625c8f1" \
  -X PUT -d '
{
  "name": "httpbin",
  "uri": "/get",
  "methods": ["GET"],
  "upstream": {
    "type": "roundrobin",
    "nodes": { "httpbin.org:80": 1 }
  }
}'
```

返回 200 或 201，且 body 里带 `"id":"1"`，说明写入 etcd 成功。

## 6.3 验证

```bash
curl -i http://127.0.0.1:9080/get
```

应该看到 `HTTP/1.1 200 OK` 和 httpbin 返回的 JSON。没有外网时，把节点换成自己的服务，例如 `"host.docker.internal:8080": 1`。

# 7. 常见错误与排错

| 现象 | 原因 | 解决 |
|------|------|------|
| 9080 拒绝连接 | 容器没起、端口没映射 | `docker compose ps`、看日志 |
| 404 | `uri`、方法或 `host` 不匹配 | 用 Admin API 读回逐字段对照 |
| Admin 401 | `X-API-KEY` 错误 | 对照 `config.yaml` |
| Admin 400 | 字段不符合 schema | 看响应 body 的具体字段 |
| 503 | 上游没有健康节点 | 查地址、端口、健康检查 |
| 启动连不上 etcd | 地址错或 etcd 未就绪 | 容器内 `curl http://etcd:2379/health` |
| 改了配置没生效 | 把 Standalone 和传统模式搞混 | 传统模式只读 etcd |

# 8. 练习

1. 创建一条带 `host` 限制的路由，用 `curl -H "Host: ..."` 验证命中，不带 Host 时返回 404。
2. 把路由的 `methods` 改成 `["POST"]`，观察 GET 请求的返回码。
3. 故意把 `etcd.host` 改错，观察启动日志，再用 `docker compose exec` 验证连通性。

# 9. 小结

| 主题 | 要点 |
|------|------|
| 定位 | 基于 OpenResty 的云原生网关，强在动态配置与插件 |
| 架构 | 控制面 Admin API + etcd，数据面 Nginx worker |
| 配置 | etcd 是唯一真相，数据面 Watch 后本地缓存 |
| 对象 | Route 匹配、Upstream 转发、Service 复用、Consumer 身份 |
| 阶段 | rewrite → access → before_proxy → proxy → header_filter → body_filter → log |
| 顺序 | 插件按 priority 降序，与 JSON 书写顺序无关 |
| 边界 | Admin API 不暴露公网，默认密钥必须替换 |

下一篇进入路由与上游，讲清 `uri`、`host`、`vars` 的组合匹配，参数路由与通配符的边界，以及负载均衡、健康检查、超时和重试的具体配置。
