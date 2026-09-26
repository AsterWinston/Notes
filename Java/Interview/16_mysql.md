# 1. 知识点
- **体系架构**：连接层（身份认证、线程复用）→ 服务层（SQL解析、优化、查询缓存8.0移除）→ 引擎层（可插拔存储引擎，InnoDB/MyISAM/Memory）→ 存储层（文件系统、日志）
- **连接层**：负责客户端连接管理、认证授权、连接线程复用（thread cache），每个连接在服务端对应一个线程
- **服务层-SQL解析**：词法分析→语法分析→生成解析树，判断SQL类型（DML/DDL/DCL）
- **服务层-查询优化**：预处理器检查语义→查询优化器（CBO基于成本）选择最优执行计划→生成执行计划
- **服务层-执行器**：调用存储引擎API逐行读取数据，返回结果集
- **引擎层**：可插拔式设计，通过Handler API与上层交互，不同引擎对表数据存取方式不同
- **存储层**：数据文件（.ibd）、日志文件（redo log、undo log、binlog）、配置文件等物理存储
- **InnoDB**：默认引擎，支持事务（ACID）、行级锁、外键约束、聚集索引、MVCC、崩溃恢复、在线DDL
- **MyISAM**：不支持事务和行锁，表级锁，非聚集索引，支持全文索引（5.6前），数据文件和索引文件分离（.MYD/.MYI），count(*)快
- **Memory**：数据全部存内存，重启丢失，默认哈希索引，支持表级锁，适合临时表/缓存表
- **B+树索引**：所有数据存叶子节点，叶子节点形成有序双向链表，非叶节点仅存key作路标，范围查询高效，高度3-4层
- **哈希索引**：Memory引擎默认，等值查询O(1)，不支持范围/排序，InnoDB自适应哈希索引由系统自动创建
- **自适应哈希索引**：InnoDB根据查询模式自动在B+树上建立哈希索引，加速热点页访问
- **全文索引**：适用于CHAR/VARCHAR/TEXT，InnoDB从5.6开始支持，MySQL 5.7+支持ngram中文分词
- **空间索引**：用于GIS地理空间数据类型（GEOMETRY），InnoDB支持，基于R树
- **索引数据结构选择**：B+树 vs B树——B+树非叶节点不存数据，扇出更大，高度更低；叶子链表支持范围扫描。B树叶节点也存数据，范围查询需中序遍历
- **为什么不是红黑树**：红黑树二叉树深度大（千万数据约24层），磁盘IO次数多；B+树多叉，每次IO加载一个节点（16KB页），3-4层即可
- **为什么不是哈希**：哈希无法处理范围查询、排序、LIKE模糊匹配，且哈希冲突后链表查找效率下降
- **聚簇索引**：主键索引的叶子节点存储整行数据，InnoDB必须有主键（无显式则选唯一非空索引或隐式rowid），数据即索引
- **非聚簇索引（二级索引）**：叶子节点存储主键值而非数据，通过主键回表查询获取完整行数据
- **回表**：通过二级索引查到主键后，再到聚簇索引中查完整行，增加IO开销
- **索引覆盖**：查询列全部在索引中，无需回表，Extra显示Using index，性能最优
- **联合索引**：多列组成索引，遵循最左前缀原则（查询条件从索引最左列开始才能使用）
- **最左前缀原则**：联合索引(a,b,c)能走索引的查询：a=?、a=? AND b=?、a=? AND b=? AND c=?；无法走索引：b=?、c=?
- **索引下推ICP**：MySQL 5.6+，在引擎层使用索引中的列条件提前过滤，减少回表和返回服务层的行数
- **索引失效场景**：WHERE子句对索引列使用函数（WHERE YEAR(date)=2024）；索引列参与运算（WHERE id+1=10）；隐式类型转换（varchar列用数字查询）；LIKE '%xx%'左侧模糊；!= / <> / NOT IN 可能导致全表扫描；OR条件非所有列都是索引列；IS NULL / IS NOT NULL 视情况而定
- **EXPLAIN type访问类型**：system > const（主键/唯一等值）> eq_ref（联表唯一匹配）> ref（非唯一索引等值）> range（索引范围）> index（索引全扫描）> ALL（全表扫描）
- **EXPLAIN extra**：Using index（覆盖索引）、Using where（存储引擎返回后服务层过滤）、Using index condition（ICP）、Using temporary（临时表）、Using filesort（额外排序）、Using join buffer（关联缓存）
- **EXPLAIN possible_keys vs key**：possible_keys为可能使用的索引，key为实际使用的索引，两者不一致说明优化器可能选择了非最优索引
- **避免SELECT ***：只取需要的列，减少网络传输和内存占用，更易触发索引覆盖
- **分页优化**：大偏移量LIMIT N,M 用延迟关联（子查询先取主键再用主键关联取完整数据）；游标分页WHERE id>last_id LIMIT N
- **关联查询**：小表驱动大表，left join左表全量右表匹配，用小表做驱动表减少循环次数
- **IN vs EXISTS**：外层大内层小用IN，外层小内层大用EXISTS（早退出）；MySQL 8.0优化器自动转换
- **ACID-原子性**：事务不可分割，undo log记录旧版本数据实现回滚
- **ACID-一致性**：事务前后数据满足所有约束，由redo log + undo log + 锁共同保障
- **ACID-隔离性**：事务并发执行互不干扰，MVCC + 锁实现
- **ACID-持久性**：提交的事务永久保存，redo log WAL机制 + 双写保证
- **读未提交**：可读到未提交数据，脏读；实现：读不加锁
- **读已提交**：每次读生成新ReadView，不可重复读；实现：MVCC每语句创建新快照
- **可重复读**：InnoDB默认，事务内读一致，可能幻读（InnoDB通过Next-Key Lock解决大部分）；实现：MVCC事务开始创建ReadView
- **串行化**：事务串行执行，读写加锁，最高隔离级别
- **MVCC**：多版本并发控制，通过隐藏列（DB_TRX_ID事务ID、DB_ROLL_PTR回滚指针、DB_ROW_ID）和undo log版本链 + ReadView实现快照读
- **快照读 vs 当前读**：普通SELECT是快照读不加锁；SELECT FOR UPDATE / UPDATE / DELETE / INSERT是当前读加锁读最新数据
- **RC和RR的ReadView差异**：RC每次SELECT生成新ReadView；RR事务第一个SELECT生成ReadView后复用，保证可重复读
- **redo log**：物理日志，记录数据页的物理变更，循环写入固定大小的ib_logfile，write-ahead日志先写redo再写磁盘，崩溃恢复用redo前滚
- **undo log**：逻辑日志，记录SQL逻辑逆操作（INSERT→DELETE，UPDATE→旧值），用于事务回滚和MVCC，链式串联旧版本
- **binlog**：Server层逻辑日志，记录SQL语句逻辑，追加写入，用于主从复制和数据恢复，三种格式：STATEMENT/ROW/MIXED
- **锁机制-表锁**：MyISAM默认，粒度大并发低，InnoDB在DDL或无索引查询时可能升级为表锁
- **锁机制-行锁**：InnoDB基于索引对记录加锁，Record Lock锁索引记录，共享锁S/排他锁X
- **Gap Lock间隙锁**：锁索引记录间的间隙，防止幻读，RR级别下生效，RC下无
- **Next-Key Lock**：Record Lock + Gap Lock，左开右闭区间，InnoDB RR下的主要加锁方式
- **插入意向锁**：INSERT操作在插入前设置，与Gap Lock冲突时等待，表示有插入意向
- **自增锁**：AUTO_INCREMENT列插入时获取，5.7后有innodb_autoinc_lock_mode参数（0传统/1连续/2交错）
- **等值查询唯一索引**：命中时退化为Record Lock锁单行；未命中时锁间隙
- **等值查询非唯一索引**：命中时对扫描到的记录加Next-Key Lock，对不满足条件的间隙加Gap Lock
- **范围查询**：对索引扫描范围加Next-Key Lock，直到第一个不满足条件的记录
- **死锁**：两个事务互相持有对方需要的锁，形成循环等待，InnoDB自动检测并回滚代价小的事务
- **死锁排查**：SHOW ENGINE INNODB STATUS查看LATEST DETECTED DEADLOCK；查询information_schema.innodb_trx / innodb_lock_waits
- **主从复制三线程**：Master的Binlog Dump线程发送日志；Slave的IO线程接收写入Relay Log；Slave的SQL线程读取Relay Log执行
- **异步复制**：主库提交事务立即返回，不等待从库确认，数据可能不一致，延迟风险
- **半同步复制**：Master提交后等待至少一个Slave收到binlog并写入Relay Log才返回，after_sync模式（5.7+默认）
- **组复制MGR**：基于Paxos协议的多主复制，自动故障检测和选主，强一致
- **并行复制**：Slave SQL线程多线程并行执行Relay Log，5.7基于库级别，8.0基于Writeset的组提交并行（binlog_transaction_dependency_tracking）
- **延迟复制**：从库故意延迟N秒执行，用于误操作快速恢复
- **主从延迟原因**：从库硬件差/从库单线程回放/大事务/网络延迟/锁竞争；排查：SHOW SLAVE STATUS看Seconds_Behind_Master
- **分库分表-水平分库**：按分片键将同一张表数据分布到不同数据库，解决单库存储和并发瓶颈
- **分库分表-水平分表**：同一库内按分片键拆分大表到多张子表，解决单表数据量过大
- **分库分表-垂直分库**：按业务模块拆分到不同数据库，如用户库、订单库
- **分库分表-垂直分表**：将宽表的冷热字段拆分，如主表存频繁访问字段，扩展表存大文本/不常用字段
- **ShardingSphere-JDBC**：客户端分片中间件，嵌入应用以jar形式，SQL解析→路由→改写→执行→结果归并
- **ShardingSphere-Proxy**：独立部署数据库代理，对应用透明，兼容MySQL协议
- **慢查询配置**：slow_query_log=ON, long_query_time（阈值秒）, slow_query_log_file路径
- **慢查询分析**：mysqldumpslow工具（-s排序、-t topN）；Percona Toolkit的pt-query-digest分析汇总报告
- **Buffer Pool**：InnoDB内存缓存区，缓存数据页和索引页，通过LRU链表管理（5/8 young区 + 3/8 old区防全表扫描污染），预读机制
- **Change Buffer**：非唯一二级索引的DML修改缓存在内存，后续合并到索引页，减少随机IO
- **自适应哈希索引**：InnoDB自动在频繁访问的索引页上建立哈希索引，加速等值查询
- **MySQL 8.0原子DDL**：CREATE/ALTER/DROP等DDL支持原子性，失败自动回滚，不再残留中间状态
- **MySQL 8.0窗口函数**：ROW_NUMBER/RANK/DENSE_RANK/SUM OVER，按窗口分区排序计算
- **MySQL 8.0 CTE**：WITH AS公用表表达式，支持递归CTE实现树形查询
- **MySQL 8.0降序索引**：联合索引支持指定列的降序，优化ORDER BY col1 ASC, col2 DESC
- **MySQL 8.0隐藏索引**：ALTER TABLE ... ALTER INDEX ... INVISIBLE，可隐藏索引测试查询影响后再删除
- **MySQL 8.0资源组**：Resource Group按CPU/线程优先级分配资源，限制不同业务的资源使用
- **MySQL 8.0双密码**：支持主密码+备用密码，安全轮转无需中断业务
- **MySQL常见日志**：binlog(Server层逻辑日志/数据变更/主从复制/数据恢复); slow query log(记录执行时间>long_query_time的SQL/排查性能瓶颈); redo log(InnoDB物理日志/崩溃恢复/循环写ib_logfile); undo log(逻辑日志/事务回滚/MVCC); error log(记录启动/运行/关闭中的错误); relay log(从库中继日志/记录从主库接收的binlog); general query log(所有SQL记录/一般关闭)
- **Binlog三种格式**：STATEMENT(记录SQL语句原文、紧凑但可能导致主从不一致如UUID()); ROW(记录每行变更前后数据、准确但日志量大); MIXED(默认STATEMENT、特殊情况下自动切换ROW)
- **索引存放位置**：索引在磁盘上持久化存储(.ibd文件); MySQL将频繁使用的索引页加载到内存Buffer Pool中以加速查询; 写入操作先在Buffer Pool中修改，后续通过checkpoint刷回磁盘

