# 1. 排查方法论：先定位引擎段，再下钻

ShardingSphere 的故障定位有固定的"漏斗模型"：**一条逻辑 SQL 要穿过 解析 → 路由 → 改写 → 执行 → 归并 → 事务 六段，每段都有独立的可观测面**。拿到告警先别翻代码，按顺序回答三个问题：Actual SQL 长什么样（改写/路由对不对）、执行了几个单元（扇出是否异常）、报错栈落在哪个引擎包（infra 包名即段落）。三个问题答完，90% 的故障能定位到具体配置项或 SQL 写法。观测入口优先级：Proxy 的 DistSQL（PREVIEW/SHOW）> JDBC 的 sql-show > Agent 指标 > 日志 grep。

# 2. 路由不正确

```sql
-- 1. 检查 SQL 是否正确路由
PREVIEW SELECT * FROM t_order WHERE order_id = 1;

-- 预期输出：
-- +------------------+------------------------------------------------+
-- | dataSourceName   | actualSQL                                      |
-- +------------------+------------------------------------------------+
-- | ds0              | SELECT * FROM t_order_1 WHERE order_id = 1    |
-- +------------------+------------------------------------------------+

-- 2. 检查分片规则配置
SHOW SHARDING TABLE RULE t_order;

-- 3. 常见原因：
-- - 分片算法表达式错误（如 user_id % 2 写成 order_id % 2）
-- - 分片键列名与 SQL 中的列名不一致
-- - 分片策略类型选择不当
```

## 2.1 路由排查的决策树

```text
PREVIEW 结果与预期不符
  ├─ 返回全部 N 片 → 分片值没提取到
  │    ├─ SQL 里确实没分片键 → 补条件/冗余键/走异构索引
  │    ├─ 有但被函数包裹（MOD/DATE_FORMAT/隐式转换）→ 改写谓词为裸列比较
  │    ├─ 范围查询 + INLINE 算法 → 换 INTERVAL/HASH_MOD 或开 allow-range-query-with-inline-sharding
  │    └─ 字符串传数字键（char(10) 列传 "007"，算法内 % 运算行为漂移）→ 统一类型
  ├─ 只路由到 1 片但查不到数据 → 算法映射与数据实际分布不符（历史片按旧规则写入）
  └─ 单元数 = M×N（两表 JOIN）→ 绑定表未生效：检查 JOIN 列是否同时含两表分片键、ON 写法、分片配置一致性
```

**JDBC 模式没有 PREVIEW**，等价手段是 sql-show 看 Actual SQL 条数，或代码里直接调 `parseSQL`/单测构造 RouteContext。还有一类"时对时不对"的路由，几乎都是**参数类型不稳定**（同列有时传 Integer 有时传 String）导致算法分支行为差异。

# 3. 改写 SQL 不对：看 SQL 日志

改写层问题的唯一真相源是 Actual SQL（逻辑 SQL 经路由+改写后的物理 SQL）。开启方式：`props: sql-show: true`（INFO 级打印 `Actual SQL: ds0 ::: SELECT ... ::: [params]`），排查完必须关（19 篇性能税）。三个典型症状与归因：

```text
症状1：逻辑 SQL 写了列 a，物理 SQL 变成 a_encrypted AS a
  → 正常（加密规则改写），ORM 映射异常时检查是否用了列下标取值
症状2：分页 LIMIT 100,10 变成 LIMIT 0,110
  → 正常（分页改写），若归并后行数不对再查 ORDER BY 是否下推
症状3：物理 SQL 完全没变（表名还是逻辑名）
  → 该语句解析失败走了降级直通（grep "SQLStatement is null" / ParserException），
    或表未配置规则落入单表模式；用 PARSE SQL 验证语法覆盖
```

> **注意**：sql-show 打印的是"发往物理库的最终文本 + 参数列表"，它无法告诉你**为什么**是这个结果；结合 15/16 篇的 Token 改写模型与 RouteContext 结构，"实际发出的 SQL 减去预期"的差集就是某类改写器干的活，按差集形态反推改写器种类是最快的归因路径。

# 4. Proxy 连接失败

```bash
# 1. 检查 Proxy 进程
ps aux | grep shardingsphere-proxy

# 2. 检查端口监听
netstat -tlnp | grep 3307

# 3. 测试连通性
mysql -h 127.0.0.1 -P 3307 -u root -p

# 4. 常见错误：
# ERROR 2003 (HY000): Can't connect → Proxy 未启动
# ERROR 1045 (28000): Access denied → 用户名/密码错误
# ERROR 1040 (HY000): Too many connections → 连接数满

# 5. 查看 Proxy 日志
tail -100 /opt/shardingsphere-proxy/logs/stdout.log | grep ERROR
```

