# 1. 概述

**分库分表（Sharding）是 Mycat 的核心能力，也是它区别于"纯读写分离代理"的标志**：把一张逻辑表水平拆成多个物理子表，分布在不同的数据库实例或同一实例的不同库上，SQL 路由由分片键（sharding column）决定。

```text
逻辑表 orders (1000万行)
    │
    │ 按 user_id % 4 分片
    │
    ├──→ dn1.orders (user_id=0,4,8,...)     # 分片 0
    ├──→ dn2.orders (user_id=1,5,9,...)     # 分片 1
    ├──→ dn3.orders (user_id=2,6,10,...)    # 分片 2
    └──→ dn4.orders (user_id=3,7,11,...)    # 分片 3
```

**Mycat 1.6 的一个硬限制要先说清：它只分库、不分表**——一个逻辑表在每个 dataNode 上只对应一张同名物理表。想要"2 实例 × 每实例 2 表"，得把每个物理表包装成独立 database 的 dataNode，或者干脆接受"分片数 = 库数"。这是 1.6 与 ShardingSphere 的功能差距之一：**Mycat 2.x 才原生支持一个逻辑表映射多张物理分表**。理解 1.6 的分片，本质是理解"分片键的值如何通过规则函数算出目标 dataNode 下标"。

路由的两条路径是全部讨论的底座：

```text
INSERT INTO orders(user_id,...) VALUES (7,...)
    → 分片键已知(user_id=7) → 代入规则函数 → 命中 dnX → 精确路由（最快路径）

SELECT * FROM orders WHERE status = 'PAID'
    → 分片键未知 → 无法定位 → 广播所有 dataNode → 结果归并（代价高）
```

> **注意**：分片键的值不允许被 UPDATE 修改——改了分片键等于数据要跨节点搬家，Mycat 或拒绝或产生错误路由（旧行留在原节点）。规划分片键时必须选**不可变字段**，这是分片方案评审的第一条检查项。

# 2. 1.6 配置分工：schema.xml 定拓扑，rule.xml 定规则

一张分片表的完整配置横跨两个文件。以"orders 按 user_id 取模分 4 片"为例，schema.xml 声明逻辑库、表的候选节点与规则名，并给出物理拓扑：

```xml
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE mycat:schema SYSTEM "schema.dtd">
<mycat:schema xmlns:mycat="http://io.mycat/">
  <schema name="TESTDB" checkSQLschema="false" sqlMaxTimeout="100">
    <table name="orders" primaryKey="id" autoIncrement="true"
           dataNode="dn1,dn2,dn3,dn4" rule="orders-partition"/>
  </schema>
  <dataNode name="dn1" dataHost="host1" database="test_db"/>
  <dataNode name="dn2" dataHost="host1" database="test_db2"/>
  <dataNode name="dn3" dataHost="host2" database="test_db"/>
  <dataNode name="dn4" dataHost="host2" database="test_db2"/>
  <dataHost name="host1" maxCon="1000" minCon="10" balance="0"
            writeType="0" dbType="mysql" dbDriver="native" switchType="1"
            slaveThreshold="100">
    <heartbeat>select 1</heartbeat>
    <writeHost host="w1" url="192.168.1.10:3306" user="mycat" password="mycat123"/>
  </dataHost>
  <dataHost name="host2" maxCon="1000" minCon="10" balance="0"
            writeType="0" dbType="mysql" dbDriver="native" switchType="1"
            slaveThreshold="100">
    <heartbeat>select 1</heartbeat>
    <writeHost host="w2" url="192.168.1.11:3306" user="mycat" password="mycat123"/>
  </dataHost>
</mycat:schema>
```

rule.xml 里 tableRule 把分片列绑定到 algorithm，function 给出算法类与参数：

```xml
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE mycat:rule SYSTEM "rule.dtd">
<mycat:rule xmlns:mycat="http://io.mycat/">
  <tableRule name="orders-partition">
    <rule>
      <columns>user_id</columns>
      <algorithm>mod-4</algorithm>
    </rule>
  </tableRule>
  <function name="mod-4" class="io.mycat.route.function.PartitionByMod">
    <property name="count">4</property>
  </function>
</mycat:rule>
```

