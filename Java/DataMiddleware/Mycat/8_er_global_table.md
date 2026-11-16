# 1. 概述：分片后 JOIN 为什么难

**分库分表把 JOIN 从数据库内部问题变成了跨网络分布问题**：单库里 `orders JOIN order_items` 由优化器用Nested Loop/Hash Join 本地完成；拆到 16 个节点后，一条跨表 SQL 的路由引擎要回答三个问题——两张表的数据在哪些节点、怎么把关联行凑到一起、凑不到一起时退化成什么代价。

```sql
SELECT o.*, i.*
FROM orders o JOIN order_items i ON o.id = i.order_id;
```

按两表的分片关系，Mycat 面对三种局面：

```text
局面 A：orders 按 id 分片，order_items 按 order_id 分片，同一算法同一分片数
        → 关联双方落在"相同下标的分片" → 可以下推到每个节点本地 JOIN ✅

局面 B：orders 与 dict_city（小字典表）JOIN
        → 把 dict_city 在每个分片各复制一份 → 任意分片都能本地 JOIN ✅（全局表）

局面 C：orders 按 id 分片，users 按 city 分片，分片键不对齐
        → 无法下推 → Mycat 拉数据回中间件做嵌套循环 JOIN ❌（性能灾难）
```

**Mycat 1.6 提供两种结构化手段消灭局面 C**：**ER 分片**（让父子表天然同分片，即局面 A）与**全局表**（让小表到处都有，即局面 B）。二者都写在 schema.xml 的 `<table>` 定义里，路由引擎识别表类型后自动选择 JOIN 策略。Mycat2 基于 Calcite 会额外尝试分片键对齐推断（如 `A.x = B.y` 且 y 是 B 的分片键时可交换路由），但对"ER 缠绕 + 全局表复制"这两种声明式优化的依赖不变。

> **注意**：跨分片 JOIN 的兜底实现（把驱动表全量拉回、逐行去被驱动表查询）在 Mycat 1.6 上是真实存在的代码路径，它会静默地跑起来并静默地慢死——**没有报错的慢查询比报错更危险**。上线前用 `EXPLAIN` 检查每条 JOIN 的路由结果，凡是出现"多个节点 + 中间件 JOIN"字样的都要治理。

# 2. ER 分片：父子缠绕

## 2.1 原理

**ER（Entity-Relationship）分片的思想**：子表不独立决定分片位置，而是**跟随父表的数据分布**——父行在哪个分片，其所有子行就在哪个分片。实现上要求分片规则保证"父键与子键映射到同一分片下标"，配置上通过 `parent` + `join` + `parentColumn` + `childColumn` 四个属性声明：

```text
分片 dn1                          分片 dn2
├── orders: id=1,5,9,...          ├── orders: id=2,6,10,...
└── order_items:                  └── order_items:
      order_id=1 → 2 行 item            order_id=2 → 3 行 item
      order_id=5 → 1 行 item            order_id=6 → 1 行 item
  （子表行与父行同库，JOIN 完全本地化）
```

## 2.2 完整配置（1.6 schema.xml）

```xml
<schema name="TESTDB" checkSQLschema="false" sqlMaxTimeout="100">
  <table name="orders" primaryKey="id" autoIncrement="true"
         dataNode="dn1,dn2,dn3,dn4" rule="orders-by-id"/>
  <table name="order_items" primaryKey="id" dataNode="dn1,dn2,dn3,dn4"
         rule="items-by-orderid"
         childTable="order_item_details"
         parent="orders"/>
  <table name="order_item_details" primaryKey="id"
         dataNode="dn1,dn2,dn3,dn4"
         parent="order_items"/>
</schema>

<!-- rule.xml：父表按 id 取模，子表按 order_id 取模，count 相同 -->
<tableRule name="orders-by-id">
  <rule>
    <columns>id</columns>
    <algorithm>mod-4</algorithm>
  </rule>
</tableRule>
<tableRule name="items-by-orderid">
  <rule>
    <columns>order_id</columns>
    <algorithm>mod-4</algorithm>
  </rule>
</tableRule>
```

