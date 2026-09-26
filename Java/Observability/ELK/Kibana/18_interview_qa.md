
# 1. 基础题

## 1.1 Kibana 是什么？在 ELK 中的角色？

Kibana 是 Elastic Stack 的可视化和管理工具。在 ELK 中负责：
- **可视化**：Dashboard、Lens 图表
- **搜索**：Discover 交互式数据探索
- **管理**：Index Management、ILM、权限
- **高级**：ML、APM、SIEM

## 1.2 Kibana 和 Grafana 的区别？

| 维度 | Kibana | Grafana |
|------|--------|---------|
| 数据源 | 仅 ES | 多数据源（ES/Prometheus/MySQL等） |
| ML/APM/SIEM | 内置 | 需插件 |
| 社区 | Elastic | 独立开源 |

纯 ES 环境用 Kibana，多数据源场景用 Grafana，两者可互补。

## 1.3 Data View 是什么？

Data View（原 Index Pattern）定义 Kibana 如何识别 ES 索引中的字段，是 Discover 和 Dashboard 的数据源基础。通过通配符 `orders-*` 匹配多个索引，支持 Runtime Field 动态计算字段。

# 2. 原理题

## 2.1 Saved Objects 是什么？存储在哪里？

Saved Objects 是 Kibana 所有配置的统一持久化模型（Dashboard/Visualization/Data View 等），存储在 ES 的 `.kibana` 系统索引中。支持导出/导入、跨 Space 复制、版本迁移。

## 2.2 KQL 和 Lucene 语法的区别？

| 维度 | KQL | Lucene |
|------|-----|--------|
| 设计 | 现代、简洁 | 传统、强大 |
| 范围 | `amount > 100` | `amount:{100 TO *}` |
| 正则 | ✗ | ✓ |
| 推荐 | 新用户默认 | 高级用户 |

Kibana 默认使用 KQL，可在搜索栏切换至 Lucene。

## 2.3 Kibana 的 Spaces 隔离原理？

每个 Space 在 `.kibana` 索引中以 `space:{id}` 区分存储，不同 Space 之间的 Saved Objects 完全隔离。通过 RBAC 控制用户对 Space 的访问权限。

# 3. 实践题

## 3.1 如何创建一个监控订单量的 Dashboard？

1. 创建 Data View：`orders-*`，time field: `created_at`
2. Lens 创建 QPS 折线图：拖拽 `created_at` + Count of records
3. Lens 创建指标卡：拖拽 `SUM(amount)`
4. Lens 创建饼图：拖拽 `status.keyword`
5. Dashboard → 添加所有面板 → 调整布局
6. Controls → 添加 `category.keyword` 下拉选择器

## 3.2 如何设置错误日志告警？

1. Rule → Elasticsearch query
2. Index: `logs-*`，Query: `level:ERROR`
3. Threshold: count > 10 (last 5 min)
4. Check every: 1 min
5. Connector: Slack Webhook
6. Action: 发送告警消息

## 3.3 如何给不同团队隔离数据和 Dashboard？

1. 创建 Spaces：Team-A, Team-B
2. 各 Space 创建独立的 Data View（指向各自的索引）
3. RBAC 创建角色：team_a_viewer（仅 Team-A Space 的 read）
4. 用户分配到对应角色
5. 各团队在自己的 Space 中创建 Dashboard

# 4. 架构题

## 4.1 Kibana 如何处理高并发？

- **无状态设计**：Kibana 多实例水平扩展（LB 分发）
- **ES 连接池**：`elasticsearch.maxSockets` 控制并发
- **Task Manager**：后台任务（告警/报告）异步执行
- **Saved Objects 缓存**：减少 ES 查询

## 4.2 Fleet 和传统 Beat 有什么区别？

Fleet 是 Kibana 的统一采集管理平台，通过 Elastic Agent 一站式管理所有数据采集。传统 Beat 需要手动在每台机器上部署和配置多个独立的 Beat（Filebeat/Metricbeat 等）。Fleet 提供 UI 集中管理、一键升级、策略下发。
