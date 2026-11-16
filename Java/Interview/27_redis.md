本篇覆盖 Redis 的高频面试题，按"主题块 → 详细分块 → 题目 → 解答"组织。五个主题块：性能模型与数据结构、缓存应用、持久化与高可用、分布式协调与原子操作、运维与开发实践。缓存与数据库一致性的 MySQL 侧配合见 16_mysql.md，Binlog 订阅见 14_canal.md，消息语义对比见 13_rocketmq.md。

# 1. 性能模型与数据结构

## 1.1 服务模型

### 1.1.1 Redis 为什么这么快？

四个层面（答题按序展开）：

1. **纯内存**：操作纳秒级，磁盘只用于持久化异步落盘；
2. **单线程执行命令**：无锁竞争、无线程上下文切换、天然串行原子——省掉并发编程最贵的东西；
3. **IO 多路复用**：epoll/kqueue，单线程 event loop 管理数万连接；命令解析、请求排队在内存完成；
4. **高效数据结构**：SDS、字典、跳表、紧凑编码（ziplist→listpack）为"内存换 CPU/CPU 换内存"精心取舍（1.2.2）——不是泛型容器，而是按数据规模自动切换编码。

**追问：单线程为什么还怕慢命令？**——事件循环串行执行，`KEYS *`/大集合 O(N) 命令**阻塞全部后续请求**（2.2.3/5.1.1）；"快"建立在"每条命令微秒级"的自律上。6.0 的多线程只解决网络收发瓶颈（1.1.2），执行仍单线程。

### 1.1.2 Redis 6.0 的多线程模型是怎样的？

**背景**：瓶颈从 CPU 转向**网络 IO**——万兆网卡 + 海量连接下，单线程读 socket、解析请求、写回响应本身成为上限。

**模型（io-threads 默认关闭）**：

- 主线程：accept、命令**执行**（仍单线程，原子性语义完全不变）；
- IO 线程池：主线程把一批已就绪 fd 的 **read+协议解析**、以及**reply 写回**分派给 2~N 个 IO 线程并行做，主线程等栅栏（job done）后再逐条执行；
- 配置：`io-threads 4`（建议 ≤ 核数、8 以内），读写分发细节各版本有差异（6.0 写并行、读并行选项）。

**结论句**："多线程收发、单线程执行"——**执行原子的承诺没变**，变的只是把 syscall/memcpy 搬离主线程。另有后台线程家族（lazy free、bio 刷盘/fsync/AOF 关闭）一直在主线程外（大 Key 删除 UNLINK 走 bio，见 2.2.3）。

## 1.2 类型与内部结构

### 1.2.1 Redis 有哪些数据类型？典型场景？

| 类型 | 核心命令 | 典型场景 |
|------|---------|---------|
| String | SET/GET/INCR/SETEX/GETSET | 对象 JSON 缓存、计数器/限流、分布式锁值、session/token |
| Hash | HSET/HGET/HINCRBY/HSCAN | 对象字段级读写（购物车、用户资料——局部更新不回写整串） |
| List | LPUSH/RPUSH/LPOP/BLPOP/LRANGE | 简单队列、最新列表（时间线） |
| Set | SADD/SISMEMBER/SINTER/SDIFF | 标签、共同好友（交集）、抽奖去重、关注关系 |
| ZSet | ZADD/ZRANGE/ZRANGEBYSCORE/ZREMRANGEBYSCORE | 排行榜、延迟队列（score=触发时间）、带权任务 |
| Bitmap | SETBIT/BITCOUNT/BITOP | 签到（天=bit）、活跃标记、按位统计 |
| HyperLogLog | PFADD/PFCOUNT/PFMERGE | UV 基数统计（12KB 固定内存、误差 0.81%） |
| Geo | GEOADD/GEOSEARCH | 附近的人/店（本质 ZSet+Geohash，1.2.2 注） |
| Stream | XADD/XREADGROUP/XACK | 可靠消息（消费组/ACK/回溯，见 4.2.3） |

**口径**：前五种是"万物之基"，后四种是派生结构（Bitmap=String 位视图、Geo=ZSet、HLL=特殊 String）——说"Redis 有 5 种基本类型 + 扩展类型"更显体系感。

### 1.2.2 各类型底层用什么编码？为什么这样设计？

| value 类型 | 编码切换（简化） |
|-----------|-----------------|
| String | `int`（可转 long）→ `embstr`（≤44B，一次分配）→ `raw`（SDS+buf 两次分配）；数字另有 float→ 字符串 |
| Hash | 7.0 前 list-max-zipmap-*；现：**listpack**（字段少）→ **hashtable**；`hash-max-listpack-entries 128/value 64B` |
| List | **quicklist**（双向链表，节点= listpack）；>100 且全索引访问老配置下可整转 listpack，默认 quicklist；7.x `lpush/rpush` 优化 |
| Set | **intset**（int16→32→64 升级不降级）→ **hashtable** |
| ZSet | **listpack**（元素少）→ **skiplist + hashtable 双持**（1.2.3） |
| Stream | **rax 基数树**（前缀压缩）+ listpack 装条目 |

**设计主线**：**小数据省内存（连续内存 listpack/intset），大数据保性能（dict O(1)/skiplist O(log n)）**，超阈值单向升级。listpack 替代 ziplist 的原因：ziplist 每节点存"前一节点长度"，连锁更新可能引发最坏 O(N) 重排；listpack 只记自身长度，无连锁问题。