**三层命名必须一一对应**：schema.xml 的 rule 属性指向 tableRule name，algorithm 指向 function name。`count=4` 决定 mod 结果落在 0..3，恰好映射 dataNode 列表下标；任何一处不一致都会在 INSERT 时抛路由越界错误，**Mycat 不做启动期校验，只在运行期炸**。


# 3. 分片算法家族

**Mycat 1.6 内置约 16 种分片函数**（`io.mycat.route.function.PartitionByXxx`），选型看三件事：**数据均匀性、扩容成本、范围查询友好度**。Mycat2 的函数类名基本相同，但配置内联进 schemas.json，不再有"三层命名对齐"陷阱。

## 3.1 取模（PartitionByMod）

`分片 = 分片键值 % count`。数据最均匀、实现最简单，但 **count 与节点数强绑定，扩容时历史数据归属全部重算，需要全量迁移**——这是取模被诟病的根源。适合规模可预判、或明确接受"扩容即搬家"的场景。

## 3.2 哈希（PartitionByString / MurmurHash）

字符串键用 PartitionByString，取分片键前 length 个字符做 hash 再分配：
```xml
<function name="str-partition" class="io.mycat.route.function.PartitionByString">
    <property name="partitionCount">2 2</property>
    <property name="length">2</property>
</function>
```
一致性 hash 用 PartitionByMurmurHash：先把分片键映射到 256 个虚拟桶，桶再映射到物理节点。**扩容时只有新增桶段的数据需要迁移**，理论迁移量约 1/新节点数，代价小：

```xml
<function name="murmur-256" class="io.mycat.route.function.PartitionByMurmurHash">
    <property name="count">256</property>
    <property name="seed">0</property>
    <property name="virtualBucketTimes">160</property>
</function>
```

## 3.3 范围分片（PartitionByRangeLong）

按数值区间划分，每段区间对应节点编号，**范围查询天然友好、追加式扩容只动最后一段**，代价是最新区间容易成热点：
```xml
<function name="range-partition" class="io.mycat.route.function.PartitionByRangeLong">
    <property name="partitionCount">3</property>
    <property name="range">0-999999=0,1000000-1999999=1,2000000-2999999=2</property>
</function>
```
## 3.4 日期分片（PartitionByDate）

按时间列自然分段，日志、账单、订单等时间序列数据的首选；配合"老分区转归档库"的冷热分离运维非常顺手：

```xml
<function name="date-partition" class="io.mycat.route.function.PartitionByDate">
    <property name="sBeginDate">2024-01-01</property>
    <property name="sPartionDay">30</property><!-- 每 30 天一个分片 -->
    <property name="dateFormat">yyyy-MM-dd</property>
</function>
```
## 3.5 枚举/映射分片（PartitionByFileMap）

分片键取值是有限集合（地区码、租户类型、月份）时，用"值→分片号"映射文件，路由逻辑完全由人工控制，可以精确地把大客户数据放到专用节点。conf 下 region-partition.txt：

```text
1=0
2=1
3=2
4=3
```

对应 function：

```xml
<function name="region-map" class="io.mycat.route.function.PartitionByFileMap">
    <property name="mapFile">region-partition.txt</property>
    <property name="type">0</property>
</function>
```
## 3.6 算法对比总表

| 算法 | 均匀性 | 扩容成本 | 范围查询 | 典型分片键 | 适用场景 |
|------|--------|---------|---------|-----------|---------|
| PartitionByMod 取模 | 高 | 高（全量迁移） | 差 | 数值 ID | 规模可预判、写入均匀 |
| PartitionByString 哈希 | 中高 | 高 | 差 | 字符串键 | 租户 ID、手机号段 |
| PartitionByMurmurHash | 高 | 低（部分迁移） | 差 | 数值/字符串 | 计划多次扩容 |
| PartitionByRangeLong | 低（看业务） | 低（追加） | 好 | 数值区间 | 区间查询频繁 |
| PartitionByDate | 中（有天然热点） | 低（按天追加） | 好 | 时间列 | 日志、账单、冷热分层 |
| PartitionByFileMap 枚举 | 人工决定 | 低（改映射文件） | 好 | 地区/类型 | 地域多活、大客户隔离 |

> **设计争议**：取模 vs 一致性 hash 之争，本质是"扩容频率"的预判之争。国内互联网公司的实践是**一次性把分片数设计为足够大（如 1024/2048 个逻辑分片映射到少量物理库）**，用"逻辑分片数远大于物理节点数"换取"扩容只搬分片不重新路由"——这个模式源自淘宝 TDDL，Mycat 1.6 因为只分库不分表，实现同样的效果要靠 PartitionByFileMap 手工维护"逻辑桶→物理节点"映射表，运维负担明显更重；这是选 1.6 做超大规模分片前必须掂量的点。

