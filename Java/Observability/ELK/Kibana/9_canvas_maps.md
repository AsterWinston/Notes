# 1. Canvas：演示级大屏

Canvas 是 Kibana 的**演示级可视化工具**，适合创建数据大屏和信息展示。它与 Dashboard 的分界不在"能不能画图"，而在**面向对象与排版自由度**：Dashboard 是"操作型"界面——面板栅格布局、过滤器联动、值班同学随时下钻；Canvas 是"表达型"界面——像素级自由排版、品牌配色、静态或定时刷新，面向汇报、作战室投屏、领导看板。一句话：**天天用、要交互 → Dashboard；给观众看、要好看 → Canvas**。

```text
Canvas vs Dashboard:
  - Dashboard: 操作型（监控、分析、下钻交互）
  - Canvas:    展示型（大屏、汇报、品牌视觉）

Canvas 元素:
  - 文本 / Markdown / 图片
  - 形状 (矩形/圆形/线条)
  - 进度条 / 指标
  - 图表 (Data Table / Area / Bar / Pie / Metric / Tagfield)
  - Elasticsearch SQL 数据源
```

Canvas 的 Workpad（作品）由多页（Page）组成，每页是无限画布，元素任意拖拽、对齐、分层；支持全局筛选器（Filter 元素可暴露给查看者）、页面自动翻页（投屏模式）、以及把整个 Workpad 导出 PDF/图片。Basic License 即可使用（部分高级元素需付费层级），是零成本做"数据汇报页"最顺手的工具。

## 1.1 创建 Workpad

```text
Canvas → Create workpad
  ├── Add element → 选择图表/文本/形状
  ├── 拖拽调整位置和大小
  └── 右侧 Panel 设置样式

数据源:
  └── Elasticsearch SQL: SELECT COUNT(*), AVG(amount) FROM "orders*"
```

元素的数据绑定走 **Elasticsearch SQL**（或 ES|QL，新版本），这是 Canvas 与 Lens 的本质差异：Lens 用拖拽聚合，Canvas 用 SQL 结果集。好处是熟悉 SQL 的分析师可以直接 `GROUP BY` 出任何形状的数据；代价是每条 SQL 都是一次完整查询，复杂报表要自己控制查询数量。

## 1.2 Canvas 表达式语言

Canvas 元素之间用一条**管道式表达式链**把"数据获取 → 变换 → 渲染"串起来，语法类似 shell pipe，每一步是内置函数：

```text
# 数据表达式：ES|QL/essql 取数后接 pie 渲染
filters
| essql query="SELECT category, COUNT(*) as cnt FROM \"orders*\" GROUP BY category"
| pie

# 样式表达式：取标量后格式化为指标卡
filters
| essql query="SELECT SUM(amount) as total FROM \"orders*\""
| math "total"
| metric "¥{0}"
| render containerStyle={containerStyle backgroundColor="#000"}
```

常用函数按用途分组：数据类 `essql`/`.es|ql`、`math`（列间运算）、`groupBy`/`pivot`/`sort`/`limit`（结果整形）；渲染类 `metric`、`bar`、`line`、`pie`、`table`、`tagcloud`、`timeSeries`；样式类 `font`、`color`、`border`、`containerStyle`、`tooltip`。表达式链的好处是**可版本化**——Workpad 导出即一段段文本表达式，review 一个数字卡片"口径是什么"直接读链条；复杂变换（如透视）在链里用 `pivot` 比在 SQL 里写更直观。

> **注意**：`filters` 作为链首是约定，代表应用当前 Workpad 级筛选器；漏写会导致该元素不受全局 Filter 元素控制，汇报现场"点了筛选怎么这个卡片不动"多半是这个原因。SQL 里的索引名要带引号转义（`\"orders*\"`），且默认只查默认时间范围（Last 15 minutes）——大屏数字"小得离谱"先检查时间窗而不是数据。

# 2. Maps：地理可视化

Kibana Maps 把 ES 里的地理字段渲染到交互地图上。地图不只是"好看"，地理维度是日志/业务数据里天然的强分析轴：攻击源分布在哪个国家、配送热力集中在哪片城区、门店业绩分区域对比——这些用表格看五十行不如地图一眼。

## 2.1 地理字段类型与数据要求

| ES 类型 | 说明 | Maps 图层支持 |
|---------|------|--------------|
| `geo_point` | 经纬度坐标（`lat,lon` / geohash / WKT 点） | 点标注、簇、网格热力、Top hits |
| `geo_shape` | 地理形状（多边形/线/集合） | 区域图层、Choropleth 着色 |

```json
# ES Mapping（正确示例）
{
  "properties": {
    "location":       { "type": "geo_point" },
    "delivery_area":  { "type": "geo_shape" }
  }
}
```

数据要求是排障高频项：**`lat`/`lon` 两个独立的 double 字段不是 geo_point**——动态推断成 `double` 后 Maps 看不到该字段；正确写法是 mapping 显式声明 `geo_point`，值可以是 `{"lat":39.9,"lon":116.4}`、`"39.9,116.4"`、geohash 字符串或 WKT `POINT(116.4 39.9)`。经纬度顺序在不同表示法里不一致（对象写法 lat 在前，数组/字符串是 `lat,lon`，WKT 是 `x y` 即 lon 在前），写入端最容易反的就是这个——地图上点全落在几内亚湾（0,0）附近，基本就是经纬度写反或漏声明类型。

## 2.2 图层模型