# 2. 面试题

## 2.1 请简述MySQL的体系架构
MySQL分为四层：连接层负责客户端连接管理和认证；服务层是核心，完成SQL解析、优化器选择执行计划、执行器调用引擎；引擎层为可插拔存储引擎（InnoDB为主）；存储层为物理文件（数据、日志、配置）。这种分层设计使引擎可替换，各层职责清晰。

## 2.2 InnoDB和MyISAM的主要区别是什么？
| 特性 | InnoDB | MyISAM |
|------|--------|--------|
| 事务 | 支持ACID | 不支持 |
| 锁粒度 | 行级锁 | 表级锁 |
| 外键 | 支持 | 不支持 |
| 索引 | 聚簇索引 | 非聚簇索引 |
| 崩溃恢复 | redo log支持 | 不支持，崩溃后需修复 |
| 全文索引 | 5.6后支持 | 原生支持 |
| count(*) | 需扫描 | 直接读表元数据 |
| 数据文件 | .ibd（共享或独立） | .MYD数据 + .MYI索引 |

8.0后系统表全部InnoDB化，MyISAM逐步废弃。

## 2.3 为什么InnoDB使用B+树而不是B树、红黑树或哈希表？
- **B+树 vs B树**：B+树非叶节点仅存key扇出更大，树更矮（3-4层即可索引千万数据）；叶子节点形成有序双向链表，范围查询只需遍历链表；B树叶节点也存数据，范围需中序遍历，磁盘IO更多。
- **vs 红黑树**：红黑树是二叉树，深度大（千万数据~24层），每次节点访问即一次磁盘IO，性能极差。
- **vs 哈希表**：哈希仅支持等值查询，不支持范围、排序、模糊匹配；且哈希冲突后性能下降；InnoDB自适应哈希是对B+树的补充而非替代。

