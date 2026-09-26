# 1. 知识点
- **基本概念-数据库**：MongoDB实例可创建多个数据库，数据库包含多个集合和索引，类似MySQL的database
- **基本概念-集合Collection**：一组Document的集合，类似关系型数据库的表，但集合无固定Schema（文档结构可不一致）
- **基本概念-文档Document**：MongoDB的基本数据单元，BSON（Binary JSON）格式，键值对有序存储，类似关系型的一行记录
- **BSON**：Binary JSON，MongoDB内部存储和网络传输的数据格式，支持更多数据类型（Date、Binary、ObjectId、Decimal128）
- **与关系型数据库对比**：嵌入式文档代替JOIN减少关联查询；动态Schema适应敏捷开发；NoSQL分布式天生支持；不支持复杂的跨文档ACID事务（4.0前）
- **动态Schema**：同一集合中不同文档的字段和类型可以不同，新增字段无需ALTER，适合需求快速变化的产品
- **ObjectId**：MongoDB默认主键类型，12字节：4字节时间戳 + 5字节随机值 + 3字节递增计数器，分布式唯一
- **索引类型-单字段索引**：对文档的某个字段创建索引，最基础索引
- **索引类型-复合索引**：多个字段组合索引，遵循前缀原则
- **索引类型-多键索引**：对数组字段的索引，MongoDB为数组中每个元素创建一个索引条目
- **索引类型-2dsphere**：地理空间球面索引，支持GEO查询（$near、$geoWithin、$geoIntersects），用于LBS位置服务
- **索引类型-TTL索引**：Time To Live索引，文档在指定时间后自动删除，适用会话数据、日志自动过期清理
- **索引类型-全文索引**：对文本字段建立倒排索引，$text进行全文搜索，支持多语言分词
- **索引类型-唯一索引**：确保字段值唯一，null值被视为唯一值
- **索引类型-稀疏索引**：仅索引包含该字段的文档，忽略该字段不存在的文档
- **索引类型-哈希索引**：对字段值做哈希后索引，用于哈希分片时配合分片键使用
- **索引结构-B树**：MongoDB使用B树而非B+树，叶子节点既存key也存data，非叶子节点也存数据，适合等值查询和文档存储
- **为什么MongoDB使用B树而不是B+树**：文档型数据库按ID等值查询为主，B树叶节点存数据减少层级IO，无需多跳一次到达叶子；MongoDB无范围查询优化需求（非关系型的扫描强度低）；WiredTiger存储引擎对B树的页管理做了深度优化
- **复制集**：MongoDB高可用方案，一个主节点Primary + 多个从节点Secondary + 可选仲裁节点Arbiter
- **Primary**：读写节点，所有写操作必须经过Primary，默认也是读节点
- **Secondary**：只读节点，从Primary异步同步oplog，分担读负载，故障时可提升为Primary
- **Arbiter**：仲裁节点，不存储数据仅参与选举投票，打破平票（偶数节点时），资源消耗极低
- **自动故障转移**：Primary故障时，复制集自动选举新Primary，心跳检测默认每2秒一次，故障检测约10秒
- **选举-Raft协议**：MongoDB使用Raft共识算法的变体，基于节点优先级priority和oplog新鲜度（optime）投票
- **Oplog**：操作日志，记录Primary的写操作，Secondary读取Oplog增量同步数据，大小固定capped collection循环写入
- **分片Sharding**：MongoDB的水平扩展方案，将数据水平拆分到多个分片节点
- **Shard分片节点**：存储实际分片数据的Mongod实例（通常为复制集），每个Shard只存数据子集
- **Mongos路由**：分片集群入口，接收客户端请求，根据配置服务器信息将请求路由到正确分片，聚合结果返回客户端
- **Config Server配置服务器**：存储分片集群的元数据（分片范围、Chunk分布），通常为复制集（3节点），均衡器和Mongos都以此为数据源
- **片键选择策略**：决定数据分布的核心，影响读写性能和数据均衡——选择查询频繁字段、基数高分布均匀的字段
- **哈希分片**：对片键做哈希后均匀分布，数据分布均衡但范围查询跨多个分片
- **范围分片**：按片键值范围分布（如userId[0,1000]在分片1），范围查询高效但热点风险大
- **Chunk**：分片数据的基本迁移单元，一个分片上连续片键范围的数据，默认大小64MB
- **Chunk分裂**：Chunk数据量超过上限（默认64MB）自动分裂为两个Chunk
- **Chunk迁移**：均衡器Balancer检测分片间数据不均匀时自动迁移Chunk，后台进行
- **聚合管道Aggregation Pipeline**：数据处理管道，文档顺序经过一系列阶段处理转换
- **$match**：过滤文档，类似WHERE，尽量在管道前部放置以利用索引减少后续处理量
- **$group**：分组聚合，类似GROUP BY，_id指定分组字段，支持$sum/$avg/$min/$max/$first/$last等累加器
- **$project**：字段投影转换，类似SELECT列 + 计算表达式
- **$sort**：排序，类似ORDER BY
- **$limit/$skip**：限制返回数和跳过前N条，与分页配合
- **$lookup**：左外连接，类似SQL LEFT JOIN，通过pipeline或localField/foreignField关联另一集合
- **$unwind**：展开数组字段，数组中每个元素生成一个文档
- **WiredTiger**：MongoDB 3.2起默认存储引擎，文档级并发控制（乐观锁MVCC的变体），压缩（snappy/zlib/zstd），支持Checkpoint和Journal
- **WiredTiger文档级并发控制**：写操作锁定单文档而非整个集合，并发写入能力强
- **Checkpoint检查点**：WiredTiger定期将内存中的数据快照写入磁盘，作为数据恢复的基准点，默认60秒
- **Journal日志**：类似redo log，写操作先记录到Journal文件，定期Checkpoint后清理旧的Journal，保证崩溃恢复
- **事务-4.0副本集多文档事务**：MongoDB 4.0引入副本集内的多文档ACID事务
- **事务-4.2分布式事务**：MongoDB 4.2扩展到分片集群的分布式事务
- **ACID与关系型差异**：MongoDB事务语法和范围类似关系型，但性能开销大（分布式事务需协调多Shard），适用强一致性场景（转账扣减库存），不建议滥用
- **Spring Data MongoDB**：Spring对MongoDB的封装，MongoTemplate提供模板式CRUD，MongoRepository提供声明式数据访问
- **MongoTemplate**：核心类，类似JdbcTemplate，提供find、update、remove、aggregate等操作
- **MongoRepository**：类似JpaRepository，接口继承即可获得CRUD方法，支持方法命名查询
- **注解@Document**：标注Java类为MongoDB文档实体，参数collection指定集合名
- **注解@Id**：标注主键字段，MongoDB帮自动生成ObjectId
- **注解@Field**：指定文档中字段名与Java属性的映射
- **注解@Indexed**：声明单字段索引
- **注解@CompoundIndex**：在类级别声明复合索引
- **GridFS**：存储超过16MB的大文件（如视频、图片），将文件切分为多个255KB的chunk分别存储，
- **适用场景-日志数据**：高写入吞吐，BSON支持嵌套结构，TTL索引自动清理过期日志
- **适用场景-时序数据**：IoT传感器数据、监控指标；高并发写入、自动过期
- **适用场景-LBS**：内建2dsphere地理空间索引，支持$near等GEO查询
- **适用场景-Json快速迭代**：产品原型、内容管理系统（CMS），Schema灵活迭代快
- **适用场景-电商目录**：商品属性差异大（衣服有尺码、手机有内存），动态Schema天然支持
- **适用场景-游戏数据**：玩家属性频繁变更、高写入、排行榜实时聚合

