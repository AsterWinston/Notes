DistSQL（Distributed SQL）是 ShardingSphere 的**在线运维语言**：用类 SQL 语法管理存储单元、规则、元数据与集群状态，配合 Cluster 模式的元数据仓库做到"一处执行、全集群生效"，不需要重启、不需要改文件下发。它主要服务于 **Proxy**（JDBC 形态也能用 `shardingsphere-parser-sql-*` 做离线解析，但治理动作以 Proxy 为管理面）。本篇按 RDL/RQL/RAL/RUL 四类语法给出可粘贴实例、与 YAML 配置的分工边界、权限管理方式，全部对齐 5.5.x 语法。

# 1. 概述与分类

```
DistSQL 四大类：
  ┌──────────────────────────────────────────────┐
  │ RDL (Resource & Rule Definition Language)    │ 增删改资源与规则
  │   REGISTER / CREATE / ALTER / DROP            │
  ├──────────────────────────────────────────────┤
  │ RQL (Resource & Rule Query Language)         │ 只读查询
  │   SHOW / COUNT                                │
  ├────────────────────────────────────────────┤
  │ RAL (Resource & Rule Administration Language)│ 管理与控制
  │   ENABLE / DISABLE / MIGRATE / LOCK / REFRESH │
  ├──────────────────────────────────────────────┤
  │ RUL (Rule Understanding Language)            │ SQL 解析辅助
  │   PARSE / FORMAT / PREVIEW                    │
  └──────────────────────────────────────────────┘
```

分类动机是**权限与审计边界**：RDL/RAL 是写操作（改变集群状态），RQL 是只读观测，RUL 是无副作用的调试工具。运维平台按四类分别授权，比"给 Proxy 一个 root"安全得多。语法层面 DistSQL 复用内核的 ANTLR 解析——与处理业务 SQL 是同一套引擎，这也是为什么 5.x 要先重写解析器才有 DistSQL。

**前置动作**（Proxy 上所有 DistSQL 的开场）：

```sql
-- 登录 Proxy 后先建逻辑库、切上下文
CREATE DATABASE sharding_db;
USE sharding_db;
```

> **注意**：DistSQL 有**版本兼容断层**。5.2 之前是 `ADD RESOURCE` / `CREATE SHARDING RULE` / `DISABLE READWRITE_SPLITTING READ xxx FROM yyy`；5.3+ 改为 `REGISTER STORAGE UNIT`，熔断类 RAL 统一收敛到 5.5 的 `ALTER ... RULE ... DISABLE/ENABLE` 形态。搜到的教程语法要先用 `SHOW PLUGINS` / 官方 5.5 文档比对，别直接执行。

# 2. RDL — 资源与规则定义

## 2.1 注册存储单元

```sql
-- HOST/PORT/DB 方式（默认 HikariCP）
REGISTER STORAGE UNIT ds0 (
    HOST="127.0.0.1", PORT=3306, DB="demo_ds_0",
    USER="root", PASSWORD="root",
    PROPERTIES("maximumPoolSize"=10, "idleTimeout"=30000)
);

-- URL 方式（复杂连接参数时用）
REGISTER STORAGE UNIT ds1 (
    URL="jdbc:mysql://127.0.0.1:3306/demo_ds_1?serverTimezone=UTC&useSSL=false",
    USER="root", PASSWORD="root"
);

-- 注册时校验账号权限（5.5）：SELECT / XA / PIPELINE / NONE
REGISTER STORAGE UNIT ds2 (
    HOST="127.0.0.1", PORT=3306, DB="demo_ds_2",
    USER="root", PASSWORD="root"
), CHECK_PRIVILEGES=SELECT,XA;
```

要点：**注册即建连接池**，连通性失败会直接报错回滚；存储单元名大小写敏感、只允许字母数字下划线且以字母开头；`PROPERTIES` 的 key 就是 HikariCP 原生属性名。`XA` 权限用于校验账号是否可执行 XA 语句（为分布式事务预热），`PIPELINE` 校验迁移所需权限。

## 2.2 创建分片规则

标准分片（DATANODES 行表达式 + 显式策略）与自动分片（STORAGE_UNITS + 单一分片列）两种形态：

