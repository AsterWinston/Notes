# 1. 数据库分库分表概述

当单表数据量超过**千万级**或单库数据量超过**百 GB** 时，MySQL 可能面临以下瓶颈：

- **磁盘 IO**：单表过大导致 B+Tree 层数增加，查询变慢
- **连接数**：单库连接数有限（通常 1000~2000）
- **写入性能**：单表写入受限于单机的 IO 能力
- **备份/恢复慢**：TB 级别数据库全量备份耗时长

**分库分表**是水平扩展数据库的核心手段——将数据**按某种规则**拆分到多个数据库或表中，每份数据量可控，整体处理能力叠加。

## 1.1 垂直拆分与水平拆分

| 维度 | 垂直拆分 | 水平拆分 |
|------|---------|---------|
| **划分方式** | 按**业务**拆分（不同表放到不同库） | 按**数据行**拆分（同一表的数据分散到多个节点） |
| **示例** | 用户库、订单库、商品库各自独立 | order 表按 user_id 拆分为 16 张表 |
| **解决瓶颈** | 业务耦合、连接数 | 单表数据量、写入性能 |
| **复杂度** | 较低 | 较高（跨分片查询、分布式事务） |

### 1.1.1 垂直拆分示例

```
单体库：
  ├── user (用户)
  ├── order (订单)
  ├── product (商品)
  └── payment (支付)

拆分后：
  ├── user_db → user
  ├── order_db → order
  ├── product_db → product
  └── payment_db → payment
```

### 1.1.2 水平拆分示例

```
order 表（单表 5000 万行）
  → order_0 (user_id % 16 = 0)
  → order_1 (user_id % 16 = 1)
  → ...
  → order_15 (user_id % 16 = 15)
```

## 1.2 分库分表需要解决的问题

| 问题 | 解决方案 |
|------|---------|
| **主键生成** | 雪花算法、号段模式、Redis 自增 |
| **跨分片查询** | 中间件支持；指定分片键在单分片内查询 |
| **跨分片排序/分页** | 中间件在内存中合并排序（性能差） |
| **跨分片 JOIN** | 尽量避免；通过数据冗余或应用层聚合替代 |
| **分布式事务** | Seata、RocketMQ 事务消息、TCC、最终一致性 |
| **全局唯一建索引** | 与分片键耦合；冗余到 ES 中 |
| **扩容/数据迁移** | 一致性 hash、双写迁移、停服迁移 |

> **核心原则**：分库分表后，**80% 以上的查询应落在单一分片内**（通过正确的分片键设计），才能获得良好的性能。跨分片查询是分库分表的代价，应最小化。

# 2. 分片策略

## 2.1 分片键（Sharding Key）

分片键决定了数据如何分布。**选择分片键是分库分表设计中最重要的决策**。

**分片键选择原则**：

| 原则 | 说明 |
|------|------|
| **高区分度** | 数据尽可能均匀分布到各分片（避免热点） |
| **业务相关** | 大部分查询包含此字段作为条件 |
| **不跨分片** | 避免 JOIN 和事务跨分片 |

```sql
-- ✓ 好的分片键：user_id（下单、查订单都带 user_id）
SELECT * FROM `order` WHERE user_id = 100;
-- → 路由到 user_id % 16 的分片

-- ✗ 差的分片键：status（区分度低：0 或 1，数据严重倾斜）
SELECT * FROM `order` WHERE status = 1;
-- → 需要扫描所有分片
```

## 2.2 常见分片算法

### 2.2.1 取模分片（Hash Mod）

```java
// 分片算法：user_id % 16
int shard = userId % 16;
```

- 优点：数据均匀分布，实现简单
- 缺点：扩容时需要**几乎全量迁移数据**（如从 16 片扩到 32 片，100% 数据位置变化）

### 2.2.2 一致性哈希（Consistent Hash）

```
环 [0, 2^32)
  ├── node1 (hash = 100)
  ├── node2 (hash = 5000)
  ├── node3 (hash = 2000000)
  └── node4 (hash = 3000000000)
```

- 优点：扩容时只需迁移部分数据（受影响节点前后之间）
- 缺点：数据分布不均匀，需要虚拟节点平衡

### 2.2.3 范围分片（Range）

```
user_id 1-10000000 → shard_0
user_id 10000001-20000000 → shard_1
...
```

- 优点：扩容简单（新增分片放后面即可）
- 缺点：热点集中（新用户都集中在最后的分片）

### 2.2.4 日期分片（按年月）

```
order_202501 → shard_0
order_202502 → shard_1
...
```

- 优点：适合时序数据，扩容天然支持
- 缺点：时间分片不均匀（大促某月数据暴增）

## 2.3 实战分片策略（推荐）

通常采用**复合分片**，在应用层或中间件层实现多层次路由：

1. 先按**业务维度**垂直拆分（用户库、订单库...）
2. 水平拆分使用**取模分片**（初期够用，后续通过一致性哈希扩容）
3. 对特定场景（如按年份查询订单），增加**二级索引表**（ES 中建 user_id → shard 映射）

