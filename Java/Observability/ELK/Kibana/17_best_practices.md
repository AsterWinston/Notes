
# 1. Spaces 规划

```
推荐 Spaces 结构：
  ├── Production (仅监控 Dashboard，权限 read-only)
  ├── Development (开发环境数据探索)
  ├── Security (SIEM 安全分析)
  ├── ML (机器学习作业)
  └── Admin (管理 Space，仅管理员)
```

# 2. Dashboard 设计原则

```
✓ DO:
  - 控制面板数 ≤ 15 个（加载性能）
  - 使用 Controls 统一过滤
  - 设置合理的时间默认值（如 Last 15m）
  - Markdown 面板添加说明
  - 定期归档不再使用的 Dashboard

✗ DON'T:
  - 不要在一个 Dashboard 堆砌 30+ 面板
  - 不要使用过小的 refresh interval（生产 ≥ 60s）
  - 不要使用 * 开头的查询（全模糊，极慢）
  - 不要在 Dashboard 中暴露原始日志数据（用 Discover）
```

# 3. Data View 命名规范

```
推荐命名: {environment}-{datatype}-{detail}

示例:
  prod-logs-application
  prod-metrics-system
  staging-logs-nginx
  dev-logs-*

索引模式:
  - prod-logs-*  → 匹配所有生产日志索引
  - metricbeat-*  → 匹配指标（带时间字段）
```

# 4. 权限最小化

```
角色划分:
  ┌──────────┬─────────┬──────────┬───────┐
  │   Role   │ Discover│Dashboard │ Admin │
  ├──────────┼─────────┼──────────┼───────┤
  │ Viewer   │   ✓     │    ✓     │   ✗   │
  │ Analyst  │   ✓     │    ✓     │   ✗   │
  │ Admin    │   ✓     │    ✓     │   ✓   │
  └──────────┴─────────┴──────────┴───────┘

建议:
  - 生产环境：仅给 read 权限
  - 开发环境：给 all 权限（允许创建 Dashboard）
  - Dev Tools：仅 Admin 可访问
```

# 5. 性能优化

```yaml
# kibana.yml
elasticsearch.requestTimeout: 30000
elasticsearch.shardTimeout: 30000
elasticsearch.maxSockets: 500

# Advanced Settings
state:storeInSessionStorage: false   # 防止 session storage 膨胀
csv:maxSizeBytes: 5242880           # 限制 CSV 导出
discover:sampleSize: 500             # 限制单次搜索返回数
```

# 6. 版本升级

```
升级步骤:
  1. 备份 Saved Objects
     GET /api/saved_objects/_export?type=dashboard&type=visualization

  2. 备份 kibana.yml 配置文件

  3. 滚动升级（多实例时）
     - 停止 Kibana-1 → 升级 → 启动
     - 停止 Kibana-2 → 升级 → 启动

  4. 验证
     - 检查 Dashboard 是否显示正常
     - 检查告警规则是否正常

  5. 升级 Saved Objects（如有版本迁移提示）
```

# 7. 监控 Kibana 自身

```
必须监控的指标:
  □ Kibana 服务健康 (GET /api/status)
  □ ES 连接状态
  □ Kibana JVM 内存使用
  □ 请求延迟 P99
  □ 错误日志数量
  □ Saved Objects 数量（过多影响启动速度）

日志:
  tail -f /var/log/kibana/kibana.log | grep -E "ERROR|WARN"
```

# 8. DON'T

```
✗ 不要在生产环境用 Dev Tools 执行删除/修改操作
✗ 不要在 Dashboard 中引用过多 Data View（增加 ES 负载）
✗ 不要给所有用户 all 权限
✗ 不要在 Kibana 中存储超大 CSV 导出结果
✗ 不要混用不同版本的 Kibana + ES（版本必须匹配）
✗ 不要在 Dashboard 中使用 1s 刷新间隔
✗ 不要忘记定期清理旧的 Saved Objects
```
