
# 1. 最佳实践的总纲

SkyWalking 落地失败的项目很少死于技术——绝大多数死于**治理纪律**：服务命名混乱导致拓扑不可读、全量采样导致存储爆炸、Agent 版本漂移导致断链、Trace 里塞大 payload 导致链路变慢。本篇把生产环境反复验证过的配置纪律、容量协同与反模式汇总成清单，按"Agent 部署 → 命名与标识 → 采样 → 性能边界 → 流程协同 → 反模式"组织。

# 2. Agent 部署策略

```text
策略对比：

Sidecar/InitContainer 模式（K8s）：
  ├── Agent 镜像作为 InitContainer 拷贝到共享卷
  ├── 应用容器挂载 -javaagent（或 JAVA_TOOL_OPTIONS）
  ├── 优点：Agent 版本独立发布轴，应用镜像零侵入
  └── 缺点：Pod 内存额外 +50~150MB，spec 复杂一层

启动参数模式（虚机/容器自建镜像）：
  ├── -javaagent:/path/to/skywalking-agent.jar
  ├── 优点：简单直接，配置就在启动脚本里
  └── 缺点：Agent 升级需重启应用

镜像内置模式：
  ├── Agent 打包在应用镜像中（Dockerfile COPY）
  ├── 优点：最简部署，版本随应用锁定
  └── 缺点：Agent 升级要全量重建镜像，周期以月计
```

选型建议：K8s 环境一律 InitContainer 或注入 Webhook；虚机环境用启动参数 + 配置中心下发的集中化 `agent.config`。**无论哪种模式，Agent 的配置文件都应视为"平台配置"而非"应用配置"**——由监控平台团队维护基准版本，业务方只允许改白名单里的项（服务名、忽略路径）。

# 3. 服务命名与实例标识纪律

命名是 SkyWalking 治理里**回报最高、成本最低**的一件事。所有 UI 检索、告警路由、仪表盘模板都按 `service/instance/endpoint` 三元组组织，命名不纪律等于给整个观测体系埋雷。

```properties
# agent.config —— 推荐用环境变量注入，与 CI/CD 对齐
# 服务名：{域}-{应用}，全小写连字符，禁止中文空格与环境词以外的动态量
agent.service_name=${SW_AGENT_NAME:order-trade-service}
# 命名空间：隔离环境，UI 服务列表按 namespace 过滤
agent.namespace=${SW_AGENT_NAMESPACE:prod}
# 实例名：稳定唯一标识；K8s 用 POD_NAME，虚机用 hostname
agent.instance_name=${SW_INSTANCE_NAME:}
# 实例属性：附加元数据，UI properties 面板可查
agent.instance_properties[region]=cn-east-1
agent.instance_properties[cluster]=hz-prod
agent.instance_properties[framework]=spring-boot
```

**三条硬规则**：其一，**服务名绝不含动态量**（Pod 名、IP、随机数）——服务名是长期实体，每次滚动重建产生新服务名会让指标、告警、拓扑全部历史断裂；其二，**环境靠 `agent.namespace` 隔离**而不是塞进服务名（`prod-order` vs `staging-order` 的做法会让跨环境对比、模板复用全部失效）；其三，**一个服务名只属于一个逻辑服务**，复制启动脚本导致多个不相关应用共用一个名字，是"拓扑图出现诡异自环"的头号原因。

`serviceNameMaxLength`（默认 70）与 `instanceNameMaxLength` 会截断超长名字——截断在 OAP 侧静默发生，命名时预留余量。实例名留空时 Agent 自动生成 UUID 型实例 ID，K8s 下 Pod 每次重建就是一个新实例，这是正常行为（实例随 TTL 过期），但**排障时用 POD_NAME 做实例名可读性远高于 UUID**。

# 4. 采样策略：10000 全采与降采的权衡

SkyWalking 有两级采样旋钮，语义容易混淆，必须先分清：

```properties
# ① Agent 侧（skywalking-java）：本地限流，决定"生成并上报多少 Segment"
# 每 3 秒最多采样的 Trace 数；-1 = 全采（默认）
agent.sample_n_per_3_secs=-1
```

