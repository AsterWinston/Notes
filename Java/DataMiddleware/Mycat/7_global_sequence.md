# 1. 概述

**Mycat 的全局序列（Global Sequence）是分布式主键背后的发号引擎**：它不只服务于表主键——业务单号、批次号、消息 seq 等任何"全局唯一递增数"都可以向它申请。第 6 篇讲"方案怎么选"，本篇讲"Mycat 1.6 原生实现的机制与运维"。

Mycat 1.6 通过 `conf/sequence_conf.properties` 与 `conf/sequnce_handler.properties`（注意官方文件名的拼写就是 sequnce）把序列模式固定为三选一：

| handlerType | 模式 | 存储 | 递增性 | 重启行为 |
|-------------|------|------|--------|---------|
| 0 | 本地文件 | sequence_conf.properties | 趋势递增 | 读回 conf 文件中记录的当前值 |
| 1 | 数据库（默认推荐） | MYCAT_SEQUENCE 表 | 严格递增（区间连续） | 丢弃当前未用完号段 → 跳号 |
| 2 | ZooKeeper | ZK 持久节点 | 严格递增 | 丢弃当前号段 → 跳号 |

> **注意**：三种模式**不能按表混用**——handlerType 是全局的，一张集群要么全走本地要么全走数据库。这是 Mycat 与"每表可选发号策略"的 ShardingSphere 明显的设计理念差异：Mycat 把序列当作全局基础设施，配置一次全局生效。

Mycat 2.x 走的是另一条路：序列定义内联在 schemas.json 的 `sequences` 段，支持 Snowflake/数据库/自定义 Handler 并存，按表引用，见第 5 节。

# 2. 数据库号段模式（模式 1）

**核心表结构与领号 SQL 是必背内容**——面试问"Mycat 序列原理"，答案就是这一段：

```sql
-- 序列元数据表（放在独立的 MySQL 实例上，库名 mycat_seq）
CREATE TABLE MYCAT_SEQUENCE (
    name          VARCHAR(50) NOT NULL COMMENT '序列名：MYCATSEQ_表名大写',
    current_value BIGINT      NOT NULL COMMENT '已分配到的水位线',
    increment     INT         NOT NULL DEFAULT 100 COMMENT '号段步长',
    PRIMARY KEY (name)
) ENGINE=InnoDB;

INSERT INTO MYCAT_SEQUENCE (name, current_value, increment)
VALUES ('MYCATSEQ_ORDERS', 0, 1000);
```

Mycat 领号时执行的原子更新（1.6 源码 `DBSeqHandler` 的实际语句）：

```sql
-- 事务内两步：更新 + 读回，靠行锁串行化并发实例
UPDATE MYCAT_SEQUENCE SET current_value = current_value + increment
WHERE name = 'MYCATSEQ_ORDERS';

SELECT current_value FROM MYCAT_SEQUENCE WHERE name = 'MYCATSEQ_ORDERS';
-- 返回新水位线 → 本次领到的区间是 (新值 - increment, 新值]
```

```properties
# conf/sequnce_handler.properties（handlerType=1 时生效）
sequnceHandlerType=1
# 序列表所在库的 JDBC 直连（绕过 8066，避免发号被路由逻辑牵连）
dataSourceURL=jdbc:mysql://192.168.1.100:3306/mycat_seq?useServerPrepStmts=false&useCursorFetch=true&autoReconnect=true
dataSourceUser=mycat
dataSourcePassword=mycat123
```

**两个容易被忽略的细节**：

1. **序列名默认规则是 `MYCATSEQ_ + 表名大写`**，表上 `autoIncrement="true"` 时 Mycat 自动按这个名字取号；想手工取指定序列可以用 `SELECT NEXT VALUE FOR MYCATSEQ_ORDERS;` 或 `SELECT MYCATSEQ_ORDERS.NEXTVAL;`
2. **领号 JDBC 串里的 `useServerPrepStmts=false` 不是可选装饰**：1.6 早期版本用服务端预编译读序列表会触发游标相关的坑，配置模板里带注释提醒，改连接串时别把它删了