# 4. 分片键选择原则

**分片键（sharding column）决定了一切查询的路由命运**，选择原则比算法选择更重要：

```text
✓ 好的分片键：
   - 高区分度（接近唯一），数据分布天然均匀
   - 高频查询条件（90% 以上 SQL 的 WHERE 会带上它）
   - 不可变（一旦写入永不修改，UPDATE 分片键被中间件禁止）
   - 与主键分离思考：主键管唯一，分片键管路由，可以不同

✗ 差的分片键：
   - 低区分度：status 只有 5 个值 → 最多 5 片，还倾斜
   - 查询不带它：90% SQL 不含分片键 → 90% 查询全分片广播
   - 会被修改：手机号做分片键，换号即搬家
```

**实战中的经典决策**：订单系统里"按 user_id 分片还是按 order_id 分片"。按 user_id：查"我的订单"精确路由，但查单个订单详情必须带 user_id，客服系统全站搜 order_id 就是广播；按 order_id 反之。**常见的折中是 order_id 用全局序列生成、且把 user_id 编码进 order_id 的低位**（如 Snowflake 低位塞 worker/取模信息，或干脆 `order_id = 号段 × 1000 + user_id % 1000`），让两个键都能算出分片——这是 Mycat 生态里普遍的应用层自定义玩法。

> **行业实践**：先拉出生产慢查询与流量 TOP SQL，统计 WHERE 条件出现频率，再定分片键——**分片键是给查询模式选的，不是给表结构选的**。上线后改分片键等于二次扩容搬家，成本极高。

# 5. 分库分表组合与扩容

## 5.1 布局组合

1.6 中"分库分表"的组合自由度来自 dataNode 的定义方式——多个 dataNode 可以指向同一 dataHost 的不同 database，从而在单实例上模拟"分表"：

```text
需求：2 实例 × 每实例 2 库 = 4 分片

host1 (192.168.1.10:3306)         host2 (192.168.1.11:3306)
  ├── database: test_db             ├── database: test_db
  │     └── orders  ← dn1           │     └── orders  ← dn3
  └── database: test_db2            └── database: test_db2
        └── orders  ← dn2                 └── orders  ← dn4
```

dataNode 列表顺序即分片下标：`dataNode="dn1,dn2,dn3,dn4"`，mod 结果 0→dn1、1→dn2、2→dn3、3→dn4。**Mycat2 的原生逻辑分表**则直接在 schemas.json 里用 `dataNodes: "db0.orders_${0..1}"` 风格的占位展开，一库多表不再需要拆 database 模拟。

## 5.2 自动扩容分片（先扩片再扩机）

Mycat 官方推荐的平滑扩容思路是"**分片数一次定够，物理节点逐步追加**"：1024 个逻辑分片先映射到 4 个 dataNode，扩容第 5 个节点时，只需把某些分片段的映射从旧节点改到新节点，并搬那几段的数据——借助 PartitionByFileMap/一致性 hash 的映射表，**路由变更与数据迁移的范围被压缩到被重分配的桶**。

停机扩容流程（小数据量）：

```bash
# 1. 停止应用写入（或全局序列号段预留 + 只读）
# 2. 按新规则导出数据
mysqldump -h old_host -u mycat -p test_db orders > orders.sql
# 3. 修改 schema.xml/rule.xml：加 dataNode、调 count
# 4. 按新路由重新导入到目标分片
# 5. reload 配置并校验路由，再放开写入
mysql -h127.0.0.1 -P9066 -umycat -p -e "RELOAD @@CONFIG"
```

在线扩容流程（生产推荐，双写 + 增量同步）：

```text
1. 配置新分片，写入仍全走旧分片
2. 全量迁移：旧分片历史数据按新规则灌入新分片
3. 增量同步：Canal/复制链路把迁移期间的变更追平
4. 双写校验：切一部分读流量到新分片比对数据一致性
5. 切换路由：reload 配置，新写按新规则落子
6. 观察期后下线旧分片数据
```

