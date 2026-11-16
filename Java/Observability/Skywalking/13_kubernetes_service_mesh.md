
# 1. K8s 部署

SkyWalking 落地 K8s 之后，服务端（OAP + UI）的部署已经被 Helm 收敛成一条命令，真正需要设计的是**探针侧**：怎么在不改应用镜像的前提下把 Java Agent 注入到 Pod 里、怎么让 Agent 的内存开销与容器 limits 相处、怎么在 Pod 频繁重建的环境里保住实例与拓扑数据的整洁。另一条线是 **Service Mesh**：不侵入应用、直接从 Envoy 拿流量的观测路线，它与 Agent 路线各有边界，生产上通常是**混合使用**而不是二选一。

## 1.1 OAP + UI Helm 部署

```bash
helm repo add skywalking https://apache.jfrog.io/artifactory/skywalking-helm
helm install skywalking skywalking/skywalking \
  --set oap.image.tag=9.7.0 \
  --set ui.image.tag=9.7.0 \
  --set oap.storageType=elasticsearch \
  --set elasticsearch.host=es-cluster:9200
```

这个 chart 会拉起 **OAP Deployment（默认 2 副本）+ UI Deployment + Service**，`oap.storageType` 支持 `elasticsearch`、`banyanDB`、`mysql` 等取值，也可以指向集群外已有的存储。OAP 多副本在 K8s 上靠 **`SW_CLUSTER=k8s` 的原生集群模式**互相发现：Pod 通过 watch 带特定 label 的 endpoint 组成集群，chart 已自动注入相关环境变量，因此需要给 OAP 的 ServiceAccount 授予 `endpoints/pods` 的 list/watch 权限——这也是官方 chart 会一并创建 RBAC 资源的原因。

> **注意**：OAP 副本数不是越多越好。每个 OAP 节点都承担 Receiver、聚合与存储写入的完整管道，扩容前先确认瓶颈在 OAP 还是存储；集群模式下所有节点共享同一存储，存储写入能力才是天花板（参见存储后端与 OAP 性能两篇）。

## 1.2 Sidecar Agent

Java Agent 无法作为真正独立的 sidecar 进程运行——它必须附着在应用 JVM 里。所以 K8s 上事实标准是 **"initContainer 拷贝 + emptyDir 共享卷"**：用一个官方 Agent 镜像把 Agent 目录复制进共享卷，主容器通过 `JAVA_TOOL_OPTIONS` 挂载，**应用镜像完全不用改**。

```yaml
apiVersion: apps/v1
kind: Deployment
spec:
  template:
    spec:
      containers:
        - name: app
          image: my-app:latest
          env:
            # JVM 自动识别该环境变量，等价于命令行 -javaagent
            - name: JAVA_TOOL_OPTIONS
              value: "-javaagent:/skywalking/agent/skywalking-agent.jar"
          volumeMounts:
            - name: agent
              mountPath: /skywalking/agent
        - name: skywalking-agent
          image: apache/skywalking-java-agent:9.7.0
          volumeMounts:
            - name: agent
              mountPath: /skywalking/agent
      initContainers:
        - name: init-agent
          image: apache/skywalking-java-agent:9.7.0
          command: ['cp', '-r', '/skywalking/agent', '/target/']
          volumeMounts:
            - name: agent
              mountPath: /target
      volumes:
        - name: agent
          emptyDir: {}
```

这个模式的好处是 **Agent 版本是一条独立的发布轴**：升级 Agent 只改 initContainer 的镜像 tag，不需要业务方重新构建镜像；回滚同理。代价是每个 Pod 多一个 initContainer，且 Agent 的 JVM 参数（服务名、OAP 地址）要靠环境变量传（见第 5 节）。

**设计考量**：三种注入方式的取舍——initContainer + emptyDir 无镜像侵入、改 tag 滚动即可升级，是 K8s 主流做法；打进应用镜像最简但升级要重新构建；Operator/Webhook 自动注入同样无侵入，适合平台团队按标签统一治理。

## 1.3 Operator 与自动注入