# 2. 面试题

## 2.1 MongoDB相比关系型数据库有什么优势？
MongoDB的优势包括：1）动态Schema，无需预先定义表结构，适合需求快速变化的场景；2）嵌入式文档模型，关联数据存一起，大部分查询无需JOIN；3）天然分布式，通过复制集和分片实现高可用和水平扩展；4）高写入性能（WiredTiger文档级锁）；5）JSON/BSON数据格式与前端/微服务天然匹配。不适合需要复杂跨文档ACID事务、结构化关联查询（多表JOIN）的场景。

## 2.2 MongoDB为什么选择B树而不是B+树作为索引结构？
1）MongoDB是文档数据库，以等值查询为主（findById），B树叶节点直接存数据只需一次查找，B+树必须到叶子才能取数据多一层IO。2）范围查询需求弱于关系型数据库，B+树的叶子链表优势不明显。3）WiredTiger的B树实现结合LSM思想做了写入优化（页内修改+checkpoint），不比B+树慢。4）B树内部节点也存数据，可以减少索引高度和IO次数。

## 2.3 复制集的工作原理是什么？自动故障转移如何实现？
复制集由1个Primary、多个Secondary和可选Arbiter组成。Primary接收所有写操作并记录oplog，Secondary异步从Primary拉取oplog重放保持数据一致。各节点间通过心跳（heartbeat，默认2秒一次）相互监控。当Primary 10秒内无响应，符合选举条件的Secondary发起选举（基于Raft协议），根据优先级priority和oplog新鲜度optime投票，选出新Primary。旧Primary恢复后自动降级为Secondary加入集群。客户端驱动自动感知Primary变更重连。

