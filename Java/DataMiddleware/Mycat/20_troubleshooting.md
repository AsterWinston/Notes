# 1. 排错总纲：先定位层次，再找原因

**Mycat 排错的第一功课是"故障发生在哪一跳"**：应用 → Mycat(8066) → 后端 MySQL(3306) 是一条三段链路，症状高度相似（超时、报错、结果不对），根因却在不同层。分诊顺序固定为四问：① 连不上还是连上后慢/错？② 是所有 SQL 还是个别 SQL？③ 是全部实例还是个别实例？④ 故障时间点前有没有变更（配置/发布/数据）？

```text
┌──────────┐   8066   ┌──────────┐  ds pool  ┌────────────┐
│ App/JDBC │ ───────→ │  Mycat   │ ────────→ │ MySQL 分片  │
└──────────┘          └──────────┘           └────────────┘
  ①驱动/池参数        ②认证/路由/归并        ③连接数/慢SQL/锁
```

> **注意**：三段链路排错有个天然陷阱——**Mycat 会把后端错误原样（或包装后）抛给客户端**，"Table doesn't exist" 可能真是表没了，也可能是路由到了错误的空分片；"Access denied" 可能是应用密码错，也可能是 datasources 里后端账号密码错。拿到任何报错，先问一句"**这个错是谁产生的**"：看错误码前缀与消息模板（Mycat 自产错误常带 mycat 字样或 ER_UNKNOWN 包装），拿不准就直接对同一 SQL 做二分——后端 MySQL 直连执行一遍，结果不同则问题在中间件层。

# 2. 启动失败

```bash
# 1. 检查日志
tail -f logs/wrapper.log

# 常见启动错误：
# - Address already in use → 端口被占用
netstat -an | grep 8066
lsof -i :8066

# - Config file parse error → JSON 格式错误
python3 -m json.tool conf/schemas.json
python3 -m json.tool conf/datasources.json

# - MySQL connection refused → 后端数据库不通
mysql -h 192.168.1.10 -u mycat -pmycat123 -e "SELECT 1"

# - OutOfMemoryError → 内存不足
java -Xms2g -Xmx4g -jar mycat2.jar
```

启动期还有一类"**半死不活**"形态最容易骗人：进程起来了、端口在听、但 schema 初始化失败导致某些逻辑表不可用——日志里翻 `ERROR` 而不是只看 `Started` 字样，再用 `showSchemas/showDataSources` 管理命令核对加载完整性。1.6.x 的配置错误是 XML 格式/引用错（rule 名不存在、dataHost 名对不上），2.x 是 JSON 语法/字段错（第 15 篇的 ConfigValidator 拒绝整个 schema 加载）；两边都有共同教训：**启动失败信息经常只打到 stdout 的第一行**，wrapper 日志与 console 输出都要看。

# 3. 连接 Mycat2 失败（8066 连不上）

```bash
# 1. 检查 Mycat2 是否启动
ps aux | grep mycat2

# 2. 检查端口是否监听
netstat -tlnp | grep 8066

# 3. 测试连接
mysql -h 127.0.0.1 -P 8066 -u root -p123456

# 4. 检查用户配置
cat conf/users.json

# 5. 检查 IP 白名单
# users.json 中的 ipWhiteList
```

**把"连不上"拆成五个具体症状，每个都有确定答案**：

| 症状 | 最可能原因 | 验证动作 |
|------|-----------|---------|
| 拒绝连接（Connection refused） | 进程未起/端口未听/防火墙 | telnet 8066、firewalld/安全组 |
| 能 TCP 连上但握手失败/无欢迎包 | 白名单拒绝、管理口当业务口连 | 9066 对比测试、看 audit/日志连接拒绝记录 |
| 认证失败（Access denied for user 'x'） | 密码错/用户不存在 | users.json 核对；**注意这是 Mycat 层认证错，不是后端** |
| Unknown database | 账号 schemas 里没该库/库名大小写 | 用该账号 `show databases` 对比 |
| 偶发连不上、池里有死连接 | VIP 漂移、Mycat 重启、LB 摘除 | 对照 LB 事件与实例重启记录 |