1045 在 Proxy 语境下多两层含义：`authority` 配置的**用户 host 通配**不匹配（`root@%` vs 客户端来源 IP），或密码用了非 MySQL 原生认证插件；**后端连不上是另一个维度**——Proxy 对客户端握手成功后执行 SQL 才暴露后端问题（register storage unit 时 `PROBE` 预检、`SHOW STORAGE UNITS` 看状态），日志关键字 `Failed to initialize` / HikariPool 连接超时。Too many connections 要先分清是**前端会话满**（proxy-frontend 无硬上限，看文件句柄与内存）还是**后端池满**（各存储单元 maxPoolSize 之和超 DB 承受），处置方向完全不同。

# 5. 跨库分布式事务异常

```bash
# 1. 检查事务超时配置
SHOW VARIABLES LIKE '%transaction%';

# 2. 查看 XA 事务状态
# MySQL:
XA RECOVER;

# 3. 手动清理悬挂的 XA 事务
XA COMMIT 'xid-xxx';     # 或
XA ROLLBACK 'xid-xxx';

# 4. 常见原因：
# - 数据库连接断开导致 prepared 事务悬挂
# - 事务执行时间超过 timeout
# - 网络波动导致 RPC 失败

# 5. 缩短事务时间
# - 业务拆分为多个小事务
# - 减少事务中的 RPC 调用
# - 增加事务超时阈值
```

## 5.1 XA 悬挂的标准处置流程

`XA RECOVER CONVERT XID` 列出 prepared 分支后，**不能盲回滚**：回滚一个"TM 已决定提交"的事务会造成跨库不一致。判定顺序：① 用 xid 的 gtrid 反查 Atomikos/Narayana 日志（`resource.log`/journal）确认全局决策；② 日志显示 COMMIT 意图 → `XA COMMIT 'gtrid','hagramid',1` 前向恢复；无决策记录 → ROLLBACK；③ 处置后核对受影响业务表的数据完整性。批量场景写恢复脚本对接 `RECOVER XID` 接口（TM 侧也有 recovery 模块可自动跑）。防复发三件套：TM 日志持久卷、`xa-timeout` 设置、业务事务瘦身（18 篇第 6 节）。

**Seata AT 的对应异常**是另一套症状：`LockConflictException`（全局锁等待，典型是长事务持锁）、undo_log 残留（回滚失败，人工比对镜像后清理）、以及"回滚了不该回滚的"（脏写，通常是同记录被绕过代理的直连修改污染）。排查抓手是 Seata 控制台按 xid 追分支状态机，而非查 SS 侧日志。

# 6. SQL 解析错误

```bash
# 1. 使用 PARSE 命令测试
PARSE SELECT * FROM t_order WHERE ...;

# 2. 常见不支持的语法：
# - 存储过程 CALL
# - 复杂子查询（嵌套超过 2 层）
# - 特定数据库的专有函数（如 MySQL 的 GROUP_CONCAT）

# 3. 解决：
# - 简化 SQL
# - 拆分为多条 SQL
# - 使用 Hint 路由绕过解析
```

解析失败的行为分两种：语句含分片表时通常抛 `SqlParsingException`（或降级直通，取决于版本与 props）；**更隐蔽的是"解析成功但语法树不完整"**——g4 未覆盖的子句被静默丢弃，表现为"ORDER BY 没生效""LIMIT 没改写"，只能用 PREVIEW 比对物理 SQL 发现。处置优先级：改写 SQL 为标准形态 > 自定义 SQL 审核拦截 > 升级方言 g4（官方 issue 驱动）> 注释 hint 指定路由绕过（仅解决路由，不解决改写缺失）。GROUP_CONCAT 这类方言函数在分片表上的正确姿势是"单片内使用"（查询带分片键）或改造成应用层拼接。

# 7. 内存溢出与连接泄漏

```bash
# 1. 查看 JVM 内存
jstat -gcutil $(pgrep -f shardingsphere) 1000

# 2. 生成 heap dump
jmap -dump:live,format=b,file=heap.hprof $(pgrep -f shardingsphere)

# 3. 常见原因：
# - 大结果集未流式归并 → 增加 executor-size 或调小结果集
# - 连接数过多 → 限制 maxPoolSize
# - 元数据缓存 → 定期 REFRESH TABLE METADATA

# 4. 增加内存
# -Xms2g -Xmx4g
```