**SDS 四大优势**（必背）：O(1) 取长度（len 字段）；二进制安全（记长度不靠 `\0`，可存图片字节流）；防溢出（拼接自动扩）；**预分配+惰性释放**减少 realloc 次数；另有 `alloc/buf` 可复用空间。

### 1.2.3 ZSet 为什么用跳表 + 字典双结构，而不用红黑树？

**双持设计**：dict（member→score，等值查 O(1)）+ skiplist（score 序，范围与排名）共享同一批对象——一个场景两种入口。

**vs 红黑树**（核心差异是范围查询形态）：

| 维度 | 跳表 | 红黑树 |
|------|------|--------|
| 范围查询 | O(log n) 定位起点后**底层双向链表直接顺扫** | 中序遍历，实现绕 |
| 实现/调试 | 改指针即可插入删除，简单 | 旋转+变色，复杂 |
| 并发改造 | 容易做局部锁/无锁 | 难 |
| 内存 | 每节点平均 1.33 层指针 | 略省 |

**结论句**：ZSet 的主战场是 `ZRANGE/ZRANGEBYSCORE`（排行、延迟队列都是范围语义），跳表"定位后线性扫"完胜红黑树中序；实现简单是作者亲口确认的第二理由。B+ 树更不行（磁盘结构，内存场景多付合并成本）。

### 1.2.4 什么是渐进式 rehash？

**问题**：dict 扩容若一次性搬迁 2^n 桶，单线程模型下 = 整库停摆毫秒~秒级。

**双表方案**：dict 持有 `ht[0]/ht[1]`；触发扩容（负载因子 >1 且无 fork 子进程 / >5 无条件；收缩 <0.1）时按 2 倍建 ht[1]、`rehashidx=0`：

1. 此后**每次对 dict 的读写都顺带搬一个桶**（rehashidx 递增），期间查询"先 ht[0] 后 ht[1]"、新增直入 ht[1]；
2. serverCron 定时辅助推进（每轮 1ms 时间片，`hz` 控制），防止 dict 无人访问停滞；
3. 搬完释放 ht[0]、ht[1] 转正。

**面试点**：这是"单线程不能被长操作阻塞"哲学在数据结构层的体现——同族方案：`UNLINK`/lazy free、big Key SCAN 分批、rehash 与 fork 互斥（有 bgsave 时只允许 >5 强扩，防内存翻倍压力）。

# 2. 缓存应用

## 2.1 读写模式与三高危

### 2.1.1 缓存有哪些读写模式？Cache-Aside 为什么是默认答案？

| 模式 | 读写职责 | 特点 |
|------|---------|------|
| **Cache-Aside**（旁路） | 读：未命中查库并回填；写：更库 + **删缓存** | 应用直管，最常用最可控 |
| Read-Through | 读时缓存组件自己回源 | 应用只依赖缓存；回源逻辑收口组件 |
| Write-Through | 写同时更缓存+库（同步） | 数据最新但写放大、写延迟 |
| Write-Behind/Back | 只写缓存，异步批量刷库 | 吞吐最高、**可能丢数据**，实现复杂（Redis 上少见，本地写缓冲/SSD 场景多） |

**Cache-Aside 两条铁律**：① 写时**删缓存而不是更缓存**——并发写序列下"更新缓存"会让旧值后到（更缓存失败/交错产生永久脏数据），删除把问题降为"最多缓存多活一会儿"；② 删除发生在**DB 提交之后**（提交前删 = 旧读请求回填旧值）。

### 2.1.2 如何保证缓存与数据库的一致性？

**先摆立场**：不做分布式事务时，缓存-DB 只能做到**最终一致**——面试答"方案谱系 + 选型逻辑"：

1. **Cache-Aside + 删后验**（主流）：更新 DB → 删缓存；极端乱序（读旧库回填在删之后）概率极低且自愈——接受它；
2. **延迟双删**：删缓存 → 延迟 T（略大于单次读写耗时）→ 再删一次——盖掉"回源竞态"回填的旧值；T 拍脑袋、重试兜底缺失是它被吐槽的点；
3. **订阅 binlog 失效缓存（Canal/Debezium）**：DB 是唯一真相，变更事件异步驱删缓存——业务代码零侵入、削峰天然，**生产推荐组合**；注意乱序（同 key 按主键分区保序）；
4. **强一致场景不用缓存**：Redis 加锁/事务消息做不了"DB+缓存原子更新"，需要就别缓（或用 `EVAL` 单实例原子协议自欺有限）。

**配套追问**："删除失败怎么办？"——删缓存重试（本地消息表/队列）+ TTL 兜底（缓存必须有期）+ 订阅 binlog 补偿，三道防线。

### 2.1.3 缓存穿透、击穿、雪崩分别是什么？怎么解？

