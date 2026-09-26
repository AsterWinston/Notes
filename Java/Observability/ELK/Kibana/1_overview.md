
# 1. 概述

Kibana 是 Elastic Stack 的**可视化窗口**，提供对 Elasticsearch 数据的搜索、分析和图表展示能力。2013 年与 Elasticsearch 一同开源，是"ELK"中的"K"。

```
┌──────────────────────────────────────────────────────┐
│                   Kibana                              │
│                                                       │
│  ┌──────────┐ ┌──────────┐ ┌──────────┐             │
│  │ Discover │ │Dashboard │ │  Lens    │ ← 数据可视化  │
│  │ (探索)    │ │ (仪表盘)  │ │ (拖拽分析)│             │
│  └──────────┘ └──────────┘ └──────────┘             │
│                                                       │
│  ┌──────────┐ ┌──────────┐ ┌──────────┐             │
│  │ Alerting │ │    ML    │ │   APM    │ ← 高级功能   │
│  │ (告警)    │ │ (机器学习)│ │ (性能监控)│             │
│  └──────────┘ └──────────┘ └──────────┘             │
│                                                       │
│  ┌──────────┐ ┌──────────┐ ┌──────────┐             │
│  │  Spaces  │ │  Fleet   │ │ Security │ ← 管理       │
│  │ (多租户)  │ │ (采集管理)│ │  (SIEM)  │             │
│  └──────────┘ └──────────┘ └──────────┘             │
└──────────────────────────────────────────────────────┘
```

# 2. 核心能力

| 能力 | 说明 | 工具 |
|------|------|------|
| **数据探索** | 交互式搜索和过滤 ES 数据 | Discover |
| **可视化** | 图表展示（柱状/折线/饼图/地图） | Lens / Visualize |
| **仪表盘** | 多面板组合的实时监控大屏 | Dashboard |
| **告警** | 基于查询结果的规则告警 | Alerting |
| **机器学习** | 异常检测、分类、回归 | Machine Learning |
| **安全分析** | SIEM 安全信息事件管理 | Security |
| **APM** | 应用性能监控 UI | APM / Observability |
| **管理** | 索引管理、ILM、权限控制 | Stack Management |

# 3. Kibana vs Grafana

| 维度 | Kibana | Grafana |
|------|--------|---------|
| **数据源** | 仅 Elasticsearch | ES/Prometheus/MySQL/InfluxDB 等 |
| **可视化** | Lens 拖拽、Canvas 大屏 | 面板丰富 |
| **告警** | 内置（ES Query/Index Threshold） | 内置（多数据源） |
| **学习曲线** | 低（与 ES 深度绑定） | 中 |
| **ML/APM/SIEM** | 内置 | 需额外插件 |
| **开源** | ✓（Elastic License） | ✓（AGPL） |

**结论**：仅使用 ES → Kibana；多数据源 → Grafana；两者可互补（Grafana 连 ES，Kibana 做 ML/APM）。

# 4. Kibana 8.x 新特性

| 特性 | 说明 |
|------|------|
| **Lens** | 拖拽式可视化，替代原 Visualize |
| **Dashboard 增强** | Controls 面板过滤器、Drilldown 下钻 |
| **Alerting 框架** | 统一 Rule + Connector 模型 |
| **Fleet** | 统一采集管理（Elastic Agent） |
| **Security SIEM** | 检测规则、Timeline、Cases |
| **Observability** | APM + Infrastructure + Uptime 统一面板 |

# 5. 核心设计理念

1. **数据驱动**：所有功能围绕 Elasticsearch 数据索引展开
2. **Saved Objects**：一切配置（Dashboard/Visualize/Index Pattern）都是可导出的 Saved Object
3. **Spaces 隔离**：多租户空间隔离，不同团队独立工作
4. **KQL 查询**：自研查询语言，降低 ES Query DSL 学习成本
5. **插件化**：功能模块可按需开启（ML/APM/SIEM 等）

# 6. 适用场景

```
✓ 适用：
  - ES 数据的可视化探索和搜索
  - 实时业务监控大屏
  - 日志集中检索分析
  - APM 应用性能监控
  - 安全事件分析（SIEM）
  - 多团队数据隔离（Spaces）

✗ 不适用：
  - 需要连接多个异构数据源（用 Grafana）
  - 独立的 BI 报表（用 Metabase/Superset）
```
