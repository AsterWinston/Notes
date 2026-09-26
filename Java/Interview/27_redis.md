# 1. 知识点
- **String（字符串）**：最基础类型，底层用 SDS（Simple Dynamic String）存储，编码分为 int（整数）、embstr（短字符串 <=44 字节，一次内存分配）、raw（长字符串）。用于缓存、计数器、分布式锁等。
- **Hash（哈希）**：键值对集合，底层编码有 ziplist（压缩列表，元素少时使用）和 hashtable（哈希表）。适合存储对象属性。
- **List（列表）**：有序字符串列表，底层编码 quicklist（3.2+），由 ziplist 组成的双向链表。适合消息队列、最新消息列表等。
- **Set（集合）**：无序且唯一的字符串集合，底层编码有 intset（纯整数小集合）和 hashtable。所有操作 O(1)。适合标签、好友关系、共同好友等。
- **Sorted Set（有序集合，ZSet）**：有序且唯一的字符串集合，基于跳表（skiplist）+ 字典（dict）结构。适合排行榜、延时队列。
- **Bitmaps（位图）**：通过 String 类型的 bit 操作实现，适合签到统计、UV 去重、布隆过滤器（内存极省，每位一个用户）。
- **HyperLogLog**：基数估算的统计数据结构，12KB 固定内存，误差 0.81%。适合 UV 统计、大流量场景的近似去重计数。
- **Geo（地理位置）**：基于 Geohash 编码的 Sorted Set，支持经纬度存储与半径查询、距离计算。适合附近的人、地理位置检索。
- **Stream（流）**：Redis 5.0 引入的持久化消息队列数据结构，支持 Consumer Group、消息确认 ACK、消息回溯。适合可靠的消息队列场景。
- **Cache-Aside（旁路缓存）**：读时先查缓存，缓存未命中再查数据库并回填缓存；写时先更新数据库再删除缓存。最常用缓存模式。
- **Read-Through**：缓存缺失时由缓存层负责从数据库加载，应用层只操作缓存。
- **Write-Through**：写时同时更新缓存和数据库，缓存层保证一致性。
- **Write-Behind（写回）**：写时只更新缓存，异步批量写入数据库。吞吐高但有数据丢失风险。
- **缓存穿透**：查询不存在的数据（缓存和 DB 都无），高并发时直接打到 DB。解决：布隆过滤器（过滤不存在的 Key）、缓存空值（设短过期时间）。
- **缓存击穿**：热点 Key 突然过期，大量请求打到 DB。解决：互斥锁（锁第一个请求回源）、逻辑过期（永不过期，异步刷新）、分布式锁。
- **缓存雪崩**：大量 Key 同时过期或 Redis 集群宕机。解决：Key 过期时间加随机值、热点数据永不过期、Redis 集群高可用、限流降级。
- **惰性删除**：访问 Key 时检查是否过期，过期则删除。节省 CPU 但内存可能堆积。
- **定期删除**：每 100ms 定时随机扫描部分 Key，删除其中过期的 Key。平衡 CPU 和内存。
- **内存淘汰策略**：
  - `noeviction`：不淘汰，内存满时写操作报错。
  - `allkeys-lru`：在所有 Key 中用 LRU 淘汰。
  - `volatile-lru`：在设置过期时间的 Key 中用 LRU 淘汰。
  - `allkeys-random` / `volatile-random`：随机淘汰。
  - `volatile-ttl`：淘汰剩余 TTL 最短的 Key。
  - `allkeys-lfu` / `volatile-lfu`（Redis 4.0+）：用 LFU 淘汰。