```yaml
# ② OAP 侧（9.x/10.x）：trace-sampling-policy-settings.yml
# rate 精度为 1/10000，10000 = 100% 全采（默认）
default:
  rate: 10000        # 全采
  duration: -1       # -1 = 不因"慢"而额外保留
# 可按服务覆盖：核心服务全采、边缘服务降采
services:
  - name: gateway-service
    rate: 10000
  - name: log-agg-service
    rate: 1000       # 10%
  - name: report-service
    rate: 100
    duration: 3000   # 且 RT>3s 的 Trace 无条件保留
```

**默认 `rate: 10000` 全采是官方立场**，理由是：降采样会破坏拓扑统计精度（边权按采到的量外推）、破坏告警的完整性、并且"故障时你最想看的那条 Trace 恰好被采掉"是墨菲定律重灾区。正确姿势是**分层降采**：入口网关与核心交易链路全采；高 QPS 的内部服务按 rate 降采；同时用 `duration` 阈值保证**慢 Trace 无条件保留**，用 `agent-analyzer.forceSampleErrorSegment=true`（默认开）保证**错误 Trace 不丢**，用 sw8 头的 sampled 标志（Forced Tracking，1 表示强制采样并沿链路传播）保证**调试中的 Trace 不丢**。

> **注意**：Agent 侧 `sample_n_per_3_secs` 与 OAP 侧 rate 都会丢数据，**排查"Trace 怎么又没了"时两级都要查**。生产上建议 Agent 侧保持 -1（生成成本低），把丢数据的决策集中到 OAP——降采样策略改动不用重启业务应用，这是集中式的最大价值。

# 5. 端点与 Tag 的基数治理

端点是指标维度的爆炸源。`/api/user/{id}` 若被记成 `/api/user/12345`，一个接口就是上万个端点，每个端点独立产出分钟级指标：

```properties
# 方式 1（首选）：代码使用路径模板，Spring MVC 的 @PathVariable 由 Agent 归一化为 {xxx}
@GetMapping("/api/user/{id}")
public User getUser(@PathVariable Long id) { ... }

# 方式 2：正则化插件兜底（对第三方/遗留代码）
plugin.springmvc.use_qualified_name_as_endpoint_name=true
agent.ignore_suffix=.jpg,.css,.js,.html

# 方式 3（9.x/10.x OAP 侧）：endpoint-group-rules.yml 按正则把命中端点归组
# 或在 UI 的 Endpoint Grouping 功能里维护规则，无需改 Agent
```

Tag 侧同理：`searchableTracesTags` 只把**声明过的 key** 建成倒排可检索维度，自定义 tag 值域必须有限（order_status 可以，order_id 不行）。OAP 对端点名也有 `endpointNameMaxLength`（150）截断保护，但截断救不了基数——**治理位置永远在产生端点的那一层**。

# 6. 字节码增强的性能边界

Agent 的开销模型（参考 4_java_agent 篇的量级）：Span 创建微秒级、拦截器调用纳秒到微秒级、异步上报序列化才到几十微秒。正常业务负载整体 CPU 开销 **3%~8%**，超出这个区间基本是配置问题而不是 Agent 问题：

```properties
# 高 QPS 接口的收敛开关（按需要）：
# 单 Segment 的 Span 数上限，超出强制截断（默认 300，防循环打 Span 撑爆段）
agent.span_limit_per_segment=300
# Agent 上报缓冲：内存 channel 与磁盘 buffer 上限，满了先丢旧数据
buffer.channel.size=60000
buffer.buffered_file_size=50mb
# 数据库参数采集默认关闭（防 PII 与大 SQL 撑爆 Span）
plugin.jdbc.trace_sql_parameters=false
```

```properties
# trace-ignore 插件（optional-plugins 拷贝后生效）：忽略健康检查与静态资源
agent.ignore_suffix=.jpg,.css,.js,.html
trace.ignore_path=/actuator/**,/health,/favicon.ico
```

**增强边界的纪律**：只增强"框架边界方法"（拦截器/代理点），不要为图方便用 `@Trace` 注解给循环体内的小方法打标——每个 Span 本身很便宜，但一个循环打上千个 Span，Segment 体积与序列化成本立刻失控；同理，**不要加载用不到的插件**（optional-plugins 按需拷贝），增强类数量直接影响启动时间与 Metaspace。

# 7. 与发布、压测、灰度的协同

