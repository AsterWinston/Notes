
Logstash 8.x 默认把 Elastic 安全特性带进了每一条链路：beats 端口裸监听、9600 API 无认证、配置文件明文写密码，是安全审计里最常见的三个"低垂的雷"。本篇按"传输加密 → 密钥管理 → 最小权限 → 监控与集中管理 → 审计"的顺序把这条链路的防护补齐，并说明 Logstash 自身（而非 ES）该看哪些监控端点。

# 1. 管道安全

## 1.1 敏感字段脱敏

**日志脱敏的合理位置是 Logstash 的 Filter 段，而不是 ES**——数据一旦写入索引，就已经进入 `_source`、倒排、副本和快照备份，事后再删成本高得多。三种手段按"破坏程度"递增：直接删除（字段整条丢弃）、掩码（保留格式但遮盖中间位）、哈希/gsub 不可逆替换。选型时注意掩码逻辑对手机号/身份证这类强格式字段有效，对自由文本里的敏感信息（`message` 内嵌的 token）要配合 `mutate gsub` 的正则替换：

```ruby
filter {
  # 方式 1: 删除敏感字段
  if [phone] or [email] {
    mutate {
      remove_field => ["phone", "email", "password", "credit_card"]
    }
  }

  # 方式 2: 替换为掩码
  ruby {
    code => '
      if event.get("phone")
        phone = event.get("phone").to_s
        event.set("phone", phone[0..2] + "****" + phone[-4..-1])
      end
    '
  }

  # 方式 3: 仅保留 Hash（不可逆）
  mutate {
    gsub => [ "password", ".*", "******" ]
  }
}
```

> **注意**：上面方式 2 用了 ruby filter，安全上没问题但性能差（ruby 是逐条调用）；量大时应改用 `mutate { gsub => [ "phone", "(\\d{3})\\d{4}(\\d{4})", "$1****$2" ] }`，效果等价、吞吐高一个量级。另外脱敏规则要覆盖 `message` 原文字段——很多团队删了结构化 `phone` 字段，却忘了 grok 之前 `%{GREEDYDATA:message}` 里还留着完整原文。

**设计考量**：

- **脱敏时机尽量前置**：在最早的 Input/Filter 段处理，越往后副本越多（PQ 落盘、DLQ、隔离索引里都可能留一份原文）
- **PQ/DLQ 也是敏感数据载体**：落盘事件包含脱敏前的字段，磁盘与备份的访问控制要按敏感级处理
- **保留可审计性**：不可逆哈希要配加盐或 HMAC 密钥，否则同值同哈希仍可被关联还原

# 2. 传输加密：Beats Input TLS

Logstash 侧配证书与私钥；若要求客户端证书（mTLS），再加 CA 校验。8.x 起 `ssl` 默认为 true（beats input 插件层面），但多数部署仍显式写出：

```ruby
input {
  beats {
    port => 5044
    ssl => true
    ssl_certificate           => "/etc/logstash/tls/logstash.crt"
    ssl_key                   => "/etc/logstash/tls/logstash.key"
    ssl_key_passphrase        => "${LS_SSL_KEY_PASSPHRASE}"   # 私钥有口令时从 keystore 注入
    ssl_certificate_authorities => ["/etc/logstash/tls/ca.crt"]
    ssl_verify_mode           => "force_peer"                  # 强制校验客户端证书（mTLS）
    ssl_supported_protocols   => ["TLSv1.2", "TLSv1.3"]
    ssl_cipher_suites         => [
      "TLS_AES_256_GCM_SHA384", "TLS_CHACHA20_POLY1305_SHA256",
      "TLS_AES_128_GCM_SHA256", "ECDHE-ECDSA-AES256-GCM-SHA384"
    ]
  }
}
```

对应的 Filebeat 侧（`filebeat.yml`）：

```yaml
output.logstash:
  hosts: ["logstash.internal.example.com:5044"]
  ssl.certificate_authorities: ["/etc/filebeat/certs/ca.crt"]
  ssl.certificate: "/etc/filebeat/certs/beat.crt"
  ssl.key: "/etc/filebeat/certs/beat.key"
  ssl.verification_mode: full        # 校验主机名与证书，不要用 none
```