- **RDB 持久化**：在指定时间间隔内执行 `bgsave`，fork 子进程生成二进制快照文件。利用写时复制（COW）机制不阻塞主线程。恢复快但可能丢失最后一个保存点到宕机之间的数据。
- **AOF 持久化**：追加记录每条写操作命令。策略：`always`（每条刷盘最安全但慢）、`everysec`（每秒刷盘，平衡）、`no`（操作系统决定）。文件大时通过 `bgrewriteaof` 重写压缩。
- **混合持久化**（Redis 4.0+）：结合 RDB 和 AOF，AOF 文件前半部分是 RDB 二进制压缩数据，后半部分是增量 AOF 命令。恢复快且数据全。
- **主从复制**：全量复制：Slave 首次连接 Master，Master 生成 RDB 发送给 Slave，Slave 加载 RDB 后通过 replication buffer 追增量数据。增量复制：Slave 短暂断开重连后，Master 从 replication buffer 或 backlog 中传输断开期间的增量数据。通过 offset 偏移量对比确定数据差。
- **Sentinel（哨兵）**：监控 Redis 主从节点，当 Master 主观下线（SDOWN）且多个 Sentinel 确认客观下线（ODOWN）后触发故障转移，选举新 Master，通知 Client 新地址。同时作为配置提供者。
- **Cluster（集群模式）**：数据分片到 16384 个 Slot，每个 Master 负责一段 Slot。通过 CRC16(key) % 16384 确定 Slot。支持 MOVED 重定向（Slot 不属于当前节点）和 ASK 重定向（Slot 正在迁移中）。节点间通过 Gossip 协议通信，支持主从故障转移。
- **分布式锁（Redisson）**：基于 Redis 的 `SET NX PX` 命令，Redisson 提供 Watchdog 自动续期（默认 30s，每 10s 续期 30s）。支持可重入锁（Hash 结构：Key 为锁名 + 线程 ID + 重入计数）、公平锁、联锁、红锁。
- **RedLock（红锁）**：在多个独立 Redis 实例上分别加锁，多数节点加锁成功且总耗时小于锁有效期才算成功。用于提高可用性，但存在争议（Martin Kleppmann 认为不安全）。
- **Pipeline（管道）**：客户端批量发送多个命令不等待响应，减少 RTT。不保证原子性，适用于无依赖关系的批量操作。
- **Lazy-Free（惰性删除/异步删除）**：Redis 4.0+ 使用 `UNLINK` 代替 `DEL` 异步释放大 Key 内存，避免阻塞。通过后台线程回收内存。
- **Lua 脚本**：通过 `EVAL` 或 `EVALSHA` 在 Redis 服务端原子执行多个命令（不会被其他命令打断），适用于需要原子性的复合操作。使用 `redis.replicate_commands()` 可实现随机写入的主从复制。
- **事务**：`MULTI` 开启事务，命令入队，`EXEC` 原子执行。不支持回滚（命令语法错误入队失败全部拒绝执行，运行时错误继续执行其他命令）。`WATCH` 实现乐观锁（CAS），`DISCARD` 取消事务。
- **布隆过滤器**：RedisBloom 模块或 Redisson 客户端实现，用于判断元素是否可能存在（有误判率，说无辜说有罪但不会说无罪为有罪）。适合缓存穿透防护。
- **Redisson**：Java 客户端，提供分布式锁、布隆过滤器、RateLimiter、原子 Long、Map/Set/List 等分布式数据结构和服务的封装。
- **大 Key 问题**：单个 Key 的 Value 过大（哈希/列表/集合元素过多）会导致慢查询、网络阻塞、主从切换卡顿。解决：拆分为多个小 Key（如 user:1:info, user:1:extend）、用 `UNLINK` 异步删除。

# 2. 面试题

## 2.1 Redis 的数据类型有哪些？每种的使用场景是什么？
- **String**：缓存（用户 Token、配置值）、分布式锁、计数器（INCR）、Session 共享。
- **Hash**：存储对象信息（用户信息字段，可部分更新）、购物车。
- **List**：消息队列（LPUSH + RPOP）、最新消息列表、时间线。
- **Set**：标签系统、好友关系、共同好友（SINTER 交集）、抽奖去重。
- **Sorted Set**：排行榜（ZADD + ZRANGE）、延时队列（Score 为到期时间戳）、分页排序数据。
- **Bitmaps**：签到打卡（SETBIT + BITCOUNT）、日活跃用户统计。
- **HyperLogLog**：UV 统计（PFADD + PFCOUNT）、大流量去重计数。
- **Geo**：附近的人、商店位置检索计算距离（GEORADIUS）。
- **Stream**：可靠消息队列（支持 Consumer Group、确认机制、消息回溯）。

## 2.2 Redis 的缓存穿透、击穿、雪崩是什么？如何解决？
- **缓存穿透**：查询一个数据库中也不存在的数据，缓存中没有对应 Key，请求直接打到数据库。解决方案：1) 布隆过滤器，先判断 Key 是否可能存在；2) 缓存空值，设置较短过期时间（如 5 分钟）。
- **缓存击穿**：热点 Key 过期瞬间，大量请求打到数据库。解决方案：1) 互斥锁/分布式锁，只有第一个请求回源加载数据，其余等待；2) 热点 Key 逻辑过期，永不过期，用后台线程异步刷新。
- **缓存雪崩**：大量 Key 同时过期或 Redis 宕机。解决方案：1) 过期时间 + 随机值，避免同时失效；2) Redis 集群高可用（主从 + Sentinel）；3) 服务限流降级，保护数据库；4) 多级缓存（本地 + 远程）。

