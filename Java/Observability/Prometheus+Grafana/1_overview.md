
# 1. 什么是 Prometheus

Prometheus 是由 SoundCloud 于 2012 年开源的一款**系统监控和警报工具集**，2016 年加入 CNCF（Cloud Native Computing Foundation），是继 Kubernetes 之后第二个毕业的项目。它以**多维数据模型**、**灵活的 PromQL 查询语言**、**拉取（Pull）采集模型**和**强大的告警能力**为核心特点，已成为云原生监控领域的事实标准。

# 2. 发展历史

| 时间 | 里程碑 |
|------|--------|
| 2012 | SoundCloud 内部开发，由 Matt T. Proud 和 Julius Volz 发起 |
| 2015 | 开源发布 0.1 版本 |
| 2016 | 加入 CNCF，成为第二个托管项目（继 Kubernetes 之后） |
| 2018 | 发布 2.0 版本，重写存储引擎为自研 TSDB |
| 2018 | 从 CNCF 毕业 |
| 2020+ | 成为云原生生态核心组件，PromQL 成为 CNCF 规范 |

# 3. 核心特性

| 特性 | 说明 |
|------|------|
| 多维数据模型 | 基于时间序列，由 Metric Name + Key/Value 标签定义 |
| PromQL | 强大的查询语言，支持聚合、计算、预测等 |
| Pull 采集模式 | Server 主动拉取指标，而非 Agent 推送 |
| 服务发现 | 原生支持 Kubernetes、Consul、DNS、文件等多种发现方式 |
| 无依赖存储 | 自研 TSDB，单节点即可运行 |
| 高可扩展性 | 通过联邦集群、Thanos、VictoriaMetrics 等方案可无限扩展 |
| 精确告警 | 基于 PromQL 定义告警规则，Alertmanager 负责去重、分组、路由 |

# 4. 生态组件

Prometheus 不是单一组件，而是一个生态系统：

```
+----------------+     +------------------+     +------------------+
|   Targets      |     |   Prometheus     |---->|   Alertmanager   |----> 通知(邮件/钉钉/微信等)
| (Exporters)    |---->|   (Server)       |     +------------------+
|   Applications |     |                  |---->|   Grafana        |
|   Pushgateway  |     +------------------+     +------------------+
|   Service Disc |             |
+----------------+             v
                        +------------------+
                        |   Data Storage   |
                        | (Local TSDB /    |
                        |  Remote Storage) |
                        +------------------+
```

| 组件 | 作用 |
|------|------|
| Prometheus Server | 核心组件，负责指标采集、存储和查询 |
| Alertmanager | 告警管理：去重、分组、路由、静默、抑制 |
| Exporters | 暴露指标的 Agent（Node Exporter、Blackbox Exporter 等） |
| Pushgateway | 短任务指标中转组件 |
| Grafana | 可视化仪表盘 |
| Service Discovery | 自动发现采集目标（K8s、Consul 等） |
| Thanos / VictoriaMetrics | 长期存储和高可用扩展方案 |

# 5. 适用场景与不适用场景

## 5.1 适用场景

- **微服务和云原生环境**：Kubernetes 原生集成，容器化场景的首选
- **时序数据分析**：基于 PromQL 的灵活聚合和预测能力
- **SRE 监控**：遵循 RED 方法（Rate/Errors/Duration）和 USE 方法（Utilization/Saturation/Errors）
- **自动化运维**：结合 Alertmanager 实现自动告警通知

## 5.2 不适用场景

- **日志监控**：Prometheus 设计为指标监控系统，不适用于日志采集和分析（应使用 ELK/Loki）
- **事件追踪**：不适用于分布式追踪（应使用 Jaeger / Skywalking）
- **高精度计费**：Prometheus 数据可能存在采样丢失，不适用于精确计费场景
- **仅需要 Push 模式**：长期运行的 Push 模型会引入单点故障和指标膨胀风险

# 6. 与其他监控系统对比

| 特性 | Prometheus | Zabbix | InfluxDB | Graphite |
|------|-----------|--------|----------|----------|
| 数据模型 | 多维标签 | 树形结构 | 标签（1.x 无标签） | 点分路径 |
| 采集方式 | Pull | Push/Pull | Push | Push |
| 查询语言 | PromQL | 表达式 | Flux | 函数式 |
| 集群化 | 联邦/Thanos | 原生 | 商业版 | 原生 |
| 云原生适配 | 原生支持 | 较弱 | 一般 | 较弱 |
| 告警 | 内置+Alertmanager | 内置 | 商业版 | 插件 |

# 7. 关键设计理念

## 7.1 为什么选择 Pull 而非 Push

| 对比项 | Pull 模型 | Push 模型 |
|--------|-----------|-----------|
| 控制权 | Server 控制采集时机，避免被目标淹没 | Agent 控制发送，Server 被动接收 |
| 健康检查 | 若目标挂掉，采集失败可立即感知 | 需要心跳机制检测 Agent 存活 |
| 安全性 | 只需 Server 可访问目标端口 | 需要 Agent 可访问 Server，暴露入口更多 |
| 配置复杂度 | 需知道所有目标地址 | Agent 只需知道 Server 地址 |

## 7.2 为什么选择本地存储

- 简化运维：单二进制启动即可运行，无需外部存储依赖
- 高性能：本地 TSDB 针对时序数据做了极致优化
- 高可用：通过 Thanos 等方案实现长期存储和高可用，而不牺牲核心的简洁性

---


# 8. PromQL 的地位

PromQL 作为 Prometheus 的查询语言，已被 CNCF 标准化（OpenMetrics 规范的一部分），成为云原生监控领域的通用查询语言，类似 SQL 在关系数据库中的地位。

# 9. OpenMetrics 标准

Prometheus 的指标暴露格式已被标准化为 OpenMetrics（CNCF 沙箱项目），这意味着任何符合 OpenMetrics 标准的应用都可以被 Prometheus 采集，反之 Prometheus 也可以采集任何符合该标准的指标端点。

# 10. 社区繁荣度

- GitHub Stars：50k+
- 贡献者：3000+
- Exporters：官方 + 社区 1000+
- 集成：几乎所有主流中间件和应用框架都原生支持或提供 Exporter