手工写 initContainer 适合试点，全集群推广时通常交给 **准入 Webhook 自动注入**：给 namespace 打标签，Pod 创建时被自动改写 spec 注入 Agent。Go 探针有官方的 `apache/skywalking-go-operator`（基于 eBPF 的 Go 自动插桩，注入逻辑由 Operator 完成）；Java Agent 没有官方 Operator，但社区有大量把上面 initContainer 模式包装成 MutatingWebhook/CRD 的实现，思路上都是"**按标签匹配 Pod → 注入卷与 JAVA_TOOL_OPTIONS → 注入环境变量**"。选型时关注三点：Webhook 挂掉会不会阻塞业务 Pod 创建（`failurePolicy: Ignore` 保可用）、注入逻辑是否支持按服务覆盖采样参数、Agent 版本是否可在 CRD 中统一管理。

# 2. k8s-event-exporter

K8s 自身的**事件**（Pod OOMKilled、CrashLoopBackOff、Node NotReady、Deployment 扩缩容）是排障时最有上下文的信号，SkyWalking 用独立的 exporter 把它们送进 OAP，展示在 UI 的 Event 页面，与服务指标、Trace 按时间线对齐。

```yaml
# 部署 k8s-event-exporter 收集 K8s 事件
apiVersion: apps/v1
kind: Deployment
metadata:
  name: skywalking-k8s-event-exporter
spec:
  template:
    spec:
      serviceAccountName: skywalking
      containers:
        - name: exporter
          image: apache/skywalking-kubernetes-event-exporter:latest
          env:
            - name: OAP_ADDRESS
              value: skywalking-oap:11800
            - name: CLUSTER_NAME
              value: prod-cluster
```

```text
K8s Events → Event Exporter → OAP Server → UI 展示
  Pod OOMKilled
  Node NotReady
  Deployment Scaled
```

链路是：exporter 用 ServiceAccount **watch API Server 的 events 资源**（需要 `list/watch events` 的 RBAC），把事件规范化成 SkyWalking Event 协议经 gRPC 上报；`CLUSTER_NAME` 决定事件归属的集群标识。两个使用要点：其一，**K8s 事件本身默认只保留约 1 小时**，exporter 必须常驻，事后补捞是捞不回来的；其二，事件在存储侧同样吃 TTL，量级远小于 Trace，但也不要和 Trace 混在同一个过期策略里省空间。

> **行业实践**：除外部事件外，9.x 的 OAP 还支持 `receiver-event`——把**指标越限**（如某服务错误率超阈值）自动转换成一条 Event 记录，与 K8s 事件并列展示在同一个页面，形成"业务异常 + 平台异常"统一时间线。K8s 集群的指标采集则走另一条链路：`receiver-k8s` 直接 watch API Server 采集集群资源指标，或用 `receiver-prometheus` 抓取 kube-state-metrics/node-exporter，两者产出的是 K8s 仪表盘，不要与事件链路混淆。

# 3. Service Mesh 集成（Istio）

Mesh 观测的核心卖点是**零应用侵入**：流量已经全部经过 Envoy，OAP 直接消费 Envoy 的访问日志与指标流，就能拿到服务间调用关系、L4/L7 指标与 mTLS 对端身份，不需要给每个服务挂探针。

```text
┌────────────────────────────────────────────┐
│               Service Mesh                  │
│                                             │
│  ┌──────────┐         ┌──────────┐        │
│  │  Envoy   │ ←────→  │  Envoy   │        │
│  │ (Sidecar)│   mTLS  │ (Sidecar)│        │
│  └────┬─────┘         └────┬─────┘        │
│       │ ALS                 │ ALS          │
│       ▼                     ▼              │
│  ┌──────────────────────────────────┐     │
│  │      SkyWalking OAP (ALS 接收)    │     │
│  └──────────────────────────────────┘     │
└────────────────────────────────────────────┘
```

## 3.1 Envoy ALS 配置

```yaml
# Istio MeshConfig
meshConfig:
  defaultConfig:
    envoyAccessLogService:
      address: skywalking-oap:11800
  enableEnvoyAccessLogService: true
```