## 2.3 Redis 的持久化方式有哪些？分别有什么优缺点？
- **RDB**：指定时间间隔将内存数据集快照写入磁盘。优点：文件紧凑，恢复速度快，适合冷备。缺点：可能丢失最近一次快照到宕机之间的数据，`bgsave` 会 fork 子进程消耗内存和 CPU。
- **AOF**：记录每条写操作命令追加到文件。优点：数据安全性更高，默认 `everysec` 策略最多丢失 1 秒数据，AOF 文件可读可修复。缺点：AOF 文件比 RDB 大，恢复速度慢。
- **混合持久化（4.0+）**：AOF 文件前部分是 RDB 压缩数据，后部分是增量 AOF 命令。结合二者优点，恢复快且数据相对全，是推荐策略。

## 2.4 Redis 的过期删除策略和内存淘汰策略是什么？
**过期删除策略**：Redis 同时使用惰性删除（访问 Key 时检查过期，过期即删）和定期删除（每隔 100ms 随机抽取部分设置过期时间的 Key，删除其中过期的）。

**内存淘汰策略**（`maxmemory-policy`）：
- `noeviction`：不淘汰，内存满时拒绝写入并报错（默认）。
- `volatile-lru`：从设置了 TTL 的 Key 中淘汰最近最少使用的。
- `allkeys-lru`：从所有 Key 中淘汰最近最少使用的。
- `volatile-lfu`：从设置了 TTL 的 Key 中淘汰最不经常使用的（4.0+）。
- `allkeys-lfu`：从所有 Key 中淘汰最不经常使用的（4.0+）。
- `volatile-random` / `allkeys-random`：随机淘汰。
- `volatile-ttl`：从设置了 TTL 的 Key 中淘汰即将过期的。

## 2.5 Redis 的 LRU 和 LFU 算法有什么区别？
- **LRU（Least Recently Used）**：淘汰最近最少使用的数据，关注最后一次访问时间。问题：周期性的批量访问可能导致缓存污染（冷数据被短暂高频的扫描操作推入缓存）。
- **LFU（Least Frequently Used）**：淘汰访问频率最低的数据，关注总访问次数。Redis 4.0 引入，通过衰减计数避免历史高频 key 长期占位。
- Redis 的 LRU 实现是近似 LRU（采样 N 个 Key 找最老的那个），不是严格的全局 LRU，内存开销小。

## 2.6 Redis 的主从复制原理是什么？
1. **全量复制**：Slave 首次连接或 `replication offset` 差距过大导致 backlog 不足时触发。Master fork 子进程生成 RDB 快照，同时将期间的变更写入 replication buffer。Master 发送 RDB 给 Slave，Slave 加载 RDB，然后 Master 发送 buffer 中的增量数据，Slave 执行追上 Master 状态。
2. **增量复制**：Slave 短暂断线重连后，Master 从 `repl_backlog` 缓冲区中查找 Slave 的 offset 之后的数据发送给 Slave，Slave 执行追上。如果 backlog 不包含所需 offset，则回退到全量复制。
3. 复制过程使用 `PSYNC` 命令，runid 标识 Master 身份，offset 标记同步位置。

## 2.7 Redis Sentinel（哨兵）如何实现故障转移？
1. **监控**：Sentinel 集群通过 PING 命令探测 Master 和 Slave 状态。
2. **主观下线（SDOWN）**：一个 Sentinel 发现 Master 在 `down-after-milliseconds` 时间内无响应，判定其主观下线。
3. **客观下线（ODOWN）**：该 Sentinel 询问其他 Sentinel 是否也认为 Master 下线，超过 `quorum` 个 Sentinel 确认后，判定为客观下线。
4. **Leader 选举**：客观下线后 Sentinel 间通过 Raft 协议选举出一个 Sentinel Leader 执行故障转移。
5. **故障转移**：Leader Sentinel 从健康的 Slave 中按照优先级（`slave-priority`）→ 复制偏移量（offset 越大越优先）→ runid（字典序越小越优先）选出一个 Slave 提升为 Master。通知其他 Slave 新 Master 地址，通知 Client 新 Master 地址。

