
# 1. 概述

Lens 是 Kibana 8.x 的**新一代可视化引擎**，支持拖拽字段即可生成图表，替代传统的 Visualize 功能。

```
传统 Visualize: 选择图表类型 → 配置聚合 → 写 JSON
Lens: 拖拽字段 → 自动推荐图表 → 一键生成
```

# 2. 支持的图表类型

| 类别 | 图表 | 适用场景 |
|------|------|---------|
| **柱状/条形** | 垂直柱状、水平条形、堆叠柱状 | 分类对比 |
| **折线/面积** | 折线图、面积图 | 时间趋势 |
| **饼图/环形** | Pie、Donut、Treemap | 占比分布 |
| **表格** | 数据表格 | 明细列表 |
| **指标** | 单值指标（带趋势对比） | KPI 展示 |
| **热力图** | Heatmap | 密度分布 |
| **仪表** | Gauge、Goal | 目标完成度 |
| **地图** | 区域地图、坐标地图 | 地理数据 |

# 3. 基础操作

```
创建 Lens:
  Visualize Library → Create visualization → Lens
  → 选择 Data View

拖拽字段:
  - 从左侧字段列表拖拽到中央配置区
  - 字段类型自动匹配推荐的图表类型

字段放置区:
  Horizontal axis (X 轴) → category.keyword
  Vertical axis (Y 轴) → COUNT of records / AVG(amount)
  Break down by (分组) → status.keyword
```

# 4. 函数与公式

```
Lens 支持对字段使用函数:
  - Average / Sum / Min / Max / Median
  - Count / Distinct Count
  - Percentile (95th / 99th)
  - Cumulative Sum（累计值）
  - Moving Average（移动平均）
  - Difference / Differences from average

自定义公式:
  Formula: count() / overall_sum(count())
  效果: 百分比计算

  Formula: average(amount) * count()
  效果: 总金额 = 均价 × 数量
```

# 5. 多层分组

```
分组 1: category.keyword (柱状图分组)
  └── 分组 2: status.keyword (堆叠/颜色区分)
        └── 分组 3: Subtype (子类别，仅表格)
```

# 6. 可视化建议

Lens 根据拖拽的字段类型**自动推荐**图表：

```
拖入 1 个 keyword 字段 → 推荐: 柱状图、饼图
拖入 1 个 date + 1 个 number → 推荐: 折线图、面积图
拖入 2 个 number → 推荐: 散点图
拖入 geo_point → 推荐: 地图
```

# 7. 样式配置

```
右侧 Panel → Appearance:
  - Color: 配色方案（单色/渐变/分类）
  - Labels: 显示/隐藏数据标签
  - Legend: 图例位置和样式
  - Y axis: 刻度、范围、单位格式
  - Tooltip: 悬停提示信息
```

# 8. 与 Visualize 对比

| 维度 | Lens | Visualize |
|------|------|-----------|
| 操作方式 | 拖拽 | 选择聚合 + 配置 |
| 图表建议 | ✓ 智能推荐 | ✗ |
| 公式 | ✓ 内置 | ✗ |
| 学习曲线 | 低 | 中 |
| 版本 | 8.x 主力 | 逐渐弃用 |
| 自定义程度 | 高 | 高（含 Vega） |
