
# 1. Agent 不生效

```bash
# 1. 确认 Agent JAR 路径正确
ls -la /path/to/skywalking-agent.jar

# 2. 检查 JVM 参数
ps aux | grep java | grep skywalking-agent

# 3. 查看 Agent 日志
tail -f logs/skywalking-agent.log
# 正常启动: "SkyWalking Agent started successfully"
# 配置错误: "Failed to connect to collector"

# 4. 检查 OAP 连通性
telnet oap-server 11800
curl http://oap-server:12800
```

# 2. OAP 启动失败

```bash
# 1. 检查端口占用
netstat -tlnp | grep 11800
netstat -tlnp | grep 12800

# 2. 检查存储连接
# ES:
curl http://es:9200/_cluster/health
# MySQL:
mysql -h localhost -u root -p skywalking -e "SELECT 1"

# 3. 查看 OAP 日志
tail -f logs/skywalking-oap-server.log
# "Could not connect to Elasticsearch" → ES 不可达
# "Table 'xxx' doesn't exist" → 数据库未初始化

# 4. 初始化数据库（MySQL 模式）
mysql -h localhost -u root -p < oap-libs/mysql-init.sql
```

# 3. Trace 断链

```
原因排查：
  1. 跨服务时 sw8 Header 丢失
     → 检查网关/LB 是否透传 sw8 Header
     → 检查 Dubbo attachment 是否被覆盖

  2. 异步线程 Trace 丢失
     → 使用 ContextManager.wrap() 包装 Runnable
     → 或升级到支持自动传播的线程池版本

  3. 采样率配置导致
     → agent.sample_n_per_3_secs 过小
     → 部分 Trace 被丢弃

  4. 不同服务的 Agent 版本不兼容
     → 统一 Agent 版本
```

# 4. ES 索引膨胀

```bash
# 1. 检查索引大小
curl "http://es:9200/_cat/indices/sw_*?v&s=store.size:desc"

# 2. 检查 TTL 配置
grep "recordDataTTL\|metricsDataTTL" config/application.yml

# 3. 手动清理过期索引
curl -X DELETE "http://es:9200/sw_segment-20240101"
curl -X DELETE "http://es:9200/sw_metrics-all-20240101"

# 4. 调整索引分片
# 在 Index Template 中设置合理的 shards 数
```

# 5. 内存过高

```bash
# 1. 检查 OAP JVM 内存
jstat -gcutil $(pgrep -f oap-server) 1000 5

# 2. 检查 Agent 内存
# Agent 默认约 100MB
# 过大可能是采样率过高或插件过多

# 3. Agent 配置优化
agent.sample_n_per_3_secs=500      # 降低采样
plugin.jdbc.trace_sql_parameters=false  # 关闭参数收集
```

# 6. 常见错误

| 错误日志 | 原因 | 解决 |
|---------|------|------|
| `Failed to connect to collector` | OAP 不可达 | 检查网络/端口 |
| `TID:N/A` | 非 HTTP 线程启动 | 正常（定时任务等） |
| `Could not find class` | 插件兼容性问题 | 升级 Agent 版本 |
| `ES index read-only` | ES 磁盘满 | 清理磁盘或扩容 |
| `gRPC DEADLINE_EXCEEDED` | 网络超时 | 检查 gRPC 配置 |

# 7. 诊断命令

```bash
# Agent 健康
curl http://localhost:12800/health

# OAP GraphQL 查询
curl -X POST http://localhost:12800/graphql \
  -d '{"query":"{ getAllServices { name } }"}'

# 查看 OAP JVM
jstat -gcutil $(pgrep -f oap) 1000
jstack $(pgrep -f oap) > oap.jstack
```