## 2.4 什么是聚簇索引和非聚簇索引？回表是什么意思？
聚簇索引（主键索引）的叶子节点存储整行数据，数据和索引存放在一起；非聚簇索引（二级索引）的叶子节点存储主键值。通过二级索引查询时，先在二级索引中找到主键，再回到聚簇索引中查找完整行，这个过程叫回表。如果查询列全部包含在二级索引中（覆盖索引），则无需回表，性能最优。

## 2.5 联合索引的最左前缀原则是什么？
联合索引(a,b,c)使用B+树结构，排序规则是先按a排序，a相同按b排序，b相同按c排序。因此查询条件必须从最左列开始才能利用索引：a=?可用索引(a)；a=? AND b=?可用索引(a,b)；b=? AND c=?（缺失a）无法使用索引。ORDER BY也遵循此原则，ORDER BY a,b可用，ORDER BY b,c不可用。

## 2.6 什么是索引下推（ICP）？
Index Condition Pushdown，MySQL 5.6+特性。联合索引(a,b)查询WHERE a>10 AND b=5，引擎层会将a>10的行中的b列直接过滤，只对b=5的行回表，减少回表次数和返回服务层的数据量，Extra显示Using index condition。

## 2.7 哪些情况会导致索引失效？
- 对索引列使用函数：WHERE YEAR(create_time)=2024
- 索引列参与运算：WHERE id+1=10
- 隐式类型转换：varchar列不加引号 WHERE phone=13800000000
- LIKE左侧模糊：WHERE name LIKE '%abc'
- != / <> / NOT IN 可能导致优化器放弃索引
- OR条件中部分列不在索引中
- 联合索引不满足最左前缀
- 优化器认为全表扫描更快（数据量小或返回比例大时）