白名单拒绝的表现很有迷惑性：TCP 握手成功（内核完成）、MySQL 协议层直接被 Mycat 关闭，客户端常报 `Connection closed`/`Packet received out of order` 而不是清晰的权限错误——排"应用能连 DBA 不能连"这类差异问题时，**对比两边的源 IP 是否在 ipWhiteList 里**是第一步（容器/代理改写源 IP 的情形见第 16 篇第 2 节）。

# 4. 9066 管理端口连不上

管理口故障通常不是"坏了"而是"不让连"：9066 默认绑定与账号体系独立于 8066，`managerUser/managerPassword`（1.6.x 在 server.xml，2.x 在 users/管理配置）错则直接 Access denied；生产环境按第 16 篇纪律不对应用网段开放，运维跳板来源 IP 变化就会被白名单/安全组拦下。验证链：本机 `mysql -h 127.0.0.1 -P 9066` → 跳板机 telnet → 安全组/网络策略。**9066 通不了时所有 `/*!mycat:* */` 诊断命令全部失效，这也是为什么日常巡检脚本要监控管理口可达性**——真出事时连不上管理口的代价是排错时间翻倍。

# 5. 查询返回空结果 / 路由不对

```bash
# 1. 检查路由是否正确
EXPLAIN SELECT * FROM orders WHERE id = 1;
# → 确认路由到正确的分片

# 2. 直接连接后端 MySQL 查询
mysql -h 192.168.1.10 -u mycat -p -e "SELECT * FROM order_db.orders WHERE id = 1"

# 3. 检查数据是否存在
# Mycat2 是透明代理，数据在后端 MySQL 中

# 4. 检查 Schema 配置
/*!mycat:showSchemas*/
```

**空结果在三类场景各有成因**：① **路由错位**——查询去了没有该数据的分片，典型诱因是分片键类型不匹配（varchar 分片键传了数字、或反之导致路由函数取到不同哈希）、以及 1.x→2.x 迁移后同名算法实现差异（第 18 篇对拍没做严）；② **元数据滞后**——后端刚建的表/刚加的列，MyCAT 未 reload 不认识（第 15 篇边界表最后一行），表现从"报错"到"空"都可能；③ **读写分离读从库**——数据写主库后从库延迟/复制中断（SQL 线程报错停住），从片上确实没有，delay 见 `showHeartbeatStatus`。二分法在这里同样锋利：拿 EXPLAIN 里的目标分片，直连该分片执行原 SQL——有数据则是路由问题，没数据则回第 2 步检查写入去哪了（写入同样 EXPLAIN）。

路由错误看 explain 的双版本差异要记住：**1.6.x `explain` 返回改写后的下发 SQL 文本**（肉眼看 WHERE/分片是否合理），**2.x EXPLAIN 返回计划树**（看 TableScan 的目标节点集合与下推形态）；1.6.x 还有 `explain @@select_type` 等扩展用法，脚本迁移时注意。

# 6. 后端心跳断开 / 连接池异常

```sql
-- 管理口三连看
/*!mycat:showHeartbeatStatus*/   -- 各节点 ONLINE/OFFLINE、delay
/*!mycat:showBackends*/          -- 各数据源连接池水位、活跃连接
/*!mycat:showSQLStat*/           -- 谁在占用连接（慢模板 Top）
```

**"心跳断开"的根因八成不在心跳本身**：连续 maxRetry 次探测失败才会摘除（第 13 篇），所以摘除意味着后端实例级问题（宕机、磁盘满、连接数打爆 max_connections 导致新探测连接建不起来、网络分区）或探测参数过紧（跨机房 RTT > heartbeatTimeout）。反过来，**心跳正常但业务仍报"连接获取超时"**，则是池容量问题：池满 + connectionTimeout 排队，或某大查询长时间占用（长事务、未 flush 的大结果）。日志特征区分两类：前者伴随 `heartbeat fail` WARN 与状态翻转记录；后者状态全程 ONLINE，瓶颈在池水位指标。处置：前者修后端/放宽超时，后者按第 17 篇公式扩池 + 找占用大户（showSQLStat 按平均 RT×并发估占连接时间）。

