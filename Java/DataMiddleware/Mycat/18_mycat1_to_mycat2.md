# 1. 为什么要迁移，以及先回答"要不要迁移"

**Mycat 1.6.x 的维护状态是迁移决策的第一推动力**：1.6.7.x 长期停在 bug-fix 缓滞状态，对新 MySQL 版本（8.0 的 caching_sha2_password、窗口函数）、对云原生部署形态的支持都停留在"能用但难受"的程度；SQL 解析依赖的 Druid Parser 也早已不再积极演进。2.x 用 Netty 重写 I/O、用 Calcite 重写 SQL 层、用 JSON 重写配置模型，等于把三个最拖累 1.x 的债务一次清算。

但迁移是高风险工程，动手前先把问题倒过来问：**你的痛点 2.x 真的解决吗？**

| 现状痛点 | 是否必须迁 2.x |
|---------|---------------|
| 需要 JOIN/子查询等复杂 SQL 语义 | 是，1.x 基本无解 |
| 需要不停机改配置（reload） | 是，1.x 的 reload 覆盖窄且不可靠 |
| 只是想要更高吞吐 | 不一定，1.x NIO 调优后差距不大，瓶颈多在后端 |
| MySQL 8.0 兼容性 | 是，1.x 认证协议兼容问题绕不开 |
| 现有 SQL 全是精确分片单表 CRUD | 不急，迁移收益低成本高 |

> **注意**：2.x 不是 1.x 的"平滑升级版"，而是**重写产品**——配置文件不兼容、管理命令不兼容、部分 SQL 改写行为不同、社区实践积累远少于 1.x。把迁移当"换个版本号启动"的团队，第一个月就会被不兼容清单教育。评估结论若是"要迁"，则按第 6 节的灰度路径走，全程保留回退能力。

# 2. 架构差异

| 维度 | Mycat 1.6.x | Mycat 2.x |
|------|------------|-----------|
| **I/O 模型** | 自研 NIO Reactor（processor/backend/business 线程组） | Netty 事件循环，线程模型简化 |
| **SQL 解析** | Druid Parser + 字符串级手工改写 | Apache Calcite，计划树级优化（RBO 为主） |
| **配置文件** | schema.xml + server.xml + rule.xml（XML） | mycat.yml + datasources.json + schemas.json + clusters.json + users.json（JSON） |
| **读写分离** | schema.xml 中 dataHost 的 balance/writeType | clusters.json 独立集群模型（MASTER_SLAVE 类型） |
| **分片算法** | rule.xml 定义、表引用 rule 名 | schemas.json 内联 function 定义 |
| **事务** | 弱 XA 支持 | 原生 XA 支持 |
| **动态配置** | 不支持，需重启 | 支持热加载 |
| **管理端口** | 9066（`reload @@config_all` 风格命令） | 9066（`/*!mycat:xxx*/` 指令风格） |
| **心跳** | dataHost 级 `<heartbeat>` | 数据源级 heartbeat 配置 |
| **ZK 集群模式** | cluster.xml 内置同步 | 配置中心（ZK/Nacos）模式 |

三组差异展开成三段散文：

**I/O 层**：1.6.x 的 Reactor 模型是项目早期自研——processor 线程 accept/read、business executor 线程池消化改写与归并、backend NIO reactor 池负责与 MySQL 的收发包，三层之间用队列与事件传递。这套设计在当时领先，但代码路径长、每层都要手工管理 ByteBuffer，长期存在堆外内存碎片与疑难连接泄漏问题。2.x 直接站上网易通道：**Netty 的事件循环 + PooledByteBufAllocator + 成熟的 ChannelPipeline 抽象**，自己不再管理 socket 细节，线程模型从"三组线程 + 队列"收敛为"event loop + 少量业务线程"。对运维的实际含义是：1.x 的 `processor` 数、`useNewBufferPool` 这类参数在 2.x 大多不存在或换了名字，历史调优经验要重新映射（对照见第 17 篇）。