## 2.8 EXPLAIN的type字段有哪些？分别代表什么？
从优到劣：system（系统表一行）> const（主键/唯一索引等值，最多一行）> eq_ref（关联查询唯一匹配，一行）> ref（非唯一索引等值，多行）> range（索引范围扫描）> index（全索引扫描）> ALL（全表扫描）。一般要求达到range级别，最好达到ref。

## 2.9 EXPLAIN中Extra字段的常见值有哪些？
- Using index：覆盖索引，最优
- Using where：存储引擎返回行后由服务层过滤
- Using index condition：索引条件下推
- Using temporary：需创建临时表（常见于GROUP BY/DISTINCT），需优化
- Using filesort：需额外排序操作（排序列无索引），需优化
- Using join buffer：关联使用join buffer
- Impossible WHERE：WHERE条件恒假

## 2.10 什么是MVCC？它的实现原理是什么？
Multi-Version Concurrency Control，通过维护数据的多个版本来实现读写不冲突。实现基于三要素：隐藏列（DB_TRX_ID事务ID标记创建该版本的事务、DB_ROLL_PTR回滚指针指向undo log中旧版本）、undo log版本链、ReadView（活跃事务ID列表，判断版本可见性）。快照读时根据ReadView找到可见版本，当前读直接读最新数据并加锁。

## 2.11 可重复读（RR）和读已提交（RC）在MVCC实现上有何区别？
关键区别在ReadView的生成时机：RC级别每次SELECT都生成新的ReadView，因此能看到别的事务已提交的修改（不可重复读）；RR级别在事务内第一次SELECT时生成ReadView，后续SELECT复用同一ReadView，因此读到的版本始终一致（可重复读）。

## 2.12 redo log和binlog的区别是什么？
| 维度 | redo log | binlog |
|------|----------|--------|
| 层级 | InnoDB引擎层 | Server层 |
| 内容 | 物理日志，数据页的变更 | 逻辑日志，SQL语句或行变化 |
| 写入方式 | 循环写（固定大小） | 追加写（无限增长） |
| 用途 | 崩溃恢复（crash-safe） | 主从复制、数据恢复 |
| 刷盘控制 | innodb_flush_log_at_trx_commit | sync_binlog |

