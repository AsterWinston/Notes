
# 1. 环境要求

| 组件 | 要求 |
|------|------|
| JDK | ES 7.x 内置 JDK，8.x 需要 JDK 17+ |
| OS | Linux/macOS/Windows (WSL2) |
| 内存 | 最小 2GB，生产 16GB+ |
| 文件描述符 | >= 65536 |
| vm.max_map_count | >= 262144 |

# 2. Docker 部署（推荐）

```bash
# 单节点
docker run -d \
  --name es01 \
  -p 9200:9200 -p 9300:9300 \
  -e "discovery.type=single-node" \
  -e "xpack.security.enabled=false" \
  -e "ES_JAVA_OPTS=-Xms512m -Xmx512m" \
  docker.elastic.co/elasticsearch/elasticsearch:8.11.0

# 验证
curl http://localhost:9200
# {
#   "name": "es01",
#   "cluster_name": "docker-cluster",
#   "version": { "number": "8.11.0" }
# }
```

# 3. Kibana Dev Tools

```bash
# 安装 Kibana
docker run -d \
  --name kibana \
  -p 5601:5601 \
  -e ELASTICSEARCH_HOSTS=http://es01:9200 \
  docker.elastic.co/kibana/kibana:8.11.0

# 访问 http://localhost:5601 → Dev Tools
```

# 4. 第一个索引与文档

```json
# 创建索引
PUT /products
{
  "settings": { "number_of_shards": 1, "number_of_replicas": 0 },
  "mappings": {
    "properties": {
      "title": { "type": "text", "analyzer": "standard" },
      "price": { "type": "double" },
      "category": { "type": "keyword" },
      "created_at": { "type": "date" }
    }
  }
}

# 插入文档
POST /products/_doc/1
{
  "title": "iPhone 15 Pro",
  "price": 8999.00,
  "category": "电子产品",
  "created_at": "2024-01-15T10:00:00"
}

POST /products/_doc/2
{
  "title": "MacBook Pro 16",
  "price": 19999.00,
  "category": "电子产品",
  "created_at": "2024-01-16T10:00:00"
}

# 全文搜索
GET /products/_search
{
  "query": {
    "match": { "title": "iPhone" }
  }
}

# 聚合统计
GET /products/_search
{
  "size": 0,
  "aggs": {
    "avg_price": { "avg": { "field": "price" } },
    "by_category": {
      "terms": { "field": "category" },
      "aggs": { "avg_price": { "avg": { "field": "price" } } }
    }
  }
}
```

# 5. 系统参数调优

```bash
# /etc/sysctl.conf
vm.max_map_count = 262144
vm.swappiness = 1

sysctl -p

# /etc/security/limits.conf
elasticsearch  -  nofile  65536
elasticsearch  -  nproc   4096
```

# 6. elasticsearch.yml 核心配置

```yaml
# 集群配置
cluster.name: my-cluster
node.name: node-1

# 数据路径
path.data: /var/lib/elasticsearch
path.logs: /var/log/elasticsearch

# 网络
network.host: 0.0.0.0
http.port: 9200
transport.port: 9300

# 集群发现（8.x）
discovery.seed_hosts: ["node1:9300", "node2:9300", "node3:9300"]
cluster.initial_master_nodes: ["node-1", "node-2", "node-3"]

# 安全（8.x 默认开启）
xpack.security.enabled: true
xpack.security.enrollment.enabled: true
xpack.security.http.ssl.enabled: true
xpack.security.transport.ssl.enabled: true
```

# 7. 常用系统 API

```bash
# 集群健康
GET /_cluster/health

# 节点信息
GET /_cat/nodes?v

# 索引列表
GET /_cat/indices?v

# 分片分配
GET /_cat/shards?v

# 集群状态
GET /_cluster/stats?human

# 任务管理
GET /_cat/tasks?v
```
