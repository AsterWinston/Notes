# 1. Kibana 内部架构

Kibana 本质上是一个 **Node.js 服务端应用 + 浏览器 SPA** 的组合：服务端负责托管静态资源、提供 REST API、代理 ES 请求、调度后台任务；浏览器端加载由 React/Angular 混合实现的前端应用（Core 插件体系），所有界面操作最终都变成对 Kibana Server HTTP API 的调用。理解这个"前后端分离 + 全量代理"的结构，是排查 Kibana 卡顿、超时、权限问题的前提。

```text
┌──────────────────────────────────────────────────────┐
│                    Kibana Server                       │
│                                                       │
│  ┌──────────┐   ┌──────────┐   ┌──────────┐         │
│  │ HTTP API │   │   Plugins│   │   Task   │         │
│  │  Server  │   │  System  │   │ Manager  │         │
│  └────┬─────┘   └────┬─────┘   └────┬─────┘         │
│       │              │              │                 │
│  ┌────┴──────────────┴──────────────┴─────┐          │
│  │              Saved Objects              │          │
│  │  (.kibana 索引，ES 持久化存储)           │         │
│  └────────────────────────────────────────┘          │
└──────────────────────────────────────────────────────┘
```

服务端几个关键组件的职责：**HTTP Server** 基于 Hapi 实现，统一监听 5601 端口，负责 TLS 终结、会话 Cookie、请求路由；**Plugin System** 把每个功能（discover、dashboard、alerting、apm）封装为声明依赖关系的插件，启动时按拓扑序注册路由与扩展点；**Task Manager** 是后台调度器，告警检查、报表生成、ML 作业心跳等异步工作都注册为任务写入专用索引排队执行；**Saved Objects 客户端**是所有元数据读写的收口层。浏览器与 Kibana Server 之间除常规 REST 外还有一条 **WebSocket 通道**（运维观察、后台任务进度推送等走这条通道）。

> **设计考量**：为什么前端不直连 ES？一是**凭据隔离**——访问 ES 的 `kibana_system` 账号密码只存在于服务端 kibana.yml，浏览器只需要 Kibana 的会话 Cookie；二是**权限收口**——Kibana 可以基于自身 Space/角色体系对查询做裁剪（如 DLS/FLS 注入）后再转发；三是**版本兼容层**——Kibana API 与 ES REST API 解耦，前端升级不必受 ES 接口变更牵制。这也是所有"Kibana 代理 ES"排障思路的基础：网络抓包永远看不到浏览器直连 9200 的流量。

# 2. 请求路径模型

一次典型的 Discover 搜索在系统内的流转如下（8.x 默认开启安全）：

```text
浏览器 (SPA)
   │  ①  KQL 文本 → 前端解析为 ES Query DSL
   ▼
Kibana Server :5601
   │  ②  校验会话 Cookie / API Key
   │  ③  Space 与角色鉴权 (security plugin)
   │  ④  注入 DLS/FLS 过滤条件（若有）
   ▼
ES Client (kibana_system 凭据, 连接池复用)
   │  ⑤  POST /orders-*/_search
   ▼
Elasticsearch 集群 :9200
   │  ⑥  分片执行、协调节点汇总
   ▼
Kibana Server ⑦  结果整形(聚合→序列)
   ▼
浏览器 ⑧  Lens/Elastic Charts 渲染
```

由此可得三条实用推论：**Kibana 响应慢**先区分是 ES 查询慢（Inspect → Query 看 took 与 DSL）还是 Kibana 自身慢（`/api/status` 看加载、Node 事件循环日志）；**浏览器 401** 通常是 Kibana 会话过期而非 ES 凭据错误；**多 Kibana 实例共享一个 ES** 时所有实例读写同一批 `.kibana*` 索引，天然配置一致，不需要额外的共享存储。

# 3. Saved Objects 存储模型

Kibana 的所有配置都以 **Saved Objects** 形式存储在 ES 的 `.kibana` 索引中：

| Object 类型 | 存储内容 | 示例 |
|-----------|---------|------|
| `index-pattern` | Data View 定义 | 字段信息、时间字段 |
| `search` | 保存的搜索 | Discover 保存的查询 |
| `visualization` | 可视化配置 | Lens 图表 JSON |
| `dashboard` | 仪表盘布局 | 面板排列、查询参数 |
| `config` | Kibana 全局设置 | 主题、默认索引 |
| `alert` | 告警规则 | Rule 定义、Actions |
| `tag` | 标签 | 对象分类标签 |