| 现象 | 本质 | 方案 |
|------|------|------|
| **穿透**：查**根本不存在**的数据，每次绕过缓存打库 | 缓存"空结果"不落 | ① **缓存空值**（短 TTL 60~300s，简单）；② **布隆过滤器**挡在入口（2.1.4，海量 key 空间适用，注意删除/更新维护）；③ 接口层参数校验/风控（恶意 id 枚举） |
| **击穿**：**单个热点 key 过期瞬间**，并发齐打库 | 回源风暴集中一度 | ① **互斥重建**：`SET lock NX PX` 只放一个请求回源，其余等待或吐旧值；② **逻辑过期**：value 内嵌过期时间，发现过期异步刷新、主流程继续用旧值（可用性优先）；③ 热点永不过期+后台更新 |
| **雪崩**：**大量 key 同时到期/Redis 宕机**，库被整波打穿 | 过期时刻共振、单点 | ① TTL 加随机扰动（基础卫生）；② 多级缓存（本地 Caffeine 挡层）；③ 集群高可用（Sentinel/Cluster，3.2 系列）；④ **限流降级**（Sentinel 对 DB 回源限并发，见 20_sentinel.md）；⑤ Redis 进程挂：熔断只走缓存侧读降级 |

**加分句**：击穿/穿透在"热点 key"场景是同一把锁治理；雪崩的"集群宕机"分支不是过期问题而是**容量与容灾**问题——两类方案别混着答。

### 2.1.4 布隆过滤器的原理和误差特性？

**结构**：m 位数组 + k 个哈希函数。加入：k 个位置 1；查询：k 位**全为 1 → 可能存在；任一位为 0 → 一定不存在**。

**误差**：只有 **false positive（误判在）**，没有 false negative；误判率随容量逼近设计值上升，近似 `(1-e^{-kn/m})^k`——设计时按"预期元素 n + 目标误判 p"反推 m、k（RedisBloom/Redisson 都自动算，k≈7~8 量级常见）。

**工程要点**：

1. 标准 Bloom **不支持删除**（一位多键共享）→ Counting Bloom/quotient filter；**业务数据被删时过滤器要重建或容忍"多放行"**（穿透防线退化为缓存空值兜底，2.1.3）；
2. 误判在 = 放行打库一次 + 落空值缓存，业务通常可接受；
3. Redis 形态：**RedisBloom 模块**（BF.ADD/BF.EXISTS）或 Redisson RBloomFilter（位数组分片 String 实现）；
4. 海量去重（判"今天已读"）用 BitMap/HyperLogLog 更合适（1.2.1 口径）。

## 2.2 内存治理

### 2.2.1 过期 key 怎么删？内存满了怎么淘汰？

**过期删除 = 惰性 + 定期双策略**：

1. 惰性删除：访问时检查过期即删——CPU 省、内存脏（过期 key 堆着）；
2. 定期删除：每 100ms（`hz=10`）从带 TTL 的库**随机抽 20 个**删已过期的，若过期比例 >25% 立刻再一轮直到低于阈值/循环 5 轮上限——在 CPU 与内存之间取平衡；极端场景（海量短 TTL 同时到期）会触发长循环抖动（latency spike 来源之一）。

**内存淘汰（`maxmemory-policy`，达 maxmemory 时）**：

| 策略 | 范围/依据 |
|------|----------|
| noeviction（默认） | 不淘汰，写命令报错（缓存+业务共库要谨慎） |
| allkeys-lru / volatile-lru | 全体 / 仅带 TTL：淘汰最久未用 |
| allkeys-lfu / volatile-lfu | 4.0+：淘汰访问频率最低（计数带衰减，机制见 2.2.2） |
| allkeys-random / volatile-random | 随机 |
| volatile-ttl | 最先过期的先走 |

**选型**：纯缓存 → allkeys-lru（热点自然留存）；缓存与临时数据混合 → volatile-*（保护无 TTL 业务 key，但易"可淘汰 key 全清光"仍报错）；访问呈"幂律热点"时 **LFU 优于 LRU**（扫描型污染免疫，10 位计数带衰减：旧频率每周期 `(age-counter)/2` 递减）。

### 2.2.2 Redis 的 LRU 是近似的？LFU 怎么统计频率？

**近似 LRU**：不给全局链表——**每个对象存 24bit lru（秒级时间戳）；淘汰时随机抽 `maxmemory-samples`（默认 5）个 key，淘汰其中最久未访问者**。O(samples) 代替 O(N)；samples 调大（如 10）逼近严格 LRU。Redis 的 LRU 时钟还兼职**空闲时间/idle 查询**与对象年龄估算。

**LFU**：对象 24bit 拆成 **16bit 分钟级时间戳 + 8bit 计数器**（255 饱和）；命中时按概率 `p = 1/(counter+1)` 决定是否 +1——对数计数思想：**热者恒热但爬升慢，冷 key 偶发访问不再污染计数**；衰减项按 `lfu-log-factor/lfu-decay-time` 周期把计数下调，"过去很热但现在没人访问"会被逐步挤出。

**追问**：为什么不用严格 LRU？——全链维护每次访问 O(1) 链表摘挂 = 每命令多两次指针改写，近似采样把成本换成了概率精度，符合"够用哲学"。

### 2.2.3 什么是大 Key？有什么危害，怎么治理？

**界定**（经验值，无官方线）：String 值 >10KB；集合元素 >5000 或总大小 >10MB；具体看**单命令耗时与传输量**。

