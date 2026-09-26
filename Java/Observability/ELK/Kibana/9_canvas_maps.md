
# 1. Canvas — 演示级大屏

Canvas 是 Kibana 的**演示级可视化工具**，适合创建数据大屏和信息展示。

```
Canvas vs Dashboard:
  - Dashboard: 功能型（监控、分析）
  - Canvas: 展示型（大屏、汇报）

Canvas 元素:
  - 文本 / Markdown / 图片
  - 形状 (矩形/圆形/线条)
  - 进度条 / 指标
  - 图表 (Data Table / Area / Pie / Metric)
  - Elasticsearch SQL 数据源
```

## 1.1 创建 Canvas 工作台

```
Canvas → Create workpad
  ├── Add element → 选择图表/文本/形状
  ├── 拖拽调整位置和大小
  └── 右侧 Panel 设置样式

数据源:
  └── Elasticsearch SQL: SELECT COUNT(*), AVG(amount) FROM "orders*"
```

## 1.2 Canvas 表达式

```
# 数据表达式
filters
| essql query="SELECT category, COUNT(*) as cnt FROM \"orders*\" GROUP BY category"
| pie

# 样式表达式
filters
| essql query="SELECT SUM(amount) as total FROM \"orders*\""
| math "total"
| metric "¥{0}" 
| render containerStyle={containerStyle backgroundColor="#000"}
```

# 2. Maps — 地理可视化

Kibana Maps 支持将地理数据展示在地图上。

## 2.1 支持的地理字段

| ES 类型 | 说明 |
|---------|------|
| `geo_point` | 经纬度坐标 |
| `geo_shape` | 地理形状（多边形/区域） |

```json
# ES Mapping
{
  "location": { "type": "geo_point" },
  "delivery_area": { "type": "geo_shape" }
}
```

## 2.2 地图图层

```
Maps → Add layer:

Vector layers (矢量):
  - Documents: ES 文档标注点
  - Clusters: 聚合点簇
  - Grids: 网格热力图
  - Top Hits: 显示 Top N 文档
  - Choropleth: 区域着色（需 geo_shape）

Reference layers (参考):
  - Elastic Maps Service (自带的行政边界)
  - Web Map Service (WMS)
  - TMS/Raster Tile
```

## 2.3 地图工具

```
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

# 3. 热力图示例

```
1. Maps → Create map
2. Add layer → Grids → Clusters and grids
3. Data view: orders*
4. Geo field: location
5. Grid aggregation: avg(amount)
6. Color ramp: Green → Yellow → Red

效果: 地图上显示订单金额的热力分布
```

# 4. 区域着色（Choropleth）

```
1. Add layer → Choropleth
2. Boundaries source: Elastic Maps Service
3. Boundary: World Countries
4. Join field: country_code
5. Metric: SUM(amount)

效果: 按国家着色，显示各国订单总额
```

# 5. 保存与分享

```
Canvas:
  Save → 保存 Workpad
  Share → 公开链接 / 嵌入 iframe / PDF 下载

Maps:
  Save → 保存为 Saved Object
  Add to Dashboard → 嵌入 Dashboard
```
