# 1.字符串函数

## 1.1常用字符串函数

| 函数 | 语法 | 说明 | 示例 |
|------|------|------|------|
| CONCAT | CONCAT(S1, S2, ...) | 字符串拼接，任一参数为 NULL 则整体返回 NULL | `SELECT CONCAT('Hello', ' ', 'World'); → 'Hello World'` |
| CONCAT_WS | CONCAT_WS(sep, S1, S2, ...) | 用分隔符拼接字符串，自动跳过 NULL | `SELECT CONCAT_WS('-', '2026', '05', '17'); → '2026-05-17'` |
| LOWER / LCASE | LOWER(str) | 转为小写 | `SELECT LOWER('MySQL'); → 'mysql'` |
| UPPER / UCASE | UPPER(str) | 转为大写 | `SELECT UPPER('mysql'); → 'MYSQL'` |
| LPAD | LPAD(str, n, pad) | 左侧填充 pad 字符至长度 n | `SELECT LPAD('5', 3, '0'); → '005'` |
| RPAD | RPAD(str, n, pad) | 右侧填充 pad 字符至长度 n | `SELECT RPAD('A', 5, '*'); → 'A****'` |
| TRIM | TRIM(str) | 去掉头尾空格 | `SELECT TRIM('  hello  '); → 'hello'` |
| LTRIM | LTRIM(str) | 去掉左侧空格 | |
| RTRIM | RTRIM(str) | 去掉右侧空格 | |
| SUBSTRING | SUBSTRING(str, start, len) | 截取子串（start 从 1 开始） | `SELECT SUBSTRING('Hello World', 7, 5); → 'World'` |
| LEFT | LEFT(str, n) | 截取左侧 n 个字符 | `SELECT LEFT('MySQL', 2); → 'My'` |
| RIGHT | RIGHT(str, n) | 截取右侧 n 个字符 | `SELECT RIGHT('MySQL', 3); → 'SQL'` |
| LENGTH | LENGTH(str) | 返回**字节数**（受字符集影响） | `SELECT LENGTH('你好'); -- utf8mb4: 6` |
| CHAR_LENGTH | CHAR_LENGTH(str) | 返回**字符数**（推荐） | `SELECT CHAR_LENGTH('你好'); -- 2` |
| REPLACE | REPLACE(str, from_str, to_str) | 字符串替换 | `SELECT REPLACE('Hello Wello', 'ello', 'i'); → 'Hi Wi'` |
| REVERSE | REVERSE(str) | 反转字符串 | `SELECT REVERSE('abc'); → 'cba'` |
| INSTR | INSTR(str, substr) | 返回子串首次出现位置（1-based） | `SELECT INSTR('Hello', 'll'); → 3` |
| LOCATE | LOCATE(substr, str, pos) | 功能同 INSTR，可指定起始位置 | `SELECT LOCATE('o', 'Hello World', 5); → 8` |
| REPEAT | REPEAT(str, count) | 重复字符串 | `SELECT REPEAT('ab', 3); → 'ababab'` |
| SPACE | SPACE(n) | 生成 n 个空格 | `SELECT CONCAT('A', SPACE(3), 'B'); → 'A   B'` |

## 1.2关键函数详解

### 1.2.1CONCAT vs CONCAT_WS

```sql
-- CONCAT：任一参数为 NULL，结果即为 NULL
SELECT CONCAT('MySQL', NULL, '8.0');  -- NULL

-- CONCAT_WS：自动跳过 NULL
SELECT CONCAT_WS('-', '2026', NULL, '17');  -- '2026-17'

-- 实际应用：生成全名
SELECT CONCAT_WS(' ', first_name, middle_name, last_name) AS full_name FROM user;
```

### 1.2.2SUBSTRING详解

```sql
-- 从位置 1 开始取 5 个字符
SELECT SUBSTRING('Hello World', 1, 5);  -- 'Hello'

-- 省略 len，截取到末尾
SELECT SUBSTRING('Hello World', 7);     -- 'World'

-- 负数 start（从末尾往前数）
SELECT SUBSTRING('Hello World', -5, 5); -- 'World'
```