证书签发建议走内部 CA 或 cert-manager 类自动化工具，`logstash.crt` 的 SAN 里必须包含 Filebeat 配置中访问它的主机名/DNS，否则 `verification_mode: full` 会握手失败。**自签叶子证书直连**只适合 demo：Filebeat 侧一旦关闭校验（`none`），中间人风险就回来了。

> **行业实践**：mTLS 在"跨安全域"边界上用（如生产网 DMZ 汇聚到日志区），同安全域内单向 TLS（服务端证书 + 客户端 CA 白名单）通常够用；万级 Filebeat 场景下逐个签发客户端证书的运维成本很高，普遍改为"单向 TLS + Filebeat 端 IP/网络层准入"。

# 3. ES 输出认证

8.x 默认开启 ES 安全，elasticsearch output 必须带凭据，推荐 HTTPS + 证书校验：

```ruby
output {
  elasticsearch {
    hosts => ["https://es1:9200"]
    user => "logstash_writer"
    password => "${ES_PASSWORD}"                # 从 keystore 注入，不写明文
    ssl_enabled => true                          # 8.x 新参数名
    ssl_certificate_authorities => ["/etc/logstash/certs/ca.crt"]
    ssl_verification_mode => "full"              # full / certificate / none
  }
}
```

早期文档里的 `ssl => true`、`cacert => ...`、`ssl_certificate_verification => true` 在 8.x 插件中已被 `ssl_enabled`、`ssl_certificate_authorities`、`ssl_verification_mode` 取代，混用会报 `Unknown settings`。凭据轮换时只需更新 keystore 值并重载管道，配置文件不动。

# 4. Keystore 密钥管理

```bash
# 创建 Keystore（生成于 $LS_HOME/logstash.keystore）
bin/logstash-keystore create

# 添加密钥
bin/logstash-keystore add ES_PASSWORD
# Enter value: mysecret

# 覆盖已存在的值 / 删除
bin/logstash-keystore add ES_PASSWORD --force-update
bin/logstash-keystore remove KAFKA_SASL_PASSWORD

# 列出
bin/logstash-keystore list
```

**优势**：敏感信息不在配置文件中明文存储。配置文件里写 `${ES_PASSWORD}` 占位，运行期从 keystore 解析。keystore 缺失或值不存在时管道启动即失败（fail-fast，不会带着空密码跑起来）。还有两种注入途径值得了解：**环境变量兜底**——keystore 没有该项时会读同名环境变量（容器场景常用 secret 挂载 env），但反过来"忘了放 keystore、恰好 env 里有"会让本地调试与生产行为不一致，建议显式约定优先级；**`-i` 交互输入**——`bin/logstash -e '...'` 场景不依赖 keystore 文件。

> **注意**：`logstash.keystore` 文件本身的权限才是最终防线（owner 属 logstash 用户、600），keystore 加密的是"静态存储中的凭据"，运行期内存里仍是明文，LS 进程账号被接管时防不住；备份 keystore 要单独走密管流程，丢了所有管道都起不来。

# 5. 监控 UI 与数据链路

```text
Kibana → Stack Monitoring → Logstash

监控面板:
  ├── Events Received / Emitted Rate   （进出速率是否背离）
  ├── Pipeline Workers Count
  ├── Queue Events Count (堆积监控)
  ├── JVM Heap Usage
  ├── CPU Usage
  └── Output Plugin Latency
```

Stack Monitoring 的数据链路是 Logstash 把自监控文档周期性写入 ES 的监控索引（`.monitoring-logstash-*`），Kibana 读取渲染：

```text
┌────────────┐  monitoring docs  ┌─────────────────┐  query  ┌────────┐
│  Logstash  │ ────────────────→ │ Elasticsearch   │ ←────── │ Kibana │
│ (采集自身)  │   每 10s 批量写    │ .monitoring-*   │         │        │
└────────────┘                   └─────────────────┘         └────────┘
```

## 5.1 配置 Monitoring