ALS（Access Log Service）是 Envoy 的流式访问日志通道，Istio 开启后**每个 Sidecar 通过 gRPC 流持续上报请求日志**。SkyWalking OAP 侧由 ALS Receiver（`receiver-als`）接收，与 Agent 共用 11800 端口，经 mesh 分析器生成 `service_mesh_*` 系列指标（请求量、成功率、响应时间）与 Mesh 层拓扑；Envoy 的指标流（Metrics Service）还可以把 mTLS 证书有效期、集群级连接数等 L4 指标一并上报。配置要点：Istio 版本与 OAP 版本要匹配（ALS 格式随 Envoy 演进过），并且 Sidecar 到 OAP 的 11800 出流量要在 NetworkPolicy/授权策略中放行。

## 3.2 Mesh 指标的边界与混合采集

**ALS 数据只有"流量视角"**：谁调了谁、多少次、多少延迟、mTLS 身份是什么，但拿不到应用内部的 SQL、Redis、方法级 Span，Trace 也止步于 Sidecar。这决定了 Mesh 集成的定位是**全局底账**而非深度诊断。生产上的标准打法是混合：用 ALS 快速铺开全集群拓扑和 L4/L7 指标，再对核心业务补充 Java/Go Agent 获得应用内深度数据——两套数据在 UI 中分属 Mesh 层与 General 层，互不冲突。

> **行业实践**：Mesh 团队普遍先只用 ALS 建立全量服务拓扑与告警底账，再按业务价值逐步给关键服务补挂 Agent；反过来，"上了 Mesh 就不再上 APM"几乎必然在第一次数据库慢查询排障时被推翻。

> **注意**：**Ambient 网格改变了游戏规则**——Istio Ambient 取消了每 Pod 的 Envoy Sidecar，L4 mTLS 由节点级 ztunnel 完成，L7 交给 waypoint，依赖 Sidecar 的 ALS 采集路径只剩 waypoint 覆盖到的流量，大量 L4-only 流量退化为连接级指标。可以理解为 Ambient 时代 Mesh 观测进一步向"OTLP + 应用探针"靠拢，纯 ALS 集成的价值在缩水。

## 3.3 mTLS 与连接安全

Sidecar 模式下 **mTLS 对探针是透明的**：应用与本地 Sidecar 走明文 loopback，出 Pod 后由 Envoy 加密封装，Agent 上报 OAP 的流量也被 Sidecar 代管，无需探针自己处理证书。要注意的反而是不装 Sidecar 的链路——**Agent/Satellite 直连 OAP 时建议自行启用 TLS**：`agent.enable_tls=true`（对应环境变量 `SW_AGENT_ENABLE_TLS`）并配置受信 CA，OAP 侧开启对应 gRPC TLS；跨信任域（如混合云）尤其必要。

# 4. SkyWalking Satellite

Satellite 是用 Go 写的**轻量级观测数据前置代理**，定位类似"观测数据的 Nginx"：Agent 不再直连 OAP 集群，而是连本节点/本集群的 Satellite，由它完成转发、聚合、采样、过滤与缓冲：

```text
Agent → Satellite (聚合/采样/过滤) → OAP Server
```

```yaml
# satellite-config.yaml（示意骨架，具体字段以 apache/skywalking-satellite 各版本示例为准）
pipeline:
  - name: trace
    type: trace
    filter:
      - type: sampling
        rate: 1000    # 每秒最多转发 1000 条
    sender:
      - type: grpc
        server: oap:11800
```

Satellite 解决三类问题：**连接数收敛**（上千 Agent 各自维护到 OAP 的 gRPC 长连接，收敛成 Satellite 到 OAP 的少数几条）、**网络缓冲**（OAP 滚动升级或网络抖动时，Satellite 本地队列/磁盘缓冲，避免 Agent 侧丢数）、**前置过滤**（在边缘先做采样、按路径丢弃健康检查等噪音数据）。典型部署是 DaemonSet 每节点一个，Agent 指向 `localhost:11800`。代价是多一跳延迟与一个需要运维的组件，**中小规模集群不必上**——这层复杂度在几百实例以内是净负担。

# 5. 容器环境 JVM 配置

