
# 1. 分片设计原则

```
分片键选择的黄金法则：
  1. 高频查询 WHERE 条件中的列
  2. 不影响业务逻辑的不可变列
  3. 区分度高的列（避免数据倾斜）
  4. 尽量与 JOIN 关联键一致（避免跨分片 JOIN）

推荐：业务主键（订单号/用户ID/SKU）
不推荐：状态、类型等低区分度列
```

# 2. 容量规划

```
分片数计算公式：
  分片数 = 预估总数据量 / 单表舒适容量
  单表舒适容量 ≈ 1000万 ~ 2000万行

扩容预留：
  初始配置 N 个分片
  预留 N 个空分片（避免紧急扩容）
  使用 2 的幂次方分片数（方便后续倍增）
```

# 3. 配置管理

```yaml
# 推荐：YAML 初始部署 + DistSQL 运维

# 初始：YAML 定义完整规则
rules:
  - !SHARDING
    tables: ...
    shardingAlgorithms: ...
    keyGenerators: ...

# 运维：DistSQL 在线调整
ALTER SHARDING TABLE RULE t_order (
    DATANODES("ds${0..3}.t_order_${0..1}")
);
```

# 4. 连接池配置

```yaml
dataSources:
  ds0:
    connectionTimeoutMilliseconds: 3000
    idleTimeoutMilliseconds: 60000
    maxLifetimeMilliseconds: 1800000
    maxPoolSize: 50          # 根据 QPS 计算
    minPoolSize: 5
```

**连接数计算**：`maxPoolSize ≈ (QPS × 平均RT/1000) + 10`

# 5. JDBC vs Proxy 选择

| 场景 | 推荐 |
|------|------|
| 纯 Java 应用，性能优先 | JDBC |
| 多语言混合 | Proxy |
| 需要 DistSQL 在线变更 | Proxy |
| 需要统一连接治理 | Proxy |
| 只需分库分表 | JDBC |
| 已有 MySQL 运维经验 | Proxy（协议兼容） |

# 6. 索引设计

```sql
-- 每个物理表独立建索引
-- 分片键在主键中 → 已覆盖

-- 非分片键的查询：考虑以下策略
-- 1. 避免（让查询包含分片键）
-- 2. 如果无法避免 → 使用 ES 等搜索引擎作为二级索引
-- 3. 广播表存储小量级数据

-- 分页查询优化：使用游标分页
SELECT * FROM t_order
WHERE order_id > ? AND user_id = ?
ORDER BY order_id LIMIT 20;
```

# 7. 生产配置模板

```yaml
mode:
  type: Cluster
  repository:
    type: ZooKeeper
    props:
      namespace: shardingsphere
      server-lists: zk1:2181,zk2:2181,zk3:2181

props:
  sql-show: false                       # 生产关闭 SQL 打印
  sql-simple: false
  max-connections-size-per-query: 1
  kernel-executor-size: 16              # 执行线程数
  proxy-frontend-executor-size: 32      # Proxy 前端线程
  proxy-backend-executor-suitable: true
  check-table-metadata-enabled: false   # 关闭元数据检查（性能优化）

authority:
  users:
    - user: root@%
      password: ${ROOT_PASSWORD}        # 环境变量注入
  privilege:
    type: ALL_PRIVILEGES
```

# 8. 监控告警

```
必须告警的场景：
□ Proxy 进程 OOM
□ 某个存储单元被 DISABLE
□ 连接数 > 80% maxPoolSize
□ SQL 解析错误持续增长
□ P99 RT 突增 2 倍以上
□ ZooKeeper 会话超时
```

# 9. DON'T

```
✗ 不要用分片键以外的列做高频全表扫描
✗ 不要在 YAML 中写死 IP（用环境变量或配置中心）
✗ 不要在生产环境 sql-show=true（影响性能）
✗ 不要在建表后频繁改分片规则（数据迁移成本高）
✗ 不要用 OFFSET 深度分页（用游标）
✗ 不要忘记定期备份注册中心元数据
✗ 不要单点部署 Proxy（至少 2 实例 + LB）
```