## 2.8 Redis Cluster 的工作原理是什么？
1. **数据分片**：将 16384 个 Slot 分布在不同 Master 节点上。客户端通过 `CRC16(key) % 16384` 计算 Key 的 Slot，路由到对应节点。
2. **MOVED 重定向**：客户端请求的 Key 不在当前节点时，节点返回 MOVED 错误，告诉客户端正确节点地址。智能客户端会缓存 Slot 映射关系。
3. **ASK 重定向**：Slot 正在迁移过程中，节点返回 ASK 错误，客户端临时向目标节点发送 ASKING 命令获取数据。
4. **Gossip 协议**：节点之间通过 Gossip 协议交换集群元数据（节点状态、Slot 分配），最终一致性。
5. **故障转移**：每个 Master 有若干 Slave，Master 宕机后 Slave 选举为新 Master。

## 2.9 Redisson 的分布式锁如何实现？Watchdog 机制是什么？
Redisson 基于 Redis 的 Hash 结构实现可重入锁。加锁时：
1. `KEYS[1]` 为锁名，`ARGV[1]` 为锁过期时间（默认 30s），`ARGV[2]` 为线程 ID + 重入计数。
2. 通过 Lua 脚本原子执行：若锁不存在则创建锁并设置过期时间并返回成功；若锁已存在且线程 ID 匹配则重入计数 +1 并续期返回成功；否则返回锁剩余过期时间。
3. **Watchdog**：加锁成功后启动定时任务，每 10s 续期 30s（通过 Lua 脚本检查线程 ID 匹配，续期过期时间）。Watchdog 在解锁或 JVM 退出时取消。
4. 解锁时：检查线程 ID 匹配，重入计数 -1，若计数为 0 则删除锁并发布锁释放通知。

## 2.10 Redis 的 Pipeline 和事务有什么区别？Lua 脚本又有什么不同？
| 特性 | Pipeline | 事务（MULTI/EXEC） | Lua 脚本 |
|------|----------|-------------------|----------|
| 原子性 | 不保证 | 保证（不可中断） | 保证（不可中断） |
| 目的 | 减少 RTT | 批量原子执行 | 复杂逻辑原子执行 |
| 回滚 | 不支持 | 不支持运行时错误回滚 | 不支持 |
| 交互性 | 命令可无依赖 | 命令彼此独立 | 可读写、条件判断 |
| 网络 | 批量发送，一次性接收 | 批量发送，EXEC 一次性返回 | 一次发送脚本，减少网络 |

Lua 脚本适合需要原子性且涉及读写判断的复合操作，Pipeline 适合独立的批量写入/读取优化。

## 2.11 Redis 为什么这么快（单线程模型）？
1. **纯内存操作**：所有数据都在内存中，读写纳秒级。
2. **单线程模型**：避免多线程的上下文切换、锁竞争和死锁问题。所有命令串行执行，天然线程安全。
3. **IO 多路复用**：基于 epoll（Linux）/ kqueue（BSD）的事件驱动模型，一个线程高效处理大量并发连接。
4. **高效数据结构**：SDS、ziplist、skiplist 等精心设计的数据结构，时间和空间高效。
5. Redis 6.0+ 引入 IO 多线程（仅网络读写多线程，命令执行仍单线程），进一步提升性能。

## 2.12 Redis 如何保证缓存与数据库的一致性？
1. **Cache-Aside 模式（最常用）**：读：先查缓存，未命中则查 DB 并回写缓存。写：先更新数据库，再删除缓存（而不是更新缓存）。删除缓存的原因：更新缓存在高并发顺序写的情况下容易出现脏数据。
2. **延迟双删**：写完 DB 后删除缓存，等待一段时间（如 500ms）后再删除一次，确保中间读操作写入的旧缓存被清除。
3. **订阅 Binlog（Canal）**：监听 MySQL Binlog 变化，异步删除/更新缓存。最终一致性方案，适用于对一致性要求不极端的场景。
4. **分布式事务**：要求强一致性时可用 TCC、本地消息表等确保 DB 和缓存同步。