每个 Saved Object 就是一个 ES 文档，`_id` 为 `类型:UUID` 形式，文档体内含 `type`、`updated_at` 和按类型存放的 `attributes`，对象之间通过 `references` 数组建立引用（dashboard 引用 visualization，visualization 引用 index-pattern）。这正是 7_dashboard 篇里导出 NDJSON 时 `includeReferencesDeep: true` 能带全依赖的原理——导出器沿引用图遍历。

```bash
# 查看 Saved Objects（Dev Tools 中执行）
GET /.kibana_8.11.0_001/_search
{ "size": 10 }
```

> **注意**：不要手工修改或删除 `.kibana*` 索引里的文档来"改配置"——Saved Objects 有乐观并发控制（`_seq_no`/`primary_term`），手改会绕过 Kibana 的校验与缓存，轻则 UI 不生效，重则引用图断裂导致仪表盘报错。批量操作配置请走 Stack Management → Saved Objects 或 `_import/_export` API。

# 4. kibana.yml 核心配置

Kibana 的全部服务端行为由单一 `kibana.yml`（或同名环境变量覆盖）控制。配置按作用域分为四组，逐组要点如下：Server 组决定监听与 TLS；Elasticsearch 组决定连接目标、凭据与超时；Logging 组决定日志级别与输出方式；其余为路径、任务与 xpack 开关。

```yaml
# ========== Server 配置 ==========
server.port: 5601
server.host: "0.0.0.0"
server.name: "kibana-prod"
server.publicBaseUrl: "https://kibana.example.com"  # 8.x 必填：告警/报表里的绝对链接用它生成
server.maxPayload: 10485760        # 10MB 最大请求体
server.compression.enabled: true
server.ssl.enabled: true
server.ssl.certificate: /etc/kibana/certs/kibana.crt
server.ssl.key: /etc/kibana/certs/kibana.key

# ========== Elasticsearch 配置 ==========
elasticsearch.hosts: ["https://es1:9200", "https://es2:9200"]
elasticsearch.username: "kibana_system"
elasticsearch.password: "${KIBANA_PASSWORD}"
elasticsearch.ssl.certificateAuthorities: /etc/kibana/certs/ca.crt
elasticsearch.requestTimeout: 30000
elasticsearch.shardTimeout: 30000

# ========== 日志配置 ==========
logging.root.level: info
logging.appenders.file.type: file
logging.appenders.file.fileName: /var/log/kibana/kibana.log

# ========== 其他 ==========
path.data: /var/lib/kibana
pid.file: /var/run/kibana.pid

# 内嵌 ES（开发用，不建议生产）
# elasticsearch.hosts: "http://localhost:9200"
```

两个容易踩的配置点：**`server.publicBaseUrl` 在 8.x 中不设置会启动警告且报表/告警消息里的链接退化为相对路径**；`elasticsearch.requestTimeout`（默认 30s）是 Kibana 等待 ES 响应的上限，Dashboard 重查询超时报 `Request Timeout` 时优先调这里，而不是盲目调大 ES 侧超时。

# 5. Kibana 系统索引

Kibana 会在 ES 中创建并维护一批以 `.kibana` 为前缀的系统索引，各司其职：

```text
.kibana                 → 7.x Saved Objects（别名入口）
.kibana_8.11.0_001      → 8.x Saved Objects（版本号区分）
.kibana_alerting_cases  → 案例数据
.kibana_task_manager    → 后台任务（告警/报告）
.kibana-event-log-*     → 审计事件日志
```

8.x 采用**版本号命名的物理索引 + `.kibana` 别名**的方式管理 schema 演进：升级后旧索引被标记只读保留，Kibana 启动时自动创建新版本索引并迁移 Saved Objects。若迁移异常（如版本混用），可用升级辅助工具或显式触发：

```bash
# 升级后旧索引会被标记为只读，新索引自动创建
# 显式迁移
POST /_kibana/_upgrade_saved_objects
```

> **注意**：`.kibana_task_manager` 膨胀导致 Kibana 变慢是经典故障——它是告警/报表任务的队列存储，若报表任务持续失败会不断堆积。观察 `GET /.kibana_task_manager*/_count`，异常增长时先修告警规则与报表配置，再考虑清理历史任务文档；这些索引同样会被 snapshot 覆盖，备份 ES 时**必须包含 `.kibana*`**，否则恢复集群后仪表盘全部丢失。

# 6. 与 ES 的通信模型

Kibana 对 ES 的使用可归纳为四类通道，它们的鉴权、超时、失败表现各不相同：

