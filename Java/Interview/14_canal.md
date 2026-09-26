# 1. 知识点
- **Canal概述**: 阿里巴巴开源的MySQL binlog增量订阅&消费组件，基于MySQL slave协议伪装成从库获取binlog
- **Canal名称由来**: Canal意为管道/运河，寓意像运河一样连接MySQL和其他数据系统，音译"开呢哦"
- **工作原理**: Canal Server伪装成MySQL Slave，向MySQL Master发送dump协议请求，Master将binlog事件推送给Canal，Canal解析后发送给MQ或直接消费
- **MySQL主从复制原理**: Master将所有变更记录写入binlog → Slave的IO线程从Master拉取binlog并写入relay log → Slave的SQL线程执行relay log中的事件
- **Canal差异**: Canal相当于特殊的Slave，只接收binlog不执行relay log，而是解析binlog事件后推送给下游系统
- **Canal Server**: Canal核心服务，负责连接MySQL、接收解析binlog、事件过滤、数据分发等
- **Canal Instance**: Canal Server中的一个实例，对应一个MySQL数据源(一个数据库连接)，每个Instance有独立的EventParser/EventSink/EventStore
- **Canal内部组件**: EventParser(连接MySQL接收binlog)、EventSink(过滤/归并/分发)、EventStore(存储binlog事件供消费)、EventConsumer(Canal Client)
- **Binlog格式(ROW)**: row模式记录每行数据的变化，包含before和after的完整数据。INSERT有after值，DELETE有before值，UPDATE有before和after值，数据最精确但日志量大
- **Binlog格式(STATEMENT)**: 记录原始的SQL语句原文，日志量小，但某些场景(如UUID()、NOW()函数)主从不一致
- **Binlog格式(MIXED)**: 混合模式，默认使用STATEMENT，特殊场景(如UUID、包含LIMIT的DELETE)自动切换ROW，是MySQL默认推荐模式
- **Canal推荐ROW模式**: ROW模式能获取到每行变化的具体数据(before/after)，适合数据同步场景；STATEMENT模式需要解析SQL，复杂且不准确
- **Canal Admin**: Canal的管理控制台，提供可视化操作：Canal Server和Instance的配置管理、监控、HA管理
- **Canal HA(高可用)**: 基于ZooKeeper实现主备切换。多个Canal Server竞争ZK的临时节点，获得锁的成为主Server。主挂掉后备选接管
- **Canal HA Instance切换**: 每个Instance在ZK上注册，Canal Server故障时Instance自动迁移到正常Server
- **数据过滤**: Canal支持表级过滤(白名单/黑名单)、字段过滤(只同步需要的列)、正则匹配表名
- **Canal数据输出**: 支持直接推送到Kafka/RocketMQ/RabbitMQ/Pulsar等MQ，也支持自定义Client直连(protobuf协议)
- **Canal Client使用**: Java客户端通过TCP连接Canal Server，订阅指定Instance，消费binlog事件。事件包含数据库名、表名、事件类型(INSERT/UPDATE/DELETE)、行数据
- **Canal Kafka适配**: Canal将binlog事件转化为JSON格式发送Kafka Topic，topic名默认=库名.表名。消费者可以从Kafka消费变更事件
- **Canal to MQ消息格式**: JSON消息包含header(binlog位点)和data(变更数据)，支持FlatMessage(展开一行)和CanalEntry两种格式
- **特定顺序消费**: 通过配置Kafka分区键(partition key)保证同一表的数据发送到同一分区，利用Kafka分区内有序保证消费顺序
- **数据一致性保证(最终一致性)**: Canal保证at-least-once投递。消费端需要：1)记录消费位点(offset/binlog position) 2)幂等处理(MD5校验或唯一ID去重) 3)异常重试
- **消费方MD5校验**: 消费端对binlog数据行计算MD5，与上次消费数据对比，重复则跳过
- **Canal与业务应用场景**: MySQL→Redis缓存同步、MySQL→Elasticsearch搜索引擎同步、异构数据同步、实时数据备份、实时数据报表、CQRS(命令查询职责分离)
- **Canal与Debezium对比**: Debezium基于Kafka Connect框架，支持多种数据库(MySQL/PostgreSQL/MongoDB/SQL Server等)，功能更全但引入Kafka Connect依赖较重。Canal专注于MySQL，轻量简单
- **Canal与DataX对比**: DataX是离线全量数据同步工具(批量ETL)，Canal是在线增量数据同步(实时)。通常DataX做全量迁移+Canal做增量同步

# 2. 面试题