**性能特征**：单实例对序列表的访问频率 = `TPS / increment`。步长 1000、写入 5 万 TPS 时，序列表每秒只有 50 次 UPDATE——行锁压力可控。**步长过小的症状**是序列表 UPDATE 排队、INSERT 出现零星毛刺；**步长过大的症状**是实例重启跳号多（每个实例最多丢弃一个号段）、以及号段在内存里长期不用导致水位线虚高。

> **行业实践**：序列表独立于业务库实例部署，且给该实例配双主/半同步——序列表挂 = 全局写入挂。多数公司进一步把发号从 Mycat 里拆出去自建发号服务（借鉴 Leaf），让 Mycat 只消费发号服务的号段，避免"改发号逻辑要重启中间件"。

# 3. 本地文件模式（模式 0）与时间戳精度

`handlerType=0` 时号段完全记在 `sequence_conf.properties`，Mycat 周期性刷写回该文件。**它的真实定位是开发/测试与"不能接受任何外部依赖"的极端场景**：多台 Mycat 的号段区间必须人工写成互不重叠，重启后从文件恢复水位线。

本地模式的一个历史大坑是**毫秒时间戳精度问题**：1.6 早期用 `System.currentTimeMillis()` 配合 1000 以内自增序列拼 id，同一毫秒内并发高时序列回绕、或多实例同毫秒生成相同 id。修复思路（1.6.5+ 与 Mycat2 均如此）是把时间戳推进到与"上次生成时刻 + 本条序列"绑定：

```java
// 时间戳精度修复示意（简化自 1.6 本地序列源码逻辑）
public synchronized long nextId() {
    if (currentTS < lastTS) {              // 检测到时钟回拨
        currentTS = lastTS;                // 借用未来时间，而不是等待
        if (sequence >= maxSequence) {     // 借来的毫秒序列也会用尽
            currentTS = waitNextMillis(lastTS);
        }
    } else if (currentTS == lastTS) {
        if (++sequence > maxSequence) {    // 同毫秒序列溢出
            currentTS = waitNextMillis(currentTS);
            sequence = 0;
        }
    } else {
        sequence = 0;
    }
    lastTS = currentTS;
    return (currentTS - startTimestamp) * MAX_SEQUENCE + sequence;
}
```

**`waitNextMillis` 是忙等自旋**：单实例单毫秒的产能上限就是 `maxSequence`（本地模式默认 1000），压测打满时会观察到 Mycat CPU 空转在这个方法上——这不是 bug，是设计上的限流。

# 4. ZooKeeper 模式（模式 2）

`handlerType=2` 把号段水位线从 MySQL 挪到 ZK 节点（`/Mycat/sequence/{序列名}`），领号动作变成一次 CAS 写：

```properties
# conf/sequnce_handler.properties
sequnceHandlerType=2
# ZK 集群地址与命名空间（server.xml 里另有 clusterHeight/clusterNode 等全局集群配置）
zookeeper.dataPath=/Mycat/sequence
zookeeper.sessionTimeoutMs=60000
zookeeper.connectionTimeoutMs=5000
zookeeper.maxRetries=3
zookeeper.namespace=mycat_sequence
```

**动机是把"发号强依赖"从 MySQL 换成 ZK**——很多公司本来就用 ZK 跑 Mycat 集群模式（切换通知、全局表心跳协调），序列顺手放 ZK 少维护一套存储。CAS 冲突时重试读取再写，高并发领号下冲突率随实例数上升，所以 ZK 模式的吞吐天花板低于数据库模式，但一致性更强（不存在"序列表被误 DROP"事故）。

**设计考量**：

- **已有成熟 ZK 运维体系的团队**：模式 2 少一个 MySQL 单点，是合理选择
- **ZK 不是为高写入设计的**：领号频率高（步长小）会放大 ZK 压力，模式 2 务必配大步长
- **ZK 断连时会话内本地号段照常用**，用完取不到新号段才阻塞业务——"ZK 抖动 → 写入毛刺"的链路要在监控上关联起来

# 5. Mycat2 的序列形态

Mycat2 把序列变成 schema 级声明、按表/序列名引用，Snowflake 与数据库模式可以共存：

