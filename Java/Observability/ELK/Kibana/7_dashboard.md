
# 1. 概述

Dashboard 是 Kibana 的**可视化组合面板**，将多个图表、表格、指标组合成一个实时监控视图。

```
┌──────────────────────────────────────────────────────┐
│  Dashboard: Order Monitoring                         │
│  ┌───────────────────────┐ ┌──────────────────────┐  │
│  │  Orders QPS (折线图)   │ │ Revenue (指标)       │  │
│  │     ╱╲   ╱╲    ╱╲     │ │    ¥1,253,680       │  │
│  │    ╱  ╲ ╱  ╲  ╱  ╲    │ │         ▲ 12%        │  │
│  │   ╱    ╳    ╳    ╳    │ │                      │  │
│  │  ╱     ╲    ╲    ╲   │ └──────────────────────┘  │
│  └───────────────────────┘                           │
│  ┌───────────────────────┐ ┌──────────────────────┐  │
│  │ Status Pie (饼图)      │ │ Top Products (表格)   │  │
│  │  60% active            │ │ iPhone   ¥500,000    │  │
│  │  25% pending           │ │ MacBook  ¥300,000    │  │
│  │  15% cancelled         │ │ iPad     ¥200,000    │  │
│  └───────────────────────┘ └──────────────────────┘  │
└──────────────────────────────────────────────────────┘
```

# 2. 创建 Dashboard

```
方式 1: 从 Discover/Lens 直接保存到 Dashboard
  Discover → 搜索 → Save → "Add to new Dashboard"

方式 2: 新建空 Dashboard
  Dashboard → Create dashboard → Add panel
    → 从 Visualize Library 选择已有图表
    → 或创建新的 Lens 图表
```

# 3. 面板操作

```
面板操作按钮 (面板右上角 ...):
  ├── Edit visualization     → 编辑 Lens 图表
  ├── Inspect               → 查看数据/请求/响应
  ├── Maximize              → 全屏展开
  ├── Copy to dashboard     → 复制到其他 Dashboard
  ├── Replace panel         → 替换为其他图表
  └── Remove from dashboard → 删除面板

拖拽: 面板右下角 → 调整大小
拖拽: 面板标题栏 → 移动位置
```

# 4. Controls 面板过滤器

```
Dashboard → Create panel → Controls

Options list (下拉选择):
  Field: status.keyword
  Label: Order Status
  效果: Dashboard 顶部出现下拉框 → 选择值 → 所有面板联动过滤

Range slider (范围滑块):
  Field: amount
  Step: 10
  效果: 滑块选择金额范围 → 所有面板联动

Time slider (时间滑块):
  Field: @timestamp
  效果: 时间轴拖拽 → 所有面板时间联动
```

# 5. Drilldown 下钻分析

```
设置 Drilldown:
  Lens 编辑 → 保存 → Drilldown → Create
    Type: Go to URL / Go to Dashboard
    传递当前上下文（filters, time range）

使用 Drilldown:
  点击图表中的数据点 → 跳转到目标 Dashboard
  例如: 点击 "cancelled" 饼图块 → 跳转到 "Cancelled Orders" Dashboard
```

# 6. Dashboard 管理

```
Dashboard 右上角:
  └── Share → 分享链接 / CSV 导出 / 嵌入 iframe

Dashboard → Edit:
  └── Controls 设置（全局过滤器）

Stack Management → Saved Objects → Dashboard:
  ├── Export → 导出为 NDJSON
  ├── Import → 导入 Dashboard
  └── Copy → 复制到其他 Space
```

# 7. 导出/导入

```bash
# 导出 Dashboard (包含依赖的 Lens/Search)
GET /api/saved_objects/_export
{
  "objects": [{ "type": "dashboard", "id": "my-dashboard" }],
  "includeReferencesDeep": true
}

# 导入
POST /api/saved_objects/_import
# 上传 NDJSON 文件
```

# 8. 最佳实践

1. **面板分组**：使用 Controls 面板统一过滤，而非每个面板单独加 filter
2. **命名规范**：Dashboard 命名 `[Team]-[Purpose]-[Env]`，如 `Order-Monitoring-Prod`
3. **时间范围**：Dashboard 设置合理的时间默认值（Last 15m / Last 24h）
4. **Markdown 面板**：用 Markdown 面板添加说明文字、链接、公告
5. **减少面板数**：一个 Dashboard 不超过 10-15 个面板（加载性能）
6. **避免实时刷新过频**：生产 Dashboard 刷新间隔 ≥ 30s