### 1.2.3字符串截取实战——提取域名

```sql
-- 从邮箱中提取域名
SELECT SUBSTRING(email, INSTR(email, '@') + 1) AS domain FROM user;

-- 或使用 SUBSTRING_INDEX
SELECT SUBSTRING_INDEX(email, '@', -1) AS domain FROM user;
```

> `SUBSTRING_INDEX(str, delim, count)`：按分隔符截取。count 正数从左数，负数从右数。`SUBSTRING_INDEX('a.b.c.d', '.', 2)` → `'a.b'`，`SUBSTRING_INDEX('a.b.c.d', '.', -1)` → `'d'`。

# 2.数值函数

| 函数 | 语法 | 说明 |
|------|------|------|
| CEIL / CEILING | CEIL(x) | 向上取整 |
| FLOOR | FLOOR(x) | 向下取整 |
| MOD | MOD(x, y) | 取模（`x % y` 等价） |
| RAND | RAND([seed]) | 返回 0~1 之间的随机浮点数 |
| ROUND | ROUND(x, y) | 四舍五入，保留 y 位小数 |
| TRUNCATE | TRUNCATE(x, y) | 截断到 y 位小数（不四舍五入） |
| ABS | ABS(x) | 绝对值 |
| POW / POWER | POW(x, y) | x 的 y 次方 |
| SQRT | SQRT(x) | 平方根 |
| GREATEST | GREATEST(v1, v2, ...) | 最大值 |
| LEAST | LEAST(v1, v2, ...) | 最小值 |
| SIGN | SIGN(x) | 符号（-1, 0, 1） |

```sql
-- 四舍五入 vs 截断
SELECT ROUND(3.14159, 2);     -- 3.14
SELECT TRUNCATE(3.14159, 2);  -- 3.14
SELECT ROUND(3.145, 2);       -- 3.15
SELECT TRUNCATE(3.145, 2);    -- 3.14

-- 生成随机验证码（6 位数字）
SELECT LPAD(FLOOR(RAND() * 1000000), 6, '0');

-- 计算分页总数
SELECT CEIL(COUNT(*) / 10.0) AS total_pages FROM user;
```

# 3.日期函数

## 3.1基本日期函数

| 函数 | 语法 | 说明 |
|------|------|------|
| CURDATE / CURRENT_DATE | CURDATE() | 当前日期（不含时间） |
| CURTIME / CURRENT_TIME | CURTIME() | 当前时间（不含日期） |
| NOW / CURRENT_TIMESTAMP | NOW() | 当前日期+时间 |
| SYSDATE | SYSDATE() | 当前日期+时间（与 NOW 的细微区别见下方） |
| UNIX_TIMESTAMP | UNIX_TIMESTAMP([date]) | 转为 Unix 时间戳 |
| FROM_UNIXTIME | FROM_UNIXTIME(ts [, format]) | Unix 时间戳转为日期 |
| YEAR / MONTH / DAY | YEAR(date) | 提取年月日 |
| QUARTER | QUARTER(date) | 提取季度（1~4） |
| WEEK | WEEK(date [, mode]) | 提取周数 |
| DAYOFWEEK / DAYOFYEAR | DAYOFWEEK(date) | 星期几（1=周日）/ 年内第几天 |
| DATE_ADD / DATE_SUB | DATE_ADD(date, INTERVAL expr unit) | 日期加减 |
| DATEDIFF | DATEDIFF(date1, date2) | 两日期相差天数（date1 - date2） |
| TIMESTAMPDIFF | TIMESTAMPDIFF(unit, dt1, dt2) | 两日期差值（可指定单位） |
| DATE_FORMAT | DATE_FORMAT(date, format) | 格式化日期 |
| STR_TO_DATE | STR_TO_DATE(str, format) | 字符串转日期 |
| LAST_DAY | LAST_DAY(date) | 当月最后一天 |