```text
Kibana Server (5601)
  │
  ├── Data Queries (Discover/Dashboard)
  │     └── 代理转发到 ES（透明）
  │
  ├── Saved Objects CRUD
  │     └── 直接操作 .kibana 索引
  │
  ├── Background Tasks (Alerting/Reporting/ML)
  │     └── Task Manager 定时执行
  │
  └── Fleet API
        └── Agent 策略下发
```

前两类是**同步请求路径**，用户可直接感知延迟；后两类是**异步任务路径**，由 Task Manager 轮询 `.kibana_task_manager` 认领任务执行，失败的告警检查会按退避策略重试。Fleet 通道面向 Elastic Agent 长连接，走 API Key 鉴权而非 kibana_system。运维视角的结论：`kibana_system` 账号权限不足时，现象通常是"能打开界面但数据/管理页报错 403"，因为它只能访问 `.kibana*` 与集群健康信息，**不能读取业务索引**——业务数据访问走的是"以登录用户自身权限代理查询"的 run-as 模型，这是 3_installation 篇强调要给普通用户单独授权的原因。

# 7. Kibana 性能配置

Kibana 是单进程 Node 应用，性能瓶颈主要在**事件循环阻塞与 ES 连接池**，关键配置及建议值：

```yaml
# ES 连接池
elasticsearch.maxSockets: 1000          # 与 ES 间最大并发连接数（默认 10）

# Saved Objects 缓存与导入体积
kibana.savedObjects.maxImportPayloadBytes: 26214400  # 25MB

# 集群 sniffing：生产建议关闭
elasticsearch.sniffOnStart: false

# 请求压缩：大结果集显著省带宽
elasticsearch.compression: true
```

补充两条不在 kibana.yml 里但同样重要的实践：Dashboard 并发查询数受 `data:pageSize` 等 Advanced Settings 与浏览器并发限制影响，面板数过多时"整屏 loading"的根因往往是 ES 聚合慢而非 Kibana 慢；多实例横向扩容时前面挂 LB 即可（会话可配 Redis 共享：`server.publicBaseUrl` + session 存储保持默认内存时**必须配粘性会话**，8.x 推荐启用 Redis 作为共享 Session Store 以保证 LB 后任意实例可服务）。

# 8. 生产配置模板

综合以上各节，一套可直接落地的生产 `kibana.yml`（TLS + 三节点 ES + 告警加密 + 日志降噪）：

```yaml
server.port: 5601
server.host: "0.0.0.0"
server.publicBaseUrl: "https://kibana.example.com"
server.ssl.enabled: true
server.ssl.certificate: /etc/kibana/certs/kibana.crt
server.ssl.key: /etc/kibana/certs/kibana.key

elasticsearch.hosts: ["https://es1:9200", "https://es2:9200", "https://es3:9200"]
elasticsearch.username: "kibana_system"
elasticsearch.password: "${KIBANA_PASSWORD}"
elasticsearch.ssl.certificateAuthorities: /etc/kibana/certs/ca.crt
elasticsearch.requestTimeout: 30000
elasticsearch.maxSockets: 1000

logging.root.level: warn

# 告警/报表等敏感元数据落盘加密所必需（32 字符以上随机串）
xpack.encryptedSavedObjects.encryptionKey: "${ENCRYPTION_KEY}"
xpack.reporting.encryptionKey: "${REPORTING_KEY}"
```

> **行业实践**：`xpack.encryptedSavedObjects.encryptionKey` 必须多实例**完全一致**且纳入密钥管理（K8s Secret/Vault）——它加密 Connector 里的 Webhook Token、ES 凭据等敏感字段，密钥不一致会导致部分写入的密文永远解不开；该密钥一旦丢失且未备份，已配置的告警连接器凭据只能重建。密码类配置一律用环境变量注入，不要把明文写进 yml 提交到 Git。

# 9. 小结

Kibana 架构一句话：**Node.js（Hapi）服务端 + 浏览器 SPA，自身无状态，全部配置以 Saved Objects 文档存进 ES 的 `.kibana*` 索引**。请求路径是"浏览器 → Kibana（鉴权/裁剪/代理）→ ES"，浏览器永不直连 9200；与 ES 的交互分数据查询、Saved Objects CRUD、后台任务、Fleet 四类通道，前两类同步、后两类异步。配置面聚焦 kibana.yml 的 Server/ES/Logging 三组，生产必设 `server.publicBaseUrl`、TLS、`kibana_system` 凭据与两个 encryptionKey。性能排障先分清"Kibana 慢还是 ES 慢"，扩容只需多实例 + LB（配 Redis 共享会话），但别忘了 snapshot 要覆盖 `.kibana*`。