## 2.13 Redis 的主从复制中数据延迟大时如何处理？
更多是从设计角度避免：1) 读写分离时对一致性要求不高的读操作走 Slave，写操作和读己之写走 Master；2) 监控 `repl_offset` 差距，超阈值告警；3) 优化网络带宽和磁盘性能，减少复制延迟；4) 使用 Sentinel/Cluster 自动故障转移，减少单点压力。

## 2.14 什么是 Redis 的大 Key？有什么危害？如何解决？
**大 Key**：单个 Key 的 Value 过大（Hash/List/Set/ZSet 元素过多或是超长 String）。
**危害**：1) 慢查询，阻塞 Redis 单线程；2) 删除大 Key 时造成 Redis 阻塞；3) 网络带宽瓶颈；4) 主从复制时可能阻塞 Slave。
**解决**：1) 设计层面拆分为多个小 Key（如 `user:1:info`、`user:1:order`）；2) 删除时使用 `UNLINK`（异步删除，4.0+）；3) 大 Hash 使用 `HSCAN` 分批删除；4) 大 Set 使用 `SSCAN` + `SREM` 分批删除。

## 2.15 RedLock 红锁是什么？有什么争议？
RedLock 是 Redis 作者 antirez 提出的分布式锁算法：在 N 个独立 Redis Master 节点（无主从关系）上依次尝试加锁，当在 `(N/2+1)` 个节点上加锁成功且总耗时小于锁的有效时间，则认为加锁成功。争议点：Martin Kleppmann 指出 RedLock 依赖的时间假设（时钟单调性）在现实中可能被 GC 暂停、时钟跳跃等打破，导致安全性问题。实际生产使用单实例 + Redisson Watchdog 已足够大多数场景。

## 2.16 Redis 的 Stream 数据结构与 List 做消息队列有什么区别？
| 特性 | List | Stream |
|------|------|--------|
| 消息持久化 | 消费即删除（LPOP） | 消息不删除，可重复消费 |
| Consumer Group | 不支持 | 支持，组内分摊消费 |
| 消息 ACK | 不支持 | 支持 ACK/NOACK |
| Pending 队列 | 无 | 有（未 ACK 消息待处理列表） |
| 消息回溯 | 不支持 | 支持（通过消息 ID 回溯） |
| 阻塞读取 | BLPOP/BRPOP | XREAD BLOCK |
Stream 更适合可靠消息队列场景，List 适合简单的「用后即焚」场景。

## 2.17 Redis 底层数据结构有哪些？每种值类型分别用什么实现？
| Redis 类型 | 底层数据结构（按数据量/元素大小自动切换） |
|-----------|--------------------------------------|
| String | SDS（Simple Dynamic String），二进制安全，记录 len/free/buf |
| List | 3.2 前：ZipList（小数据）/ LinkedList；3.2+：QuickList（ZipList + 双向链表混合） |
| Hash | 7.0 前：ZipList / HashTable（Dict）；7.0+：ListPack / HashTable |
| Set | 纯整数小集合：IntSet；否则：HashTable（value 为 NULL 的 Dict） |
| ZSet | 7.0 前：ZipList / SkipList + Dict；7.0+：ListPack / SkipList + Dict |
| Stream | Rax（Radix Tree 基数树）存储消息；ZipList/ListPack 存小批量消息 |

**核心底层结构详解**：
- **SDS**：C 字符串的替代品。优点：O(1) 获取长度（len 字段）、杜绝缓冲区溢出（自动扩容）、二进制安全（不依赖 `\0`）、惰性空间释放。
- **Dict（哈希表）**：两个 HashTable 交替使用，支持渐进式 Rehash。使用 MurmurHash2 计算 hash。
- **SkipList（跳表）**：多层有序链表，每层概率上升，查找 O(log n)，支持范围查询。
- **ZipList（压缩列表）**：连续内存块，header + entries + end，每个 entry 记录前一个 entry 的长度，支持双向遍历。省内存但当元素多时增删需要连锁更新。
- **ListPack（紧凑列表）**：替代 ZipList（7.0+），每个 entry 只记录自身长度（不记录前一个），避免连锁更新问题。
- **QuickList**：链表中每个节点是一个 ZipList/ListPack。平衡链表灵活性和压缩列表内存效率。
- **IntSet**：纯整数小集合，内部用有序数组存储，二分查找。编码自动升级（int16→int32→int64），不降级。

## 2.18 Redis 的渐进式 Rehash 是什么？
Redis 的 Dict（哈希表）扩展/收缩时使用渐进式 Rehash，避免一次性迁移所有数据导致服务阻塞。