## 3.2关键函数详解

### 3.2.1NOW vs SYSDATE

```sql
-- NOW() 在语句开始时获取时间（整个语句中不变）
SELECT NOW(), SLEEP(2), NOW();  -- 两个时间相同

-- SYSDATE() 在函数执行时获取时间（动态）
SELECT SYSDATE(), SLEEP(2), SYSDATE();  -- 两个时间不同（差 2 秒）
```

> **工程意义**：表中 `create_time` 用 `DEFAULT CURRENT_TIMESTAMP`（等价于 NOW()），`update_time` 用 `ON UPDATE CURRENT_TIMESTAMP`，两者都基于语句开始时间。SYSDATE 通常仅用于特殊监控场景。

### 3.2.2日期加减

```sql
-- 加 7 天
SELECT DATE_ADD(NOW(), INTERVAL 7 DAY);

-- 减 1 个月
SELECT DATE_SUB(NOW(), INTERVAL 1 MONTH);

-- 简写方式（MySQL 特有）
SELECT NOW() + INTERVAL 1 MONTH;

-- 常用 INTERVAL 单位
-- MICROSECOND, SECOND, MINUTE, HOUR, DAY, WEEK, MONTH, QUARTER, YEAR
-- HOUR_MINUTE（如 '2:30'）, DAY_HOUR（如 '3 12'）
```

### 3.2.3日期差值

```sql
-- 天数差
SELECT DATEDIFF('2026-06-01', '2026-05-17');  -- 15

-- 小时差 / 月差（更灵活）
SELECT TIMESTAMPDIFF(HOUR, '2026-05-17 08:00:00', '2026-05-17 18:00:00');  -- 10
SELECT TIMESTAMPDIFF(YEAR, birth_date, NOW()) AS age FROM user;
```

### 3.2.4DATE_FORMAT格式化

```sql
-- 格式说明符
-- %Y：4 位年  %y：2 位年  %m：月(01-12)  %d：日(01-31)
-- %H：24 小时  %h：12 小时  %i：分钟  %s：秒
-- %W：星期全名  %M：月份全名

SELECT DATE_FORMAT(NOW(), '%Y-%m-%d %H:%i:%s');  -- '2026-05-17 14:30:00'
SELECT DATE_FORMAT(NOW(), '%Y年%m月%d日');        -- '2026年05月17日'
```

# 4.流程控制函数

## 4.1IF

```sql
SELECT IF(age >= 18, '成年', '未成年') AS category FROM user;
```

## 4.2IFNULL

```sql
-- 如果 email 为空，显示 "未填写"
SELECT username, IFNULL(email, '未填写') AS email FROM user;

-- IFNULL 与 COALESCE 的区别
-- IFNULL 只接受两个参数
-- COALESCE 接受多个参数，返回第一个非 NULL 值
SELECT COALESCE(mobile, phone, email, '无联系方式') AS contact FROM user;
```

## 4.3NULLIF

```sql
-- 两个参数相等则返回 NULL，否则返回参数一
-- 常用于防止除零错误
SELECT amount / NULLIF(quantity, 0) AS unit_price FROM order_item;
```

## 4.4CASE WHEN（两种写法）

```sql
-- 写法一：搜索 CASE（最常用）
SELECT
  username,
  score,
  CASE
    WHEN score >= 90 THEN '优秀'
    WHEN score >= 80 THEN '良好'
    WHEN score >= 60 THEN '及格'
    ELSE '不及格'
  END AS grade
FROM student;

-- 写法二：简单 CASE（等值判断）
SELECT
  status,
  CASE status
    WHEN 0 THEN '禁用'
    WHEN 1 THEN '启用'
    ELSE '未知'
  END AS status_name
FROM user;
```

> **CASE 表达式是标量表达式**，可以出现在任何允许表达式的位置：SELECT 列、WHERE 条件、ORDER BY、GROUP BY、UPDATE SET 等。

# 5.聚合函数

