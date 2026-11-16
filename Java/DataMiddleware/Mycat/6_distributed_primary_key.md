# 1. 为什么需要分布式主键

**分库分表后，单库 AUTO_INCREMENT 立刻失效**：每个物理库各自从 1 开始自增，不同分片生成完全相同的 id；而业务又普遍要求 id 全局唯一（对账、跨库关联、消息路由、ES 文档 _id）。

```text
db1.orders:  INSERT → id = 1 (auto_increment)
db2.orders:  INSERT → id = 1 (auto_increment)
→ 两条记录 id 冲突，且都"插入成功"，错误在下游才爆发
```

合格的分布式主键需要同时满足四个条件：

1. **全局唯一**：跨所有分片、所有中间件实例不重复，这是一票否决项
2. **趋势递增**：InnoDB 主键是聚簇索引，乱序插入导致页分裂、写放大与碎片率上升
3. **高性能**：本地生成或极低延迟获取，不能每条 INSERT 都串行 RPC 一次
4. **信息可控**：最好携带生成时间与来源节点，方便排查；同时不宜让外部轻易推算业务规模

> **注意**：Mycat 1.6 的 `schema.xml` 里给表标了 `autoIncrement="true"` 只是告诉中间件"这张表的主键需要我接管"，**真正发号的是全局序列组件**（第 7 篇）。没有配置序列时，带 `autoIncrement="true"` 的表插入不指定 id 会直接报错，而不是回退到 MySQL 自增。

# 2. 方案总表

业界生成全局唯一 ID 的方案收敛为四类，先对比再逐个展开：

| 方案 | 唯一性机制 | 有序性 | 性能 | 依赖 | 典型代表 |
|------|-----------|--------|------|------|---------|
| UUID | 随机数/时间+MAC，128 位 | 无序 | 本地生成极高 | 无 | MySQL `UUID()` |
| 号段（Segment） | 中心表批量领取区间 | 趋势递增 | 中（缓存段内本地分配） | DB/ZK/Redis | Mycat 序列、美团 Leaf-segment |
| Snowflake | 时间戳+机器号+序列号拼 64 位 | 趋势递增 | 极高（本地） | 时钟+workerId 分配 | Mycat 本地序列、Leaf-snowflake、百度 UidGenerator |
| MySQL 自增+步长 | 每库设不同起点、相同步长 | 各库内递增 | 极高 | 需运维配置 | `auto_increment_increment` |

**为什么不用 UUID**——这是面试高频题，答案落在三个点上：

1. **无序插入是 InnoDB 之殇**：UUID 是 36 字符随机串，作为主键每插一行都是随机页插入，页分裂频繁，缓冲池命中率暴跌；实测同数据量下 UUID 主键表的体积与写入 TPS 都显著差于自增主键表
2. **存储与传输开销**：`VARCHAR(36)` 相比 BIGINT 每行多 28 字节，二级索引全部要携带主键值，放大效应传导到所有索引；作为 URL 参数、日志字段也冗长
3. **可读性与可追溯性差**：无法从 ID 看出生成时间与顺序，排障时"按 id 区间捞数据"这类基本操作都做不到

```sql
-- 错误示范：分库分表 + UUID 主键
CREATE TABLE orders (
    id VARCHAR(36) PRIMARY KEY,   -- 随机插入，页分裂严重
    ...
);

-- 正确做法：BIGINT 主键 + Snowflake/号段生成，趋势递增
CREATE TABLE orders (
    id BIGINT PRIMARY KEY,
    ...
);
```

> **设计争议**：UUID v7（前 48 位为毫秒时间戳）近年被 PostgreSQL 社区捧为主键新标准，它用"时间前缀"解决了 UUID 的有序性问题。但对 MySQL 而言它仍是 16 字节二进制/36 字符文本，聚簇索引宽度与二级索引膨胀的问题没有消失——**UUID v7 是"不想引入发号器"时的妥协，不是"比 Snowflake 更好"**。

**MySQL 自增+步长**方案值得单独说明：给 db1 设 `auto_increment_offset=1, auto_increment_increment=4`，db2 设 `offset=2, increment=4`，四库各发各的、永不相撞。看似优雅，但**它把路由责任压在了"插入落到哪个库"上**——一旦某库成为瓶颈要拆分，或数据迁移重分布，位点规划全部重来；且全局并非单调递增，只是"每个库内递增"。仅适合分片数永久固定的小规模场景。

# 3. Mycat 1.6 的三种内置实现

Mycat 1.6 的主键能力由全局序列（sequence）提供，配置在 `sequence_conf.properties`（本地模式）或 `sequnce_handler_type` 指定 Handler（数据库/ZK 模式），共三种实现，机制详解见第 7 篇：