> **行业实践**：把"MySQL 主库 max_connections 与 Mycat 后端 maxCon 之和"做成容量看板——**Mycat 扩池、多实例叠加时把后端连接配额顶爆是"心跳断开"最经典的连锁反应**（第 17 篇 Little's Law 修正项之一）：3 台 Mycat × 200 maxCon = 600 后端连接，加上运维直连与复制线程，MySQL max_connections=600 直接撞墙，新连接（含心跳）被拒。任何一侧调池都要同步另一侧的上限。

# 7. 分片数据倾斜

```sql
-- 1. 检查各分片数据量
-- 连接每个后端 MySQL
SELECT COUNT(*) FROM orders;  -- 比较各分片行数

-- 2. 检查分片算法
/*!mycat:showSchemas*/
-- 确认分片算法的 clazz 和 count 配置

-- 3. 分析原因
-- 取模算法 → 一般均匀，不均是数据分布问题
-- 范围算法 → 某范围数据过多
-- 枚举算法 → 某枚举值数据过多
```

**解决**：

```json
// 调整分片算法
{
  "clazz": "io.mycat.router.function.PartitionByMurmurHash",
  "properties": {
    "columnName": "id",
    "count": 8,
    "seed": 0,
    "virtualBucketTimes": 160    // 增加虚拟桶提高均匀度
  }
}
```

注意这段配置的适用边界：**"改算法修倾斜"在存量系统里几乎不可直接用**——换 clazz/count 意味着全部路由重算，等价于一次全量 rehash 扩容（第 15 篇的争议条目）。它的合理使用时机是**迁移/扩容的窗口里顺带修正**，或者逻辑分片体系下把热点逻辑分片整体搬去大实例（count 不动）。纯取模出现倾斜的更常见根源在 ID 生成侧（第 19 篇：趋势递增 ID + 取模 = 热点）或某大客户数据独大（业务键天然不均），前者修 ID 生成、后者走租户单独分片，都不是调分片函数参数能解决的。

# 8. 读写分离不一致

```text
问题：写后立即读取不到最新数据

排查：
  1. 检查 Slave 复制延迟
     SHOW SLAVE STATUS; → Seconds_Behind_Master

  2. 检查代码是否强制走 Master
     /*!mycat:db_type=master*/ SELECT ... WHERE id = 1;

  3. 关键读操作使用 Hint 强制 Master
```

排查顺序按**概率从高到低**：① 复制延迟（大促/批量任务时段 Seconds_Behind_Master 飙升，第 13 篇 slaveThreshold 已自动踢延迟从库，此时读应已回主——若没回，检查阈值配置与从库状态上报）；② **复制中断**（SQL 线程报错停止，Seconds_Behind_Master 显示 NULL 而心跳仍 ONLINE——`SHOW SLAVE STATUS` 的 `Last_SQL_Error` 才是真相，主从表结构漂移、主键冲突都可能让线程停掉）；③ 事务内读从库（正常应在同一连接走主库；若应用把一个写事务拆成多连接则超出中间件保证范围）；④ Hint 位置错误（`/*!mycat:db_type=master*/` 必须紧跟 SELECT 且在语句正确位置，写错前缀不报错、静默失效）。

> **注意**：验证"不一致"是不是延迟造成的，用**同一时刻直连主/从各查一遍**对比即可下结论，不要在应用层猜。业务侧的长效方案见第 19 篇：写后读窗口内强制主库、或应用显式双数据源路由。

# 9. OOM（内存溢出）

