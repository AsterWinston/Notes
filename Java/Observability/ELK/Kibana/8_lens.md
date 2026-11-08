# 1. 概述

Lens 是 Kibana 8.x 的**新一代可视化构建器**，支持拖拽字段即生成图表，取代传统的 Visualize 编辑器。它是声明式理念在 BI 前端的落地：用户只描述"用什么数据、怎么聚合、怎么摆"（一份可序列化的 State JSON），具体的 ES 聚合查询、时间分桶、分页重试全部由 Lens 引擎生成并持续同步——与 Grafana 面板 JSON、Superset 的 Query Object 是同一思路。

对比三代编辑器的形态差异：

```text
传统 Visualize: 选择图表类型 → 配置聚合 → （必要时）调 JSON
Lens:           拖拽字段 → 自动推荐图表 → 一键生成
```

声明式带来的最大红利是**编辑即预览、改动即查询**：每次拖拽/换函数，Lens 立刻重发一条由当前 State 编译的搜索请求并把结果重绘，反馈周期从"改配置→Apply→等渲染"压缩到一次拖拽。同时 Lens State 是纯数据对象，可进 Saved Object 导出、可被 Dashboard 引用、可被 API 复制——旧 Visualization（尤其基于 timeseries/TSVB 的）配置晦涩、迁移困难的问题在源头被解决。8.x 中 Lens 已覆盖柱线面饼表指标仪表热力地图全部常用图型，官方立场明确：**新图表一律用 Lens**，Visualize 编辑器仅剩 Vega 等少数场景。

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

图表切换零成本是 Lens 的特性之一：底部图表类型条随时可切，State 不变、仅渲染层变化，"这组数据用折线还是柱状更好"从判断题变成看图比较题。注意各图型对字段的"轴要求"不同（Metric 只要数值、Table 接受任意、Map 需要 geo 字段），切换时 Lens 会提示缺失的必需轴配置。

# 3. 数据源与轴模型

```text
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

Lens 的数据模型围绕三种"轴角色"组织，理解它们是配好图表的前提：

- **Category（分类轴）**：keyword、date、number 皆可作为分组维度。date 类型自动变为 date_histogram（粒度可改自动），keyword 类型编译为 terms 聚合（按计数或指标排序）。高基数 keyword 放分类轴会得到一张"全是碎条"的图——排序方式改为按指标降序 + Limit 截断（默认 10，最大可配 100）是标准处理。
- **Metric（指标轴）**：数值聚合，默认 `Count of records`，可对数值字段换 Average/Sum/Min/Max/Median/Percentile，也可用 Formula（见 4 节）。一个图表可放多个指标轴，形成多序列或分组柱。
- **Break down by / Split（拆分）**：再加一层 keyword 维度，把单一聚合变成按值分组的子聚合（折线多序列、堆叠柱的分段、饼图的扇区来源）。

> **设计考量**：Lens 把"聚合树"隐藏在拖拽交互后面，编译产物仍是标准 ES 聚合（date_histogram 包 terms 包 avg）。多拆一层高基数 terms，聚合内存与协调成本指数上涨——Lens 界面里"Break down by 建议 ≤ 2 层、每层基数 ≤ 千级"不是性能优化建议而是能不能出图的边界。

# 4. 函数与公式

Lens 支持对字段使用函数：

```text
聚合函数（Metric 配置面板内）:
  - Average / Sum / Min / Max / Median
  - Count / Distinct Count
  - Percentile (95th / 99th)

链式函数（结果之上再计算）:
  - Cumulative Sum（累计值）
  - Moving Average（移动平均）
  - Difference / Differences from average
```

Distinct Count 的口径要警惕：它编译为 ES 的 **cardinality 聚合（HyperLogLog++）**，是近似值，默认 `precision_threshold=3000` 内基本准确，基数很大时误差可达百分之几——用它做"精确用户数"报表前先接受这一点，或降基数分桶。链式函数作用在时间桶序列之上（Moving Average 对每个桶计算前 N 桶均值），做趋势平滑、环比差值非常实用。

自定义公式（Formula）是 Lens 相对旧编辑器的杀手锏，用类 Excel 语法组合多个聚合：

```text
Formula: count() / overall_sum(count())
效果: 当前桶计数占全时间范围总计数之比（每个扇区/柱条的"份额"）

Formula: average(amount) * count()
效果: 总金额 = 均价 × 数量