## 2.4 MongoDB的分片集群由哪些组件组成？各自作用是什么？
- Shard（分片节点）：存储实际分区数据，通常为复制集保证高可用
- Mongos（路由节点）：客户端入口，接收请求后查询Config Server获取元数据，将请求路由到正确Shard，归并结果返回；无状态可部署多实例
- Config Server（配置服务器）：存储分片元数据（Chunk分布、片键范围映射），为复制集保证可靠性；均衡器基于元数据决策Chunk迁移

三者协作：客户端 → Mongos → Config Server查分片位置 → 路由请求到对应Shard → Mongos聚合返回。

## 2.5 聚合管道是什么？$match和$group怎么用？
聚合管道是MongoDB的数据处理框架，文档依次经过多个处理阶段（Stage），每个阶段对输入文档进行处理后将结果传递给下一阶段，类似Unix管道。$match阶段过滤文档（类似WHERE），$group阶段按_id分组并执行聚合计算（如$sum、$avg）。示例：db.orders.aggregate([{$match: {status: "completed"}}, {$group: {_id: "$customerId", total: {$sum: "$amount"}}}]）。

## 2.6 什么是片键？片键选择有哪些策略？
片键决定数据如何在分片集群中分布，是分片的核心设计决策。策略有：
- 哈希分片：hash(片键)均匀分布，数据均衡但范围查询需跨多Shard（全Shard广播）
- 范围分片：按片键值连续范围分布，范围查询高效（只落一个Shard），但容易产生热点（如按时间分片，新数据都写到一个Shard）
- 复合片键：结合多个字段，如{region:1, userId:1}兼顾区域过滤和均匀分布
- 选择原则：基数高、查询常在条件中、分布均匀避免热点

## 2.7 MongoDB的事务支持情况如何？
MongoDB自4.0版本起支持副本集内多文档ACID事务（multi-document transactions），提供startSession→startTransaction→commitTransaction/abortTransaction API。4.2扩展到分片集群分布式事务。事务语义（可快照读提交/可重复读）与关系型相似，但底层实现不同——MongoDB基于WiredTiger的MVCC快照隔离，分布式事务需两阶段提交协调。适用强一致性场景（如转账），但因性能开销不建议滥用，仍应优先用嵌入式文档设计避免跨文档事务。

## 2.8 什么是Oplog？它的作用是什么？
Oplog（操作日志）是MongoDB复制集的核心机制，属于Primary上的capped collection（固定大小，循环写入）。记录所有修改数据的写操作（增删改），Secondary通过长轮询从Primary拉取Oplog并重放（idempotent命令形式，如insert→实际存完整文档以便幂等重放），实现数据同步。Oplog大小需合理设置，太小可能因写量大被覆盖导致Secondary全量同步。

## 2.9 $lookup和SQL的JOIN有什么区别和联系？
$lookup是MongoDB聚合管道中的左外连接操作符（3.2引入），与SQL LEFT JOIN功能类似：将左表（当前集合）与右表（关联集合）按匹配条件关联，左表全部保留，右表无匹配填null。但MongoDB不建议频繁使用$lookup——它仍是跨集合查询，无优化器自动索引选择能力，大集合性能差。MongoDB最佳实践是用嵌入式文档模型避免JOIN，$lookup仅用于遗留设计的补救。

## 2.10 Spring Data MongoDB中MongoTemplate和MongoRepository有什么区别？如何使用？
MongoTemplate是命令式模板类，提供细粒度的数据操作API（find/updateOne/remove/aggregate/createCollection等），灵活但代码量大，与Spring的JDBC Template风格类似。MongoRepository是声明式数据访问接口，继承接口即可自动获得CRUD方法、方法命名查询、@Query注解自定义查询，类似Spring Data JPA的JpaRepository。一般组合使用：MongoRepository处理简单CRUD，MongoTemplate处理复杂聚合和多条件查询。

## 2.11 MongoDB最佳适用场景有哪些？
- 游戏数据：玩家属性频繁变更、高并发在线写入、数据模型灵活
- 日志/时序数据：高吞吐写入、BSON存储异构日志结构、TTL索引自动清理
- LBS地理位置服务：2dsphere索引原生支持附近的人/范围查询
- CMS/电商目录：商品属性差异大（SKU不同属性数量类型不一），动态Schema天然适配
- IoT物联网：海量设备数据高并发写入、时序数据+半结构化数据混合
- 实时分析/仪表盘：聚合管道实时计算高性能

不适用的场景：复杂关联查询（多表JOIN）、强ACID事务密集型（金融核算）、数据仓库报告型SQL复杂分析。

## 2.12 GridFS是什么？如何使用它存储大文件？
GridFS是MongoDB的大文件存储规范，用于存储超过16MB（BSON文档限制）的文件。原理：将大文件切分成多个255KB的chunk（块文档），同时创建files文档记录元数据（文件名、大小、上传时间、MD5）。读取时根据files文档找到所有chunk并拼装还原。适用：视频、大图片、备份文件。不适用：CDN/对象存储方案更合适（带宽便宜、CDN加速更好），GridFS常用于内部文件存储。

## 2.13 MongoDB的WiredTiger存储引擎有什么特点？
1）文档级并发控制：写操作锁单文档非整个集合，并发能力强。2）数据压缩：支持snappy/zlib/zstd压缩算法，减少磁盘空间。3）Checkpoint检查点：定期全量快照写入磁盘（默认60秒），配合Journal实现崩溃恢复（重放Checkpoint后的Journal）。4）B树 + LSM思想：B树内部页缓存；内存数据定期Checkpoint落盘。5）MVCC：读操作不加锁看到快照，写操作文档级锁。

## 2.14 MongoDB中如何设计数据模型？嵌入式文档 vs 引用哪个更好？
设计的核心决策是"嵌or引"：
- 嵌入式文档：将关联数据直接嵌入当前文档中（如用户文档嵌入地址数组）。优点：一次查询获取所有数据，无JOIN开销；缺点：嵌入式数据无法独立查询和分页，文档体积膨胀。适用：一对一（如用户+个人资料）、一对少量且子项数量可控（如订单+订单条目）。
- 引用：存储ID指向关联文档（类似外键）。优点：子数据独立、可单独查询分页；缺点：需$lookup或多次查询。适用：一对多（父文档含子文档ID）、多对多。

一般原则：优先内嵌除非有充分理由引用（子文档独立访问、无限增长、共享引用）。

## 2.15 什么是TTL索引？它怎么实现数据自动过期？
TTL（Time To Live）索引是MongoDB的一种特殊单字段索引，对日期类型字段建索引并设expireAfterSeconds参数。MongoDB后台线程每60秒扫描一次TTL索引，删除字段值早于（当前时间 - expireAfterSeconds秒）的文档。底层原理是索引扫描而非全表扫描，所以依赖索引存在。典型场景：用户Session自动过期、验证码、临时数据、日志保留策略。