**危害链**：① 单线程被 O(N) 操作卡住（GET 大 value、HGETALL 万级 field、SMEMBERS 百万成员）→ 全局延迟尖刺；② **DEL 大 Key 同步释放阻塞秒级**（`UNLINK` 异步化，4.0+，5.0 默认化方向）；③ 网络带宽瞬时打满（一次回传几百 MB）；④ 主从复制/迁移集中卡顿、集群 slot 不均衡（一个节点 50% 数据压一个 key）；⑤ 触发 fork COW 内存暴涨（页被打散）。

**治理**：

1. **发现**：`redis-cli --bigkeys`（按类型采样）、`--memkeys`（按内存加权）、4.0 `MEMORY USAGE key` 精算、云控制台/PROTOCOL INFO；
2. **拆分**：逻辑分桶（`user:1:info`/`user:1:ext`）、大 Hash 按 hash tag 前缀散成 N 份（`uid:hash:{bucket}`）；
3. **删除**：UNLINK；HSCAN/SSCAN/ZSCAN 游标分批删；
4. **协议侧**：避免 HGETALL/SMEMBERS 拉全量，改 HMGET 指定字段、SCAN pipeline 分批。

### 2.2.4 什么是热 Key？怎么解决？

**界定**：单 key QPS 超出单分片承载能力（典型：秒杀商品详情、爆款榜单、春晚红包配置），瓶颈不是总量而是**同一个 slot 落在同一个节点上**——Cluster 分片救不了单 key。

**方案谱系**：

1. **多副本读**：Redis 从库分摊读 + 客户端加权（简单，扩副本线性缓解）；
2. **本地缓存（多级）**：key 打到应用进程 Caffeine（毫秒级失效策略：短 TTL+事件广播失效）——单机挡掉 90% 热读；
3. **key 打散**：`hotkey:{0..N-1}` 写入时随机选分桶、读时先查桶再回源合并（排行榜常用"分段汇总"）；
4. **探测**：4.0+ `HOTKEY`（redis-cli --hotkeys）、`OBJECT FREQ`（需 LFU）、云厂商热 key 实时分析；
5. **终极**：热点对象改推送模式（配置中心/长连接推全量副本，见 17_nacos.md 思路）。

**与击穿的区分**（追问高频）：击穿是"过期瞬间并发回源"，热 Key 是"持续高并发命中同一 key"——前者防回源风暴（互斥/逻辑过期），后者防单分片过载（副本/本地缓存/打散），方案正交。

# 3. 持久化与高可用

## 3.1 持久化机制

### 3.1.1 RDB 和 AOF 的区别？怎么选？混合持久化是什么？

| 维度 | RDB | AOF |
|------|-----|-----|
| 本质 | **某时刻全量内存快照**（二进制） | **写命令追加日志**（文本/RESP 协议） |
| 触发 | `save` 规则（`save 3600 1` 等）/`bgsave` | 每条写命令 append，`bgrewriteaof` 压缩 |
| 恢复 | 快（直接载入内存映射） | 慢（重放命令） |
| 数据安全 | 两个快照间宕机**全丢**（分钟级） | 取决于刷盘策略（everysec 丢 ≤1s） |
| 体积/开销 | 紧凑，fork 瞬间有内存/COW 压力 | 文件大，重写时同样 fork |

**选型**：纯缓存（数据可重建）可只 RDB 甚至关持久化；有"数据资产"含义（锁、计数器、消息体）必须 AOF；**生产默认 4.0+ 混合持久化**：`aof-use-rdb-preamble yes`——AOF 重写时先以 RDB 形式写文件头、增量命令续后，恢复=载快照+重放尾部，兼得速度与新鲜度。

**fork 机制（高频追问）**：bgsave/bgrewriteaof 都靠 **fork + 写时复制（COW）**——子进程共享页表，主进程继续服务；写命令越多被复制的页越多（大内存+高写入实例 fork 时内存瞬时近 2 倍，OS 页表拷贝也有毫秒级停顿）；"持久化期间延迟尖刺"根因即在此（配 `save` 规则别在业务峰值触发，云实例用 replica 做落盘基准）。

### 3.1.2 AOF 刷盘策略和重写机制？

**appendfsync 三档**（对齐 16_mysql.md 的 WAL 权衡观）：

- `always`：每条命令 fsync——最安全最慢（单线程下直接限死 QPS）；
- **`everysec`（默认）**：每秒 fsync 一次——**最多丢 1 秒**，性能与安全的行业默认；
- `no`：只 write 到 OS page cache，何时刷由 OS——最快，宕机丢失不可控（机器断电全没）。

**写流程分解**：命令先 append 内核缓冲区 →（每 1s）write →（依策略）fsync——两层节流独立。

**重写（bgrewriteaof）**：不是"压缩旧文件"而是**按当前内存态生成等价命令集**（`SET k v` 一条替代 N 次 `INCR`）；期间新写命令进 rewrite buffer，子进程完成新文件后合并替换——同样走 fork/COW；触发线 `auto-aof-rewrite-min-size`（64MB）+ `percentage`（100%）。**坑**：`no-appendfsync-on-rewrite`（重写期间暂停 fsync 降抖动 vs 丢更多秒数的取舍）；混合持久化下重写同时产出 RDB 头。

## 3.2 复制与高可用架构

### 3.2.1 Redis 主从复制的原理？全量和增量怎么触发？