两者通过两阶段提交保证一致性：prepare（写redo）→ commit（写binlog→redo标记commit）。

## 2.13 InnoDB有哪些锁？Next-Key Lock是什么？
InnoDB实现了：表锁（意向锁IS/IX，DDL时）；行锁（Record Lock锁索引记录，共享锁S/排他锁X）；Gap Lock（间隙锁，锁索引记录间的间隙，防幻读）；Next-Key Lock（Record+Gap，左开右闭区间，RR下加锁的主要形式）；插入意向锁（INSERT时设置，与Gap Lock互斥）；自增锁（AUTO_INCREMENT）。Next-Key Lock = Record Lock + Gap Lock，是InnoDB RR隔离级别下防止幻读的核心机制。

## 2.14 如何排查数据库死锁？
1. SHOW ENGINE INNODB STATUS 查看LATEST DETECTED DEADLOCK段落，包含最近一次死锁的详细信息（涉及事务、持有的锁、等待的锁、回滚的事务）。
2. 查询information_schema.innodb_trx查看当前活跃事务，innodb_lock_waits查看锁等待关系。
3. 应用层可设置innodb_print_all_deadlocks=ON将所有死锁信息输出到错误日志。
4. 定期分析慢SQL日志中是否有大量锁等待超时。

## 2.15 主从复制的三个线程分别做什么？主从延迟如何排查？
Master端Binlog Dump线程：读取binlog发送给Slave。Slave端IO线程：接收binlog写入Relay Log。Slave端SQL线程：读取Relay Log重放执行。延迟排查：SHOW SLAVE STATUS查看Seconds_Behind_Master；原因可能有从库硬件差、单线程回放（无并行复制）、大事务阻塞、网络延迟、主库写入压力大从库跟不上。解决：升级硬件、开启并行复制、拆分大事务、使用半同步复制。

## 2.16 分库分表有哪些方式？各适用什么场景？
- 垂直分库：按业务模块拆分（用户库、订单库、商品库），解决业务耦合度高、数据库连接数瓶颈
- 垂直分表：宽表拆分冷热字段（主表+扩展表），减少IO、提升缓存命中
- 水平分库：按分片键数据分布到多个数据库，解决单库存储和性能瓶颈，高并发场景
- 水平分表：同一库内按分片键拆分，解决单表数据量过大，中低并发但数据量大的场景

## 2.17 如何分析定位慢查询？
1. 开启慢查询日志：slow_query_log=ON，设置long_query_time和日志路径。
2. 使用mysqldumpslow -s t -t 10 slow.log 查看TOP 10耗时最长SQL。
3. 使用pt-query-digest（Percona Toolkit）生成详细分析报告，包括总耗时、调用次数、锁等待时间等。
4. 拿到慢SQL后用EXPLAIN分析执行计划，检查type是否ALL/index，是否有Using filesort/Using temporary，索引是否生效。
5. 根据分析结果优化索引、改写SQL、或调整表结构。

## 2.18 Buffer Pool的工作原理是什么？
Buffer Pool是InnoDB在内存中的缓存区域，缓存数据页和索引页，使用改进版LRU管理（分young区5/8和old区3/8）。新页先插入old区头部，被再次访问才移入young区，防止全表扫描一次性污染整个缓存。搭配Change Buffer缓冲非唯一二级索引的DML修改，减少磁盘随机IO；后台定期刷脏页（flush list LRU），通过checkpoint机制保证crash recovery时扫描redo log范围可控。

## 2.19 MySQL 8.0有哪些重要新特性？
原子DDL（CREATE/ALTER/DROP支持回滚不回滚中间状态）；窗口函数（ROW_NUMBER、RANK、DENSE_RANK等分析函数）；公用表表达式CTE（WITH AS + 递归CTE）；降序索引；隐藏索引（invisible index）；双密码安全轮转；Resource Group资源组管理；JSON增强（JSON_TABLE、->>操作符）；直方图统计信息；Hash Join（8.0.18）；Writeset并行复制等。

## 2.20 如何优化大数据量下的分页查询？
传统LIMIT offset, size在大偏移量时需要扫描前offset+size条记录后丢弃前offset条。优化方案：1）延迟关联——先子查询取前N条主键，再关联取完整数据；2）游标分页——记录上一页最大ID，用WHERE id>last_id LIMIT size；3）如果允许，业务上限制翻页深度（如仅前100页可翻）；4）使用覆盖索引+关联减少回表。