堆分析三板斧落点到 SS 具体对象：MAT 里 dominator 树顶若是 `MemoryQueryResult`/`ArrayList` 行缓存 → 内存归并/物化被触发（17 篇 8.1，加 ORDER BY 或限结果集）；若是 Netty `ByteBuf` 堆积 → Proxy 大结果集导出未流控（调 flush-threshold、导出改游标拉取）；若是 `RouteContext`/SQLStatement 缓存 → 超长 SQL 模板爆炸（检查未用 PreparedStatement 的代码路径，开解析缓存治理）。

**连接泄漏的特征**：池内 active 数只涨不降、`getConnection` 超时，栈上泄漏点常是**手工拿 `ShardingSphereConnection` 未 close** 或 **HintManager 未 close（ThreadLocal 泄漏导致路由错 + 连接占用双重症状）**。Proxy 侧会话泄漏看 `SHOW PROCESSLIST` 里 Sleep 长连接堆积，配会话空闲回收（`proxy-session` 相关参数）与前端超时。JDBC 泄漏定位开池 `leakDetectionThreshold`（Hikari 60s 打栈），比事后 dump 快一个数量级。

# 8. 性能：全路由扫描识别

```bash
# 指标化识别扇出异常（三种途径）
# 1) sql-show 临时开启，统计单位时间 Actual SQL 条数 / 逻辑 SQL 条数 比值
# 2) Agent 指标：route 相关 counter 按逻辑表打标，比值 > 2 持续告警
# 3) 库侧旁证：所有分片同时出现相同模板慢查询 = 全路由广播
mysql> SELECT DIGEST_TEXT, SCHEMA_NAME, COUNT_STAR,
    ->        SUM_ROWS_EXAMINED/COUNT_STAR AS avg_rows
    -> FROM performance_schema.events_statements_summary_by_digest
    -> ORDER BY avg_rows DESC LIMIT 10;
```

全路由的**业务侧指纹**很好认：单条逻辑查询 RT ≈ 最慢分片 RT + 归并开销（非等比放大则说明并发扇出健康）、连接池瞬时占用 = 扇出数、数据库全片同刻 IO 尖刺。识别后的治理路径回到 19 篇第 6 节四姿势；临时止血可用 `max-connections-size-per-query` 压并发防打爆池子——**注意它是限流不是优化**，扇出总量不变，只是排队。写类全路由（UPDATE/DELETE 无分片键）在 `sql-audit` 里直接拒绝，不进入性能讨论。

# 9. 分布式主键冲突

```sql
-- 症状：Duplicate entry 'xxx' for key 'PRIMARY'，但业务层没传过重复值

-- 排查顺序
-- 1) worker-id 是否重复：SNOWFLAKE 靠 (workerId, datacenterId) 区分实例
SHOW DIST VARIABLE;  -- 或查看实例启动日志中 worker-id 分配记录
-- Standalone 模式默认按 IP/进程内分配，容器化重启/克隆机器会重号

-- 2) 时钟回拨：NTP 校时把机器时间拨回 → 雪花生成器抛异常或复用序列
-- 日志关键字：Clock moved backwards

-- 3) 规则外的直写：某链路绕过 SS 用数据库 AUTO_INCREMENT 写同一列
```

治理：生产用 **Cluster 模式让 worker-id 由注册中心统一分配**（ZooKeeper 抢占节点、持久化，实例重启复用不冲突）；容器场景给每实例固定 `properties` 覆盖 worker-id 也可，但扩缩容脚本要防重。时钟纪律：禁 NTP 步进 slew 模式、或选带回拨保护的生成器（COSID/Leaf 号段模式）。**主键空洞**（不连续）不是故障——雪花本就不连续，把它当 bug 修是新手误区；要连续语义只有号段模式（Leaf/DB 号段），接受吞吐损耗。

# 10. 数据不一致与读写分离不一致

```bash
# 1. 检查各分片数据量
# 直接连接每个物理库查询
SELECT COUNT(*) FROM t_order_0;
SELECT COUNT(*) FROM t_order_1;

# 2. 验证分片算法
# 计算某个 ID 应该路由到哪个分片
# user_id = 5 → user_id % 2 = 1 → ds1

# 3. 常见原因：
# - 分片规则修改后历史数据未迁移
# - 数据写入时直连了物理库而非经过 ShardingSphere
# - 分布式主键生成器 worker-id 冲突

# 4. 修复：
# - 全量迁移到新分片规则
# - 启用 sql-show 确认 SQL 正确路由
```

