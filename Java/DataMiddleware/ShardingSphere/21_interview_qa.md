
# 1. 基础题

## 1.1 ShardingSphere 是什么？有哪些核心能力？

Apache ShardingSphere 是分布式数据库生态系统，通过 JDBC 和 Proxy 两种模式提供：
- 数据分片（分库分表）
- 读写分离
- 数据加密
- 影子库压测
- 分布式事务（XA / Seata AT）

## 1.2 JDBC 模式和 Proxy 模式有什么区别？

| 维度 | JDBC | Proxy |
|------|------|------|
| 部署 | SDK 嵌入应用 | 独立部署代理 |
| 性能 | 高（直连 DB） | 中（+1 跳） |
| 语言限制 | 仅 Java | 无限制 |
| 配置运维 | YAML + 重启 | DistSQL 在线变更 |

## 1.3 ShardingSphere 5.x 有哪些主要变化？

- ANTLR 4 解析器（替换自研）
- DistSQL 在线配置语言
- 插件化 SPI 架构
- 全新治理中心（Cluster 模式）
- 支持 MySQL/PG/openGauss/SQLServer/Oracle

# 2. 原理题

## 2.1 ShardingSphere 的五大引擎是什么？

```
Parse → Rewrite → Route → Execute → Merge

1. Parse: ANTLR 解析 SQL → AST
2. Rewrite: 逻辑表名 → 物理表名、补充主键、分页改写
3. Route: 分片键 → 目标数据源和表
4. Execute: 多线程并发执行 → 各数据源
5. Merge: 归并排序/聚合/分页 → 统一结果
```

## 2.2 ShardingSphere 如何处理跨分片的 ORDER BY + LIMIT？

1. 每个分片执行 ORDER BY LIMIT（取 offset + limit 条）
2. 各分片结果集归并排序（K 路归并 / 优先队列）
3. 跳过 offset 条，取 limit 条返回

## 2.3 ShardingSphere 的分布式事务如何实现？

- **XA 模式**：集成 Atomikos/Narayana，两阶段提交
- **Seata AT 模式**：Seata 代理数据源，undo_log 自动回滚
- **自动判断**：涉及多个数据源时自动升级为 XA/Seata，单数据源用本地事务

## 2.4 广播表和绑定表有什么区别？

| 维度 | 广播表 | 绑定表 |
|------|--------|--------|
| 数据分布 | 每个数据源全量 | 子表与父表同分片 |
| 数据量 | 小（< 10万） | 随业务增长 |
| 写入开销 | 多数据源写入 | 单分片写入 |
| 典型场景 | 字典表、配置表 | 订单 + 订单明细 |

# 3. 实践题

## 3.1 如何选择分片键？

1. 查询中最频繁的 WHERE 条件列
2. 不可变列（避免修改触发数据迁移）
3. 区分度高（避免数据倾斜）
4. 与 JOIN 关联键一致（避免跨分片 JOIN）
5. 业务主键优先（订单号、用户ID）

## 3.2 分库分表后如何做跨分片分页？

深度分页性能差，推荐游标分页：

```sql
-- 替代 OFFSET 10000 LIMIT 20
SELECT * FROM t_order
WHERE order_id > ? AND user_id = ?
ORDER BY order_id LIMIT 20;
```

## 3.3 分布式主键方案怎么选？

| 方案 | 场景 |
|------|------|
| SNOWFLAKE | **首选**，高性能趋势递增 |
| UUID | 不需要排序 |
| COSID_SNOWFLAKE | 需要更好的时钟回拨保护 |
| AUTO_INCREMENT | 不推荐（单点瓶颈） |

# 4. 对比题

## 4.1 ShardingSphere vs Mycat2？

| 维度 | ShardingSphere | Mycat2 |
|------|---------------|--------|
| 接入 | JDBC + Proxy | Proxy |
| SQL 引擎 | ANTLR (多方言) | Calcite (MySQL 8.0) |
| 配置 | DistSQL 在线 | JSON 热加载 |
| 治理 | ZooKeeper/Nacos | ZK/Nacos |
| 社区 | Apache 顶级 | 社区维护 |

## 4.2 ShardingSphere vs Vitess？

| 维度 | ShardingSphere | Vitess |
|------|---------------|--------|
| 接入 | JDBC + Proxy | Proxy |
| 弹性伸缩 | ElasticJob 集成 | 原生 vtgate |
| 分布式事务 | XA/Seata | 2PC |
| 生态 | Java 生态 | CNCF / K8s 生态 |

# 5. 架构题

## 5.1 ShardingSphere 如何保证高可用？

- **Proxy 集群**：多实例 + LB，通过注册中心同步配置
- **Cluster 模式**：ZooKeeper/etcd/Nacos 存储元数据
- **熔断机制**：DISABLE 故障存储单元，自动摘除
- **数据库发现**：MGR 主从自动发现

## 5.2 数据加密的查询辅助器原理？

加密字段无法直接 LIKE 查询。查询辅助器对原文做**确定性哈希**（如 MD5），查询时对查询条件同样计算哈希，通过辅助列做精确匹配。支持等值查询和前缀查询，不支持全模糊 LIKE。