```sql
-- 标准分片表规则：分库 user_id + 分表 order_id + 雪花主键
CREATE SHARDING TABLE RULE t_order (
    DATANODES("ds${0..1}.t_order_${0..1}"),
    DATABASE_STRATEGY(TYPE="standard", SHARDING_COLUMN=user_id,
        SHARDING_ALGORITHM(TYPE(NAME="inline",
            PROPERTIES("algorithm-expression"="ds${user_id % 2}")))),
    TABLE_STRATEGY(TYPE="standard", SHARDING_COLUMN=order_id,
        SHARDING_ALGORITHM(TYPE(NAME="inline",
            PROPERTIES("algorithm-expression"="t_order_${order_id % 2}")))),
    KEY_GENERATE_STRATEGY(COLUMN=order_id, TYPE(NAME="snowflake"))
);

-- 自动分片表规则：hash_mod，分片键可不出现在 SQL 中
CREATE SHARDING TABLE RULE t_account (
    STORAGE_UNITS(ds0,ds1),
    SHARDING_COLUMN=account_id,
    TYPE(NAME="hash_mod", PROPERTIES("sharding-count"="4")),
    KEY_GENERATE_STRATEGY(COLUMN=account_id, TYPE(NAME="snowflake"))
);

-- 绑定表（主子表同分片算法）
CREATE SHARDING TABLE REFERENCE RULE t_order_ref (t_order, t_order_item);

-- 广播表
CREATE BROADCAST TABLE RULE t_config;
```

## 2.3 读写分离 / 加密 / 影子规则

```sql
-- 读写分离组：TYPE 指负载均衡算法（round_robin/random/weight）
CREATE READWRITE_SPLITTING RULE readwrite_ds (
    WRITE_STORAGE_UNIT=ds_master,
    READ_STORAGE_UNITS(ds_slave0, ds_slave1),
    TRANSACTIONAL_READ_QUERY_STRATEGY=PRIMARY,
    TYPE(NAME="round_robin")
);

-- 加密规则：密文列 + 等值辅助列 + 模糊辅助列
CREATE ENCRYPT RULE t_user (
    COLUMNS(
        (NAME=phone, CIPHER=phone_cipher,
            ENCRYPT_ALGORITHM(TYPE(NAME="AES",
                PROPERTIES("aes-key-value"="1234567890abcdef",
                           "digest-algorithm-name"="SHA-1")))),
        (NAME=email, CIPHER=email_cipher, ASSISTED_QUERY=email_query,
            ENCRYPT_ALGORITHM(TYPE(NAME="AES",
                PROPERTIES("aes-key-value"="1234567890abcdef"))),
            ASSISTED_QUERY_ALGORITHM(TYPE(NAME="MD5")))
    )
);

-- 影子库规则：SOURCE=生产存储单元，SHADOW=影子存储单元，逐表指定算法
CREATE SHADOW RULE shadow_rule (
    SOURCE=demo_ds, SHADOW=demo_ds_shadow,
    t_order(TYPE(NAME="SQL_HINT")),
    t_order_item(TYPE(NAME="VALUE_MATCH",
        PROPERTIES("operation"="insert", "column"="user_id", "value"="1")))
);
```

> **设计考量**：RDL 建出来的规则**立即写入元数据仓库并广播**，与 YAML 启动配置等价生效，但后续修改走的是 DistSQL——两套配置系统的持久化位置不同（YAML 在文件/启动参数，DistSQL 在注册中心），混用会让"当前生效配置"变成罗生门。规则分工见第 6 节。

# 3. RQL — 查询

```sql
SHOW STORAGE UNITS;                    -- 全部存储单元
SHOW SHARDING TABLE RULES;             -- 所有分片规则
SHOW SHARDING TABLE RULE t_order;      -- 指定表规则
SHOW SHARDING TABLE NODES t_order;     -- 数据节点分布（排错利器）
SHOW SHARDING ALGORITHMS;              -- 已定义算法
SHOW UNUSED SHARDING ALGORITHMS;       -- 孤儿算法（可 DROP）
SHOW READWRITE_SPLITTING RULES;
SHOW ENCRYPT RULES;
SHOW BROADCAST TABLE RULES;
SHOW SINGLE TABLES;                    -- 已加载单表
SHOW LOGICAL TABLES;
COUNT SHARDING RULE;                   -- 规则计数
SHOW RULES USED STORAGE UNIT ds0;      -- 反向依赖：删存储单元前先查
```

