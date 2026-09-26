
# 1. 基础题

## 1.1 Mycat 是什么？Mycat 1.x 和 2.x 有什么区别？

Mycat 是一个开源的分布式数据库中间件，作为 MySQL 协议代理层，对应用透明地提供分库分表、读写分离能力。

| 维度 | Mycat 1.x | Mycat 2.x |
|------|----------|-----------|
| I/O 模型 | BIO | NIO（Vert.x） |
| SQL 解析 | Druid | Apache Calcite |
| 配置格式 | XML | JSON |
| 动态配置 | 不支持 | 支持热加载 |
| 分布式事务 | 弱 XA | 原生 XA |
| 查询优化 | 简单改写 | RBO + CBO |

## 1.2 Mycat vs ShardingSphere-Proxy 如何选择？

| 维度 | Mycat2 | ShardingSphere-Proxy |
|------|--------|---------------------|
| 社区活跃度 | ★★★☆☆ | ★★★★★ |
| SQL 兼容性 | MySQL 8.0 完整 | MySQL + PostgreSQL |
| 配置方式 | JSON 文件 / ZK | DistSQL 在线变更 |
| 分布式事务 | XA | XA/Seata |
| 治理能力 | 基础 | 完善（APM/治理中心） |
| 文档丰富度 | 中 | 高 |

**建议**：新项目优先考虑 ShardingSphere-Proxy；Mycat2 适合已深度定制的 Mycat1 迁移场景。

## 1.3 Mycat 的分片策略有哪些？

| 策略 | 算法 | 适用场景 |
|------|------|---------|
| 取模 | `id % N` | 均匀分布主键 |
| 范围 | `id BETWEEN a AND b` | 区间查询 |
| 一致性Hash | MurmurHash | 扩容友好 |
| 日期 | `yyyy-MM-dd` 分区 | 时间序列 |
| 枚举 | 映射表 | 地区/类型 |

# 2. 原理题

## 2.1 Mycat2 如何处理一条跨分片的 COUNT(*) 查询？

```
1. SQL 解析 → Calcite Parser
2. 路由 → 发现跨所有分片
3. 并行发送 COUNT(*) 到每个分片
4. 各分片返回 count 值
5. Mycat2 求和合并 → 返回最终结果
```

## 2.2 Mycat2 的 ER 表和全局表有什么区别？

| 维度 | ER 表 | 全局表 |
|------|-------|--------|
| 数据分布 | 子表与父表在同一分片 | 全量副本在每个分片 |
| 数据量 | 随业务增长 | 小（< 10 万行） |
| 写入开销 | 单分片 | 所有分片（XA 事务） |
| JOIN | 与父表在本分片 JOIN | 任意分片可 JOIN |

## 2.3 Mycat2 如何实现读写分离？

通过 `clusters.json` 配置 `MASTER_SLAVE` 集群类型，Mycat2 根据 SQL 类型自动路由：
- INSERT/UPDATE/DELETE → Master
- SELECT → Slave（负载均衡）
- 特殊场景通过 Hint 强制路由

## 2.4 Mycat2 的 XA 事务执行流程？

1. 各分支 MySQL 执行 SQL
2. XA PREPARE（MySQL 写 binlog prepare 状态）
3. 所有分支 PREPARE OK → XA COMMIT
4. 任一分支失败 → XA ROLLBACK

## 2.5 Calcite 在 Mycat2 中的作用？

- SQL 解析（兼容 MySQL 完整语法）
- SQL 校验（表名、列名、类型检查）
- 查询优化（谓词下推、列裁剪、LIMIT 下推、常量折叠）
- 执行计划生成

# 3. 实践题

## 3.1 如何选择分片键？

1. 查询中使用频率最高的列
2. 区分度高的列（避免数据倾斜）
3. 不可变列（修改分片键可能导致数据迁移）
4. 业务主键（如订单号、用户 ID）

## 3.2 分库分表后如何做跨分片分页？

```sql
-- 深度分页性能差
SELECT * FROM orders ORDER BY id LIMIT 20 OFFSET 10000;

-- 优化方案：游标分页
SELECT * FROM orders
WHERE id > 9999
ORDER BY id LIMIT 20;
```

## 3.3 Mycat2 如何实现不停机扩容？

```
1. 准备新 MySQL 实例
2. 配置新分片（status=backup）
3. 全量迁移 + Canal 增量同步
4. 验证一致性
5. 热加载新配置 → 生效
```

# 4. 架构题

## 4.1 Mycat2 自身如何保证高可用？

- 多实例部署 + VIP/LVS 负载均衡
- 配置存储在 Nacos/ZK（共享）
- 无状态设计（实例故障不影响其他实例）
- 健康检查自动摘除故障实例

## 4.2 Mycat2 的性能瓶颈在哪里？

1. **跨分片查询**：全分片扫描 + 归并排序/聚合
2. **XA 事务**：多节点两阶段提交
3. **后端连接数**：连接池耗尽 → 排队
4. **SQL 解析**：Calcite 解析器开销（通常可忽略）

## 4.3 分布式主键方案如何选择？

| 方案 | 适用场景 |
|------|---------|
| Snowflake | **首选**，无依赖，高性能 |
| 数据库序列 | 需要严格递增的场景 |
| UUID | 不需要排序的场景 |
| Redis INCR | 已经使用 Redis 的场景 |

# 5. 对比题

## 5.1 Mycat2 分库分表 vs 应用层分库分表？

| 维度 | Mycat2 | 应用层（ShardingSphere-JDBC） |
|------|--------|---------------------------|
| 部署方式 | 独立中间件 | SDK 嵌入 |
| 语言限制 | 无（MySQL 协议） | Java |
| 网络开销 | +1 跳 | 直连数据库 |
| 运维复杂度 | 需维护中间件 | 随应用发布 |
| 性能 | 中（代理层开销） | 高（直连） |