**灰度升级 Agent**：Agent 与 OAP 的兼容策略是"新 OAP 兼容老 Agent，老 OAP 不保证兼容新 Agent 的新数据格式"，所以升级顺序永远是**先升 OAP、后升 Agent**；Agent 灰度按部署批次推进（K8s 改 initContainer tag 即天然分批），保留上一版本镜像以便回滚。重大版本（如 8.x→9.x）先在 staging 全链路验证，重点看断链与自定义插件兼容性。

**与压测协同**：压测流量的 Trace 会污染生产指标与告警。两种正规做法：压测环境独立 `agent.namespace`/独立 OAP 集群；或在入口网关给压测请求打标，利用 `trace.ignore_path`/OAP 侧 ignore 规则丢弃影子流量。**绝对禁止**用"压测时临时关 Agent"的方式——那恰好让压测报告失去了最需要的链路证据。

**与 CI 协同**：把 `agent.service_name`、OAP 地址、namespace 作为部署平台的必填项做准入校验（CI 里 lint Helm values/启动脚本），命名不合规拒绝发布——治理规则不进流水线就等于没有规则。

> **行业实践**：成熟团队会把"服务注册表"（服务名 → 负责人 → SLO）与 SkyWalking 服务名一一对齐，告警按注册表路由到人；SkyWalking 的 `instance_properties` 常用来携带团队约定的元数据（机房、版本、灰度标记），成本极低。

# 8. 常见反模式清单

```text
✗ Trace 里塞大 payload：把请求/响应体、整段 JSON、堆栈全文写进 tag/log 字段
   —— Span 膨胀 → 上报带宽与序列化暴涨 → ES segment 索引爆炸
   正确做法：tag 只放摘要与业务 ID，大内容进日志系统用 TraceId 关联
✗ 用 Pod 名/IP 做服务名（每次重建产生新服务，指标拓扑断裂）
✗ 多个不相关应用共用一个 agent.service_name（拓扑出现假自环）
✗ 生产全量开启 SQL 参数追踪（PII 泄露 + Span 膨胀双重风险）
✗ 不设 Trace TTL 或 TTL 设 90 天（存储无限增长，代价全在磁盘上）
✗ 在循环/高频小方法上打 @Trace（Segment 体积失控）
✗ Agent 比 OAP 先升级大版本（数据协议不兼容，静默丢数据）
✗ 压测流量混入生产监控且不打标（容量指标失真，告警疲劳）
✗ 忽略端点不设 trace.ignore_path（健康检查淹没 Trace 列表与 TopN）
✗ 告警规则全发同一个群（关键告警被噪声淹没）
```

**设计考量**：以上反模式的共同根源是把 SkyWalking 当"免费的日志放大器"用——它的每个数据点都参与聚合与索引，**写入侧省下的每一字节，都在查询侧和存储侧加倍偿还**。观测数据的纪律是：高价值、低基数、小体积。

# 9. 容量规划速查

```text
每日 Trace 量 ≈ 实例数 × 每实例采样 Trace 速率 × 86400 × 采样率

示例：10 个服务 × 50 实例 × 高 QPS，OAP rate 全采
  假设每服务每秒 500 Trace：10 × 500 × 86400 ≈ 4.32 亿条/天 → 必须降采
  网关全采 + 内部服务 rate=1000（10%）后 ≈ 数千万条/天
  存储 ≈ 条数 × 平均 Segment 数 × 单 Segment KB 级 × 副本/索引放大(ES 2~3x)
```

按这个口径先算一遍，再对照 15 篇的单机吞吐上限（8C16G 约 1~2 万 Segment/s）决定 OAP 节点数与是否拆分集群——**先算后配，永远比先配后调省心**。

# 10. 小结

| 领域 | 一句话纪律 |
|------|-----------|
| 部署 | K8s 用 InitContainer/注入，Agent 版本是独立发布轴 |
| 命名 | 服务名无动态量、环境用 namespace、一名一服务 |
| 采样 | 默认全采（rate 10000），降采分层做，慢/错 Trace 无条件保留 |
| 基数 | 端点与 tag 治理前移到 Agent/代码层，别指望存储兜底 |
| 性能 | 开销 3%~8% 为正常，超了查配置；只增强框架边界 |
| 协同 | 先升 OAP 后升 Agent；压测流量打标；命名进 CI 校验 |
| 反模式 | 大 payload 入 Trace、共用服务名、无 TTL、全开 SQL 参数——四害必除 |