**SQL 层**：1.6.x 拿到 Druid AST 后的处理方式是"**判断路由 → 把原 SQL 字符串按规则裁剪改写 → 直接下发**"，跨片归并逻辑硬编码在 Merge 实现类里，支持的形态（单表、分页、有限聚合）之外一律报"不支持"。2.x 换成 Calcite 后，SQL 先变成关系代数计划树，**投影下推、谓词下推、LIMIT 下推、聚合改写（AVG→SUM/COUNT）、子查询展开都是规则驱动的统一变换**，JOIN 与子查询因此获得数量级的支持度提升。排错体验也同步换轨：1.6.x 用 `explain select ...` 看到"改写后的下发 SQL"，2.x 用 EXPLAIN 看到"计划树"（见第 11 篇）。

**配置层**：XML → JSON 不只是格式变化。1.x 的三份 XML 互相引用（schema 引用 dataNode，dataNode 引用 dataHost，table 引用 rule.xml 里的规则名），改一个概念要在三个文件里跳；2.x 按职责拆成正交文件：**datasources.json 管物理连接、clusters.json 管主从/多主拓扑、schemas.json 管逻辑表与分片规则、users.json 管账号**，引用链变短且配合配置中心可整体版本化（第 15 篇）。

# 3. 配置文件映射

## 3.1 schema.xml → schemas.json

**Mycat 1.6 (schema.xml)**：

```xml
<schema name="order_db" checkSQLschema="true" sqlMaxLimit="100">
    <table name="orders" primaryKey="id"
           dataNode="dn0,dn1" rule="mod-long" />
</schema>
<dataNode name="dn0" dataHost="dh1" database="order_db_0" />
<dataNode name="dn1" dataHost="dh2" database="order_db_1" />
```

**Mycat 2.x (schemas.json)**：

```json
{
  "schemas": {
    "order_db": {
      "tables": {
        "orders": {
          "type": "sharding",
          "dataNodes": "dn0.orders,dn1.orders",
          "sharding": {
            "function": {
              "clazz": "io.mycat.router.function.PartitionByMod",
              "properties": { "columnName": "id", "count": 2 }
            }
          }
        }
      }
    }
  }
}
```

迁移要点：1.x 的 `rule="mod-long"` 是"规则名 → rule.xml 算法定义"两跳间接，2.x 把算法**内联到表定义**，一眼可见分片键与分片数；`sqlMaxLimit="100"` 这类 schema 级默认 LIMIT 保护在 2.x 的配置位置不同，务必确认新配置里等价项存在，否则原本被截断的 `SELECT *` 会突然开始拖全表。dataNode 在 2.x 里退化为"dataNode.table"寻址语法糖，物理库名映射移到 datasources/clusters 层。

## 3.2 dataHost → datasources.json + clusters.json

**Mycat 1.6 (schema.xml)**：

```xml
<dataHost name="dh1" maxCon="100" minCon="10" balance="1"
          writeType="0" dbType="mysql" dbDriver="native">
    <heartbeat>select user()</heartbeat>
    <writeHost host="master1" url="192.168.1.10:3306"
               user="root" password="root" />
    <readHost host="slave1" url="192.168.1.11:3306"
              user="root" password="root" />
</dataHost>
```

**Mycat 2.x (datasources.json + clusters.json)**：

```json
// datasources.json
{
  "prototype": {
    "name": "master1",
    "url": "jdbc:mysql://192.168.1.10:3306/order_db",
    "user": "root", "password": "root",
    "minCon": 10, "maxCon": 100
  },
  "slave1": {
    "name": "slave1",
    "url": "jdbc:mysql://192.168.1.11:3306/order_db",
    "user": "root", "password": "root"
  }
}

// clusters.json
{
  "clusters": {
    "dh1": {
      "clusterType": "MASTER_SLAVE",
      "readBalanceType": "BALANCE_ALL",
      "master": "master1",
      "replicas": ["slave1"]
    }
  }
}
```