```bash
# 1. 查看 GC 日志
jstat -gcutil $(pgrep -f mycat2) 1000

# 2. 生成 heap dump
jmap -dump:live,format=b,file=heap.hprof $(pgrep -f mycat2)

# 3. 常见 OOM 原因
# - 前端连接数过多 → 增大 maxCon 或限制
# - 大结果集未流式处理 → 检查 SQL 是否少了 LIMIT
# - 堆外内存不足 → 增大 MaxDirectMemorySize

# 4. 临时缓解
/*!mycat:reloadConfig*/
```

heap dump 里 Mycat 的 OOM 形态很好认：**堆内**被归并结构占满时，支配树顶端通常是排序/分组缓冲与大结果集 RowBatch——对应第 12 篇的危险 SQL 形态（`DISTINCT+ORDER BY`、高基数 GROUP BY、无 LIMIT 大表扫描），处置是把 SQL 揪出来（慢日志/审计按 rows 与分片数排 Top）；**堆外** direct OOM（`OutOfMemoryError: Direct buffer memory`）则与 Buffer Pool 配额和连接数挂钩（第 17 篇第 6 节，`pageSize × maxPages` 对齐 MaxDirectMemorySize）。第 3 步注释里"前端连接数过多 → 增大 maxCon"这句要读对方向：**连接数逼近上限时应当限客户端（用户级配额/池上限），同时核对内存是否按连接数规划过**，盲目增大 maxCon 只会把 OOM 提前。

# 10. SQL 执行慢

```bash
# 1. 查看慢 SQL
/*!mycat:showSlowSQL*/

# 2. 查看执行计划
EXPLAIN SELECT ... FROM orders WHERE ...;
# 关注是否全分片扫描（node 列是否有多个分片）

# 3. 直接在后端 MySQL 执行
mysql -h backend -e "SELECT ..."
# 判断慢在 Mycat2 层还是 MySQL 层

# 4. 检查分片命中
# WHERE 条件缺少分片键 → 全分片扫描
```

三段计时把"慢"归到唯一层：单片直连耗时 ≈ Mycat 端到端耗时 → 慢在 MySQL（索引/锁/数据量）；单片快、Mycat 慢 → 慢在中间件层，按 **分片数（路由宽度）→ 结果行数（归并量）→ 池等待（排队）** 三个候选排查，分别对应 showSQLStat 的 avgShards、慢日志 rows、showBackends 池水位。还有一类"**偶发慢**"最难抓：全分片查询的木桶效应（一个分片抖动整条查询抖动，第 12 篇第 9 节）——特征是所有单片指标正常但 P99 高，解法是降低路由宽度而不是修那个"最慢的分片"。

# 11. 事务异常

| 症状 | 根因 | 处置 |
|------|------|------|
| `No operations allowed after connection closed` 在事务中途 | 后端连接被摘（心跳）/池回收了事务中的连接 | 事务时长 < 池 idleTimeout；故障期报错由应用重试 |
| XA 提交报错、留下悬挂分支 | PREPARE 成功后协调者/某分支失联 | 查 `XA RECOVER` 后端各片，手工 XA COMMIT/ROLLBACK 后重启实例 |
| 事务里读不到自己刚写的数据 | 读请求路由到了从库（延迟）或不同连接 | 事务内读强制主库（见第 8 节） |
| 跨片事务部分成功 | 弱 XA/无 XA 形态下逐语句执行失败不回滚 | 业务改造单片事务或引入补偿 |

**1.6.x 与 2.x 在事务上的排错差异**要点名：1.6.x 默认事务模型是"BEGIN 透传到各用到的连接"，出错时的实际状态高度依赖"哪条语句在哪个片失败了"，日志里看到的是若干独立连接的错误；2.x 原生 XA 有明确的两阶段语义，悬挂分支可在 MySQL 侧 `XA RECOVER` 查到。**悬挂 XA 是重启 Mycat 前必须清掉的**——实例重启不自动回滚后端已 PREPARE 的分支，它们会一直锁住行。

```sql
-- 查看锁等待
SELECT * FROM information_schema.innodb_lock_waits;

-- 查看当前事务
SELECT * FROM information_schema.innodb_trx;

-- 杀死长时间持有锁的事务
KILL <trx_id>;
```