## 2.21 MySQL常见的日志都有什么用？
| 日志 | 层级 | 内容 | 用途 |
|------|------|------|------|
| binlog | Server层 | 逻辑日志，记录数据变更的SQL或行变化 | 主从复制、数据恢复、CDC |
| slow query log | Server层 | 执行时间超过 long_query_time 的SQL | 定位性能瓶颈，发现慢SQL |
| redo log | InnoDB引擎层 | 物理日志，记录数据页的变更 | 崩溃恢复（crash-safe） |
| undo log | InnoDB引擎层 | 逻辑日志，记录事务逆操作 | 事务回滚、MVCC快照读 |
| error log | Server层 | 启动/运行/关闭中的错误信息 | 排查MySQL服务异常 |
| relay log | Slave端 | 从主库接收的binlog副本 | 从库数据同步重放 |
| general query log | Server层 | 所有客户端连接和执行的所有SQL | 调试审计（生产环境一般关闭） |

## 2.22 Binlog日志有哪三种格式？各自优缺点？
| 格式 | 记录内容 | 优点 | 缺点 |
|------|---------|------|------|
| STATEMENT | 原始SQL语句 | 日志量小，节省磁盘和网络 | 可能导致主从不一致（如 UUID()、NOW() 等函数在主从执行结果不同） |
| ROW | 每行数据变更前后的具体值 | 数据一致性强，不会出现主从不一致 | 日志量大，批量更新时会记录大量数据 |
| MIXED | 默认STATEMENT，特殊场景自动切换ROW | 兼顾紧凑性和一致性 | 需要额外判断逻辑，不是全部ROW |

生产环境推荐 ROW 格式，保证主从数据一致性。

## 2.23 索引存放在哪里？
索引持久化存储在磁盘上的表空间文件（.ibd）中，但MySQL运行时会维护一个Buffer Pool内存缓冲区。查询时：将频繁使用的索引页从磁盘加载到Buffer Pool中以内存速度加速查询。写入时：先在Buffer Pool中修改索引页并记录redo log，后续通过checkpoint机制将脏页刷回磁盘上的.ibd文件。这种设计兼顾了持久化（磁盘不丢数据）和性能（内存高速访问）。

## 2.24 count(*) 为什么会慢？InnoDB 和 MyISAM 有什么不同？
| 引擎 | count(*) 实现 | 性能 |
|------|-------------|------|
| MyISAM | 维护表级计数器，每次 INSERT/DELETE 时更新 | O(1)，极快 |
| InnoDB | 全表扫描/全索引扫描（遍历所有行或索引） | 随数据量增加线性变慢 |

**InnoDB 为什么慢**：InnoDB 的 MVCC 机制导致同一行在不同事务中可能可见/不可见，数据库无法维护一个全局精确的行数计数器。每次 count(*) 需要扫描索引中所有可见的行。优化方法：
1. 使用最小的二级索引扫描（因为二级索引叶子只存主键，一个页装的数据更多，IO 更少）
2. 使用 EXPLAIN 的 rows 列获取估算值（非精确）
3. 用 Redis/缓存表维护行数（适合近似精确的场景）
4. 使用 `SHOW TABLE STATUS` 获取估算值（误差可达 40%~50%）

## 2.25 CHAR 和 VARCHAR 的区别？存储上有什么差异？
| 维度 | CHAR | VARCHAR |
|------|------|---------|
| 存储方式 | 定长，不足用空格填充 | 变长，前缀记录实际长度 |
| 最小长度 | 0 字节（存空就是空） | 1 字节（长度标识） |
| 最大长度 | 255 字符 | 65535 字节（受行大小限制） |
| 尾部空格 | 取出时自动去除 | 保留 |
| 存储效率 | 定长字段多时浪费空间 | 变长节省空间 |
| 更新效率 | 定长不产生页分裂，更新快 | 变长可能导致页分裂 |
| 适用场景 | 固定长度的短字符串（MD5、手机号、身份证、状态码） | 长度变化大的字段（姓名、地址、描述） |

**底层存储差异**：InnoDB 的 Compact/Dynamic 行格式下，CHAR 也按变长存储（只占用实际字符数），但尾部空格不会去除。VARCHAR 额外需要 1~2 字节记录长度。如果所有列都是定长（CHAR + INT），InnoDB 会优化为定长行格式，更新性能更好。

## 2.26 MySQL 的 Online DDL 原理是什么？
Online DDL 允许在 DDL 操作期间不阻塞 DML（增删改查），通过三种算法实现：