参见 [2_sql.md 第 4.6 节](#46聚合函数)，这里补充几个要点：

### 5.1COUNT的几种写法性能对比

```sql
SELECT
  COUNT(*)        AS c1,   -- 统计总行数（推荐：语义最清晰）
  COUNT(1)        AS c2,   -- 统计总行数（与 COUNT(*) 性能相同）
  COUNT(id)       AS c3,   -- 统计 id 非 NULL 的行数
  COUNT(email)    AS c4    -- 统计 email 非 NULL 的行数（最慢）
FROM user;
```

InnoDB 下 `COUNT(*)` 和 `COUNT(1)` 等价，会选择最小的二级索引进行遍历。`COUNT(字段)` 需要判断字段是否为 NULL，且字段不在索引中时需全表扫描，性能最差。

### 5.2GROUP_CONCAT——行转列

```sql
-- 将同一部门的员工姓名拼接成一个字符串
SELECT dept_id, GROUP_CONCAT(name ORDER BY name SEPARATOR ', ') AS members
FROM employee
GROUP BY dept_id;

-- 限制拼接长度（默认 1024 字节，超出截断）
SET SESSION group_concat_max_len = 10240;
```

# 6.MySQL 8.0 窗口函数

窗口函数是 MySQL 8.0 最重要的新增功能之一，解决了传统 GROUP BY 只能输出一行、无法同时保留明细行的痛点。

## 6.1窗口函数核心语法

```sql
function_name([expr]) OVER (
  [PARTITION BY partition_expr, ...]
  [ORDER BY order_expr [ASC | DESC], ...]
  [frame_clause]
)
```

- **PARTITION BY**：数据分区（类似 GROUP BY，但不合并行）
- **ORDER BY**：分区内排序
- **frame_clause**：定义窗口范围（如 `ROWS BETWEEN 1 PRECEDING AND 1 FOLLOWING`）

## 6.2常用窗口函数

| 函数 | 说明 |
|------|------|
| ROW_NUMBER() | 连续编号（1, 2, 3, ...） |
| RANK() | 跳跃排名（1, 1, 3, ...） |
| DENSE_RANK() | 连续排名（1, 1, 2, ...） |
| NTILE(n) | 分成 n 个桶（如分 4 组） |
| LAG(expr, n, default) | 向下取前 n 行 |
| LEAD(expr, n, default) | 向上取后 n 行 |
| FIRST_VALUE(expr) | 窗口内第一个值 |
| LAST_VALUE(expr) | 窗口内最后一个值 |
| SUM / AVG / COUNT OVER | 累计聚合 |

## 6.3实战案例

```sql
-- ROW_NUMBER：各部门薪资排名
SELECT name, dept, salary,
  ROW_NUMBER() OVER (PARTITION BY dept ORDER BY salary DESC) AS rn
FROM employee;

-- RANK vs DENSE_RANK
SELECT name, score,
  RANK() OVER (ORDER BY score DESC) AS `rank`,         -- 1,1,3
  DENSE_RANK() OVER (ORDER BY score DESC) AS dense_rank -- 1,1,2
FROM exam_result;

-- LAG / LEAD：计算环比增长
SELECT month, revenue,
  LAG(revenue, 1) OVER (ORDER BY month) AS prev_revenue,
  ROUND((revenue - LAG(revenue) OVER (ORDER BY month)) / LAG(revenue) OVER (ORDER BY month) * 100, 2) AS growth_rate
FROM monthly_sales;

-- 累计求和（移动平均）
SELECT date, amount,
  SUM(amount) OVER (ORDER BY date) AS cumulative_sum,
  AVG(amount) OVER (ORDER BY date ROWS BETWEEN 2 PRECEDING AND CURRENT ROW) AS moving_avg_3days
FROM daily_transaction;
```

> **窗口函数 vs GROUP BY 的核心区别**：GROUP BY 将多条记录合并为一条，丢失明细；窗口函数保留每条明细行，同时附加统计值。可以理解为窗口函数是在**不丢失行的前提下**进行了分组统计。

# 7.其他常用函数

## 7.1类型转换函数

```sql
-- CAST（SQL 标准）
SELECT CAST('123' AS SIGNED INTEGER);          -- 123
SELECT CAST('2026-05-17' AS DATE);             -- '2026-05-17'

-- CONVERT（MySQL 特有，功能类似）
SELECT CONVERT('123', SIGNED INTEGER);         -- 123

-- 隐式类型转换（不推荐，可能索引失效）
SELECT * FROM user WHERE phone = 13800138000;  -- phone 是 VARCHAR，数字导致隐式转换
```

## 7.2加密函数

```sql
-- MD5（32 位十六进制，已不安全，不用于密码存储）
SELECT MD5('hello');

-- SHA-2 系列（推荐，长度可选 224/256/384/512）
SELECT SHA2('hello', 256);

-- AES 加解密（对称加密，可用于脱敏展示）
SELECT TO_BASE64(AES_ENCRYPT('sensitive_data', 'secret_key'));
SELECT AES_DECRYPT(FROM_BASE64(encrypted_text), 'secret_key');
```

> **密码存储最佳实践**：使用 bcrypt / scrypt / Argon2（应用层），不建议在数据库层做密码哈希。MySQL 8.0 的 `caching_sha2_password` 认证插件已使用 SHA-256。

## 7.3UUID函数

```sql
-- 生成 UUID v1（基于时间戳+MAC 地址）
SELECT UUID();  -- '6ccd780c-333e-11eb-9f6a-0242ac110002'

-- UUID 转二进制（节省空间，16 字节而非 36 字符）
SELECT UUID_TO_BIN(UUID());
SELECT BIN_TO_UUID(uuid_binary);
```

> **UUID 做分布式主键**：不推荐直接用 UUID 字符串做主键（非递增导致页分裂、占用空间大）。推荐使用**雪花算法（Snowflake）**或**Twitter Snowflake 变体**生成有序分布式 ID。MySQL 8.0.17+ 支持 `UUID_TO_BIN(UUID(), TRUE)` 转换时重新排列时间部分，使 BINARY(16) 存储在 B+Tree 中保持有序。

## 7.4JSON函数（MySQL 5.7+）

```sql
-- JSON 提取
SELECT JSON_EXTRACT('{"name": "张三", "age": 25}', '$.name');  -- "张三"
-- 简写语法
SELECT JSON_UNQUOTE(JSON_EXTRACT('{"name": "张三"}', '$.name'));
-- 或使用 -> 和 ->>
SELECT '{"name": "张三"}' -> '$.name';        -- "张三"（带引号）
SELECT '{"name": "张三"}' ->> '$.name';       -- 张三（去引号）

-- JSON 修改
SELECT JSON_SET('{"name": "张三"}', '$.age', 25);  -- 添加字段
SELECT JSON_REMOVE('{"name": "张三", "age": 25}', '$.age');  -- 删除字段

-- JSON 数组操作
SELECT JSON_ARRAY(1, 2, 3);  -- [1, 2, 3]
SELECT JSON_OBJECT('name', '张三', 'age', 25);  -- {"name": "张三", "age": 25}
SELECT JSON_CONTAINS('["a", "b", "c"]', '"b"');  -- 1
```

## 7.5位运算函数（分表/权限场景）

```sql
-- 位运算
SELECT 5 & 3;  -- 1（0101 & 0011 = 0001）
SELECT 5 | 3;  -- 7（0101 | 0011 = 0111）
SELECT 5 ^ 3;  -- 6（0101 ^ 0011 = 0110）
SELECT ~5;     -- 按位取反（取决于 BIGINT 范围）
SELECT 1 << 3; -- 8（左移）
SELECT 8 >> 2; -- 2（右移）

-- 权限判断：字段存储 bitmask，查询是否有某权限
SELECT * FROM user WHERE role & 4 = 4;  -- 是否有角色 4 对应的权限
```