**拓扑职责**：主节点 `replicaof` 指向（或独立）；从节点重放写命令实现数据副本——**从默认只读**（`replica-read-only yes`），复制命令本身不记录。

**握手链路（PSYNC，2.8+）**：

1. 从节点带 **replrunid + 自身 offset** 发 PSYNC；
2. 主节点判断：**完整重同步（全量）**——首次连接/runid 不匹配（主重启未持久化）、offset 不在 **repl_backlog**（环形缓冲区，默认 1~10MB）范围内：主 fork 生成 RDB（或磁盘已有则直接传）发给从加载，期间新命令进 **replication buffer**，随后统一转发并进入命令流实时复制；
3. **部分重同步（增量）**：runid 一致且 backlog 覆盖从节点落后 offset——只补发缺口命令（断线重连的标准路径）；
4. 稳定态：主把写命令同步进所有从的连接缓冲，从按序重放——**异步复制**，主不等等。

**配套参数**：`repl-backlog-size`（调大保断线增量窗口）、`repl-backlog-ttl`、无盘复制 `repl-diskless-sync`（大 RDB 走网络不落盘）；从节点服务旧数据窗口：`replica-serve-stale-data`（同步中是否继续应答——击穿式缓存读会拿到旧库，知晓即可）。

**危险配置**：`replicaof` 写在 redis.conf 里，主节点重启会**变从**（脑裂/拓扑错乱源）——云/K8s 场景用外部编排管理角色。

### 3.2.2 主从延迟与数据不一致怎么处理？

**延迟来源**：网络带宽抖动、从节点重放单线程（主 10 万 QPS 从必然跟不上）、从做大 Key 删除/fork、`repl-diskless` 配置不当。观测：`INFO replication` 各从节点 `master_link_status/second_repl_offset` 差值、`lag` 字段。

**后果分级与对策**：

1. **缓存场景**：容忍旧读（本就是弱一致），TTL 兜底；
2. **一致性要求高**："读己之写"——写后立即读走主（会话打标：写事务后短窗口内强制路由主库，语义同 16_mysql.md 6.1.3）；
3. **等待复制**：客户端拿主写响应里的 offset，`WAIT numreplicas timeout` 等 N 个从确认（**弱保证**：超时返回不足数、从 ack 后仍可能丢——不能当强一致用）；
4. **不丢数据**：RedLock/共识路线（3.2.4 与 4.1.3）；**主从异步 = 切换即可能丢未复制写**（这是 Redis HA 的共识级局限，必答）。

### 3.2.3 Sentinel（哨兵）的原理与故障转移流程？

**哨兵集群做什么**：监控（ping）、通知、**自动故障转移**、配置提供（客户端问哨兵"现在主是谁"）。部署：≥3 节点奇数，独立进程。

**五步流程**：

1. **主观下线（SDOWN）**：单哨兵在 `down-after-milliseconds` 内没拿到 PONG；
2. **客观下线（ODOWN）**：该哨兵问其他哨兵，达到 `quorum` 同意——解决"网络分区误判单点"；
3. **哨兵 Leader 选举**：Raft 式投票（`sentinel-parallel-syncs` 控迁移并发）；
4. **故障转移**：Leader 在健康从中排序选新主——**优先级（replica-priority）> 复制数据新（offset）> runid 小**；
5. **收尾**：其余从 `replicaof` 新主、旧主复活降为从、发布配置变更、客户端重定向。

**易错追问**：① 脑裂——旧主分区期仍接写、恢复后成从被"清库"追平（`min-replicas-to-write/min-replicas-max-lag` 让旧主自我禁写缩小窗口，不能根治）；② 故障期间**未复制写丢失**（异步复制）；③ 哨兵不做负载均衡。

### 3.2.4 Redis Cluster 的工作原理？

**分片模型**：**16384 个 hash slot** 分配给 N 个主节点，每个主可带从；`CRC16(key) % 16384` 定 slot → 定节点。**hash tag**：`user:{1001}:info` 与 `user:{1001}:orders` 按 `{}` 内内容同 slot——跨 key 命令（MSET/Lua/多键事务）必须同槽的官方解法。

**协议与路由**：

- 节点间 **Gossip（cluster bus）**：PING/PONG 交换 slot 表、节点状态（最终一致，规模上限 ~数百节点）；
- **MOVED**：请求打到"slot 归属明确不是你"的节点 → 返回正确节点地址，客户端**更新 slot 缓存**重发；
- **ASK**：slot 迁移中（3.2.5）→ 临时重定向，不更新缓存；
- 客户端智能（Jedis/Lettuce 维护 slot map）；另有 `CLUSTER SHARDS`/拓扑发现新版协议。

**高可用**：每 slot 主挂 → **集群内从自动选新主**（类似哨兵内嵌：投票基于 cluster 协议）；全主存活但某 slot 主从皆挂 → 该 slot 不可用（**partial down**，`cluster-require-full-coverage` 控制整体是否停服）；不支持 `SELECT` 多库与跨 slot 事务/广播命令（除 PUBLISH）——"cluster 下 keyspace 通知/事务设计要重做"是迁移大坑。

**追问：为什么 16384 个 slot？**——① 心跳包大小：slot 位图 16384bit≈2KB，65536 会让 PING 负载×4；② 节点规模上限经验 1000 内绰绰（每主分十几槽）；③ 迁移粒度足够细。（作者官方回答）