| 算法 | 原理 | 是否阻塞 DML | 性能 |
|------|------|------------|------|
| INSTANT | 只修改元数据，不涉及数据重组织 | 不阻塞 | 最快 |
| INPLACE | 在源表上原地重建，不复制全表 | 大部分不阻塞（准备/提交阶段短暂锁表） | 较快 |
| COPY | 创建新临时表→拷贝全表数据→删旧表→重命名 | 全表拷贝期间阻塞 | 最慢 |

**INPLACE 实现原理**（以添加索引为例）：
1. **Prepare 阶段**：获取 MDL 排他锁，创建新索引的空 B+ 树结构。
2. **Execute 阶段**：降级为 MDL 共享锁，扫描聚簇索引的所有记录，将记录导入新索引。此阶段可正常 DML，DML 变更通过 row log 记录。
3. **Commit 阶段**：重新获取 MDL 排他锁，将 row log 中的变更应用到新索引，然后启用新索引。

INSTANT 适用场景（MySQL 8.0）：添加列（只能加在末尾）、修改列默认值、修改索引可见性、修改生成列表达式等。

## 2.27 Double Write（双写）机制是什么？
Double Write 是 InnoDB 保证数据页写入完整性的机制，解决**部分写失效（Partial Page Write）**问题。

**问题场景**：InnoDB 的页大小默认 16KB，但操作系统磁盘写入的最小单位是 512B（扇区）。如果脏页刷盘过程中机器宕机，可能出现"写了一半"的页——既不能用 redo log 恢复（redo log 按完整页记录物理变更，半写页可能丢失页头信息），也无法从磁盘读取。

**Double Write 流程**：
1. 脏页刷盘前，先将所有待写入的脏页**顺序写入** Double Write Buffer（共享表空间 `ibdata1` 中的 2MB 连续区域）。
2. Double Write Buffer 写完成后立即 `fsync()` 到磁盘。
3. 再将脏页**离散写入**各自的数据文件。

**崩溃恢复**：如果步骤 3 写入期间宕机，恢复时检查 Double Write Buffer 中是否有完整的页副本，有则用它覆盖损坏的数据页。如果步骤 1 或 2 失败，原数据页未受影响，通过 redo log 重做即可。

**性能考虑**：Double Write 写了两次磁盘（2MB 顺序写 + 离散写），对 SSD 影响较小，对 HDD 有一定性能消耗。MySQL 8.0.20+ 支持 `innodb_doublewrite` 关闭，但仅限特定场景（如 Fusion-io 等支持原子写的设备）。

## 2.28 MySQL 的三种 Join 算法（NLJ, BNL, Hash Join）
| 算法 | 版本 | 原理 | 适用场景 |
|------|------|------|---------|
| Index Nested-Loop Join (NLJ) | 所有版本 | 驱动表每行通过被驱动表的**索引**查找匹配行 | 被驱动表关联字段有索引 |
| Block Nested-Loop Join (BNL) | 5.6+ | 驱动表数据分块存入 Join Buffer，批量扫描被驱动表匹配 | 被驱动表关联字段无索引 |
| Hash Join | 8.0.18+ | 构建阶段：驱动表构建 Hash 表；探测阶段：被驱动表逐行哈希查找 | 大表关联、无可用索引，替代 BNL |

**NLJ**：假设驱动表 1000 行，被驱动表 100W 行，B+Tree 索引查找 ~3 次磁盘 IO → 总 IO = 1000 × 3 = 3000。性能最高。

**BNL**：驱动表不分批 → 被驱动表需全表扫描。驱动表分 N 批更少 → 被驱动表扫描 N 次。

**Hash Join**（MySQL 8.0 默认启用）：对 BNL 的全面优化。构建 Hash 表用内存或磁盘（超过 `join_buffer_size` 时落盘分批）。注意 Hash Join 只能用于等值连接，不能用于非等值连接（如 `<`、`>`）。

## 2.29 utf8 和 utf8mb4 有什么区别？
| 维度 | utf8（MySQL 中的） | utf8mb4 |
|------|-------------------|---------|
| 最大字节 | 3 字节/字符 | 4 字节/字符 |
| 字符范围 | 基本多语言平面（BMP），覆盖大部分常用文字 | 完整 Unicode，含 emoji、生僻字 |
| emoji 表情 | ❌ 不支持（如 😊 是 4 字节） | ✅ 支持 |
| 存储 | `VARCHAR(255)` 最多 255 字符 → 765 字节 | `VARCHAR(255)` 最多 255 字符 → 1020 字节 |

