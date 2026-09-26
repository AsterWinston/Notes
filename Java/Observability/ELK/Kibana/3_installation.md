
# 1. Docker 部署

```bash
# Kibana 8.x
docker run -d \
  --name kibana \
  -p 5601:5601 \
  -e ELASTICSEARCH_HOSTS=http://es:9200 \
  -e ELASTICSEARCH_USERNAME=kibana_system \
  -e ELASTICSEARCH_PASSWORD=${KIBANA_PASSWORD} \
  docker.elastic.co/kibana/kibana:8.11.0

# 验证
curl http://localhost:5601/api/status
# {"status": {"overall": {"level": "available"}}}
```

# 2. Linux 安装

```bash
# CentOS/RHEL
rpm --import https://artifacts.elastic.co/GPG-KEY-elasticsearch

cat > /etc/yum.repos.d/kibana.repo << EOF
[kibana-8.x]
name=Kibana repository for 8.x packages
baseurl=https://artifacts.elastic.co/packages/8.x/yum
gpgcheck=1
gpgkey=https://artifacts.elastic.co/GPG-KEY-elasticsearch
enabled=1
EOF

yum install -y kibana
systemctl start kibana
systemctl enable kibana
```

# 3. 导入示例数据

```
Kibana Home → Try sample data
  ├── Sample eCommerce orders   → 电商订单数据
  ├── Sample flight data        → 航班数据
  └── Sample web logs           → Web 访问日志

点击 "Add data" → 自动创建索引 + Dashboard + Visualizations
```

# 4. 第一个 Dashboard

```
1. 创建 Data View
   Stack Management → Data Views → Create
   Name: orders
   Index pattern: kibana_sample_data_ecommerce
   Time field: order_date

2. 进入 Discover
   选择 orders Data View
   搜索: customer_gender:FEMALE

3. 创建 Lens 图表
   拖拽 category.keyword → 柱状图
   拖拽 taxful_total_price → 求平均值作为 Y 轴

4. 保存到 Dashboard
   新建 Dashboard → 添加 Lens 面板 → 调整布局
```

# 5. Spaces 多空间

```
Stack Management → Spaces → Create Space
  Name: Team A
  Features: Discover, Dashboard, Visualize
  Permissions: 仅 Team A 成员可访问

创建 Space 后：
  - Discover 仅显示该 Space 的 Data View
  - Dashboard 仅显示该 Space 的仪表盘
  - 切换 Space 顶栏下拉菜单
```

# 6. 核心 UI 导航

```
┌─────────────────────────────────────┐
│  Main Menu                          │
│  ├── Discover          (数据探索)    │
│  ├── Dashboards        (仪表盘)     │
│  ├── Visualize Library (可视化库)   │
│  ├── Maps              (地图)       │
│  └── Stack Management  (管理)       │
│                                     │
│  Observability                       │
│  ├── Overview                        │
│  ├── APM                             │
│  ├── Infrastructure                  │
│  └── Uptime                          │
│                                     │
│  Security                            │
│  ├── Alerts                          │
│  └── Cases                           │
│                                     │
│  Analytics                           │
│  ├── Machine Learning               │
│  ├── Graph                          │
│  └── Canvas                         │
└─────────────────────────────────────┘
```

# 7. Dev Tools 控制台

```json
# Dev Tools → Console（类似 ES REST 客户端）
GET /_cat/indices?v
GET /orders/_search
PUT /test/_doc/1
{ "name": "test" }
```

# 8. 常用配置速查

```yaml
server.port: 5601
server.host: "0.0.0.0"
elasticsearch.hosts: ["http://es:9200"]
kibana.defaultAppId: "discover"
logging.root.level: info
```