```bash
# K8s 环境中的 Java Agent 参数（环境变量均通过 Downward API 注入）
java -javaagent:/skywalking/agent/skywalking-agent.jar \
     -XX:MaxRAMPercentage=70.0 \
     -DSW_AGENT_NAME=${SERVICE_NAME} \
     -DSW_AGENT_INSTANCE_NAME=${POD_NAME} \
     -DSW_AGENT_NAMESPACE=${ENV} \
     -DSW_AGENT_COLLECTOR_BACKEND_SERVICES=skywalking-oap:11800 \
     -jar app.jar
```

Agent 的 `agent.config` 全部支持用 `SW_` 前缀的系统属性/环境变量覆盖，这正好契合 K8s "一份镜像多环境跑"的形态。命名上有一条硬纪律：**`SW_AGENT_NAME`（服务名）必须取 Deployment/工作负载名这类稳定值，Pod 名只能进实例名 `SW_AGENT_INSTANCE_NAME`**——若把 Pod 名写成服务名，每次滚动升级都会冒出一堆"新服务"，拓扑与历史指标全部碎裂；实例名用 POD_NAME 则天然对应"一个实例=一个 Pod"，旧 Pod 消亡后实例由 TTL 自动清理。

**Agent 自身的容器开销**要计入 limits：上下文栈、上报缓冲队列、gRPC 通道与增强类的元空间，一般让进程 RSS 增加约 **50~150MB**（与 QPS、插件数量相关）。容器 JVM 最常见的坑是给内存留得太满：`limits 2Gi` 而堆也配 2Gi，加上 Agent 与堆外就必然 **OOMKilled**——用 `MaxRAMPercentage` 按比例留头寸（堆 60%~70%，余量给元空间/线程栈/Agent/堆外），比硬写 `-Xmx` 更抗配置漂移。

> **注意**：`JAVA_TOOL_OPTIONS` 与 `-javaagent` 二选一即可，同时存在会加载两个 Agent 实例导致重复上报；另外 initContainer 里用 `cp -r` 拷贝后，主容器的挂载路径必须与 `cp` 的目标结构一致（常见故障是拷贝出 `/target/agent` 双层目录，`-javaagent` 路径多算一层导致 Agent 静默不加载）。

# 6. 最佳实践

1. **InitContainer 拷贝 Agent**：避免在应用镜像中打包 Agent，让 Agent 版本成为独立发布轴
2. **POD_NAME 做实例名**：服务名固定用工作负载名，环境靠 `SW_AGENT_NAMESPACE` 隔离
3. **Satellite 汇聚**：数百实例以上使用 Satellite 降低 OAP 连接压力并缓冲网络抖动
4. **Service Mesh ALS**：Mesh 场景优先使用 ALS 建立全局底账；深度诊断仍需补 Agent
5. **内存留头寸**：容器 limits 按 `MaxRAMPercentage` 留出 Agent 与堆外余量，防 OOMKilled

**设计考量**：K8s/Mesh 场景下选择采集路线，按"侵入成本—数据深度"权衡——Agent 路线数据最深（方法/SQL/日志关联）但每语言每服务都要治理；ALS 路线覆盖最快但只到流量层；OTel SDK 路线（应用接 OTLP，OAP 做后端）介于两者之间。三条路线可在同一个 OAP 共存。

# 7. 小结

| 主题 | 关键结论 |
|------|---------|
| 服务端部署 | 官方 Helm chart + `SW_CLUSTER=k8s` 原生发现，瓶颈通常在存储不在 OAP |
| Agent 注入 | initContainer + emptyDir + `JAVA_TOOL_OPTIONS` 是事实标准；规模化交给 Operator/Webhook |
| 命名纪律 | 服务名=工作负载名（稳定），实例名=Pod 名，环境用 namespace |
| Mesh 集成 | ALS 零侵入拿 L4/L7 拓扑指标，拿不到应用内部；生产用"ALS 底账 + Agent 深度"混合 |
| Satellite | 连接收敛/缓冲/前置过滤，大集群才值得引入 |
| 容器 JVM | 给 Agent 与堆外留内存头寸，防 OOMKilled |

存储后端与 OAP 性能两篇继续展开"数据进来之后"的容量与调优问题。