### 3.2.5 Cluster 扩缩容时 slot 怎么迁移？

**四步**（`redis-cli --cluster reschedule/rebalance` 编排）：

1. 目标节点 `CLUSTER SETSLOT <slot> IMPORTING <源node-id>`；
2. 源节点 `SETSLOT <slot> MIGRATING <目标node-id>`；
3. 循环：源节点对 slot 内 key 执行 **`GETKEYSINSLOT` + `MIGRATE key`**（原子：从源删、到目标插；迁移中的 key 读请求走 **ASK** 重定向到目标）；
4. 迁完：`SETSLOT <slot> NODE <目标>` 广播归属变更，客户端收 **MOVED** 更新缓存。

**要点**：迁移期间 slot 上仍有流量（MIGRATING/IMPORTING 各管一段）；**ASK 不更新映射**因归属未正式变更；批量大 slot 会引发延迟毛刺与 backlog 膨胀（rebalance 限流、低峰执行、先迁"冷 slot"）。

# 4. 分布式协调与原子操作

## 4.1 分布式锁

### 4.1.1 用 Redis 实现分布式锁的正确姿势？会踩哪些坑？

**最小可用四命令语义**：

```text
加锁： SET lock:<res> <requestId> NX PX 30000     ← NX（不存在才写）+ PX（有效期）必须同时，否则"加锁进程宕机锁永驻"
解锁： Lua 原子比较再删—— GET==requestId ? DEL   ← 防误删别人的锁（自己超时后别人加成功）
看门狗/重试：失败侧自旋 + 退避
```

**坑位清单（逐条都是追问）**：

1. **setnx+expire 分两步**：两步间宕机=死锁——`SET key val NX PX ms` 一条搞定；
2. **value 必须是唯一请求标识**（uuid+线程），解锁 Lua 校验——否则 TTL 内锁易主，你删了新锁；
3. **业务超过 TTL → 锁提前失效、并发进入**（最危险）：要么把 TTL 拍得足够大+业务可拆（短事务），要么**续期**（4.1.2 watchdog），要么**Fencing token**（锁带版本号，资源侧拒绝旧版本写入——Redis 天然不自带，强要求转 ZK/etcd 的 zxid/revision 语义，见 34_distributed_theory.md）；
4. **主从异步丢锁**：主加锁成功未复制即宕机、从升主锁消失（3.2.2）——单实例 Redis 锁的定位是"**效率锁**（防重复执行）"而非"正确性锁（互斥保证）"；
5. **GC/时钟停顿**：持锁线程长时间 STW，醒来以为还持有；
6. 解锁后读-改-写非原子等逻辑要业务自己兜。

**结论**：绝大多数"防并发重复"（幂等下单、定时任务互斥、库存扣减入口）用 Redis 锁 + watchdog 足够；要求"绝不双写"的场景选 ZK/etcd 临时顺序节点（CP 系统），或干脆 DB 唯一约束/乐观锁。

### 4.1.2 Redisson 分布式锁做了什么工程封装？Watchdog 原理？

**结构**：锁 key 用 **Hash**：field=客户端ID:线程ID（UUID:threadId），value=**重入计数**——同线程重复加锁 HINCRBY +1、解锁递减，为 0 才 DEL；全流程 Lua 保证原子。订阅 `channel` 等待释放通知（Pub/Sub `redis_lock` 解锁广播）——避免客户端轮询空耗。

**Watchdog（看门狗）**：`lock()` 不指定 leaseTime 时启用——加锁成功返回后，**定时任务每 `lockWatchdogTimeout/3`（默认 30s/3=10s）检查锁仍被本线程持有则 EXPIRE 续期**；线程/连接存活期间无限续，客户端宕机则不再续期，锁最多 30s 自然过期。**权衡**：leaseTime 显式给定时不启用 watchdog（锁到期硬释放，业务未完=裸奔窗口）；"watchdog + 业务超长"要配幂等兜底。

**全家桶**：可重入锁 RLock、读写锁（RReadWriteLock）、联锁（RMultiLock 全锁才通）、红锁（RRedLock，作者因 4.1.3 的争议已弃用）、信号量 RSemaphore、闭锁 RCountDownLatch、限流器 RRateLimiter（4.2.2）、公平锁（队列+发布订阅）——"分布式同步器"是 Redisson 相对裸客户端的核心价值（5.2.1）。

### 4.1.3 RedLock 是什么？为什么有争议？

**算法**（antirez）：向 **N 个完全独立**的 Redis 主（无主从关系）依次加锁；全部耗时 < 请求锁 TTL 且**多数（N/2+1）成功** → 持锁有效时间 = 最小 TTL - 已耗时；失败则向全部节点解锁。

**争议（Kleppmann 2016 "How to do distributed locking"）**：

1. **依赖时钟假设**：节点 GC 暂停/时钟回拨/NTP 步进导致"你以为还有 20s，实际锁已过期"——多数派也无法弥补；
2. 主从/哨兵环境不适用（它要求独立节点，部署代价大）；
3. 锁的作用定位混淆：**正确性互斥**（不能同时有两个写者）不能靠 Redis 这类非共识时钟系统保证，应该用 fencing token（ZK/etcd 提供）；**效率**（少做无用功）才轮到 RedLock。

