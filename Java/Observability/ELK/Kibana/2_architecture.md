
# 1. Kibana 内部架构

```
┌──────────────────────────────────────────────────────┐
│                    Kibana Server                       │
│                                                       │
│  ┌──────────┐   ┌──────────┐   ┌──────────┐         │
│  │ HTTP API │   │   Plugins│   │   Task   │         │
│  │  Server  │   │  System  │   │ Manager  │         │
│  └────┬─────┘   └────┬─────┘   └────┬─────┘         │
│       │              │              │                 │
│  ┌────┴──────────────┴──────────────┴─────┐          │
│  │              Saved Objects              │          │
│  │  (.kibana 索引，ES 持久化存储)           │         │
│  └────────────────────────────────────────┘          │
└──────────────────────────────────────────────────────┘
```

# 2. Saved Objects 存储模型

Kibana 的所有配置都以 **Saved Objects** 形式存储在 ES 的 `.kibana` 索引中：

| Object 类型 | 存储内容 | 示例 |
|-----------|---------|------|
| `index-pattern` | Data View 定义 | 字段信息、时间字段 |
| `search` | 保存的搜索 | Discover 保存的查询 |
| `visualization` | 可视化配置 | Lens 图表 JSON |
| `dashboard` | 仪表盘布局 | 面板排列、查询参数 |
| `config` | Kibana 全局设置 | 主题、默认索引 |
| `alert` | 告警规则 | Rule 定义、Actions |
| `tag` | 标签 | 对象分类标签 |

```bash
# 查看 Saved Objects
GET /.kibana_8.11.0_001/_search
{ "size": 10 }
```

# 3. kibana.yml 核心配置

```yaml
# ========== Server 配置 ==========
server.port: 5601
server.host: "0.0.0.0"
server.name: "kibana-prod"
server.maxPayload: 10485760        # 10MB 最大请求体
server.compression.enabled: true
server.ssl.enabled: true
server.ssl.certificate: /etc/kibana/certs/kibana.crt
server.ssl.key: /etc/kibana/certs/kibana.key

# ========== Elasticsearch 配置 ==========
elasticsearch.hosts: ["https://es1:9200", "https://es2:9200"]
elasticsearch.username: "kibana_system"
elasticsearch.password: "${KIBANA_PASSWORD}"
elasticsearch.ssl.certificateAuthorities: /etc/kibana/certs/ca.crt
elasticsearch.requestTimeout: 30000
elasticsearch.shardTimeout: 30000

# ========== 日志配置 ==========
logging.root.level: info
logging.appenders.file.type: file
logging.appenders.file.fileName: /var/log/kibana/kibana.log

# ========== 其他 ==========
path.data: /var/lib/kibana
pid.file: /var/run/kibana.pid

# 内嵌 ES（开发用，不建议生产）
# elasticsearch.hosts: "http://localhost:9200"
```

# 4. Kibana 系统索引

```
.kibana                 → 7.x Saved Objects
.kibana_8.11.0_001      → 8.x Saved Objects（版本号区分）
.kibana_alerting_cases  → 案例数据
.kibana_task_manager    → 后台任务（告警/报告）
.kibana-event-log-*     → 审计事件日志
```

```bash
# 升级后旧索引会被标记为只读，新索引自动创建
# 显式迁移
POST /_kibana/_upgrade_saved_objects
```

# 5. 与 ES 的通信模型

```
Kibana Server (5601)
  │
  ├── Data Queries (Discover/Dashboard)
  │     └── 代理转发到 ES（透明）
  │
  ├── Saved Objects CRUD
  │     └── 直接操作 .kibana 索引
  │
  ├── Background Tasks (Alerting/Reporting/ML)
  │     └── Task Manager 定时执行
  │
  └── Fleet API
        └── Agent 策略下发
```

# 6. Kibana 性能配置

```yaml
# 请求批处理
kibana.defaultAppId: "discover"

# ES 连接池
elasticsearch.maxSockets: 1000
elasticsearch.compression: true

# 搜索性能
elasticsearch.includePortInHostHeader: true
elasticsearch.sniffOnStart: false         # 生产建议关闭 sniffing

# Saved Objects 缓存
kibana.savedObjects.maxImportPayloadBytes: 26214400  # 25MB
```

# 7. 生产配置模板

```yaml
server.port: 5601
server.host: "0.0.0.0"
server.ssl.enabled: true
server.ssl.certificate: /etc/kibana/certs/kibana.crt
server.ssl.key: /etc/kibana/certs/kibana.key

elasticsearch.hosts: ["https://es1:9200", "https://es2:9200", "https://es3:9200"]
elasticsearch.username: "kibana_system"
elasticsearch.password: "${KIBANA_PASSWORD}"
elasticsearch.ssl.certificateAuthorities: /etc/kibana/certs/ca.crt
elasticsearch.requestTimeout: 30000

logging.root.level: warn
xpack.encryptedSavedObjects.encryptionKey: "${ENCRYPTION_KEY}"
xpack.reporting.encryptionKey: "${REPORTING_KEY}"
```