```text
Maps → Add layer:

Vector layers (矢量):
  - Documents: ES 文档逐点标注（默认类型）
  - Clusters: 聚合点簇（缩放展开）
  - Grids: 网格热力图
  - Top Hits: 每网格显示 Top N 文档
  - Choropleth: 区域着色（需 geo_shape 或边界连接）

Reference layers (参考):
  - Elastic Maps Service (自带的行政边界/底图)
  - Web Map Service (WMS)
  - TMS/Raster Tile
```

图层分两类：**数据图层**（Documents/Clusters/Grids/Choropleth 等，渲染 ES 文档）与**参考图层**（底图、行政边界，提供上下文背景）。Choropleth（分级统计图）是业务大屏最常用图层：用 Elastic Maps Service 的世界/国家/省级边界做区域形状，再把业务指标按"连接字段"挂到区域上着色——关键点是**连接字段类型要匹配**：ES 里存 ISO 国家码（`CN`/`US`）就选 `iso3166-1-alpha-2` 连接层，存中文名连不上（连接字段两侧需为同一编码体系），这是"区域着色全灰"的首要原因。内网隔离环境拿不到 Elastic Maps Service 时，需配置自定义 Mapbox token 或内网瓦片服务（`maps.*` 系列设置）。

## 2.3 地图工具与样式

```text
地图工具（左侧工具栏）：
  ├── 框选过滤 → 选择地图区域 → 过滤文档
  ├── 距离测量 → 测量两点距离
  ├── 绘制形状 → 自定义标注
  └── 缩放/平移

图层样式：
  - 点颜色/大小按字段值映射
  - 热力图颜色渐变
  - Tooltip 自定义显示字段
```

框选（box select）会把屏幕区域编译成 geo bounding box / geo_shape 过滤，联动 Dashboard 全局 filters——"圈一块区域，看该区域订单明细表"是地图作为**过滤器 UI** 的正确用法，比 KQL 手写 geo_bbox 自然得多。点样式三档：按类别上色（status）、按数值映射大小/颜色（amount）、或统一样式；网格聚合并用 color ramp（绿→黄→红）是热力表达标配。

## 2.4 典型场景

**物流配送密度**：Documents → Grids 图层，geo field 取订单收货点 `location`，Metric 用 `AVG(taxful_total_price)`，color ramp 暖色系 → 一眼看出高价值订单密集区，叠加 Clusters 图层显示门店位置，两张图层上下叠看即"供给-需求"匹配分析。

**攻击源分布**（Security 团队高频用法）：`source.ip` 经 Ingest GeoIP 处理器富化为 `source.geo.coordinates`（geo_point），Documents 图层按国家着色计数，Top Hits 图层看单国攻击明细，框选某国 + Dashboard filter 联动深挖。Security 的 Dashboards 应用里预置的地理面板就是基于 Maps 图层体系。

# 3. 热力图示例

```text
1. Maps → Create map
2. Add layer → Grids → Clusters and grids
3. Data view: orders*
4. Geo field: location
5. Grid aggregation: avg(amount)
6. Color ramp: Green → Yellow → Red

效果: 地图上显示订单金额的热力分布
```

调优两个旋钮：`Precision`（geohash 精度等级，越高网格越细、聚合桶越多查询越慢——全国范围 5-6 级、城市级 8-10 级经验值）与 `Type`（`Clustered grid` 适合热点概览，`Scaled points` 让点大小随值变化更适合"少而精"的点位场景）。

# 4. 区域着色（Choropleth）

```text
1. Add layer → Choropleth
2. Boundaries source: Elastic Maps Service
3. Boundary: World Countries
4. Join field: country_code
5. Metric: SUM(amount)
6. Color mode: Continuous / Discrete 分档

效果: 按国家着色，显示各国订单总额
```

Join field 取自 Data View 字段列表，类型须为 keyword 且存标准码值；Metric 支持 Count/Sum/Average/Max/Min；配色分 `Continuous`（渐变）与 `Discrete`（分档），指标长尾分布（少数区域占大头）时选 Quantiles 分档或改对数刻度，避免"只有一块红其余全灰"。

# 5. 保存与分享

```text
Canvas:
  Save → 保存 Workpad
  Share → 公开链接 / 嵌入 iframe / PDF 下载

Maps:
  Save → 保存为 Saved Object
  Add to Dashboard → 嵌入 Dashboard
```

Maps 与 Lens 图表面板一样是 Saved Object，可进 Dashboard、可导出 NDJSON 迁移；Canvas Workpad 是独立的 Saved Object 类型，Share 的链接支持有效期/公开设置（依赖 `server.publicBaseUrl` 与匿名访问策略，内网大屏注意鉴权配置，别为投屏开全域匿名）。

> **注意**：Canvas、Maps、Lens 三者的"分享"语义完全不同——Canvas 面向"展示"（导出 PDF、自动翻页、公开链接）、Maps/Lens 面向"分析"（嵌入 Dashboard、带 filter 联动）。选型时先问受众是"观众"还是"操作员"，避免把给操作员的交互地图做成静态大屏截图。

# 6. 小结

Canvas 与 Maps 补齐 Kibana 的两端表达：**Canvas 是"汇报大屏"**——自由排版 + Elastic SQL 数据源 + 管道式表达式链（`filters | essql | math | metric | render`），面向观众不面向操作；**Maps 是"地理分析"**——geo_point 出点与热力网格、geo_shape/边界连接出 Choropleth，框选即过滤、与 Dashboard 天然联动。数据侧记住两个坑：经纬度必须 mapping 成 geo_point 且注意 lat/lon 顺序，Choropleth 的连接字段必须是标准 ISO 编码。选型判断：要交互下钻回 Dashboard + Lens，要静态投屏好看上 Canvas，要空间视角上 Maps。
