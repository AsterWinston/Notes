# 1.数值类型

## 1.1整数类型

| 类型 | 字节 | 有符号范围 | 无符号范围 |
|------|------|-----------|-----------|
| TINYINT | 1 | -128 ~ 127 | 0 ~ 255 |
| SMALLINT | 2 | -32768 ~ 32767 | 0 ~ 65535 |
| MEDIUMINT | 3 | -8388608 ~ 8388607 | 0 ~ 16777215 |
| INT / INTEGER | 4 | -2147483648 ~ 2147483647 | 0 ~ 4294967295 |
| BIGINT | 8 | -2^63 ~ 2^63-1 | 0 ~ 2^64-1 |

### 设计建议

- **主键用 BIGINT UNSIGNED**：`BIGINT UNSIGNED AUTO_INCREMENT`，INT 最大值 21 亿在很多业务中不够用（电商订单表极易突破）
- **状态字段用 TINYINT**：如 `status TINYINT NOT NULL DEFAULT 1`
- **年龄/数量用 TINYINT/SMALLINT**：不要用 INT
- **布尔用 TINYINT(1)**：MySQL 不原生支持 BOOLEAN，`BOOLEAN` 底层映射为 `TINYINT(1)`
- **选择能满足需求的最小类型**：更小的类型 = 更紧凑的索引 = 更高的 Buffer Pool 命中率

### ZEROFILL（不推荐）

```sql
-- ZEROFILL 会自动添加 UNSIGNED，不足位数前面补 0
CREATE TABLE t (id INT(5) ZEROFILL);
INSERT INTO t VALUES (42);  -- 存储 00042

-- 注意：MySQL 8.0.17+ 已废弃 INT(N) 语法中的显示宽度（N）
-- 只保留功能含义，不再影响显示
```

## 1.2浮点数与定点数

| 类型 | 字节 | 精度 | 适用场景 |
|------|------|------|---------|
| FLOAT | 4 | 单精度（约 7 位有效数字） | 科学计算、评分（允许小误差） |
| DOUBLE | 8 | 双精度（约 15 位有效数字） | 同上 |
| DECIMAL(M, D) | 可变 | **精确**（定点数，字符串存储） | **金额、财务计算（必须！）** |

### 金额为什么不能用 FLOAT/DOUBLE

```sql
-- 浮点数的精度丢失问题
SELECT 0.1 + 0.2;                    -- 0.30000000000000004
SELECT CAST(0.1 AS FLOAT) + CAST(0.2 AS FLOAT);  -- 同上有误差

-- DECIMAL 无此问题
SELECT CAST(0.1 AS DECIMAL(10,2)) + CAST(0.2 AS DECIMAL(10,2));  -- 0.30
```

> **金融铁律**：涉及金额的字段**必须**使用 DECIMAL（或用 BIGINT 存储分为单位的整数）。

### DECIMAL 存储特点

- 每 9 个十进制数字占 4 字节
- DECIMAL(18, 2)：18 位数字，2 位小数 → 约 9 字节
- `balance DECIMAL(10, 2) COMMENT '余额（元）'` 可存储 -99,999,999.99 ~ 99,999,999.99

# 2.字符串类型

## 2.1CHAR vs VARCHAR

| 特性 | CHAR(N) | VARCHAR(N) |
|------|---------|-----------|
| **长度** | 定长，永远占用 N 个字符 | 变长，占用 = 数据 + 1~2 字节 |
| **最大长度** | 255 字符 | 65535 字节（实际受行大小限制 ~16383 字符） |
| **存储效率** | 浪费空间，但无碎片 | 节省空间，可能产生碎片 |
| **读写性能** | 极好（定长，无需计算偏移量） | 略差（变长，需前缀记录长度） |
| **适用场景** | 定长数据：MD5(32)、身份证(18)、手机号(11)、UUID、ISO 国家码(2) | 变长数据：姓名、邮箱、地址、标题 |

```sql
-- CHAR 适用：长度固定不变
password_hash CHAR(32),   -- MD5
credit_card CHAR(16),     -- 卡号固定 16 位
country_code CHAR(2),     -- ISO 国家代码
md5 CHAR(32);             -- MD5 哈希固定 32 字符

-- VARCHAR 适用：长度可变
username VARCHAR(50),
email VARCHAR(100),
bio VARCHAR(500);
```

## 2.2TEXT 和 BLOB 类型

| 类型 | 最大长度 | 说明 |
|------|---------|------|
| TINYTEXT | 255 字节 | 极少用 |
| TEXT | 65535 字节（64KB） | 文章内容、JSON 字符串 |
| MEDIUMTEXT | 16MB | 更长内容 |
| LONGTEXT | 4GB | 极少用 |
| BLOB 系列 | 同上 | 二进制大对象（图片、文件） |

### TEXT/BLOB 的问题

- **不能有默认值**
- **索引受限**：只能前缀索引（`TEXT(100)`）
- **存储溢出**：数据通常在独立的外部页存储，读取需要额外 IO
- **排序临时表**：TEXT 排序时使用磁盘临时表（MEMORY 引擎不支持 TEXT）

> **替代方案**：
> - 文章内容存 MySQL TEXT 仅用于检索，展示内容从对象存储（OSS/S3）加载
> - 图片/文件不存数据库，存 OSS，数据库只存 URL
> - 大 JSON 考虑 MongoDB 等文档数据库

## 2.3ENUM和SET

```sql
-- ENUM：单选（最多 65535 个值）
CREATE TABLE user (
  gender ENUM('M', 'F', 'O') COMMENT '性别'
);

-- SET：多选（最多 64 个值）
CREATE TABLE user (
  hobbies SET('读书', '运动', '音乐', '旅行')
);
```

