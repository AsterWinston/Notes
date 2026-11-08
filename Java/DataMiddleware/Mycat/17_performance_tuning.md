# 1. 调优全景图

**Mycat 的性能上限由 SQL 路由质量决定，下限由中间件自身资源决定**——绝大多数"Mycat 慢"的案例，调参数之前先数 EXPLAIN 里的分片数：全分片扫描的查询配再好的 JVM 也救不回来。因此本文的调优顺序是自上而下：先确认路由形态（第 5 节），再谈线程、连接池、内存（第 2-4、6 节），最后用压测验证（第 7 节）。

Mycat2 性能调优维度：

```text
Mycat2 性能调优维度：
┌─────────────┬──────────────┬──────────────┬─────────────┐
│  JVM 层      │   Reactor    │   连接池      │  SQL 路由    │
├─────────────┼──────────────┼──────────────┼─────────────┤
│ 堆内存 / GC  │ 线程数        │ 前端/后端连接  │ 分片命中     │
│ 堆外内存     │ Buffer 大小  │ 超时配置      │ 避免全扫描   │
│ 启动参数     │ 序列化框架    │ 连接复用      │ 结果合并     │
└─────────────┴──────────────┴──────────────┴─────────────┘
```

**Mycat 在链路中是"额外一跳"**，它的开销模型是：每条 SQL 付出解析/路由/改写的 CPU、协议编解码的 CPU、结果归并的内存与网络中转。精确分片命中时这一跳约 0.5-2ms；跨片归并时按分片数与结果量放大。调优的目标不是把这跳压到零（做不到），而是让它**稳定、可预期、不成为木桶最短那块**。

# 2. JVM 调优

```bash
# Mycat2 启动 JVM 参数
java -server -Xms2g -Xmx4g \
     -Xmn1g \
     -XX:+UseG1GC \
     -XX:MaxGCPauseMillis=200 \
     -XX:InitiatingHeapOccupancyPercent=45 \
     -XX:+DisableExplicitGC \
     -XX:MaxDirectMemorySize=1g \
     -Dio.netty.allocator.type=pooled \
     -jar mycat2.jar
```

| 参数 | 说明 | 推荐值 |
|------|------|--------|
| `-Xms/-Xmx` | 堆内存 | 2G/4G |
| `-Xmn` | 年轻代 | 堆的 1/4 |
| `-XX:MaxDirectMemorySize` | 堆外内存（Netty/Vert.x 使用） | 1G |
| `-Dio.netty.allocator.type` | Netty 内存分配器 | pooled |

**Mycat 的内存分布与普通 Java 应用不同，调参要按用途切**：堆内存主要被归并结构（排序缓冲、GROUP BY 哈希表、语句/计划缓存）占用，与**并发大查询数**成正比，而不是与 QPS 成正比；堆外（direct）内存被 Netty 的读写缓冲占用，与**在途连接数 × 单连接缓冲页**成正比。`-XX:+DisableExplicitGC` 要谨慎评估——它屏蔽 `System.gc()`，而 Netty 老版本依赖显式 GC 回收 DirectByteBuffer，堆外吃紧时可能反而引发 OOM 而非 Full GC 兜底；保守做法是 `-XX:+ExplicitGCInvokesConcurrent`（显式 GC 转并发执行）而不是全禁。

> **注意**：Mycat 对 GC 停顿的敏感度高于普通业务应用——GC 期间 Reactor 线程全停，所有在途请求同时超时，表现为"整点尖刺一起抖"。这也是 1.6.x 时代就推荐 G1/ZGC、堆不要给太大（4G-8G 而非 16G+）的原因：**堆越大，一次停顿的绝对时间越长**。容量不足优先加实例横向扩，不是加单实例堆。

# 3. Reactor 线程调优

Mycat 1.6.x 的线程模型是"processor（NIO 读写）+ backend NIO reactor + business executor"三组线程，2.x 迁移到 Netty 事件循环后概念简化为 Reactor 线程组。参数含义都是同一件事：**协议处理线程的数量与职责边界**。

```yaml
# mycat.yml
server:
  reactorNumber: 8          # Reactor 线程数
  bufferPoolPageSize: 4194304    # 4MB 页大小
  bufferPoolChunkSize: 4096      # 4KB 块大小
  processorBufferPoolType: "direct"  # direct / heap
```

**Reactor 线程数建议**：

| CPU 核心数 | 推荐线程数 | 说明 |
|-----------|-----------|------|
| 4 | 4 | — |
| 8 | 8-16 | I/O 密集型可多配 |
| 16 | 16-32 | — |
| 32 | 32-48 | — |

```java
// 原则：Reactor 线程数 = CPU 核心数 × 2（I/O 密集型）
// 过多：上下文切换开销 > 并行收益
// 过少：无法充分利用多核
```