**实践结论**：单集群 Redis 锁 + watchdog + 幂等设计覆盖 95% 业务；对锁语义苛刻（金融账务）改 ZK/etcd/数据库——"要不要用 RedLock"的面试答到争议层面即满分。

## 4.2 原子操作、计数与消息

### 4.2.1 Redis 事务、Pipeline、Lua 脚本有什么区别？

| 维度 | 事务 MULTI/EXEC | Pipeline | Lua |
|------|----------------|----------|-----|
| 目的 | 一组命令**连续执行不被打断** | 减少 RTT（批量发、批量收） | 读写判断的**复合原子逻辑** |
| 原子性 | EXEC 串行原子；但**非 all-or-nothing** | 无（服务端仍逐条独立执行） | 整段脚本单线程原子执行 |
| 回滚 | **不支持**：语法错误整队拒执（EXEC 前发现）、运行时错误其余命令照常执行 | 谈不上 | 运行错误脚本终止，已执行部分**不回滚** |
| 读己之写/条件逻辑 | 不支持 | 不支持 | **支持**：先 GET 判断再 SET |
| 典型场景 | WATCH+CAS（乐观锁） | 批量 MSET 无依赖命令 | 解锁 Lua（4.1.1）、限流令牌桶、秒杀扣减（见 16_mysql.md 3.2.4 同族思想） |

**三个"原子"要分清**：事务的原子=EXEC 后不打断；Pipeline 的"原子"=不存在；Lua 的原子=执行期独占。Redis 事务没有回滚 = "原子事务"名不副实——答出这点才见深度；EVAL 内死循环=整库卡死（`lua-time-limit` 只到 5s 后开始拒绝，SCRIPT KILL 救不了写操作后的脚本）。

### 4.2.2 Redis 如何做计数器、限流器和分布式数据结构？

- **计数器**：`INCR/INCRBY key` 单命令原子；按窗口 key（`api:{uid}:{minute}`）+ EXPIRE 实现简单限流——**注意 INCR 后设 TTL 的非原子窗口**（宕机留下无 TTL key 永久计数）：用 Lua 打包——INCR 返回值==1 时才 EXPIRE，两条命令一次原子执行；
- **限流器**：Redisson **RRateLimiter**（令牌桶：Lua 计算补令牌+扣减，支持 rate/interval）；云原生 Sentinel 集群限流同样以 Redis/Tair 为配额中心（见 20_sentinel.md）；滑动窗口 ZSet（score=时间戳，ZREMRANGEBYSCORE 清窗 + ZCARD 计数）；
- **分布式数据结构**（Redisson 的卖点）：RMap/RSet/RQueue（共享缓存集合）、RAtomicLong/RBitSet、RSemaphore（信号量：许可数即任务并发上限，适合分布式"最多同时 N 个"）、RLock；
- **UV/统计**：HyperLogLog（1.2.1）+ PFMERGE 按天/渠道合并；Bitmap 签到 + BITCOUNT 日活。

### 4.2.3 Stream 做消息队列和 List 有什么区别？

| 特性 | List（LPUSH/BRPOP） | Stream |
|------|--------------------|--------|
| 消费语义 | 取出即删，**崩溃即丢** | 消息常驻（MAXLEN 裁剪），**ACK 制** |
| 消费组 | 无 | **XGROUP/XREADGROUP**：组内分摊、组间广播 |
| 失败处理 | 自实现（取出暂存→处理完删） | **PEL 待确认列表 + XACK + XCLAIM 接管**（未 ACK 超时由同组其他消费者领取重试——RocketMQ 重试队列同款思路，见 13_rocketmq.md） |
| 回溯/历史 | 不能 | 按 **消息 ID（毫秒seq）** 范围重读（XREADGROUP from、XRANGE） |
| 顺序 | 单 key 有序 | 单 key 内 ID 单调 |
| 适用 | 简单任务池、可容忍丢失 | 轻量可靠消息；**重量级场景仍用专业 MQ**（无消费端限流/死信/事务消息） |

**Stream 的裁剪**：`XADD MAXLEN ~100000`（近似裁剪省内存）、`XDEL`+`XTRIM`；消费组无 ACK 策略需 `XPENDING` 巡检告警。

### 4.2.4 如何用 Redis 实现延迟队列？

**主流方案：ZSet + 轮询**：

```java
zadd delay_queue <执行时间戳> <taskPayload>
while(true) {
    List<String> due = zrangebyscore(key, 0, now, LIMIT 0 100);  // Lua 打包"取出+删除"防并发重复消费
    for (String t : due) handle(t);
    sleep(50~500ms);
}
```

要点：**ZRANGEBYSCORE + ZREM 用 Lua/事务合并**防多实例重复弹出（或弹出前 ZPOPMIN 式原子）；score 精度=轮询精度（50ms 级）；积压保护：单次 LIMIT 防 OOM、失败任务回写 score 退避重试。

**Stream 方案**：投递即 XADD，延迟消费端按"score 列不原生支持"——需要延迟投递时仍回 ZSet 当"定时器表"，到期再转 Stream 分发（两层结构：延迟索引 + 可靠分发）。