**ENUM 优缺点**：

| 优点 | 缺点 |
|------|------|
| 存储紧凑（1~2 字节存 65535 个值） | 不易扩展（ALTER TABLE 改 ENUM 需重建表） |
| 查询效率高 | 排序按内部数字而非字符串 |
| 强类型约束 | 可移植性差（非 SQL 标准） |

> **建议**：少用 ENUM/SET。状态字段用 `TINYINT` + 代码枚举（Java Enum），扩展更灵活。

# 3.日期时间类型

| 类型 | 字节 | 范围 | 时区感知 | 精度 |
|------|------|------|---------|------|
| DATE | 3 | 1000-01-01 ~ 9999-12-31 | ✗ | 天 |
| TIME | 3 | -838:59:59 ~ 838:59:59 | ✗ | 时分秒 |
| DATETIME | 5~8 | 1000-01-01 00:00:00 ~ 9999-12-31 23:59:59 | ✗ | 秒(默认)/微秒 |
| TIMESTAMP | 4~7 | 1970-01-01 00:00:01 ~ 2038-01-19 03:14:07 | ✓ | 秒(默认)/微秒 |
| YEAR | 1 | 1901 ~ 2155 | ✗ | 年 |

## 3.1DATETIME vs TIMESTAMP

| 特性 | DATETIME | TIMESTAMP |
|------|---------|-----------|
| **时区** | 存储时区无关，存什么显示什么 | 存储时转为 UTC，读取时转回当前时区 |
| **范围** | 1000 ~ 9999 | 1970 ~ 2038 |
| **存储空间** | 5 + 小数秒字节 | 4 + 小数秒字节 |
| **默认值** | 支持 `DEFAULT CURRENT_TIMESTAMP` | 可指定 `ON UPDATE CURRENT_TIMESTAMP` |
| **2038 问题** | 无 | 有（Unix 时间戳溢出） |

```sql
-- 推荐：业务时间用 DATETIME
create_time DATETIME NOT NULL DEFAULT CURRENT_TIMESTAMP,
update_time DATETIME NOT NULL DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP;
```

> **推荐使用 DATETIME**：
> 1. 没有 2038 年问题
> 2. 时区语义清晰（存储的是"实际时间"，而非"UTC 的时间戳"）
> 3. 如果业务跨时区，由应用层处理时区转换

## 3.2关于 2038 年问题

TIMESTAMP 使用 32 位有符号整数存储 Unix 时间戳，最大值 `2147483647` → **2038-01-19 03:14:07 UTC**，超过此时间点会溢出。

> MySQL 8.0 尚未解决此问题。保险起见，所有新表统一使用 DATETIME。

# 4.JSON类型（5.7.8+）

MySQL 5.7 引入原生的 JSON 类型，提供 JSON 校验、高效存储和索引支持。

```sql
CREATE TABLE user (
  id BIGINT PRIMARY KEY,
  attrs JSON COMMENT '扩展属性'
);

INSERT INTO user VALUES (1, '{"name": "张三", "tags": ["vip", "new"]}');

-- JSON 路径访问
SELECT attrs->>'$.name' AS name FROM user;
SELECT * FROM user WHERE JSON_CONTAINS(attrs->'$.tags', '"vip"');

-- JSON 索引（MySQL 8.0.17+ 支持多值索引）
-- 对 JSON 中的数组字段建立索引
ALTER TABLE user ADD INDEX idx_tags ((CAST(attrs->'$.tags' AS CHAR(50) ARRAY)));
```

**使用建议**：
- 适合存储**格式不确定的扩展属性**（如用户标签、商品属性等异构数据）
- 不适合存储核心业务字段（无法利用 B+Tree 索引，查询优化困难）
- 不适合大 JSON（MySQL 限制 JSON 最大 1GB，按经验几百 KB 以内）
- JSON 列不能有默认值（除了 NULL）

# 5.空间数据类型（GIS）

```sql
-- 地理坐标
CREATE TABLE store (
  id BIGINT PRIMARY KEY,
  name VARCHAR(100),
  location POINT SRID 4326,   -- WGS 84 坐标系
  SPATIAL INDEX idx_location (location)
);

INSERT INTO store VALUES (1, '门店A', ST_GeomFromText('POINT(116.404 39.915)', 4326));

-- 查询距离
SELECT name,
  ST_Distance_Sphere(location, ST_GeomFromText('POINT(116.40 39.90)', 4326)) AS distance
FROM store
WHERE ST_Distance_Sphere(location, ST_GeomFromText('POINT(116.40 39.90)', 4326)) < 5000;
```

# 6.类型选择总结

| 场景 | 推荐类型 | 反例 |
|------|---------|------|
| 主键 | BIGINT UNSIGNED AUTO_INCREMENT | VARCHAR / UUID 字符串 |
| 金额 | DECIMAL(18, 2) | FLOAT / DOUBLE |
| 手机号 | CHAR(11) | VARCHAR(20) |
| 邮箱 | VARCHAR(100) | TEXT |
| 哈希值(MD5) | CHAR(32) | VARCHAR(64) |
| 状态/类型 | TINYINT | VARCHAR(10) |
| 文章内容 | TEXT / MEDIUMTEXT | VARCHAR(65535) |
| 扩展属性 | JSON | VARCHAR 存 JSON 字符串 |
| 时间 | DATETIME(3) | TIMESTAMP |
| 性别 | TINYINT | ENUM |
| 布尔 | TINYINT(1) | CHAR(1) 存 'Y'/'N' |
| 图片/文件 | VARCHAR(255) 存 URL | BLOB |
| IP 地址 | VARBINARY(16) 或 INT UNSIGNED | VARCHAR(45) |