## 2.1 Canal的工作原理是什么？
Canal伪装为MySQL的Slave节点：
1) Canal向MySQL Master发送dump协议请求，注册为Slave
2) MySQL Master收到请求后，将binlog事件实时推送给Canal(与主从复制一致)
3) Canal的EventParser接收并解析binlog二进制流，转为CanalEntry事件对象
4) EventSink过滤(按库/表/字段规则)并归并数据
5) EventStore存储事件(内存RingBuffer)，供客户端拉取
6) Canal Client通过TCP连接消费事件，获取变更数据
整个过程对MySQL无侵入(只需开启binlog)，对性能影响极小(像普通Slave一样)。

## 2.2 Binlog的三种格式有什么区别？为什么Canal推荐ROW？
| 格式 | 内容 | 优点 | 缺点 |
|------|------|------|------|
| ROW | 记录每行数据before/after值 | 数据精确，包含完整行信息 | 日志量大 |
| STATEMENT | 记录SQL语句原文 | 日志量小 | 函数(UUID/NOW)导致不一致 |
| MIXED | 混合，默认STATEMENT，特殊转ROW | 折中 | 不确定解析格式 |

Canal推荐ROW模式原因：ROW模式提供每行变更的before/after完整数据，可以直接用于数据同步(如Redis缓存更新知道最新值、ES索引更新知道文档内容)。STATEMENT模式需要解析SQL语句，复杂度高、准确性差，且无法获取函数计算后的具体值。

## 2.3 Canal如何保证消息的顺序消费？
MySQL binlog本身是严格有序的(事务内有序)，Canal解析后按binlog顺序投递。

对于Kafka适配：通过配置`canal.mq.partition`指定分区策略：
- 按`库名.表名`分区：同一表的所有变更进同一分区，保证单表顺序
- 按主键hash分区：同一行的变更进同一分区，保证单行顺序
- 按事务ID分区：同一事务的所有变更进同一分区

Kafka分区内天然有序，消费者按分区顺序消费即可保证消息顺序消费。

## 2.4 Canal如何实现高可用(HA)？
Canal HA基于ZooKeeper：
1) 多个Canal Server节点(通常2个)同时启动，但只有主Server进行binlog消费
2) 每个Server启动时在ZK的`/otter/canal/destinations/{instance}/cluster/`下创建临时顺序节点
3) 序号最小的节点对应的Server成为主(Primary)，开始从MySQL拉取binlog
4) 主Server定时更新ZK节点的心跳信息(binlog位点等)
5) 主Server宕机时，临时节点自动删除，其他Server监听ZK变化，其中最小序号的Server自动升级为主
6) 新主从ZK获取上次记录的binlog位点，继续消费(不允许丢数据)

## 2.5 如何利用Canal实现MySQL到Redis的数据同步？
1) 搭建Canal Server，配置MySQL数据源(开启ROW格式binlog)
2) Canal Client订阅目标表的binlog事件
3) 根据事件类型处理缓存：
   - INSERT：将新数据写入Redis(如Hash结构存储行数据)
   - UPDATE：更新Redis中的对应数据
   - DELETE：删除Redis中的对应Key
4) 缓存Key设计：`{表名}:{主键ID}`，如`user:1001`
5) 幂等处理：Canal可能重复投递，每条消息带上binlog位点或行数据MD5，消费端去重
6) 保证最终一致性：消费失败重试(MQ重试队列)，记录消费位点便于找回

## 2.6 使用Canal同步数据到ES需要注意什么？
1) **ROW格式**：确保MySQL binlog格式为ROW，才能获取变化后的完整数据
2) **全量同步前置**：先用DataX/Logstash做全量同步到ES，Canal从binlog位点开始增量同步
3) **幂等处理**：根据ES文档ID(对应MySQL主键)做INSERT/UPDATE(Upsert操作)和DELETE
4) **批量处理**：收集多条变更(如100条或1秒)批量推送ES，减少ES写入压力(Bulk API)
5) **位点记录**：记录消费到的binlog位点，便于重置消费位置
6) **数据一致性校验**：定期对比MySQL和ES数据量，或抽样做数据MD5校验

## 2.7 Canal与Debezium如何选择？
| 维度 | Canal | Debezium |
|------|-------|----------|
| 专注度 | 仅MySQL | MySQL/PG/Mongo/SQL Server等 |
| 依赖 | 轻量，只需ZK(MQ可选) | 重，依赖Kafka Connect |
| 社区 | 阿里主导，中文文档多 | Red Hat主导，国际化 |
| MQ适配 | RocketMQ/Kafka/RabbitMQ | 主要Kafka |
| 控制台 | Canal Admin | Debezium UI |
| 适用 | MySQL为主的中文技术栈 | 多数据库、Kafka技术栈 |

选型：如果只用MySQL+中文团队优先Canal；若已有Kafka生态或多数据库支持需求选用Debezium。