**机制**：
- Dict 内部持有两个 HashTable：`ht[0]`（旧表）和 `ht[1]`（新表，容量为旧表 2 倍或收缩为 1/2）。
- 设置 `rehashidx=0` 表示开始 Rehash。
- 每次对 Dict 的增删改查操作，同时将 `ht[0]` 中 `rehashidx` 位置的桶迁移到 `ht[1]`，`rehashidx++`，直到 `rehashidx` 达到 `ht[0].size` 时 Rehash 完成→释放 `ht[0]`→`ht[1]` 变为 `ht[0]`。
- **定时辅助迁移**：Redis 的 `serverCron` 每 100ms 检查并迁移 1ms 的时间片（配置项 `hz`），避免无操作时的 Rehash 停滞。

**期间的查询**：先在 `ht[0]` 查，查不到再到 `ht[1]` 查。新增操作直接写入 `ht[1]`。删除/更新需同时检查两个表。

**触发时机**：
- 扩展：负载因子 `used/size > 1`（无 BGSAVE/AOF 重写时）或 `> 5`（任何时候强制扩展）。
- 收缩：负载因子 `< 0.1`。

## 2.19 Redis 6.0+ 多线程模型详解
Redis 6.0 引入了**IO 多线程**，但命令执行仍然是**单线程**。本质是"多线程处理网络读取/写入"+"单线程执行命令"。

**为什么 Redis 6.0 之前是单线程还能这么快？**
1. 纯内存操作。
2. 高效的数据结构（SkipList、Dict、SDS 等）。
3. I/O 多路复用（epoll），单线程处理所有连接请求，无上下文切换和锁竞争。
4. 瓶颈在网络 I/O 而非 CPU（单个 Redis 实例 QPS 可达 10W+）。

**6.0 为什么要引入多线程？**
随着硬件发展（多核 CPU）和网络带宽提升（10Gbps+），单线程处理网络 I/O 成为瓶颈——大量连接同时收发数据时，一个线程对 Socket 的读/写操作成为限制。

**6.0 多线程模型**：
- 主线程（单线程）：接收客户端连接（accept），执行读写事件分发，**执行所有 Redis 命令**（命令处理仍是单线程，保持原子性）。
- IO 线程池（可配置 2~N 个线程）：主线程将多个 Socket 的读/写操作分发给 IO 线程并行处理，处理完成后交回主线程。

**配置方式**：`io-threads 4`（建议不超过 CPU 核心数）、`io-threads-do-reads yes`（开启多线程读取）。

**核心结论**：Redis 6.0+ 仍然是"单线程执行命令"，多线程只用于网络 I/O 加速。在高并发连接场景下（数千甚至数万连接），多线程可以显著提升吞吐。

## 2.20 Redis Cluster 的 Hash Slot 迁移过程（MOVED 和 ASK）
Redis Cluster 使用**16384 个 Hash Slot** 分布数据（`CRC16(key) % 16384`）。

**MOVED 重定向**：当客户端请求的 Slot 不在当前节点的服务范围内时，节点返回 `MOVED <slot> <ip>:<port>`。客户端收到后**永久更新** Slot→节点映射表，重新向正确节点发起请求。这是标准的 Slot 归属变更通知。

**ASK 重定向**：发生在 **Slot 正在迁移过程中**。迁移时源节点对已迁移的 Key 返回 `ASK <slot> <ip>:<port>`。客户端收到后：
1. 临时向目标节点发送 ASKING 命令（告知目标节点"我虽然 Slot 不在你这里，但请处理这个请求"）。
2. 再重新发送原请求。
3. ASK 重定向**不更新客户端的 Slot 映射表**——因为迁移尚未完成，Slot 仍然正式归属于源节点。

**Hash Slot 迁移流程**：
1. 通过 `CLUSTER SETSLOT <slot> MIGRATING <source-id>` 标记源节点为 MIGRATING。
2. 通过 `CLUSTER SETSLOT <slot> IMPORTING <target-id>` 标记目标节点为 IMPORTING。
3. 用 `MIGRATE` 命令逐个迁移 Key 中的数据。
4. 迁移完成后，通过 `CLUSTER SETSLOT <slot> NODE <target-id>` 向整个集群宣布 Slot 归属变更。

迁移过程中：
- 源节点：对请求的 Key 存在则处理，不存在返回 ASK 重定向。
- 目标节点：收到带 ASKING 的请求时正常处理（否则返回 MOVED）。