**×2 只是起点，判定看线程忙碌度**：`showReactors` 输出的各线程负载（processor/executor 队列积压）才是调参依据——若存在明显积压且 CPU 未饱和，加线程；若 CPU 已 90%+，加线程只会恶化，应该加实例。一个容易踩的反模式是把 SQL 改写、结果归并这类**CPU 密集操作放到 Reactor 线程里**：单个慢归并把整条 event loop 上几百个连接的读写全拖住，表现为"无辜请求一起超时"。1.6.x 专门有 business executor 线程池隔离这类工作，2.x 同理——**任何超出"收包-查路由-转发"的操作都不该占着 event loop**。

# 4. 连接池调优

```yaml
server:
  maxCon: 10000              # 最大前端连接数

# 数据源连接池
datasources:
  ds1:
    minCon: 5                # 最小空闲连接
    maxCon: 100              # 最大连接
    maxRetryCount: 3         # 获取连接重试
    idleTimeout: 60000       # 空闲超时(ms)
    connectionTimeout: 3000  # 获取连接超时(ms)
```

**连接数计算公式**：

```text
后端连接数 ≈ (前端 QPS × 平均 RT / 1000) / 每个分片

示例：
  前端 QPS = 5000
  平均 RT = 10ms
  分片数 = 4

  每分片连接数 = (5000 × 0.01) / 4 ≈ 12.5 → 配 20
  总后端连接数 = 20 × 4 = 80
```

公式出自 Little's Law（在途请求数 = QPS × RT），但要打上三个中间件特有的修正项：**其一，跨片查询的连接放大**——一条全分片 SQL 同时借出 N 个后端连接，若此类查询占比 p，需求近似乘 (1-p+p×N)，16 分片 20% 全扫意味着连接需求翻倍不止；**其二，多 Mycat 实例叠乘**——每台都按自己的 maxCon 建池，后端 MySQL 的 max_connections 要 ≥ 实例数 × Σ各数据源 maxCon × 1.2；**其三，minCon 的冷热语义**——空闲缩到 minCon 后再遇流量高峰要付建连+认证成本，跨机房建连几十 ms，低峰频繁缩容会让早高峰抖一波，minCon 建议覆盖日常 QPS 基线而不是 5。

> **行业实践**：`connectionTimeout`（从池取连接的等待超时）**必须显式设置且不宜过长**（1-3s）。不设时连接池满会无限排队，前端请求全部 hang 死、故障向上传染成应用线程池雪崩；设了超时，故障表现变成"快速报错"，应用侧可降级/熔断。这是"MySQL 慢导致 Mycat 雪崩"与"MySQL 慢但系统可预期劣化"的分水岭。

# 5. SQL 路由优化

**路由质量是 Mycat 性能的第一决定项**，其余参数都在这个前提成立之后才有意义。

```sql
-- ✓ 最优：精确分片查询（只访问 1 个分片）
SELECT * FROM orders WHERE id = 1;

-- △ 可接受：IN 查询（访问已知的 N 个分片）
SELECT * FROM orders WHERE id IN (1, 5, 9);

-- ✗ 差：全分片扫描（访问所有分片）
SELECT * FROM orders WHERE user_id = 100;  -- user_id 不是分片键
SELECT * FROM orders ORDER BY create_time;  -- 无分片键
```

**分片命中率监控**：

```sql
/*!mycat:showSQLStat*/
-- +------------------+--------+----------+
-- | sql              | count  | avgShards|
-- +------------------+--------+----------+
-- | SELECT ... id=1  | 10000  | 1.0      | ← 精确路由
-- | SELECT ... uid=? | 5000   | 4.0      | ← 全分片扫描
-- +------------------+--------+----------+
```

优化路径按性价比排序：**给高频非分片键查询建映射表**（以 user_id 查 orders：维护 user_id→order_id 的索引表/走 ES，先查键再精确路由）；**把 JOIN 改造成 ER 表或全局表**（机制见第 11、12 篇）；**给报表类流量改道**（从机直连/数仓，不占在线集群的分片扫描预算）。`avgShards` 这个指标建议做成看板趋势——分片命中率下降通常意味着新上线的 SQL 没按规范写，它是**变更引入劣化**的早期信号，比容量告警来得更早。

# 6. Buffer Pool 调优

```yaml
server:
  bufferPool:
    type: direct
    pageSize: 4194304      # 4MB 页大小
    chunkSize: 4096        # 4KB 块大小
    maxPages: 256          # 最大页数（256 × 4MB = 1GB）
```

**Buffer Pool 用途**：Mycat2 内部数据缓冲（SQL 解析、结果集临时存储、协议编解码）。

**建议**：`pageSize × maxPages` ≈ `-XX:MaxDirectMemorySize`

**Buffer Pool 满了的表现是"分配阻塞"而不是 OOM**——新连接/新结果拿不到页就等待或报错，此时前端吞吐骤降但 CPU 未必高，容易误判为数据库问题。容量估算按"峰值在途字节数"：并发连接数 × (前端读缓冲 + 后端借出连接数 × 结果缓冲)。chunkSize 影响碎片率：结果包普遍 1-16KB 时 4KB chunk 合理，若业务有大批量结果中转（大 IN、大 INSERT），适当调大 chunkSize 或 pageSize 能减少页申请次数。**1.6.x 的 bufferPool 调优经验基本适用于 2.x**，参数名从 server.xml 挪到了 mycat.yml。

