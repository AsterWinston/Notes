
# 1. 概述

DistSQL（Distributed SQL）是 ShardingSphere 的**在线运维语言**，类 SQL 语法，支持在不重启的情况下动态管理数据源、规则、权限。

```
DistSQL 三大类：
  ┌──────────────────────────────────────┐
  │ RDL (Resource & Rule Definition)     │
  │  数据源注册、规则创建、表定义           │
  ├──────────────────────────────────────┤
  │ RQL (Resource & Rule Query)          │
  │  查询数据源状态、规则配置              │
  ├──────────────────────────────────────┤
  │ RAL (Resource & Rule Administration) │
  │  管理操作：熔断、禁用、迁移            │
  └──────────────────────────────────────┘
```

# 2. RDL — 资源与规则定义

## 2.1 注册存储单元

```sql
-- 注册数据源
REGISTER STORAGE UNIT ds0 (
    URL="jdbc:mysql://127.0.0.1:3306/demo_ds_0?serverTimezone=UTC",
    USER="root",
    PASSWORD="root",
    PROPERTIES("maximumPoolSize"=10)
);

REGISTER STORAGE UNIT ds1 (
    URL="jdbc:mysql://127.0.0.1:3306/demo_ds_1?serverTimezone=UTC",
    USER="root",
    PASSWORD="root"
);
```

## 2.2 创建分片规则

```sql
-- 创建分片表规则（一站式）
CREATE SHARDING TABLE RULE t_order (
    DATANODES("ds${0..1}.t_order_${0..1}"),
    DATABASE_STRATEGY(TYPE="standard", SHARDING_COLUMN=user_id,
        SHARDING_ALGORITHM(TYPE(NAME="inline",
            PROPERTIES("algorithm-expression"="ds${user_id % 2}")))),
    TABLE_STRATEGY(TYPE="standard", SHARDING_COLUMN=order_id,
        SHARDING_ALGORITHM(TYPE(NAME="inline",
            PROPERTIES("algorithm-expression"="t_order_${order_id % 2}"))))
);
```

## 2.3 创建读写分离规则

```sql
CREATE READWRITE_SPLITTING RULE readwrite_ds (
    WRITE_STORAGE_UNIT=ds_master,
    READ_STORAGE_UNITS(ds_slave0, ds_slave1),
    TYPE(NAME="static")
);
```

## 2.4 创建加密规则

```sql
CREATE ENCRYPT RULE t_user (
    COLUMNS(
        (NAME=user_name, CIPHER=user_name_cipher, ENCRYPT_ALGORITHM(TYPE(NAME="AES",
            PROPERTIES("aes-key-value"="1234567890abcdef")))),
        (NAME=phone, CIPHER=phone_cipher, ENCRYPT_ALGORITHM(TYPE(NAME="AES",
            PROPERTIES("aes-key-value"="1234567890abcdef"))))
    )
);
```

# 3. RQL — 查询

```sql
-- 查询所有存储单元
SHOW STORAGE UNITS;

-- 查询分片表规则
SHOW SHARDING TABLE RULES;

-- 查询指定表的分片规则
SHOW SHARDING TABLE RULE t_order;

-- 查询未使用的存储单元
SHOW UNUSED STORAGE UNITS;

-- 查询读写分离规则
SHOW READWRITE_SPLITTING RULES;

-- 查询加密规则
SHOW ENCRYPT RULES;
```

# 4. RAL — 管理操作

```sql
-- 禁用/启用从库（从读负载中摘除/恢复）
DISABLE READWRITE_SPLITTING READ ds_slave0 FROM readwrite_ds;
ENABLE READWRITE_SPLITTING READ ds_slave0 FROM readwrite_ds;

-- 禁用/启用存储单元
DISABLE STORAGE UNIT ds0;
ENABLE STORAGE UNIT ds0;

-- 刷新表元数据
REFRESH TABLE METADATA;

-- 显示 SQL 解析结果
PARSE SELECT * FROM t_order WHERE order_id = 1;

-- 预览 SQL 路由
PREVIEW SELECT * FROM t_order;

-- 导出当前配置
EXPORT DATABASE CONFIGURATION;
```

# 5. 在线扩容流程

```sql
-- 1. 注册新存储单元
REGISTER STORAGE UNIT ds2 (
    URL="jdbc:mysql://127.0.0.1:3306/demo_ds_2",
    USER="root", PASSWORD="root"
);

-- 2. 迁移数据（结合 ElasticJob）
MIGRATE TABLE ds0.t_order_0 INTO ds2.t_order_0;

-- 3. 修改分片规则（追加新数据节点）
ALTER SHARDING TABLE RULE t_order (
    DATANODES("ds${0..2}.t_order_${0..1}")
);

-- 4. 验证路由
PREVIEW SELECT * FROM t_order WHERE user_id = 2;
```

# 6. DistSQL vs YAML 配置

| 维度 | DistSQL | YAML |
|------|---------|------|
| 在线变更 | ✓ | ✗（需重启） |
| 版本管理 | 脚本化 | Git 管理 |
| 学习成本 | 低（类 SQL） | 中（YAML 语法） |
| 适用场景 | 运维管理 | 初始部署 |
| 自动化 | ✓ 可脚本化 | 中 |

# 7. Best Practices

1. **初始配置用 YAML**：启动时通过 YAML 加载基础配置
2. **运维变更用 DistSQL**：日常管理使用 DistSQL，无需重启
3. **DistSQL 脚本化管理**：将 DistSQL 脚本纳入 Git，变更可追溯
4. **Export 备份**：变更前 `EXPORT DATABASE CONFIGURATION` 备份当前配置
5. **分步执行**：重大变更分步执行，每步验证路由正确性
