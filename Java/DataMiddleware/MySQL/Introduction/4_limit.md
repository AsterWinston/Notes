# 1.约束概述

约束（Constraint）是作用于表中字段上的**规则**，用于限制存储在表中的数据，保证数据库中数据的**正确性、有效性和完整性**。

约束在数据库层面的意义：

- **数据完整性保障**：防止脏数据进入系统（如年龄为负数、邮箱格式错误）
- **减少业务代码校验**：将数据规则下沉到数据库层，避免"防御式编程"在各处重复校验
- **提升查询性能**：主键和唯一约束自动创建索引，加速查找
- **维护表间关系**：外键约束确保引用完整性

> **设计争议**：是否应该在数据库层使用外键约束？互联网高并发场景下，往往**不使用物理外键**，而是由应用层保证数据一致性，原因：外键带来额外的锁开销和级联复杂性，在分库分表场景下更是无法使用。但在金融、ERP 等数据一致性要求极高的系统中，物理外键仍被广泛采用。

# 2.非空约束（NOT NULL）

限制该字段的数据不能为 NULL。

```sql
CREATE TABLE user (
  id BIGINT NOT NULL,
  username VARCHAR(50) NOT NULL
);
```

**设计考量**：

- **尽量使用 NOT NULL**：NULL 占用额外 1 字节标记位，且 NULL 的比较逻辑复杂（任何值与 NULL 比较结果都是 NULL），容易引发 BUG
- **ID 类字段必须 NOT NULL**：主键、外键、索引字段不应允许 NULL
- **可选字段**：如 `email` 允许 NULL 表示"未填写"，但也可以使用空字符串 `''` 替代（需权衡：`''` 无法区分"未填写"和"填写了空字符串"）

# 3.唯一约束（UNIQUE）

保证该字段所有数据都是唯一、不重复的（允许 NULL，且多个 NULL 互不相同）。

```sql
CREATE TABLE user (
  id BIGINT NOT NULL,
  email VARCHAR(100) UNIQUE,       -- 列级约束
  phone VARCHAR(20),
  UNIQUE KEY uk_phone (phone)      -- 表级约束（可命名，便于管理）
);

-- 添加唯一约束
ALTER TABLE user ADD UNIQUE INDEX uk_email_phone (email, phone);  -- 联合唯一
```

> **联合唯一约束**：`UNIQUE(a, b)` 表示 (a, b) 的组合值不能重复，但单独的 a 或 b 可以重复。例如 `(email='a@a.com', phone='123')` 和 `(email='a@a.com', phone='456')` 不冲突。

**唯一约束与 NULL**：MySQL 中 UNIQUE 约束允许插入多个 NULL 值（认为 NULL 彼此不相等）。

# 4.主键约束（PRIMARY KEY）

主键是一行数据的**唯一标识**，要求非空且唯一。每张表有且仅有一个主键。

```sql
-- 单列主键
CREATE TABLE user (
  id BIGINT PRIMARY KEY,
  username VARCHAR(50)
);

-- 复合主键
CREATE TABLE order_item (
  order_id BIGINT,
  product_id BIGINT,
  quantity INT,
  PRIMARY KEY (order_id, product_id)
);
```

**主键设计原则**：

| 原则 | 说明 |
|------|------|
| **唯一性** | 全局唯一标识一行 |
| **非空** | 主键不能为 NULL |
| **不可变** | 主键值一旦生成，不应修改 |
| **简短** | 主键越短，B+Tree 索引越紧凑，二级索引越小 |
| **有序** | 递增主键减少页分裂，写入性能最优 |
| **无业务含义** | 避免使用身份证号、手机号等业务字段做主键（业务规则可能变化） |

**常见主键生成策略**：

| 策略 | 优点 | 缺点 | 适用场景 |
|------|------|------|---------|
| **自增 ID** | 简单、有序、索引友好 | 单机唯一，分库需改造 | 单机 MySQL |
| **雪花算法** | 全局唯一、趋势递增 | 需额外组件，长度较长 | 分布式系统 |
| **UUID** | 全局唯一、无需中心节点 | 无序导致页分裂，36 字符太长 | 对性能不敏感的系统 |
| **数据库号段** | 一次获取一批 ID | 需额外表或中间件 | 大型分布式系统 |
| **Redis 自增** | 性能高 | 依赖外部存储 | 已有 Redis 的系统 |

```sql
-- 使用自增主键
CREATE TABLE user (
  id BIGINT UNSIGNED NOT NULL AUTO_INCREMENT,
  username VARCHAR(50),
  PRIMARY KEY (id)
);

-- 查看下一个自增值
SHOW CREATE TABLE user;
SELECT AUTO_INCREMENT FROM information_schema.tables
WHERE table_schema = 'mydb' AND table_name = 'user';
```

# 5.默认约束（DEFAULT）

保存数据时，如果未指定该字段的值，则采用默认值。

```sql
CREATE TABLE user (
  id BIGINT PRIMARY KEY AUTO_INCREMENT,
  status TINYINT NOT NULL DEFAULT 1 COMMENT '状态：0禁用，1启用',
  create_time DATETIME NOT NULL DEFAULT CURRENT_TIMESTAMP,
  update_time DATETIME NOT NULL DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP
);
```