# 7. 压测方法

**压测前先明确测什么**：测 Mycat 转发极限（后端用 mock 或轻量查询）与测真实业务容量（SQL 形态、数据分布还原生产）是两件事，混在一起得出的数字没有指导意义。标准动作：sysbench/jmeter 按 SQL 模板配比（如 80% 精确路由点查 + 15% IN + 5% 聚合）阶梯加压（20%、50%、80%、100%、120% 预期峰值），每档稳定 10 分钟，采集 QPS/RT 分位/错误率/Mycat CPU/GC/后端连接池水位五条曲线，**拐点即容量线**（RT P99 陡升或错误率非零的第一档）。

```text
测试环境：Mycat2 4C8G，后端 4 个 MySQL 5.7

| 场景                      | QPS    | AVG RT | P99 RT |
|--------------------------|--------|--------|--------|
| 精确分片 (WHERE id=?)    | 25000  | 3ms    | 8ms    |
| IN 查询（3 个 ID）        | 12000  | 8ms    | 20ms   |
| 全分片 COUNT(*)          | 5000   | 20ms   | 50ms   |
| 全分片 ORDER BY LIMIT    | 3000   | 35ms   | 80ms   |
| 读写分离 SELECT          | 20000  | 4ms    | 10ms   |
| XA 事务（2 分片）         | 3000   | 30ms   | 60ms   |
```

> **注意**：这组数字只能作为**量级参照**（精确路由与全分片扫描差一个数量级、XA 相对本地事务的开销约 2-3 倍），不能当 SLA 承诺引用。压测结论要注明：Mycat 版本、JVM 参数、分片数、SQL 配比、后端实例规格五项，缺一项数字就无法复现。

# 8. 慢 SQL 日志与可观测性

调优的前提是定位，Mycat 提供三层观测：**语句统计**（showSQLStat，按模板聚合次数/RT/分片数）、**慢 SQL 记录**（阈值触发写入慢日志，含改写后的下发语句）、**全链路耗时拆分**（前端接收→解析路由→后端执行→归并回写各段计时）。排查"某接口变慢"的标准动线：先在 MySQL 侧确认单片执行时间没有恶化 → 再看 Mycat 慢日志里该模板的分片数是否上升（缓存失效/数据倾斜导致路由变宽）→ 最后看归并段耗时（结果量增大）。**慢 SQL 阈值设多少**：在线交易系统常用 100-500ms，设得太高漏掉劣化早期，太低日志淹没；配合语句表统计的 Top N 报表看趋势比盯单条更有用。

# 9. 常见性能问题

| 问题 | 原因 | 优化 |
|------|------|------|
| 全分片扫描过多 | WHERE 缺少分片键 | 优化 SQL，添加分片键 |
| 后端连接池满 | minCon/maxCon 不足 | 按公式计算增大 |
| 排序/分页慢 | 深度分页 + 跨分片排序 | 使用游标分页替代 OFFSET |
| GC 频繁 | 堆内存不足 | 增大 -Xmx，使用 G1GC |
| Reactor 线程满 | 线程数不足 | 增大 reactorNumber |

这张表的五行动作背后是同一句话：**中间件层的性能问题，六成靠 SQL 规范、三成靠容量规划、一成靠调参**。把"必带分片键、必带 LIMIT、禁跨片大 JOIN、报表走旁路"落进 CR 与防火墙（第 16 篇），参数表里的数字就很少需要动。

# 10. 快速调优清单

```text
□ JVM: -Xms2g -Xmx4g -XX:+UseG1GC -XX:MaxDirectMemorySize=1g
□ Reactor: 线程数 = CPU 核数 × 2
□ 连接池: 后端连接数 = 前端 QPS × RT / 1000 / 分片数
□ Buffer: pageSize × maxPages ≈ MaxDirectMemorySize
□ SQL: 确保 WHERE 包含分片键
□ 监控: Prometheus + Grafana 实时观察
□ 日志: 生产环境设置 WARN 级别
```

# 11. 小结

| 层面 | 关键参数/手段 | 一句话原则 |
|------|--------------|-----------|
| SQL 路由 | 分片键进 WHERE、ER 表、映射表 | 路由质量决定性能上限，先看 EXPLAIN 分片数 |
| 连接池 | maxCon 公式 × 跨片放大系数；connectionTimeout | 后端连接是稀缺资源，快速失败优于无限排队 |
| 线程 | reactorNumber、event loop 隔离重活 | 线程忙看队列积压说话，别让归并卡住收包 |
| 内存 | 堆(归并)与 direct(中转)分账、G1 | Mycat 怕停顿不怕小堆，容量不够加实例 |
| 观测 | SQLStat 分片数、慢 SQL 阈值、五曲线压测 | 数字必须带版本与配比语境才可复现 |

下一篇（18 Mycat1→2 迁移）把 1.6.x 的这些经验映射到 2.x 的参数体系上。