Formula: sum(cost) / sum(revenue)
效果: 成本率（比先算每篇比值再平均更正确——加权而非简单平均）
```

公式里的 `overall_*` 函数族（`overall_sum`/`overall_average`/`overall_min`/`overall_max`）聚合的是**整个查询范围的总计**，不带时间/分组上下文——分母用它才能算占比。聚合顺序决定业务含义：`sum(x)/sum(y)` 是加权比率，`average(x/y)` 需要脚本逐文档除法（Lens 不支持，这也是"为什么我用 Lens 算不出某指标"的正确答案）。公式支持多变量命名（如 `A=sum(price); B=sum(qty); A/B`），复杂指标建议写清变量名，Saved Object 里可读性完全不同。

# 5. 多层分组与排序

```text
分组 1: category.keyword (柱状图分组)
  └── 分组 2: status.keyword (堆叠/颜色区分)
        └── 分组 3: Subtype (子类别，仅表格)
```

多层分组编译为嵌套子聚合，只有 Table 类型支持第三层（子表行内展开）。排序控制两个维度：**行的排序**（按计数或某个指标列，terms 聚合的 `order`）与**分组内的次序**（堆叠图的 stack 顺序）。"Top 10 类别 + Others"是 Lens 自动处理的：超过 Limit 的 terms 值被折叠进 Others 桶（可显式开关），饼图默认只展示 Top N 正是防止几十个小扇区糊成色环。

# 6. 可视化建议

Lens 根据拖拽的字段类型**自动推荐**图表：

```text
拖入 1 个 keyword 字段 → 推荐: 柱状图、饼图
拖入 1 个 date + 1 个 number → 推荐: 折线图、面积图
拖入 2 个 number → 推荐: 散点图
拖入 geo_point → 推荐: 地图
```

推荐逻辑基于"字段类型组合 → 图型约束求解"：date 在分类轴优先时序图型、两个数值维度触发散点、geo 字段只有地图可用。自动推荐解决"用什么图"，选图的经验则解决"该问什么问题"：占比用堆叠/饼但分类数 ≤ 6 才有效；趋势用折线，面积图只在讲"量的累积"时用（堆叠面积易夸大视觉占比）；比较大小用柱状（基线对齐），横向条形适合长类目名；单指标盯盘用 Metric，加 compare overall 做环比箭头。

# 7. 样式配置

```text
右侧 Panel → Appearance:
  - Color: 配色方案（单色/渐变/分类）
  - Labels: 显示/隐藏数据标签
  - Legend: 图例位置和样式
  - Y axis: 刻度、范围、单位格式
  - Tooltip: 悬停提示信息
```

样式层三个细节：Color 可按维度值固定映射（status=cancelled 永远红），避免同一颜色在不同面板指代不同值——这是大屏可读性第一规则；Y axis 默认从 0 开始（柱状必须，折线可手动截断但要警惕视觉夸大）；Unit/format 与 Data View 的字段格式化联动（bytes 字段自动带单位）。样式改动同属 State，一样可导出复现。

# 8. Lens 家族与旧方案对比

| 维度 | Lens | Visualize（旧） | TSVB | Vega/Vega-Lite |
|------|------|----------------|------|----------------|
| 操作方式 | 拖拽 | 聚合下拉配置 | 指标+面板配置 | 写 JSON 规格 |
| 图表建议 | ✓ 智能推荐 | ✗ | ✗ | ✗ |
| 公式 | ✓ 内置 | 部分（scripted field） | ✓ formula | ✓ transform |
| 多数据源 | ✓（8.8+，限同源） | ✗ | ✓ | ✓（需数据源配置） |
| 学习曲线 | 低 | 中 | 高 | 高 |
| 定位 | 8.x 主力 | 遗留编辑器 | 遗留（可迁 Lens） | 高度定制仍推荐 |
| 状态 | 活跃演进 | 逐步弃用 | 逐步弃用 | 活跃维护 |

TSVB（Time Series Visual Builder）与 Region Map、Coordinate Map、Tile Map 等专用编辑器都已被 Lens 对应图型吸收，存量图表 8.x 仍可渲染但"Edit"会建议另存为 Lens；迁移成本主要是把 `series.formula` 翻译到 Lens Formula。Vega 不在"被替代"之列——它面向完全自定义的视觉语法（桑基图、日历图、自定义交互），Lens 不做这类表达，两者定位互补。

# 9. 小结

Lens 是声明式可视化构建器：拖字段定 Category/Metric/Split 三角色，State JSON 自动编译为 date_histogram+terms+avg 等 ES 聚合树，编辑即预览。函数层记住三点——Distinct Count 是 HLL 近似、链式函数作用在时间桶上、Formula 用 `overall_*` 算占比、比率一律"先 sum 后除"。图型选择上自动推荐够用，经验补充"分类 ≤ 6 用饼、长名用条、盯盘用 Metric"。家族层面：新图全部 Lens，Vega 留给深度定制，旧 Visualize/TSVB/地图编辑器按节奏迁移。性能红线只有一条——控制每层分组基数与桶数，它直接决定这条聚合查询能不能在 Dashboard 的超时预算内跑完。