概念对应关系必须一张表说清，这是迁移中最容易理解错位的地方：

| 1.6.x 概念/参数 | 2.x 对应 | 备注 |
|----------------|---------|------|
| dataHost | datasources（物理）+ clusters（拓扑） | 一拆二：连接属性与主从结构解耦 |
| balance="0/1/2/3" | readBalanceType 枚举 | 1.x 靠数字记忆，2.x 见名知义 |
| writeType="0" | 主库写（集群模型固定） | 2.x 无多写位图概念 |
| dbDriver="native"/"jdbc" | datasource 统一驱动模型 | 1.x native/jdbc 双轨在 2.x 取消 |
| `<heartbeat>` + heartbeat-frequency | datasource.heartbeat.* | 周期与 SQL 都进 JSON 结构 |
| writeHost 下挂多个从（层次） | replicas 平铺数组 | 1.x 二三层写主从、2.x 集群统一声明 |
| dataHost maxCon/minCon | datasource maxCon/minCon | 语义一致，粒度从 host 到源 |

# 4. 分片算法迁移

| Mycat 1.6 | Mycat 2.x |
|-----------|-----------|
| `PartitionByMod` | `io.mycat.router.function.PartitionByMod` |
| `PartitionByLong` | `io.mycat.router.function.PartitionByRange` |
| `PartitionByMurmurHash` | `io.mycat.router.function.PartitionByMurmurHash` |
| `PartitionByDate` | `io.mycat.router.function.PartitionByDate` |
| `AutoIncrement` | `SnowflakeSequenceHandler` |

**迁移的铁律：分片函数的映射名可以换，分片的"结果"必须一个都不能变**。1.x 的 `PartitionByLong` 是"预定义区间范围"算法，2.x 的 `PartitionByRange` 语义相近但区间表达方式不同（sharePath 目录分片 vs 显式 range 配置），必须逐值对拍：把生产分片键的样本集（覆盖每个分片边界 ±1）同时喂给 1.x 路由与 2.x 新配置，**逐条比对目标分片**，任何一条不一致都可能意味着存量数据在 2.x 下读不到。MurmurHash 这类看似同名的实现，虚拟桶数量、seed 参数默认值也可能不同——"同名不同实现"是分片迁移最大的暗坑，对拍不是保险程序而是唯一验收标准。

# 5. SQL 兼容性变更

| 功能 | Mycat 1.6 | Mycat 2.x |
|------|----------|-----------|
| JOIN | 有限支持 | ER 表 + 全局表增强 |
| 子查询 | 部分 | 更完善（Calcite 优化） |
| 存储过程 | 部分 | 不支持（设计限制） |
| XA 事务 | 实验性 | 正式支持 |
| ORDER BY + LIMIT | 支持 | 优化（LIMIT 下推） |
| MySQL 8.0 语法 | 不支持 | 支持 |

兼容性变化**不是单向变好**，两个方向都要盘：2.x 多出来的能力（复杂 JOIN、窗口函数逐步支持）是迁移收益；1.x 勉强能用而 2.x 明确砍掉的东西（存储过程/函数透传、部分老 hint 语法、以及行为差异——如 1.x 对无 LIMIT 查询自动补 sqlMaxLimit 的兜底在 2.x 默认策略不同）则可能造成**存量 SQL 从"能跑"变"报错"或从"截断"变"全量"**。迁移验证的 SQL 样本必须从 1.x 的语句统计表里捞真实的 Top 200，而不是测试用例清单。

# 6. 迁移步骤