| 实现 | 原理 | 依赖 | 递增性 | 性能 |
|------|------|------|--------|------|
| 0 本地文件（默认） | 进程内号段，重启读 conf 文件 | 无 | 趋势递增（各实例号段错开） | 最高 |
| 1 数据库 | mycat_sequence 表批量领号段 | MySQL | 严格递增 | 中（领号时一次 DB 写） |
| 2 ZooKeeper | ZK 节点存号段指针 | ZK | 严格递增 | 中高 |

```properties
# conf/sequence_conf.properties —— 本地号段模式
# 表名(大写).当前值(min值,max值)
ORDERS.HISIDS=
ORDERS.MINID=10001
ORDERS.MAXID=20000
# 多台 Mycat 时每台配置互不重叠的号段区间，例如：
# mycat1: ORDERS.MINID=10001  ORDERS.MAXID=20000
# mycat2: ORDERS.MINID=20001  ORDERS.MAXID=30000
```

**接入方式对应用完全透明**：INSERT 不带 id 列，Mycat 自动填充并支持 `LAST_INSERT_ID()` 取回：

```sql
-- 通过 8066 插入，id 由 Mycat 全局序列生成
INSERT INTO orders(user_id, amount) VALUES (100, 50.0);
SELECT LAST_INSERT_ID();
-- → 7234567890123456789（数据库模式下）或 10001（本地模式下）
```

```java
// JDBC：与单库 MySQL 用法完全一致，无需任何改造
PreparedStatement ps = conn.prepareStatement(
    "INSERT INTO orders(user_id, amount) VALUES (?, ?)",
    Statement.RETURN_GENERATED_KEYS);
ps.setLong(1, 100L);
ps.setBigDecimal(2, new BigDecimal("50.00"));
ps.executeUpdate();
ResultSet keys = ps.getGeneratedKeys();   // Mycat 把序列值塞进 generated keys
keys.next();
long id = keys.getLong(1);
```

# 4. Snowflake：算法与落地

## 4.1 位布局

```text
64 位 long 型 ID：

┌─┬──────────────────────────┬──────────┬──────────┐
│0│     41 位时间戳 (ms)      │ 10 位机器 │ 12 位序列 │
└─┴──────────────────────────┴──────────┴──────────┘

41 位时间戳：自定义纪元起可用约 69 年
10 位机器码：workerId（常拆成 5 位 dataCenterId + 5 位 workerId）= 1024 节点
12 位序列号：同一毫秒内 4096 个 ID → 单节点理论峰值 409.6 万/秒
```

**同一毫秒内序列用尽时自旋等待下一毫秒**——这是 Snowflake 的背压机制，意味着单节点吞吐有硬上限但绝不需要任何 RPC。ID 整体按时间趋势递增，完美契合 InnoDB 聚簇索引。

## 4.2 workerId 分配问题

Snowflake 唯一的协调需求是**每个生成节点的 workerId 全局唯一**。Mycat 1.6 的本地模式实际是"简化版雪花"：用 `sequence_conf.properties` 里互不重叠的号段区间替代了机器码协调——配置文件就是 workerId 登记表，代价是扩容/缩容要人工规划区间。Mycat2 与工业级实现（百度 UidGenerator、美团 Leaf-snowflake）的做法：

```java
// 常见方案：启动时从 DB/ZK 抢占注册 workerId（示意）
public long allocateWorkerId(String instanceName) {
    // INSERT INTO worker_registry(instance, worker_id, expire_time)
    // 依赖唯一键抢占，或 ZK 顺序节点 + 临时节点自动回收
    return registryDao.register(instanceName);
}
```

> **行业实践**：自建 Mycat 集群时，**workerId/号段区间的分配表必须由配置管理平台统一维护**，靠 Excel 记"哪台机器用哪段"是翻车之母——新扩容一台机器忘了登记号段，与老机器区间重叠，重复主键事故在各大社区案例里反复上演。

## 4.3 时钟回拨处理

Snowflake 依赖本地时钟，NTP 校时/虚拟机迁移导致的时钟回拨会让"未来时间"倒退回已发号区间，产生重复 ID。工业级容错策略收敛为四档：

```text
1. 回拨极小（≤5ms）   → 阻塞自旋等到追平（Leaf-snowflake 默认）
2. 回拨较小          → 借用"未来时间"：扩展位记借用量，事后归还（UidGenerator）
3. 回拨较大（>1s）    → 直接抛异常拒绝服务，等人工/NTP 纠正
4. 预防             → 关闭 NTP 平滑校时改用渐进调整；容器场景锁定宿主机时钟源
```

Mycat 1.6 的本地序列**不依赖时钟**（纯号段），天然免疫回拨问题；Mycat2 的 `SnowflakeSequenceHandler` 需要按上表在应用侧或运维侧自行保障时钟纪律。**设计考量**：