```yaml
# logstash.yml
xpack.monitoring.enabled: true
xpack.monitoring.elasticsearch.hosts: ["https://es:9200"]
xpack.monitoring.elasticsearch.username: "logstash_system"
xpack.monitoring.elasticsearch.password: "${MONITORING_ES_PASSWORD}"
xpack.monitoring.elasticsearch.ssl.certificate_authority: "/etc/logstash/certs/ca.crt"
xpack.monitoring.elasticsearch.timeout: 20s
```

`xpack.monitoring.*` 这套设置名在 8.x 中仍可用但已标记向非 xpack 命名迁移，新部署按所用小版本官方参考核对。两个运维要点：监控写入**用独立账号**（下节角色表里的 `logstash_system`），避免与业务写入账号互相牵连；监控索引本身要走 ILM 设保留期（如 30 天），不然 `.monitoring-*` 是集群里典型的"隐形容量消耗者"。

> **设计考量**：监控链路有自指风险——ES 故障时业务写不进、监控文档同样写不进，监控恰好在你最需要它的时候失明。所以关键告警（队列水位、进程存活）至少要有一条**不依赖 ES** 的旁路，例如基于 `_node/stats` 的脚本/Prometheus exporter，或把 logstash.yml 日志接独立 syslog。

# 6. API 监控端点与加固

```bash
# 节点信息
curl http://localhost:9600/_node

# 管道统计
curl http://localhost:9600/_node/stats/pipelines

# JVM 信息
curl http://localhost:9600/_node/stats/jvm

# 热线程
curl http://localhost:9600/_node/stats/hot_threads?human=true

# 插件信息
curl http://localhost:9600/_node/plugins
```

9600 默认只绑 localhost（`http.host: "127.0.0.1"`），这是它没开认证也相对安全的原因；一旦为集中采集改成 `0.0.0.0`，就必须同时开启 TLS 与认证：

```yaml
# logstash.yml
http.host: "0.0.0.0"
api.ssl.enabled: true
api.ssl.keystore.path: /etc/logstash/tls/api.p12
api.ssl.keystore.password: "${API_KEYSTORE_PASS}"
api.auth.type: basic
api.auth.basic.username: "admin"
api.auth.basic.password: "${API_PASSWORD}"     # 建议 credentials 文件替代，见官方文档
```

生产常见拓扑是把 API 端口留给 Prometheus logstash-exporter 抓取：exporter 与 Logstash 同机部署、走 localhost + 只读账号，比对外开放 9600 加全套 TLS 更稳妥。

# 7. Central Management（集中管理）

多实例部署时可以把"管道配置的存放与下发"集中到 Kibana：**配置在 Kibana 的 Management → Logstash Central Management 编辑，Logstash 定时拉取**，适合"少量运维统一管几十台采集端"的场景。前提：License 需 Basic 及以上、ES 安全开启、Kibana 可写 `logstash-*` 系统索引。

```yaml
# logstash.yml
xpack.management.enabled: true
xpack.management.elasticsearch.hosts: ["https://es1:9200"]
xpack.management.elasticsearch.username: "logstash_mgmt"
xpack.management.elasticsearch.password: "${MGMT_ES_PASSWORD}"
xpack.management.pipeline.id: ["nginx-prod", "app-prod"]   # 本实例认领的管道
xpack.management.reload_interval: 5s                       # 拉取配置变更的周期
```

与热加载的取舍：Central Management 下 **`config.reload.automatic` 不生效**（以拉取周期 `xpack.management.reload_interval` 为准）；本地 `path.config` 与集中管道**不能同名**，冲突时启动报错。灰度能力也更弱——它按 pipeline.id 认领实例分组，版本控制、审计回滚不如 Git 化配置直观。

> **行业实践**：中大规模团队更多走 **Git + CI 校验 + 配置下发（Ansible/ArgoCD）** 而非 Central Management——配置的版本化、Review、回滚都在既有发布体系里完成；Central Management 的价值在"没有配置管理基建、又需要 Web 界面小改小调"的运维团队。

# 8. Dead Letter Queue 运维

```bash
# 查看 DLQ 文件
ls -la /var/lib/logstash/dead_letter_queue/

# 查看 DLQ 统计
curl http://localhost:9600/_node/stats | jq '.pipelines."main".dead_letter_queue'
```