# 12. 常见错误码

| 错误 | 含义 | 解决 |
|------|------|------|
| `Can't connect to MySQL server on '...'` | 后端 MySQL 不可达 | 检查网络和 MySQL 状态 |
| `Access denied for user` | 后端 MySQL 认证失败 | 检查数据源用户密码 |
| `Table doesn't exist` | 后端 MySQL 中表不存在 | 确认表已创建 |
| `XA_RBDEADLOCK` | XA 事务死锁 | 优化事务，缩短持锁时间 |
| `Too many connections` | 后端连接池满 | 增大 maxCon 或减少并发 |
| `No dataNode found` | 路由找不到数据节点 | 检查 Schema 配置 |

补三条高频的：`Found exception when handling query: unsupported` 类报错在 1.6.x 意味着撞上 SQL 支持边界（第 10 篇），2.x 下若仍出现多半是 Calcite 方言差异，换等价写法；`Connection reset`/`Broken pipe` 在**应用→Mycat** 段多是连接空闲被网络设备（LB/云 NAT）掐掉而池不知情——应用池配 testWhileIdle + keepalive 间隔小于网络设备空闲超时；同错误出现在 **Mycat→后端** 段则查后端重启/网络与 wait_timeout 错配（连接空闲超过 MySQL wait_timeout 被服务端单方关闭）。

# 13. 快速诊断命令

```bash
# Mycat2 进程
ps aux | grep mycat2
jstack $(pgrep -f mycat2) > mycat2.jstack

# 连接状态
netstat -an | grep 8066 | wc -l          # 前端连接数
netstat -an | grep 3306 | wc -l          # 后端连接数

# 日志
tail -100 logs/wrapper.log | grep ERROR
tail -100 logs/mycat2.log | grep -i exception

# 管理命令
mysql -h 127.0.0.1 -P 9066 -u root -p -e "/*!mycat:showReactors*/"
mysql -h 127.0.0.1 -P 9066 -u root -p -e "/*!mycat:showHeartbeatStatus*/"
```

jstack 输出里认三类线程就能覆盖大部分卡顿定位：**event loop/processor 线程**大量 RUNNABLE 且堆栈在 socket 读 → 后端返回慢；线程大量 WAITING 在 business/executor 队列 → 归并重活积压（看队列参数与慢模板）；`showReactors` 各线程负载不均衡 → 连接分布不均或某线程被慢归并卡住（第 17 篇 event loop 隔离原则）。

# 14. 小结：启动失败原因表 + 排错口诀

| 启动/运行问题 | 看哪里 | 一句话结论 |
|--------------|--------|-----------|
| 起不来 | wrapper.log 首行 + console | 端口占用/JSON 语法/后端不可达/内存参数，各占其一 |
| 8066 连不上 | telnet → 白名单 → users.json | TCP 通但握手死，先怀疑 ipWhiteList |
| 9066 连不上 | 管理账号 + 来源 IP 策略 | 多半是"被安全策略拦"而非故障 |
| 空结果/路由怪 | EXPLAIN + 直连分片二分 | 类型不匹配与元数据未 reload 是两大惯犯 |
| 心跳断开 | showHeartbeatStatus + 后端日志 | 连接配额撞墙最隐蔽 |
| 倾斜/慢/OOM | avgShards、rows、GC | 全是 SQL 形态问题的不同表情 |
| 事务异常 | XA RECOVER + innodb_trx | 重启前清悬挂分支 |

**排错口诀按序执行**：先分层（三段链路二分）→ 再分型（连不上/慢/错/空）→ 看变更（时间点 ± 配置/发布/数据）→ 用工具（explain、show 系列、直连对比）→ 最后才改参数。绝大多数 Mycat 故障在这套流程下 30 分钟内可定位；定位超过一小时的，九成是**跳过了二分直接改参数**。下一篇（21 面试问答）把这 20 篇的知识按面试问题重新切片。