**Mycat 1.6 对 ER 的处理机制**：路由时若 JOIN 的驱动表带了分片键（如 `WHERE orders.id = 1`），子表不再按自己的规则计算，**直接用父表算出的节点集合**；子表单独查询仍走自己的 rule。`childTable` 属性用于三层以上的级联声明。注意 Mycat 的 ER 是"声明增强路由"，**并不强制约束你的写入逻辑**——往 order_items 插入一条 order_id=1 但路由结果不在 dn1 的记录，Mycat 会照按子表规则执行，ER 关系就被破坏了。

## 2.3 查询效果

```sql
-- 带父表分片键：精确路由 + 本地 JOIN
SELECT o.*, i.*
FROM orders o JOIN order_items i ON o.id = i.order_id
WHERE o.id = 1;
-- EXPLAIN 显示只落在 dn1，两表同库，MySQL 原生 JOIN

-- 不带分片键：各分片本地 JOIN 后归并（仍可接受，无跨库拉取）
SELECT o.*, i.*
FROM orders o JOIN order_items i ON o.id = i.order_id
WHERE o.status = 'PAID';
-- 广播到 4 个分片，每个分片内部 JOIN，结果拼接

-- 子表反向查父：order_items 的分片键在 WHERE 里即可精确路由
SELECT o.*, i.*
FROM orders o JOIN order_items i ON o.id = i.order_id
WHERE i.order_id = 1;
```

> **设计考量**：ER 缠绕的前提是**"子键与父键的分片函数等价"**——上面配置里 mod-4 对 `id` 与 `order_id` 用同一个函数同一个 count，这要求 order_id 的值域与 id 一致。实践中更常见的做法是用全局序列同时生成两表主键，天然满足。一旦父表改规则（count 变了），所有子表必须同步改，**ER 表组的规则变更是原子事务**，这也是 Mycat 把 `rule` 拆独立文件带来的隐性耦合。

## 2.4 三层 ER

```xml
<table name="order_item_details" primaryKey="id"
       dataNode="dn1,dn2,dn3,dn4"
       parent="order_items"/>
```

```text
orders ──┬── order_items ──┬── order_item_details
         │                 └── ...
         └── order_payments（同为 orders 的子表）

三层关联时 Mycat 沿 parent 链逐级取父表已算好的节点集合，
JOIN 越深、路由越依赖"链顶带分片键"；链顶没分片键则全分片各自本地 JOIN。
```

**避免过深的 ER 链**：链上任何一环的分片键计算错误或数据错位，都会让整条链的 JOIN 结果静默不全。三层是社区实践的上限共识。

# 3. 全局表：广播复制

## 3.1 原理与配置

**全局表（Global Table）= 每个分片存一份完整副本**，用写放大换 JOIN 本地化，适合"数据量小、读多写少、变更低频"的字典/配置/地区表：

```xml
<schema name="TESTDB" checkSQLschema="false" sqlMaxTimeout="100">
  <table name="orders" primaryKey="id" dataNode="dn1,dn2,dn3,dn4" rule="orders-by-id"/>
  <!-- type="global"：写广播，读任选一节点本地 JOIN -->
  <table name="dict_province" primaryKey="code" type="global"
         dataNode="dn1,dn2,dn3,dn4"/>
  <table name="currency_rate" primaryKey="code" type="global"
         dataNode="dn1,dn2,dn3,dn4"/>
</schema>
```

```text
dn1: orders + dict_province(全量) + currency_rate(全量)
dn2: orders + dict_province(全量) + currency_rate(全量)
dn3: orders + dict_province(全量) + currency_rate(全量)
dn4: orders + dict_province(全量) + currency_rate(全量)

任意分片上的 orders JOIN dict_province 都是"本地表 JOIN 本地副本"
```

## 3.2 行为矩阵