```text
第 1 步：环境准备
  - 部署 Mycat2 实例（新端口）
  - 编写新格式配置文件

第 2 步：配置验证
  - 使用 Mycat2 连接测试库验证
  - 执行常见 SQL 确保路由正确

第 3 步：数据迁移（如分片规则变更）
  - 使用 mysqldump + 脚本重新分布数据
  - 或使用 Canal 增量同步

第 4 步：灰度上线
  - 先切换 1% 流量 → Mycat2
  - 观察无异常 → 逐步切换到 100%
  - 保留 Mycat1.6 作为回退

第 5 步：完全切换
  - 关闭 Mycat1.6
  - 监控 Mycat2 稳定性
```

第 4 步展开成可执行的灰度设计：**推荐的切流单位是"应用 × 流量比例"，通道是前端负载均衡或应用连接池多 URL**——让 A 组应用先只读流量指向 Mycat2 实例，跑通一致性比对（同一查询新旧双发、结果 diff），再放写流量，再扩到 B 组。1.x/2.x **可以长期并存**：它们只是同一批 MySQL 前面的两个代理，读写路由到同样的分片上不存在冲突；真正要防的是**并存期间配置漂移**——扩容、改规则这类元数据变更必须在两套配置中同步执行，建议用同一份源数据生成两套配置文件（配置即代码），变更单同时覆盖两版。**回滚**在灰度期是"把连接指回 1.x"这一动作，秒级完成；完全切换（第 5 步）之后回滚成本陡增，所以关闭 1.x 之前留至少一个完整业务周期的双跑观察。

# 7. 不兼容项

| 项目 | Mycat 1.6 | Mycat 2.x | 应对 |
|------|----------|-----------|------|
| ZK 配置中心 | 内置 | 需配置 | 启用 Nacos/ZK 配置中心 |
| 管理命令 | `show @@datasource` | `/*!mycat:showDataSources*/` | 更新运维脚本 |
| 全局表 | `type="global"` | `"type": "global"` | 配置迁移 |
| 自增 ID | `autoIncrement="true"` | `"autoIncrement": true` | 配置迁移 |
| 日志格式 | 自定义 | 标准化 | 更新日志收集 |

清单之外补三项常被漏掉的：**① 全局序列表结构**——1.x 的 `MYCAT_SEQUENCE` + `mycat_seq_ranges/mycat_seq_current_value` 三表模型与 2.x 的序列实现不同，切换存量业务 ID 段需要显式设置起始值（避免新旧序列重叠，见第 7 篇全局序列），这是迁移中唯一可能产生**数据级事故**的点，必须专项验证；**② 客户端 jar 与连接参数**——2.x 握手行为对老版本 Connector/J 的兼容性不同（尤其 8.0 认证插件），应用统一升级驱动版本并回归连接池参数；**③ hint 注释语法**——`/*!mycat:db_type=master*/` 类路由 hint 在两代间的解析严格程度不同，依赖 hint 的代码要逐个回归。

> **行业实践**：迁移项目的时间预算普遍低估，经验配比是**配置翻译与 SQL 回归约占 30%、双跑灰度与差异修复占 50%、监控/脚本/文档改造占 20%**。验收口径不要定"跑通了测试用例"，而定"生产流量在 2.x 上连续 N 周，慢 SQL/错误率/路由分布三曲线与 1.x 基线无劣化"。

# 8. 小结

| 要点 | 结论 |
|------|------|
| 迁移动因 | 1.6.x 停滞的解析器/I/O/配置模型；但纯 CRUD 场景收益有限，先评估再动手 |
| 三大重写 | Druid→Calcite（计划级优化）、自研 NIO→Netty、XML→JSON+配置中心 |
| 分片迁移铁律 | 配置映射名可变、路由结果不可变，同名算法也要逐值对拍 |
| SQL 兼容 | 双向盘点：新能力是收益，被砍项与默认行为差异是风险，用生产 Top SQL 回归 |
| 迁移路径 | 双代理并存 + 按应用灰度 + 结果 diff；序列表切换是唯一数据级风险点 |

下一篇（19 最佳实践）把两代产品通用的工程纪律沉淀成清单。