```json
{
  "schemas": {
    "order_db": {
      "sequences": {
        "order_no_seq": {
          "clazz": "io.mycat.router.sequence.SnowflakeSequenceHandler",
          "properties": {
            "workerId": 1,
            "dataCenterId": 1,
            "startTimestamp": 1700000000000
          }
        },
        "user_id_seq": {
          "clazz": "io.mycat.router.sequence.DatabaseSequenceHandler",
          "properties": {
            "url": "jdbc:mysql://192.168.1.100:3306/mycat_seq",
            "user": "mycat",
            "password": "mycat123",
            "tableName": "sequence",
            "step": 1000
          }
        }
      },
      "tables": {
        "orders": {
          "type": "sharding",
          "autoIncrement": true,
          "sequence": "order_no_seq"
        }
      }
    }
  }
}
```

```sql
-- 两种使用入口与 1.6 对齐：自动填充 + 显式取名
INSERT INTO orders(user_id, amount) VALUES (100, 50.0);
SELECT NEXT VALUE FOR order_no_seq;
```

2.x 还支持实现 `SequenceHandler` 接口注入自定义发号（如对接 Redis INCR 集群或公司统一发号服务），热加载配置即可生效，不用重启——这是相对 1.6 "改序列配置必须重启"的实质性改进。

# 6. 应用接入方式汇总

```java
// 方式一：什么都不做（推荐）——表标 autoIncrement=true，Mycat 自动填充
// JDBC 取回主键
Statement st = conn.prepareStatement(sql, Statement.RETURN_GENERATED_KEYS);
st.executeUpdate();
st.getGeneratedKeys();          // 拿到 Mycat 序列生成的 id

// 方式二：SQL 内显式引用序列
// INSERT INTO orders(id, ...) VALUES (NEXT VALUE FOR MYCATSEQ_ORDERS, ...);

// 方式三：先取号再写入（发号与落库解耦，便于把 id 同步给缓存/ES）
ResultSet rs = stmt.executeQuery("SELECT MYCATSEQ_ORDERS.NEXTVAL");
long id = rs.next() ? rs.getLong(1) : -1;
```

> **行业实践**：方式三在多语言系统中也最通用——**先发号后写库**意味着 id 可以在写事务开始前就流入消息、缓存与搜索引擎，避免"写完再查回主键"的额外往返。代价是发号与写入不再原子，失败的 INSERT 同样消耗号段，跳号更密。

# 7. 性能与运维要点

```text
各模式性能量级参考（单表发号，increment=1000）：

模式0 本地     ~百万/秒   纯内存，受 waitNextMillis 自旋限流
模式1 数据库   ~10万+/秒  领号摊薄后主要成本在内存分配
模式2 ZK      ~万级/秒    CAS 冲突与 ZK 写延迟主导
```

**运维清单**：

1. 监控序列表 `current_value` 增速与业务写入速率是否匹配（虚高 = 大量号段被丢弃，检查重启频率）
2. 数据库模式下 `MYCAT_SEQUENCE` 表禁止走 Mycat 路由（用独立 JDBC 直连），也别被业务误 TRUNCATE
3. 号段交接：换模式/换区间时先让旧区间"只读耗尽"，再切换新区间，避免重叠
4. 9066 管理端口 `SHOW @@SEQUENCE`（1.6.7+）可查看各序列当前水位与缓存剩余
5. 实例上下线登记：本地模式改 conf 文件、Snowflake 改 workerId，都要纳入发布流程，杜绝"手工改一边忘另一边"

# 8. 小结

**Mycat 1.6 全局序列是"三种模式选一、全局生效"的发号器**：数据库号段模式以序列表行锁 + 批量领号段成为生产主流，本地模式胜在无依赖但受毫秒序列自旋限流、且有历史时间戳精度坑（修复靠"借未来时间 + 忙等下一毫秒"），ZK 模式用 CAS 换掉 MySQL 单点但吞吐最低。接入层永远只有三招——自动填充取 `LAST_INSERT_ID`、SQL 内 `NEXT VALUE FOR`、先取号后写库。Mycat2 把序列内联进 schemas.json 并支持按表混用与热加载，但存量大厂的既有系统仍以 1.6 数据库模式为主，并把"号段区间登记、序列表独立部署、跳号预期管理"列为序列运维三板斧。