`SHARDING TABLE NODES` 回答"逻辑表的每个分片到底落在哪个存储单元"——扩容后路由异常先看它；`UNUSED` 系列用于变更收尾，孤儿算法/存储单元不清理会让后续 ALTER 引用到错误对象。

# 4. RAL — 管理与控制

```sql
-- 熔断：摘除/恢复读库（5.5 语法；旧版为 DISABLE ... READ ... FROM）
ALTER READWRITE_SPLITTING RULE readwrite_ds DISABLE ds_slave0;
ALTER READWRITE_SPLITTING RULE readwrite_ds ENABLE ds_slave0;
SHOW STATUS FROM READWRITE_SPLITTING RULE readwrite_ds;

-- 计算节点管理（Proxy 集群实例）
SHOW COMPUTE NODES;   -- 列含 instance_id/instance_type/host/port/status/worker_id/labels/version
-- 下线/上线单个实例（instance_id 取自 SHOW COMPUTE NODES；当前连接实例不可 disable）
DISABLE COMPUTE NODE '3e84d33e-cb97-42f2-b6ce-f78fea0ded89';
ENABLE  COMPUTE NODE '3e84d33e-cb97-42f2-b6ce-f78fea0ded89';
-- LABEL / UNLABEL COMPUTE NODE ... —— 给实例打标做流量分组

-- 元数据刷新：后端表结构变了但元数据缓存旧
REFRESH TABLE METADATA t_order;
REFRESH DATABASE METADATA;

-- 配置导出/导入（灾备与迁移）
EXPORT DATABASE CONFIGURATION;
IMPORT DATABASE CONFIGURATION '...';

-- YAML → DistSQL 语句转换（只输出语句，不影响当前元数据）
CONVERT YAML CONFIGURATION FROM FILE '/xxx/config_sharding_db.yaml';

-- 集群锁（WRITE 排他 / READ_WRITE 读写，TIMEOUT 毫秒可选）
LOCK CLUSTER WITH WRITE;
UNLOCK CLUSTER;

-- 变量/状态查看
SHOW DIST VARIABLE;                    -- 集群拓扑+规则+属性全景
SHOW TABLE METADATA t_order;
```

`CONVERT YAML CONFIGURATION` 是 YAML→DistSQL 迁移的官方通道，把存量 `config-xxx.yaml` 转成语句列表后在 Proxy 重放，避免手工翻译规则出错。

# 5. RUL — SQL 解析与路由预览

```sql
-- 预览执行计划：逻辑 SQL → 物理 SQL（排查全路由神器）
PREVIEW SELECT * FROM t_order WHERE user_id = 1;
+------------------+--------------------------------------------+
| data_source_name | actual_sql                                 |
+------------------+--------------------------------------------+
| ds1              | SELECT * FROM t_order_1 WHERE user_id = 1 |
+------------------+--------------------------------------------+

-- 查看解析结果 AST
PARSE SELECT * FROM t_order WHERE order_id = 1;

-- 格式化 SQL
FORMAT SQL 'select * from t_order where order_id=1';
```

RUL 不改变任何东西，但它是 DistSQL 里**日常使用频率最高**的一类：规则改完先 `PREVIEW` 一条真实 SQL，确认路由宽度符合预期，再放开业务流量。5.5.x 语法为 `PREVIEW sqlStatement`（不带引号），返回 `data_source_name + actual_sql` 两列。

# 6. DistSQL vs YAML 分工

| 维度 | DistSQL | YAML |
|------|---------|------|
| 生效方式 | 在线执行，Cluster 全节点同步 | 启动加载，变更需滚动重启 |
| 持久化位置 | 元数据仓库（ZooKeeper/JDBC） | 本地文件 / classpath |
| 版本管理 | 需自管：脚本化 + EXPORT 备份 | Git 天然管理 |
| 语法覆盖 | 全功能（含 RAL/RUL 运维面） | 全功能（含 mode/authority 全局项） |
| 学习成本 | 类 SQL，低 | 结构化缩进，中 |
| 适用 | 运行时变更、应急、审计 | 初建、基线、声明式部署 |

**推荐的分工纪律**：

- **基线用 YAML**：新环境用 YAML（或 CONVERT 后的脚本）把规则一次建好，保证可重复部署。
- **变更用 DistSQL**：上线后的所有增删改走 DistSQL + 变更单，操作自然带上时间与执行者。
- **避免双写混用**：同一集群选定一条主路径。DistSQL 主路径下，YAML 只保留 `mode/authority/props` 全局骨架——这些本来也不能用 DistSQL 修改（authority 变更需重启或用 `ALTER USER` 类扩展）。
- **每次 RDL/RAL 前后各做一次 `EXPORT DATABASE CONFIGURATION`**：导出当前全量配置作为回滚依据。