**过期通知（KeySpace Notification）**：订阅 `__keyevent@0__:expired` 拿"key 到期事件"当定时器——**不可靠**：惰性/定期删除导致事件延迟与不保证送达（3.x 起事件不持久化、重启丢），生产只配玩具场景；**严肃需求直接用 MQ 延时消息/RocketMQ 定时（13_rocketmq.md）或时间轮调度**。

# 5. 运维与开发实践

## 5.1 诊断与运维

### 5.1.1 慢查询和延迟毛刺如何排查？

**慢日志配置**：`slowlog-log-slower-than 10000`（10ms 微秒计）、`slowlog-max-len 128`（内存环形）；`SLOWLOG GET/LEN/RESET`——记录**命令执行耗时**，不含排队等待（客户端看到慢但服务端不慢=排队或网络）。

**原因树**：

1. **命令本身重**：O(N) 大集合（KEYS/HGETALL/SMEMBERS）→ SCAN/HSCAN 分批、UNLINK 异步删（2.2.3）；
2. **阻塞在别人的慢命令**：单线程排队效应——查同窗口其他慢记录；
3. **持久化抖动**：fork 页表拷贝、AOF rewrite、`bgsave` 周期（错峰、`save ""` 禁自动触发）；
4. **内存**：swap、大 Key 淘汰风暴、rehash 风暴（1.2.4）；
5. **外部**：THP 透明大页（关）、CPU 绑核跑满、网络、宿主机超卖；
6. **客户端**：连接池小/跨机房 RTT/DNS——`latency:server` 侧指标对比（`LATENCY DOCTOR`/`SLOWLOG` 交叉看）。

**工具链**：`INFO`（keyspace/stats/replication）、`MONITOR`（**临时**：全命令流打印自身就拖慢，仅调试用）、云控制台/RedisExporter→Prometheus 曲线（20_sentinel.md 同生态）。

### 5.1.2 线上如何安全地遍历 key、分析内存？

**永远不用 KEYS**：O(N) 遍历阻塞事件循环，生产应 rename-command 禁用（云实例默认封 KEYS/FLUSHALL/CONFIG）。**SCAN 家族**：`SCAN / HSCAN / SSCAN / ZSCAN` 游标分批（COUNT 控制每批），**遍历期间新增 key 不保证被见到、删除的不会重访**（弱一致快照语义要答出）；集群模式逐节点游标或 `--cluster scan`。

**内存分析**：`redis-cli --bigkeys`（按类型统计最大元素，采样非全量）、`--memkeys`（按内存字节加权）、`MEMORY USAGE key`（精确估算）、`MEMORY DOCTOR` 建议、`OBJECT ENCODING` 看是否退化大编码；`INFO memory` 的 `used_memory_dataset/vshrink` 与碎片率 `mem_fragmentation_ratio`（>1.5 考虑重启/`activedefrag`）。

**危险命令治理**：rename-command 封 KEYS/FLUSHALL/CONFIG；`acl`（6.0+）按用户限命令——与 33_cicd.md 的"变更门禁"同属"线上不留手滑通道"哲学。

## 5.2 客户端与 Spring 集成

### 5.2.1 Jedis、Lettuce、Redisson 怎么选？

| 维度 | Jedis | Lettuce | Redisson |
|------|-------|---------|----------|
| 模型 | 直连 socket，**Jedis 实例非线程安全** | Netty NIO 异步，**单连接线程安全** | 独立 API（自研 Netty 客户端） |
| 连接池 | 必须（commons-pool2） | 共享连接+池可选 | 连接池内置 |
| 集群/哨兵 | 支持 | 支持（自动拓扑刷新） | 支持 |
| 功能定位 | 命令级轻量 | **Boot 默认客户端**（spring-data 首选） | **分布式对象/锁/限流全家桶**（4.1.2/4.2.2） |
| 版本演进 | 5.x 起线程安全改造 | Reactive 原生 | 需独立学习成本 |

**选型**：纯缓存用 Boot 默认 Lettuce；要锁/信号量/限流器/布隆——**叠一层 Redisson**（可共存，或干脆 RClient 统一）；团队存量 Jedis 不必强迁。

### 5.2.2 Spring 使用 Redis 有哪些高频坑？

1. **序列化**：JdkSerializationRedisTemplate 默认存字节码——**换实体字段/改 serialVersionUID 即读炸**、体积大不可读；统一 `GenericJackson2JsonRedisSerializer`（StringRedisSerializer 管 key）——**注意多态类型 `@class` 的类迁移兼容**（改包名=缓存全失效，TTL 兜底）；
2. **RedisTemplate 与 StringRedisTemplate 混用**：key 序列化不一致导致"程序写入、控制台查不到"；
3. **@Cacheable 三坑**（见 7_spring.md 4.3.2 同族自调用失效）：同类调用不过代理；`key` 表达式写错导致大 key/雪崩；null 不缓存=穿透（`cache-null-values`）；
4. **超时与重试**：`timeout` 缺省无限等待拖崩业务线程池；失败降级直查库要显式设计（2.1.3 雪崩防线）；
5. **Lua/批量与集群**：`EVAL` 跨 slot 报 CROSSSLOT（hash tag 治理）；`Pipeline` 结果按顺序对齐但**中途网络错误可能部分执行**；
6. **连接池参数**：maxTotal/maxWait 拍脑袋——压测定容量，`lettuce pool` 借还校验别每命令 ping。