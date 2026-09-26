# 1. 知识点
- **ShardingSphere生态**：Apache顶级项目，包含ShardingSphere-JDBC（客户端分片）、ShardingSphere-Proxy（服务端代理）、ShardingSphere-Sidecar（云原生Sidecar）三大产品
- **ShardingSphere-JDBC**：轻量级Java框架，以jar包形式嵌入应用，增强JDBC驱动，无代理层无网络开销；对代码有侵入（需配置分片规则）
- **ShardingSphere-Proxy**：独立部署数据库代理，对应用透明（应用连接Proxy即可），兼容MySQL/PostgreSQL协议，可异种语言接入
- **JDBC vs Proxy选择**：JDBC性能高但Java专属，需改配置部署；Proxy对应用零侵入，适合异构语言或多应用统一管理，多一跳网络开销
- **分片键**：数据分布的维度字段，决定数据路由到哪个数据源和哪张表
- **分片键选择原则**：查询频繁的字段优先、数据分布均匀避免热点、最好是业务主键（如用户ID/订单ID）、避免NULL值
- **标准分片策略Standard**：单一分片键，支持PreciseShardingAlgorithm（精确分片，=、IN）和RangeShardingAlgorithm（范围分片，BETWEEN）
- **复合分片策略Complex**：多个分片键，需ShardingStrategyConfiguration配置ComplexShardingAlgorithm，适用于多维度共同决定分片
- **行表达式分片Inline**：简洁的Groovy表达式配置分片规则，如t_order_$->{user_id % 4}，适用于简单取模分片，无需编写Java算法类
- **强制分片Hint**：通过HintManager强制指定分片值和路由目标，绕过SQL解析和常规分片策略，适合特殊场景（如DBA手动迁移数据）
- **广播表Broadcast**：所有数据源中都存在相同的全量数据（如字典表、配置表），每个分片库都存一份，写操作同步所有节点
- **绑定表Binding Table**：具有关联关系的多张分片表（如order和order_item），使用相同分片键，绑定后JOIN查询避免笛卡尔积
- **分布式主键生成**：内置雪花算法Snowflake（默认）、UUID；也支持自定义KeyGenerateAlgorithm（如号段模式）
- **ShardingSphere雪花算法**：workerId自动生成（基于IP或PID），时钟回拨有容错处理（最大容忍回拨毫秒可配）
- **分布式事务-Seata集成**：通过Seata AT模式实现BASE柔性事务；支持XA强一致性事务，配置选择事务类型
- **BASE事务**：ShardingSphere默认集成Seata AT模式，通过全局锁+undo日志实现最终一致性
- **XA强一致**：支持基于Atomikos/Narayana等XA事务管理器，适用于对一致性要求极高的场景（代价：性能开销大）
- **读写分离**：配置主从数据源，写操作路由到主库，读操作路由到从库，支持一主多从
- **读写分离负载均衡**：多从库支持轮询Round Robin和随机Random两种负载均衡策略；从库可设权重
- **从库延迟容错**：可配置强制走主库的Hint、或通过心跳检测根据延迟自动摘除从库
- **数据脱敏**：内置加密算法（AES/MD5），自动对配置的列在写入时加密、读取时解密；查询WHERE条件自动转换密文
- **数据脱敏场景**：手机号、身份证、银行卡号等敏感字段，存储加密存储，应用层透明加解密
- **影子库**：压测流量路由到影子库（镜像表结构），与生产数据隔离，压测不影响真实数据
- **影子表**：同一库中创建前缀/后缀隔离的镜像表用于压测，逻辑表名相同物理表名不同
- **弹性伸缩Scale Out**：数据在线迁移至新分片节点，支持扩容缩容，基于集群计算分片规划
- **弹性伸缩原理**：全量数据同步（存量数据迁移）+增量binlog订阅（增量数据追赶）→数据校验→切换路由，实现平滑扩缩容
- **Hint强制路由**：编程式HintManager.getInstance().setDatabaseShardingValue()，强制指定数据源和表，适用于非SQL解析路由场景
- **分片算法SPI扩展**：实现ShardingAlgorithm接口，通过SPI机制注册自定义分片算法
- **ShardingSphere配置方式**：YAML配置文件、Java API配置、Spring Boot Starter自动配置、ZooKeeper/Etcd/Nacos配置中心动态配置
- **逻辑表vs物理表**：逻辑表名（如t_order）为应用使用的统一表名，物理表名（如t_order_0, t_order_1）为实际分片后的表
- **结果归并**：分片查询后需将多结果集归并：流式归并（内存友好）、内存归并（排序/分组/分页等）、装饰者归并（聚合函数）
- **分页优化**：分片分页时性能挑战，ShardingSphere支持流式分页归并和内存归并（极限情况下性能降级但功能正确）
- **SQL支持范围**：支持DML全部和大部分DQL/DDL/DCL，复杂子查询、跨库JOIN、存储过程等有限制
- **治理中心**：通过注册中心（ZooKeeper等）实现配置动态管理、元数据管理、数据库/表节点上下线监听

# 2. 面试题

## 2.1 ShardingSphere生态有哪些组件？各有什么特点？
ShardingSphere-JDBC：轻量级Java框架，以jar包嵌入应用，无代理层无网络开销，性能最高，但仅限Java且对应用有配置侵入。ShardingSphere-Proxy：独立部署数据库代理，兼容MySQL/PostgreSQL协议，对应用完全透明，支持多语言接入，多一层网络转发。ShardingSphere-Sidecar（规划中）：云原生Sidecar模式，与Service Mesh结合。选择上单Java应用用JDBC，多异构语言或统一管理用Proxy。