```bash
# 1. 检查主从延迟
# 连接从库
SHOW SLAVE STATUS\G
# Seconds_Behind_Master

# 2. 强制走主库（代码中）
HintManager hintManager = HintManager.getInstance();
hintManager.setWriteRouteOnly();

# 3. 检查读写分离规则
SHOW READWRITE_SPLITTING RULES;
```

"写后读不到"先区分三种世界：**主从延迟**（秒级、可容忍读最终一致）、**事务内混合读写路由**（SS 已保证同事务强制主库，异常多为代码在事务外重查）、**动态数据源/缓存层叠加**（绕过 SS 的读路径用了独立连接池直连从库）。修复选项按敏感度排序：关键读走 hint 主库、读链路统一延迟双删+位点检查、或从库侧 `mysql` 半同步。

# 11. 快速诊断命令

```bash
# ShardingSphere Proxy
SHOW STORAGE UNITS;                    # 查看所有存储单元
SHOW SHARDING TABLE RULES;             # 查看所有分片规则
PREVIEW SQL;                           # 预览 SQL 路由
SHOW COMPUTE NODES;                    # 查看集群计算节点（5.5 替代旧版 SHOW INSTANCE LIST）
SHOW READWRITE_SPLITTING RULES;        # 查看读写分离
SHOW TRANSACTION RULE;                 # 全局事务默认类型
PARSE SQL "...";                       # 验证语法是否可解析
EXPORT DATABASE CONFIGURATION;         # 规则存档（改前必做）
REFRESH TABLE METADATA;                # 物理 DDL 后刷新元数据缓存

# 系统层面
netstat -an | grep 3307 | wc -l        # Proxy 连接数
jstack $(pgrep -f proxy) > jstack.log  # 线程栈
jstat -gcutil $(pgrep -f shardingsphere) 1000  # GC 水位
```

# 12. 常见报错对照表

| 报错 | 引擎段 | 根因 | 处置 |
|------|--------|------|------|
| `Can not find data source or tables` | 路由 | 逻辑表未配规则且非单表模式 | 补规则/LOAD SINGLE TABLE |
| `Table or view 'xxx' does not exist`（物理名） | 改写/执行 | 物理表未建、actualDataNodes 写错 | SHOW SHARDING TABLE NODES 比对 |
| `There is no SQL parsing rule` / ParserException | 解析 | 方言不匹配或语法超 g4 覆盖 | PARSE 验证、指定 databaseType |
| `Duplicate entry for key PRIMARY` | 主键 | worker-id 冲突/时钟回拨/直写 | 见第 9 节 |
| `XAER... / Unknown XID` | 事务 | 悬挂被对端清理后重复 commit | 恢复流程 5.1 |
| `Could not open JDBC Connection for transaction` | 执行 | 池耗尽（扇出放大） | 二维账本核算、压 maxPoolSize |
| `Result set is not streaming` / OOM | 归并 | 内存归并/物理全量缓冲 | fetchSize + 流式条件（17 篇 8.1） |
| `Namespace ... is occupied` | 模式 | Cluster 命名空间冲突（多环境共用 ZK） | 隔离 namespace/overwrite 参数 |
| `Connection refused (ZooKeeper...)` | 治理 | 注册中心不可达，规则无法加载/热更 | ZK 健康、本地配置兜底策略 |

> **行业实践**：把本表进团队 runbook 时补一列"值班自救命令"，保证任何一次 P2 以上故障的**前 5 分钟动作是确定性的**（看 PREVIEW/Actual SQL、查 SHOW STORAGE UNITS 状态、开/关 sql-show 的决策树），而不是全员进源码找异常栈。

# 13. 小结

1. **漏斗模型定位**：先看 Actual SQL/扇出数/异常栈落点，再进对应引擎段下钻
2. **路由问题查提取（PREVIEW），改写问题查日志（sql-show），解析问题查 PARSE**——三大手段各管一段
3. **XA 悬挂不可盲回滚**，以 TM 日志判定前向/反向恢复；AT 异常按 xid 追分支状态机
4. **内存与连接问题看对象指纹**：MemoryQueryResult/ByteBuf/HintManager 泄漏各有特征栈
5. **主键冲突查 worker-id 与时钟，数据不一致查直写**，报错对照表进 runbook 并配"前 5 分钟动作"