这是 MySQL 的历史遗留问题：MySQL 的 `utf8` 不是标准 UTF-8，仅支持 3 字节编码。标准 UTF-8 需要 4 字节支持 BMP 之外的字符。MySQL 在 5.5.3 引入 `utf8mb4`（mb4 = most bytes 4）来完整支持 UTF-8。

**迁移注意事项**：
1. 修改列编码：`ALTER TABLE t CONVERT TO CHARACTER SET utf8mb4 COLLATE utf8mb4_general_ci;`
2. MySQL 索引前缀最大 767 字节（InnoDB 默认）→ utf8mb4 下最多 191 字符索引（767/4），需调整 `innodb_large_prefix`（5.7+ 默认）和行格式 `DYNAMIC`/`COMPRESSED`。
3. 连接字符集需要同步修改。

## 2.30 自增 ID 用完了怎么办？如何设计主键避免这个问题？
**自增 ID 用完后**：如果定义了 `INT AUTO_INCREMENT`（最大值约 21.47 亿），达到最大值后继续插入会报错 `Duplicate entry`（MySQL 不再自增，重复使用最大值）。`BIGINT`（最大值约 922 京）在业务层面几乎不会耗尽。

**避免方案/主键设计**：
1. **分布式 ID 生成方案**：
   - **Snowflake（雪花算法）**：64 位 Long（1 位符号 + 41 位时间戳 + 10 位机器 ID + 12 位序列号），不依赖数据库。
   - **号段模式（Leaf-Segment）**：从数据库批量申请 ID 号段，应用层内存分发，减少数据库交互。
   - **Redis 自增**：`INCR` 原子命令整体自增。
2. **UUID**：缺点是 36 字符（长）、非递增（导致 B+ 树页分裂、写入性能差），不适合做主键。
3. **雪花算法的改进**：
   - **百度 UidGenerator**：基于 Snowflake，使用 RingBuffer 预分配提升性能。
   - **美团 Leaf**：双 Buffer 预加载，解决号段模式"双写"瞬时空洞问题。
4. **分库分表场景**：使用全局唯一 ID 生成器（如 Leaf、TinyId），避免多库自增 ID 冲突。

## 2.31 DATETIME 和 TIMESTAMP 的区别和选择？
| 维度 | DATETIME | TIMESTAMP |
|------|----------|-----------|
| 范围 | 1000-01-01 00:00:00 ~ 9999-12-31 23:59:59 | 1970-01-01 00:00:01 UTC ~ 2038-01-19 03:14:07 UTC |
| 时区 | 无时区概念，存入/取出值不变 | 存储时转为 UTC，取出时转为当前会话时区 |
| 存储 | 8 字节（5.6.4 前）/ 5 字节（5.6.4+ 含小数秒） | 4 字节（不含小数秒）/ 7 字节（含小数秒） |
| 默认值 | 需显式设置 | 可设 `DEFAULT CURRENT_TIMESTAMP`、`ON UPDATE CURRENT_TIMESTAMP` |
| 2038 问题 | 不受影响 | 2038-01-19 后溢出（MySQL 5.6.4+ 暂未修复） |

**选择建议**：
- 跨时区业务 → TIMESTAMP（自动时区转换）
- 历史数据（如生日、历史事件） → DATETIME（值固定不变）
- 业务创建/更新时间 → TIMESTAMP（支持自动更新）
- 需要 2038 年后支持 → DATETIME

## 2.32 前缀索引是什么？适用场景和注意事项？
前缀索引是对列值的**前 N 个字符**创建索引，减少索引大小，提高索引扫描效率。

**语法**：`CREATE INDEX idx_name ON table (column(N));`

**适用场景**：长字符串字段（URL、长文本、长 VARCHAR）且该列的前几个字符已经有较高的区分度。

**注意事项**：
1. **前缀长度选择**：通过 `SELECT COUNT(DISTINCT LEFT(column, N)) / COUNT(*)` 计算区分度，接近 1 越好。一般 `N=6~10` 区分度较高。
2. **无法用于覆盖索引**：前缀索引只存储了列的前缀，无法通过索引直接返回完整列值（必须回表）。
3. **无法用于 ORDER BY / GROUP BY**：前缀索引不保证完整字符串的顺序。
4. **ICP 限制**：前缀索引在 WHERE 条件中需要与 ICP（索引下推）结合减少回表。MySQL 5.7 前对前缀索引不支持 ICP。
5. **替代方案**：可以考虑对长字符串的 Hash 值（如 CRC32）建立索引，结合原值做精确匹配。