## 2.2 什么是逻辑表和物理表？它们的关系是什么？
逻辑表是应用侧使用的统一表名（如t_order），物理表是分片后在数据库中实际存在的表（如t_order_0、t_order_1、t_order_2、t_order_3）。ShardingSphere负责将逻辑表的操作SQL路由到正确的物理表，对应用层屏蔽分片细节。

## 2.3 分片键的选择原则有哪些？
1）选择查询中最常出现在WHERE条件中的字段，确保单次查询尽量落在一个分片上；2）数据分布均匀，避免热点分片（如按用户ID哈希，不能用地区/日期这种易倾斜的字段）；3）不能为NULL（NULL无法哈希路由）；4）尽量与业务主键一致，或通过基因法将分片键信息融入主键。

## 2.4 什么是绑定表？解决了什么问题？
绑定表是将使用相同分片键且存在关联关系的多张分片表进行绑定配置，如t_order和t_order_item都按user_id分片且JOIN查询时WHERE条件中包含user_id。未绑定时，关联查询会产生笛卡尔积（每个分片组合都查一遍），绑定后只需在相同分片的分片组合间关联，大幅减少无效查询。

## 2.5 ShardingSphere-JDBC和ShardingSphere-Proxy如何选择？
- 选JDBC：Java技术栈、追求极致性能（无网络跳转）、能接受修改配置后重启应用、不需要异种语言接入
- 选Proxy：多语言技术栈混合、需要透明代理（无代码侵入）、需要统一数据治理入口、对性能损耗容忍度高（多一跳网络延迟）
- 混合部署：JDBC + Proxy组合，Proxy做治理统一入口，JDBC做高性能分片

## 2.6 ShardingSphere支持哪些分片策略？
- Standard（标准）：单一分片键，支持=、IN、BETWEEN AND
- Complex（复合）：多分片键共同决定分片
- Inline（行表达式）：Groovy表达式简洁配置，如$->{id % 4}
- Hint（强制）：编程式指定分片值，绕过SQL解析
- None：不分片

Standard和Inline最常用，Hint用于特殊场景如DBA操作。

## 2.7 ShardingSphere如何实现读写分离？
配置主库和多从库数据源，写操作（INSERT/UPDATE/DELETE）自动路由到主库，读操作（SELECT）路由到从库。支持一主多从，从库间通过Round Robin或Random负载均衡。可配置Hint强制读主库保证一致性。从库延迟检测配合心跳机制，自动摘除延迟过大的从库。

## 2.8 ShardingSphere的分布式事务方案有哪些？
1）Seata集成（BASE AT模式）：默认方案，通过全局锁+undo日志实现最终一致性，性能较好，适合多数互联网场景。2）XA强一致事务：基于Atomikos等XA事务管理器，两阶段提交保证强一致，性能开销大，适合对一致性要求极高的场景。通过配置transactionType切换。

## 2.9 数据脱敏是如何工作的？
配置需要脱敏的列和加密算法（AES/MD5等），ShardingSphere在写入时自动加密原始值为密文存储；查询时自动解密返回明文给应用，对应用层透明。同时支持查询条件自动转换（WHERE列名=明文自动转为密文条件），即查询时也可用明文条件。

## 2.10 影子库/影子表的作用是什么？
用于生产环境压力测试，将压测流量路由到影子库/影子表，与真实数据完全隔离。影子库是单独的数据库实例镜像；影子表是同一库中通过前缀/后缀隔离。压测时不影响线上数据，压测完成后可快速清理，是保障生产环境安全的重要手段。

## 2.11 ShardingSphere实现弹性伸缩（Scale Out）的原理是什么？
当需要扩缩分片节点时，数据需要重新分布。流程：1）计算旧分片到新分片的规规则并初始化目标库表结构；2）全量同步存量数据（dump旧分片全部数据至目标分片）；3）增量同步（订阅binlog同步增量变更）；4）数据一致性校验；5）切换路由规则到新分片，下线旧分片。全程在线，业务感知极低。

## 2.12 广播表是什么？适用什么场景？
广播表是在每个数据源分片中都存储全量数据的表，即每片库都有一份相同数据。所有分片的广播表数据完全一致，写操作需要同步到所有分片。适用于数据量小、变动频率低、但查询频繁的公共基础表，如字典表、配置表、国家地区表。优点：关联查询时不需要跨库，性能好。

## 2.13 分片查询的结果归并面临哪些挑战？
多分片查询结果在ShardingSphere合并时需处理：排序（各分片merge排序）、分组聚合（COUNT/SUM/AVG需合并分片结果再计算——AVG不能直接AVG of AVG）、分页（深分页需从每个分片取limit+offset条再归并排序取前limit，性能下降）、DISTINCT（多分片去重）。ShardingSphere对常用的流式归并和内存归并做了优化，但深分页仍是挑战。

## 2.14 ShardingSphere Hint强制路由的使用场景和方式？
场景：某些SQL无法通过解析自动路由（如存储过程调用）、DBA手动运维、特定业务需要跨常规路由规则的数据操作。方式：HintManager hintManager = HintManager.getInstance(); hintManager.setDatabaseShardingValue(逻辑表, 分片值); hintManager.addTableShardingValue(逻辑表, 分片值); try { // 执行业务SQL } finally { hintManager.close(); } 使用后务必关闭。

## 2.15 分布式主键在ShardingSphere中如何工作？
ShardingSphere内置雪花算法生成64位趋势递增ID作为分布式主键，workerId可基于IP或PID自动生成，支持时钟回拨容错（配置最大容忍回拨毫秒数，超出则抛出异常或等待时钟追上）。使用时只需在表配置中指定keyGenerateStrategy，或让应用层通过ShardingSphere的API获取。也可SPI扩展自定义主键生成算法（如号段模式）。