## 2.21 ZSet 为什么用跳表而不是红黑树？
| 维度 | 跳表（SkipList） | 红黑树 |
|------|-----------------|--------|
| 范围查询 | O(log n) 找到起点后，只需线性遍历（第一层双向链表直连） | O(log n + k)，但需中序遍历，实现复杂 |
| 实现复杂度 | 简单（构建/删除只需修改相邻节点指针） | 复杂（插入/删除需旋转+变色维持平衡） |
| 内存占用 | 略高（平均每节点约 1.33 倍额外指针） | 紧凑（每节点一个额外颜色位 + 左右指针） |
| 并发友好性 | 可做无锁实现 | 难以无锁化 |
| 范围查询效率 | 极高（底层有序双向链表直接遍历） | 一般（需要递归/栈式中序遍历） |

**Redis 选择跳表的核心原因**：ZSet 最核心的操作为 `ZRANGE`（按排名范围查询）和 `ZRANGEBYSCORE`（按分数范围查询）。这两种范围查询在跳表上只需 O(log n) 定位起点后直接线性遍历即可，而在红黑树上需要复杂的中序遍历。跳表实现简单、调试直观，且 Redis 作者认为同样 O(log n) 下跳表更好维护。

**Redis ZSet 的双重结构**：ZSet 同时使用 **SkipList + Dict**，Dict 按成员名 O(1) 查找分数，SkipList 按分数 O(log n) 范围查询。两者共用同一份元素（指针指向同一对象）。

## 2.22 Redis 如何实现延迟队列？
**方案一：ZSet 实现（最常用）**
- 将延时任务放入 ZSet，score = 执行时间的时间戳（毫秒）。
- 后台轮询线程：`ZRANGEBYSCORE key 0 now_timestamp LIMIT 0 N` 获取到期的任务。
- 取出后 `ZREM` 原子删除，避免重复消费。
- 缺点：依赖轮询精度（秒级），多个实例竞争时可能重复消费，可结合分布式锁解决。

**方案二：Redis Stream + 消费者组**
- 不支持延迟投递，但可以通过消息的 `IDLE` 机制实现延迟重试。
- `XREADGROUP` 的 Pending Entries List (PEL) + `XCLAIM` 转移超时未 ACK 的消息（类似 RocketMQ 的重试机制）。

**方案三：Redis 7.0+ 的过期监听（Keyspace Notifications）**
- 设置 Key 的过期时间（`EXPIRE`），通过 `__keyevent@*__:expired` 频道订阅过期事件。
- 缺点：过期事件的实时性和可靠性无法 100% 保证（Redis 过期删除是惰性+定期），不适合严谨的延迟队列场景。

**生产实践**：精确延迟用 ZSet + 轮询；秒级以上精度、任务量不大的场景足够。微秒级或需要 ACK 机制的建议直接用 RocketMQ 延时消息或专业的调度中间件。

## 2.23 Redis 的慢查询日志如何使用？
**配置**：
```conf
slowlog-log-slower-than 10000  # 超过 10000 微秒（10ms）的命令记为慢查询
slowlog-max-len 128            # 最多保存 128 条慢查询记录（FIFO 队列，内存中）
```

**查看慢查询**：
- `SLOWLOG GET [N]`：返回最近 N 条慢查询。每条记录含：日志 ID、时间戳、执行耗时（微秒）、命令和参数、客户端 IP:Port。
- `SLOWLOG LEN`：当前慢查询总数。
- `SLOWLOG RESET`：清空慢查询日志。

**慢查询原因排查**：
1. **Value 太大**（Big Key）：`GET` 大 Key 造成网络传输慢；`DEL` 大 Key 阻塞主线程（用 `UNLINK` 异步删除）。
2. **命令复杂度高**：`KEYS *`（全量扫描，禁止生产使用）、`HGETALL` 大 Hash、`SMEMBERS` 大 Set（改用 `SCAN`、`HSCAN`、`SSCAN` 分批）。
3. **O(N) 命令在 N 大时**：`SORT`、`ZRANGE` 全部元素、`SUNION` 多个大 Set。
4. **持久化影响**：BGSAVE 期间 fork() 复制页表导致短暂延迟。
5. **CPU 繁忙**：Redis 绑定单核时跑满 100%，需考虑多实例绑定不同核或迁移至更强 CPU。