| 操作 | 行为 | 一致性风险 |
|------|------|-----------|
| INSERT | 按顺序广播到所有 dataNode | 1.6 非事务广播：中途失败 → 副本不一致 |
| UPDATE | 同上 | 同上 |
| DELETE | 同上 | 同上 |
| SELECT | 任选一个节点执行 | 无 |
| 与分片表 JOIN | 下推到各分片本地完成 | 依赖副本一致 |

**广播写的原子性问题**是全局表最大的坑：1.6 逐节点执行不包 XA，第 3 个节点失败时前 2 个已生效——**没有内置补偿与对账**。生产对策是两条：把全局表变更收敛到专用维护窗口（人工保证全成功），或应用层把全局表当"低频最终一致"数据并加定期 MD5 对账任务（比对各分片 `CHECKSUM TABLE`）。

> **行业实践**：配置类全局表在成熟团队里普遍**不放 Mycat 全局表**，而是通过配置中心下发 + 本地 Caffeine 缓存——让"字典"离开数据库层，比让数据库做广播更可控。真正适合全局表的是"必须参与 SQL JOIN 的行级字典"（省市区、币种、税率），这类表 < 10 万行、一年改不了几次。

# 4. ER 表 vs 全局表：选型对照

| 维度 | ER 分片表 | 全局表 |
|------|----------|--------|
| 数据形态 | 大表，随业务增长 | 小表，行数基本稳定（< 10 万行经验线） |
| 每分片存储 | 只存本分片子集 | 存全量副本 |
| 写放大 | 1（单分片写） | N（每分片各写一份） |
| 读 | 带键精确 / 不带键广播 | 任意节点本地 |
| JOIN 本地化条件 | 父子键分片函数等价 | 恒成立（到处都有） |
| 一致性风险 | 规则变更需父子同步 | 广播写非原子，需对账 |
| 典型表 | orders / order_items / payments | dict_province / currency_rate |

**组合用法**：`orders（分片）+ order_items（ER 子表）+ dict_city（全局）` 三者 JOIN，在任意分片上都是纯本地三连 JOIN——这是 Mycat 能给出最好 JOIN 体验的形态。

# 5. 本地 JOIN 与跨库 JOIN 的代价

| JOIN 形态 | 物理过程 | 相对耗时 | 内存占用 |
|-----------|---------|---------|---------|
| 单库本地 JOIN（含 ER、全局表下推） | MySQL 原生执行 | 1x | 无 |
| 广播式本地 JOIN（父表无分片键，各分片本地 JOIN 再拼接） | N 次本地 JOIN + 结果合并 | 2~5x | 小 |
| 中间件跨库 JOIN（1.6 兜底） | 驱动表全量拉回 + 逐行/批量查被驱动表 | 数十x 起 | 驱动表驻留，可能 OOM |

**设计考量**：

- **能用 ER 就不用广播 JOIN**：ER 是一次性声明、永久本地化；全局表写放大是持续成本
- **ER 与全局表都是"声明式优化"**，改表类型不搬数据（全局表例外——加副本要初始化数据），这是它们相对"应用层手工拼 JOIN"的核心优势
- **别指望 Mycat2 的 Calcite 拯救不对齐 JOIN**：优化器能做的分片键对齐推断有限，局面 C 依旧要靠改分片键、改查询或反范式冗余列（在 orders 里冗余 city_name）来消灭

# 6. 小结

**ER 分片与全局表是 Mycat 对分片 JOIN 的两条声明式出路**：ER 让父子表在分片键函数等价的前提下"缠绕"进同一分片，JOIN 全程本地化，代价是父子规则必须成组演进；全局表把小字典广播到每个分片换取任意位置本地 JOIN，代价是写放大与广播写非原子的副本漂移。选型记三条：大表关联大表用 ER、大表关联小表用全局表、小表且低频变更；三层 ER 链是上限，全局表变更要么进维护窗口要么配对账。所有声明都无法覆盖的不对齐 JOIN，只能靠冗余列或改查询模式在应用层消灭——Mycat 不会替你兜底，它的兜底是静默的慢。
