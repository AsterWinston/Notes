
# 1. RBAC 安全体系

```yaml
# 启用安全（8.x 默认）
xpack.security.enabled: true
xpack.security.transport.ssl.enabled: true
xpack.security.http.ssl.enabled: true
```

```bash
# 生成证书（自动）
elasticsearch-certutil ca
elasticsearch-certutil cert --ca elastic-stack-ca.p12

# 设置密码（自动生成）
elasticsearch-setup-passwords auto
```

## 1.1 内置用户

| 用户 | 角色 |
|------|------|
| `elastic` | 超级管理员 |
| `kibana_system` | Kibana 连接到 ES |
| `logstash_system` | Logstash 连接到 ES |
| `beats_system` | Filebeat 连接到 ES |

## 1.2 自定义角色

```json
POST /_security/role/order_read_role
{
  "cluster": ["monitor"],
  "indices": [
    {
      "names": ["orders"],
      "privileges": ["read", "view_index_metadata"]
    }
  ]
}

POST /_security/role/order_write_role
{
  "indices": [
    {
      "names": ["orders"],
      "privileges": ["create", "index", "delete", "write"]
    }
  ]
}

# 创建用户并分配角色
POST /_security/user/order_user
{
  "password": "secret123",
  "roles": ["order_read_role", "order_write_role"]
}
```

# 2. 字段级/文档级安全

## 2.1 FLS — 字段级安全

```json
POST /_security/role/masked_role
{
  "indices": [
    {
      "names": ["orders"],
      "privileges": ["read"],
      "field_security": {
        "grant": ["order_no", "amount", "status"],
        "except": ["customer_phone", "customer_email"]
      }
    }
  ]
}
```

## 2.2 DLS — 文档级安全

```json
POST /_security/role/self_orders
{
  "indices": [
    {
      "names": ["orders"],
      "privileges": ["read"],
      "query": {
        "term": { "user_id": "_user_id" }
      }
    }
  ]
}
```

# 3. TLS 配置

```yaml
xpack.security.http.ssl.enabled: true
xpack.security.http.ssl.keystore.path: certs/http.p12
xpack.security.http.ssl.truststore.path: certs/http.p12

xpack.security.transport.ssl.enabled: true
xpack.security.transport.ssl.verification_mode: certificate
xpack.security.transport.ssl.keystore.path: certs/transport.p12
xpack.security.transport.ssl.truststore.path: certs/transport.p12
```

# 4. 监控

## 4.1 Cat API

```bash
GET /_cat/health?v
GET /_cat/nodes?v&h=name,heap.percent,ram.percent,cpu,load_1m
GET /_cat/indices?v&h=index,pri,rep,store.size
GET /_cat/shards?v&h=index,shard,prirep,state,unassigned.reason
GET /_cat/thread_pool/write?v
GET /_cat/thread_pool/search?v
GET /_cat/allocation?v
```

## 4.2 Task Management

```bash
# 查看活跃任务
GET /_cat/tasks?v

# 取消慢查询
POST /_tasks/task_id:xxx/_cancel

# 查看热点线程
GET /_nodes/hot_threads?interval=1s
```

# 5. Slow Log

```json
PUT /orders/_settings
{
  "index.search.slowlog.threshold.query.warn": "2s",
  "index.search.slowlog.threshold.query.info": "1s",
  "index.search.slowlog.threshold.fetch.warn": "1s",
  "index.indexing.slowlog.threshold.index.warn": "5s"
}
```

# 6. Prometheus Exporter

```bash
# 安装插件
elasticsearch-plugin install \
  https://github.com/vvanholl/elasticsearch-prometheus-exporter/releases/download/8.11.0/prometheus-exporter-8.11.0.zip

# 访问指标
curl http://localhost:9200/_prometheus/metrics
```