- **金融交易号要求"严格单调"时不要用 Snowflake**：它只保证趋势递增，多节点并发生成的 ID 到达顺序可能乱序
- **ID 位数要预留够**：41 位时间戳从自定义纪元算 69 年，纪元设成项目立项日而不是 1970，可用年限立刻延长
- **暴露风险**：ID 可解出生成时间与机器号，对外展示单号时通常要再做一次混淆或映射（如订单号 = 雪花 ID + 随机盐的编码）

# 5. 号段方案：数据库序列表

**Mycat 1.6 的数据库模式**就是在一张序列表上"批量领号段"，机制与美团 Leaf-segment 同构：

```sql
-- 序列表结构（Mycat 1.6 数据库模式）
CREATE TABLE MYCAT_SEQUENCE (
    name        VARCHAR(50) NOT NULL COMMENT '序列名(表名大写)',
    current_value BIGINT    NOT NULL COMMENT '当前已分配到的值',
    increment   INT         NOT NULL DEFAULT 100 COMMENT '每次领取步长',
    PRIMARY KEY (name)
) ENGINE=InnoDB;

INSERT INTO MYCAT_SEQUENCE VALUES ('ORDERS', 0, 1000);
```

```properties
# conf/sequnce_handler.properties —— 启用数据库模式
sequnceHandlerType=1
# 序列表所在数据源（走 JDBC 直连，不经过 8066 路由）
dataSourceURL=jdbc:mysql://192.168.1.100:3306/mycat_seq?useServerPrepStmts=false&useCursorFetch=true
dataSourceUser=mycat
dataSourcePassword=mycat123
```

工作原理：`UPDATE ... SET current_value = current_value + increment` 然后读回新值，**一次 DB 交互领取一段 increment 个号**，Mycat 实例在内存里逐个分配；号段用完再领下一段。1000 步长意味着平均 1000 次 INSERT 才访问一次序列表，性能足够。

```text
Mycat 实例 A                    序列表(mycat_sequence)
     │  UPDATE current_value=1000  │
     │ ──────────────────────────→ │
     │ ←── 领到区间 1~1000 ─────── │  内存逐号分配：1,2,3,...
     │                             │
Mycat 实例 B 同时 UPDATE           │  领到区间 1001~2000（行锁串行保证不重叠）
```

> **注意**：序列表的 UPDATE 靠**行锁**保证并发安全，这张表成为全局单点——所有发号实例高频领号最终都打在同一行上。对策是加大 increment（减少领号频率）与把序列表放在独立的高规格实例上。**实例重启会丢弃当前未用完的号段**，所以号段模式"趋势递增"但必然有跳号，把 id 当"总数"统计业务规模会严重失真。

# 6. 选型与最佳实践

| 场景 | 推荐方案 | 理由 |
|------|---------|------|
| 通用分库分表主键 | 数据库号段（Mycat 模式 1） | 实现简单、严格递增、无 workerId 协调 |
| 高并发写入（>5 万 TPS） | 本地号段（模式 0）/ Snowflake | 零 RPC，本地分配 |
| 多语言 heterogeneous 系统共用发号 | 独立发号服务（Leaf/UidGenerator 思想） | 发号能力与中间件解耦 |
| 需要 ID 携带业务含义（分库路由） | Snowflake + 低位编码分片键 | 由 ID 直接算出分片，免去二次查询 |
| 强安全要求、对外暴露 | 号段 + 映射混淆 | 防爬取业务规模 |

**最佳实践清单**：

1. **全局序列的 DDL 前置**：换/加序列配置（如从本地改数据库模式）涉及号段交接，必须低峰期操作并保留旧区间避免重叠
2. **workerId/号段登记集中化**：任何"人工保证唯一"的配置都要进 CMDB
3. **NTP 纪律**：时钟源、同步策略（渐进校正）写进部署基线；时钟回拨告警接入监控
4. **不要把 id 当业务序列**：跳号是特性不是缺陷；需要"连续单号"的场景（发票号）应该用独立的业务单号生成器而不是主键
5. **压测覆盖领号路径**：数据库模式下 increment 设置过小时，序列表行锁会成为整个集群的写入总闸

# 7. 小结

**分布式主键的本质矛盾是"唯一性协调成本"与"有序性收益"的平衡**：UUID 用零协调换来无序与索引膨胀（除 v7 外基本退出分片主键选型）；号段方案以"每次领一段"摊薄中心协调成本，是 Mycat 生态的默认答案；Snowflake 把协调收敛到 workerId 分配这一件事上，换来本地无限发号与趋势递增，代价是时钟纪律。Mycat 1.6 提供本地/数据库/ZK 三种内置序列实现，应用侧只需 INSERT 不写主键 + `LAST_INSERT_ID()` 取回；落地时盯住三件事——号段/workerId 的集中登记、序列表单点的步长设计、以及"趋势递增允许跳号"的业务预期管理。