```ruby
# 重放 DLQ
input { dead_letter_queue { path => "/var/lib/logstash/dead_letter_queue" } }
output { elasticsearch { ... } }
```

安全视角补一句：DLQ 里是**带失败原因的全量原始事件**，敏感字段脱敏规则若配在 filter 段，DLQ 数据可能仍含原文；`path.dead_letter_queue` 目录权限与清理流程要和 keystore、PQ 目录同等级对待。

# 9. 审计日志与最小权限

```yaml
# logstash.yml
log.format: json                    # JSON 格式日志，便于进 SIEM
path.logs: /var/log/logstash
log.level: info

api.auth.type: basic
api.auth.basic.username: "admin"
api.auth.basic.password: "${API_PASSWORD}"
```

`log.format: json` 让 Logstash 自身日志（管道启停、reload、插件错误、认证失败）可被日志管道结构化消费；把这份日志接入 ES 时记得**用独立索引并收紧读权限**，否则"谁改了配置、什么时候启动过 DLQ 重放"这类审计线索对所有人可见。

Logstash 侧没有内建的 RBAC，最小权限实际落在 **ES 角色设计**上，按用途拆账号而不是共用一个超管：

| 账号/角色 | 用途 | 关键权限 |
|-----------|------|----------|
| `logstash_writer` | 业务输出管道 | 目标索引 pattern 的 `create_index` + `write`（不给 `all`、不给别的索引） |
| `logstash_template` | 模板/ILM 管理（独立低峰任务） | `manage_index_templates`、ILM manage |
| `logstash_system` | Stack Monitoring 写入 | `.monitoring-logstash-*` 写入，成员组 `logstash_system` |
| `logstash_mgmt` | Central Management 拉取配置 | `logstash-*` 系统索引读写 |
| Kibana 查看账号 | 人 | 索引 pattern 只读 + 监控页查看 |

> **注意**：`manage_template => false` 是安全与稳定的双重建议——模板变更是**高危动作**（改 mapping 会引发全索引重建风险），应由 CI 单独用低峰任务账号执行并走 Review，而不是每台 Logstash 启动时抢着写。

# 10. 生产安全检查清单

```text
□ Keystore 存储所有密码（ES/ApI/Kafka）
□ 监控 ES 输出的认证错误
□ 敏感字段在管道中脱敏或删除
□ PQ 路径加密（如有合规要求）
□ API 端点认证开启（生产环境）
□ 定期检查 DLQ（发现下游异常）
□ 日志级别: info（生产）/ warn（稳定）
□ SSL/TLS 启用（Beats→Logstash、Logstash→ES）
```

补充几条同样常被漏项的：证书私钥文件权限 600 且属 logstash 用户；`logstash.keystore` 纳入备份与轮换流程；监控链路与业务链路账号隔离、告警至少一条不依赖 ES 的旁路；`file`/`exec` 类插件慎用（exec 等于给了配置文件持有者任意命令执行权）；集中管理或 Git 仓库中**永远不提交**任何明文密码样例。

# 11. 小结

| 主题 | 关键结论 |
|------|----------|
| 传输加密 | beats input 配 `ssl_certificate`/`ssl_key`，mTLS 加 `ssl_certificate_authorities` + `force_peer`；Filebeat 侧 `verification_mode: full` |
| 密钥管理 | `logstash-keystore` + `${VAR}` 占位，环境变量兜底但注意行为一致性；keystore 文件权限是最后防线 |
| ES 输出 | 8.x 参数改名：`ssl_enabled` / `ssl_certificate_authorities` / `ssl_verification_mode`，别混用旧参数 |
| 监控 | Stack Monitoring 写 `.monitoring-*`；`_node/stats` 是旁路；告警至少一条不依赖 ES |
| Central Management | Kibana 下发配置，禁用本地热加载，License ≥ Basic；大规模仍推荐 Git + CI |
| 最小权限 | 按 writer/template/system/mgmt 拆 ES 账号；模板管理独立低峰执行 |
| 审计 | `log.format: json` 接 SIEM；DLQ/PQ 目录含原始数据，权限与清理同敏感级对待 |