> **CURRENT_TIMESTAMP vs NOW()**：
> - 两者在 DEFAULT 中效果相同
> - `ON UPDATE CURRENT_TIMESTAMP` 使字段在行更新时自动刷新为当前时间（非常适合 `update_time`）
> - MySQL 5.6.5 之前一张表只能有一个 `CURRENT_TIMESTAMP` 列，之后无此限制

# 6.检查约束（CHECK，8.0.16+）

保证字段值满足指定条件。MySQL 8.0.16 开始真正支持 CHECK 约束（之前版本语法通过但被忽略）。

```sql
CREATE TABLE user (
  id BIGINT PRIMARY KEY,
  username VARCHAR(50) NOT NULL,
  age TINYINT NOT NULL CHECK (age >= 0 AND age <= 150),
  email VARCHAR(100) CHECK (email REGEXP '^[a-zA-Z0-9._%+-]+@[a-zA-Z0-9.-]+\\.[a-zA-Z]{2,}$'),
  gender CHAR(1) CHECK (gender IN ('M', 'F', 'O')),
  CONSTRAINT chk_balance CHECK (balance >= 0)
);
```

**CHECK 约束的使用建议**：

- 用于简单的值域校验（如 age >= 0）
- 复杂业务规则（如"状态为已发货的订单不允许修改金额"）建议在应用层或触发器中处理
- 生产环境避免过于复杂的正则表达式 CHECK，影响写入性能

# 7.外键约束（FOREIGN KEY）

外键用来让两张表的数据之间建立连接，保证数据的**一致性和完整性**。

## 7.1添加外键

### 建表时添加

```sql
CREATE TABLE department (
  id BIGINT PRIMARY KEY AUTO_INCREMENT,
  name VARCHAR(50) NOT NULL
) ENGINE=InnoDB;

CREATE TABLE employee (
  id BIGINT PRIMARY KEY AUTO_INCREMENT,
  name VARCHAR(50) NOT NULL,
  dept_id BIGINT NOT NULL,
  CONSTRAINT fk_emp_dept FOREIGN KEY (dept_id)
    REFERENCES department(id)
) ENGINE=InnoDB;
```

### 后期添加

```sql
ALTER TABLE employee
ADD CONSTRAINT fk_emp_dept
FOREIGN KEY (dept_id) REFERENCES department(id);
```

### 删除外键

```sql
ALTER TABLE employee DROP FOREIGN KEY fk_emp_dept;
```

## 7.2删除/更新行为（级联策略）

当父表（被引用表）的记录发生删除或更新时，子表的外键数据如何处理：

| 行为 | 说明 | 适用场景 |
|------|------|---------|
| **NO ACTION** | 拒绝父表的删除/更新操作（与 RESTRICT 行为一致） | 默认行为 |
| **RESTRICT** | 与 NO ACTION 行为相同，拒绝父表操作 | 父表记录有关联子表数据时不允许删除 |
| **CASCADE** | 父表变更时，子表同步删除或更新 | 订单明细随订单一起删除 |
| **SET NULL** | 父表变更时，子表外键字段设为 NULL（要求外键允许 NULL） | 部门被删除，员工变为"未分配" |
| **SET DEFAULT** | 父表变更时，子表外键设为默认值（**InnoDB 不支持**） | — |

```sql
-- 级联删除：删除部门时，部门下所有员工也删除
ALTER TABLE employee
ADD CONSTRAINT fk_emp_dept
FOREIGN KEY (dept_id) REFERENCES department(id)
ON DELETE CASCADE ON UPDATE CASCADE;

-- SET NULL：删除部门时，员工 dept_id 设为 NULL
ALTER TABLE employee
ADD CONSTRAINT fk_emp_dept
FOREIGN KEY (dept_id) REFERENCES department(id)
ON DELETE SET NULL ON UPDATE CASCADE;
```

## 7.3外键的使用建议

**使用外键的场景**：
- 数据一致性要求极高的系统（金融、ERP）
- 数据量较小（百万级以内），外键带来的额外开销可接受
- 单机 MySQL，不需要分库分表

**不使用外键的场景（推荐）**：
- 互联网高并发系统：外键在 DML 时需要检查父表，带来额外的共享锁
- 分库分表系统：无法跨库建立外键
- 历史数据归档：外键限制使数据操作变复杂
- 即使用了外键，也应同时在应用层做好数据校验（防御式设计）

> **行业实践**：大多数互联网公司不使用物理外键，而是通过**应用层代码**维护数据一致性。外键关系通过数据字典/ER 图记录，代码 review 时关注关联数据的操作完整性。DBA 也会定期扫描"孤儿数据"（orphan data）来发现 BUG。

# 8.约束管理

```sql
-- 查看表的约束信息
SELECT
  CONSTRAINT_NAME,
  CONSTRAINT_TYPE,
  TABLE_NAME
FROM information_schema.TABLE_CONSTRAINTS
WHERE TABLE_SCHEMA = 'mydb' AND TABLE_NAME = 'user';

-- 删除主键（注意：AUTO_INCREMENT 字段不允许直接删除主键）
ALTER TABLE user DROP PRIMARY KEY;

-- 删除唯一约束
ALTER TABLE user DROP INDEX uk_email;

-- 删除检查约束
ALTER TABLE user DROP CONSTRAINT chk_age;

-- 禁用/启用外键检查（数据迁移时常用）
SET FOREIGN_KEY_CHECKS = 0;  -- 禁用
-- ... 执行数据操作 ...
SET FOREIGN_KEY_CHECKS = 1;  -- 启用
```