> **注意**：Mycat **没有内置数据再平衡（rebalance）能力**，第 2~3 步的搬迁全部依赖外部工具（mysqldump、Canal、自研迁移程序），且切换瞬间未带分片键的广播查询可能读到新旧两份数据。Mycat2 宣称支持在线扩容迁移，成熟度仍待验证；对比之下 ShardingSphere 的 DistSQL `MOVE TABLE`/弹性迁移是这类需求做得最完整的开源实现。
# 6. 全局表与分片表的路由差异

**同一份 SQL 在不同表类型上的路由行为完全不同**，这张对照表是理解 Mycat 路由器的钥匙（全局表/ER 表机制详见第 8 篇）：

| SQL 场景 | 普通分片表 | 全局表（type=global） | ER 子表（parent 已声明） |
|----------|-----------|---------------------|------------------------|
| INSERT（带分片键） | 精确路由 1 个节点 | 广播到所有节点 | 按父表分片键跟随父节点 |
| INSERT（不带分片键） | 广播/报错（无法定位） | 正常广播 | 若父键在则跟随，否则退化 |
| SELECT 带分片键 | 精确路由 | 任选一个节点 | 精确路由 |
| SELECT 不带分片键 | 全节点广播 + 归并 | 任选一个节点 | 全节点广播 |
| JOIN 分片表 × 全局表 | 可下推到各节点本地 JOIN | — | 同左 |
| JOIN 分片表 × 分片表 | 除非分片键对齐，否则中间件级 JOIN（慢） | — | ER 对齐则本地 JOIN |

**设计考量**：

- **全局表把"复制成本"花在写上、把"路由收益"花在读上**，只适合低频变更的小表（字典、地区、汇率），高频写的全局表会让每个分片都承受同样流量，等于没有分片
- **分片表 JOIN 分片表是路由最难的情况**：Mycat 1.6 的处理顺序是"先尝试 ER/全局表优化 → 再做分片键对齐推断 → 都失败就拉回中间件做嵌套循环 JOIN（性能灾难）"
- **不带分片键的 INSERT 在取模规则下是致命的**：值决定路由，INSERT 时必须计算，Mycat 会解析 VALUES 里的分片键列；如果该列不在 INSERT 列清单里（指望数据库默认值），直接报错——这是"批量导入脚本在 Mycat 上失败"的头号原因

# 7. 跨分片查询的处理方式

| SQL 类型 | 处理方式 | 代价 |
|---------|---------|------|
| `WHERE user_id = 7` | 精确路由 → 单节点执行 | 无额外代价 |
| `WHERE user_id IN (1,5,9)` | 计算 3 个目标节点并行 → 结果拼接 | 多连接并发 |
| `ORDER BY age LIMIT 10 OFFSET 5` | 全分片各取 `OFFSET+LIMIT` 条 → 归并排序后截取 | 深分页放大 N 倍 |
| `GROUP BY type` | 全分片分组聚合 → 中间件二次聚合 | 内存聚合 |
| `COUNT(*)` | 全分片计数 → 求和 | 低 |
| `AVG(col)` | 各分片返回 SUM 与 COUNT → 全局重算 | 改写逻辑复杂 |
| 跨分片 JOIN | ER/全局表下推，否则中间件 JOIN | 高，尽量避免 |

**深分页是跨分片查询的第一杀手**：`LIMIT 100000, 10` 在 16 分片上意味着每个分片都要取回前 100010 行——中间件侧内存与网络全部爆炸。通用解法是把分页改成"游标式"：记住上一页最后一条的全局序列 id，用 `WHERE id > 上次最大id ORDER BY id LIMIT 10` 翻页，让排序键与分片键语义对齐。

# 8. 小结

**Mycat 分片 = 分片算法 + 分片键 + 路由规则三要素**：1.6 用 schema.xml 定拓扑、rule.xml 定算法，count 与 dataNode 数量必须一致；算法家族覆盖取模/哈希/一致性哈希/范围/日期/枚举，均匀性、扩容成本、范围查询友好度三者不可兼得，**先按 TOP SQL 的 WHERE 分布定分片键，再按扩容预期选算法**；全局表与 ER 表是 JOIN 优化的两条腿；扩容没有内置 rebalance，要么一次把逻辑分片定够（1024 桶映射），要么走"全量迁移 + 增量追平 + 路由切换"的双写流程。Mycat2 补齐了一库多表与在线迁移，但生产验证仍以 1.6 存量为主。