# 7. 权限与安全

```yaml
# server.yaml：Proxy 接入账号体系（5.5.x）
authority:
  users:
    - user: root@%
      password: root
      admin: true                      # 管理员：拥有全部权限（含 DistSQL 写）
    - user: app
      password: app
    - user: test
      password: test
  privilege:
    type: DATABASE_PERMITTED           # 推荐；ALL_PERMITTED 已标注"未来移除"
    props:
      user-database-mappings: app@%=sharding_db, test@%=report_db
```

- 授权模型：`privilege.type` 选权限提供者——`ALL_PERMITTED`（人人全权，官方标注将在未来移除）与 `DATABASE_PERMITTED`（**推荐**，用 `props.user-database-mappings` 限定"用户→可访问逻辑库"，`*` 表示全部）；`admin: true` 的用户拥有最高权限，业务账号不给 admin。认证方式默认按前端协议自动选择（`authenticators`/`authenticationMethodName` 可显式指定 MD5/SCRAM 等）。
- **后端库账号权限与前端权限分离**：`CHECK_PRIVILEGES` 校验的是 ShardingSphere→MySQL 方向（SELECT/XA/PIPELINE），Proxy 用户权限管的是接入方方向——两层都收紧才安全。
- DistSQL 没有独立口令体系：能连 Proxy 的 admin 账号即可改集群，Proxy 管理面应走内网 + 堡垒机，并把 `SHOW DIST VARIABLE`、`EXPORT DATABASE CONFIGURATION` 的输出视为敏感物料。

# 8. 在线变更示例：完整走一遍

```sql
USE sharding_db;
LOCK CLUSTER WITH WRITE;
-- 1. 注册新从库
REGISTER STORAGE UNIT ds_slave2 (
    HOST="10.0.0.12", PORT=3306, DB="demo_ds",
    USER="root", PASSWORD="root"
);
-- 2. 修改读写分离规则加入读池
ALTER READWRITE_SPLITTING RULE readwrite_ds (
    WRITE_STORAGE_UNIT=ds_master,
    READ_STORAGE_UNITS(ds_slave0, ds_slave1, ds_slave2),
    TYPE(NAME="round_robin")
);
-- 3. 验证：规则已含新从库
SHOW READWRITE_SPLITTING RULES;
-- 4. 验证路由（业务 SQL 预览，看负载集合是否更新）
PREVIEW SELECT * FROM t_order WHERE user_id = 1;
UNLOCK CLUSTER;
```

弹性扩容的完整流程（迁移数据 + 切规则）见 10_governance 的 Migration Pipeline 章节，DistSQL 只负责其中的控制面语句。

> **行业实践**：把 DistSQL 脚本纳入 Git 并按"变更单"组织（每个 PR 一次变更，附 PREVIEW 输出截图），用 CI 连测试 Proxy 重放校验语法——这是把"运维 SQL"工程化的最低成本方案；`ALTER SHARDING TABLE RULE` 这类高危语句评审时重点看**行表达式范围是否只扩不缩**，缩了会丢路由。

# 9. 小结

| 类别 | 代表语句 | 定位 |
|------|---------|------|
| RDL | REGISTER STORAGE UNIT / CREATE SHARDING TABLE RULE / CREATE ENCRYPT RULE | 在线定义资源与规则 |
| RQL | SHOW STORAGE UNITS / SHOW SHARDING TABLE NODES / SHOW UNUSED ... | 只读观测与依赖检查 |
| RAL | ALTER ... RULE DISABLE/ENABLE / REFRESH ... METADATA / MIGRATE / LOCK CLUSTER | 治理控制与迁移 |
| RUL | PREVIEW / PARSE / FORMAT SQL | 无副作用验证 |
| 分工 | YAML 做基线，DistSQL 做变更，EXPORT 做备份 | 避免双配置系统漂移 |
| 安全 | server.yaml 按用户配 privilege + CHECK_PRIVILEGES 校验后端 | 管理面内网化 |

下一篇（10_governance）看 Cluster 模式与弹性伸缩：元数据如何同步、在线迁移如何做。