# 3. Mycat 中间件

**Mycat** 是一个开源的数据库中间件，通过实现 MySQL 协议，对应用层透明地实现分库分表、读写分离等功能。应用代码无需修改，就像操作普通 MySQL 一样连接 Mycat。

## 3.1 Mycat 架构

```
应用层 (JDBC)
    │
    ▼
┌─────────┐
│  Mycat  │  ← 伪装成 MySQL Server
└─────────┘
    │
    ├── MySQL-0 (shard_0)
    ├── MySQL-1 (shard_1)
    └── MySQL-2 (shard_2)
```

## 3.2 核心概念

| 概念 | 说明 |
|------|------|
| **Schema** | 逻辑库，对应应用代码中的 database 名称 |
| **DataNode** | 数据节点，对应一个真实的 MySQL 数据库 |
| **DataHost** | 数据库主机，物理 MySQL 实例地址 |
| **分片规则** | 决定数据路由到哪个 DataNode |

## 3.3 核心配置文件

### 3.3.1 server.xml（Mycat 服务器配置）

```xml
<user name="root" defaultAccount="true">
  <property name="password">123456</property>
  <property name="schemas">mydb</property>
</user>
```

### 3.3.2 schema.xml（逻辑库和分片配置）

```xml
<!-- 逻辑库定义 -->
<schema name="mydb" checkSQLschema="true" sqlMaxLimit="100">
  <table name="order" dataNode="dn0,dn1" rule="mod-long" />
  <table name="user" dataNode="dn0,dn1" rule="mod-long" />
</schema>

<!-- 数据节点 -->
<dataNode name="dn0" dataHost="host1" database="order_0" />
<dataNode name="dn1" dataHost="host2" database="order_1" />

<!-- 数据主机 -->
<dataHost name="host1" maxCon="1000" minCon="10" balance="0"
          writeType="0" dbType="mysql" dbDriver="jdbc">
  <heartbeat>select user()</heartbeat>
  <writeHost host="master1" url="jdbc:mysql://192.168.1.100:3306" user="root" password="123456" />
  <writeHost host="slave1" url="jdbc:mysql://192.168.1.101:3306" user="root" password="123456" />
</dataHost>
```

### 3.3.3 rule.xml（分片规则）

```xml
<!-- 取模分片 -->
<tableRule name="mod-long">
  <rule>
    <columns>id</columns>
    <algorithm>mod-long</algorithm>
  </rule>
</tableRule>

<function name="mod-long" class="io.mycat.route.function.PartitionByMod">
  <property name="count">2</property>  <!-- 分片数 -->
</function>

<!-- 范围分片 -->
<tableRule name="auto-sharding-long">
  <rule>
    <columns>id</columns>
    <algorithm>rang-long</algorithm>
  </rule>
</tableRule>
```

## 3.4 Mycat 的局限性

- **跨分片 JOIN 性能差**（需在内存中聚合）
- **不支持完整的 SQL 标准**（如复杂子查询可能不支持）
- **社区活跃度下降**（Mycat2 开发缓慢）
- **运维成本高**（需要额外维护 Mycat 集群）

> **现代替代方案**：**ShardingSphere-Proxy**（Apache 项目，社区活跃，功能更全）、**Vitess**（YouTube 开源的分布式数据库群集，基于 MySQL）、**TiDB**（分布式 SQL 数据库，兼容 MySQL 协议）。

# 4. 数据迁移方案

当现有分片不满足需求需要扩容时，数据迁移是最大的挑战。

## 4.1 停服迁移

```
1. 停止服务（选择低峰期，如凌晨 3 点）
2. 全量导出数据
3. 按新分片规则导入
4. 切换应用配置
5. 恢复服务
```
- 优点：最简单、最安全
- 缺点：需要停服（有损）

## 4.2 双写迁移（推荐）

```
1. 旧数据全量同步到新集群
2. 开启双写（应用同时写新旧集群）
3. 对比数据一致性
4. 按新分片规则修正增量数据
5. 将读切换到新集群
6. 关闭旧集群写入
7. 下线旧集群
```
- 优点：不停服、平滑过渡
- 缺点：实现复杂，需要数据校验

## 4.3 基于 binlog 的增量同步

```
1. 全量备份恢复
2. Canal/DTS 监听 binlog  增量同步到新集群
3. 数据校验一致后切换流量
```
- 优点：增量实时、延迟低
- 缺点：依赖 Canal 等组件

# 5. 分库分表最佳实践

- **能不分就不分**：单表千万级用 InnoDB + 索引 + 读写分离完全能抗住，不要过早优化
- **按业务垂直拆分优先**：在水平拆分之前，先做垂直拆分
- **选好分片键**：80% 的查询应该命中单一分片
- **避免跨分片 JOIN**：冗余必要字段或应用层聚合
- **全局 ID 用雪花算法**：有序且全局唯一
- **建立监控**：分片数据量、热点分布、慢查询告警
- **定期归档**：老数据归档到独立库或对象存储
- **善用 ES**：复杂搜索场景将数据同步到 ES，MySQL 只做精准查询
