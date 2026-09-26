# 1.视图（View）

## 1.1概念与应用场景

视图是一种**虚拟存在的表**，视图中的数据并不在数据库中实际存在，行和列数据来自定义视图的查询中使用的表（基表），并且是在使用视图时**动态生成**的。

**视图的作用**：

| 作用 | 说明 |
|------|------|
| **简化查询** | 将复杂 JOIN、子查询封装为视图，业务代码只需 `SELECT * FROM view` |
| **安全隔离** | 隐藏表中敏感列（如密码、身份证号），只暴露必要字段给不同角色 |
| **数据独立** | 视图提供稳定的接口，基表结构变更只需修改视图定义，应用层无感 |
| **权限控制** | 可对视图授权，精确到行和列级别的权限控制 |

## 1.2创建视图

```sql
-- 基本创建
CREATE VIEW v_user_basic AS
SELECT id, username, email FROM user;

-- OR REPLACE：存在则替换
CREATE OR REPLACE VIEW v_order_summary AS
SELECT
  u.id AS user_id,
  u.username,
  COUNT(o.id) AS order_count,
  IFNULL(SUM(o.amount), 0) AS total_amount
FROM user u
LEFT JOIN `order` o ON u.id = o.user_id
GROUP BY u.id, u.username;

-- 指定字段别名
CREATE VIEW v_user(id, 姓名, 邮箱) AS
SELECT id, username, email FROM user;
```

## 1.3查询视图

```sql
-- 查询视图（就像操作普通表一样）
SELECT * FROM v_user_basic WHERE id > 100;

-- 查看视图定义
SHOW CREATE VIEW v_user_basic;

-- 查看所有视图
SELECT TABLE_NAME FROM information_schema.VIEWS
WHERE TABLE_SCHEMA = 'mydb';
```

## 1.4修改视图

```sql
-- 方式一：CREATE OR REPLACE（推荐，简洁）
CREATE OR REPLACE VIEW v_user_basic AS
SELECT id, username, email, phone FROM user;

-- 方式二：ALTER VIEW
ALTER VIEW v_user_basic AS
SELECT id, username, email, phone, create_time FROM user;
```

## 1.5删除视图

```sql
DROP VIEW IF EXISTS v_user_basic, v_order_summary;
```

## 1.6CHECK OPTION

WITH CHECK OPTION 用于约束对视图的插入/更新操作，防止插入超出视图范围的数据：

```sql
-- 创建视图：仅能看到 status=1 的用户
CREATE VIEW v_active_user AS
SELECT id, username, status FROM user WHERE status = 1
WITH CHECK OPTION;

-- ✓ 允许（插入 status=1 的行）
INSERT INTO v_active_user (username, status) VALUES ('张三', 1);

-- ✗ 拒绝（status=2 不满足视图的 WHERE 条件）
INSERT INTO v_active_user (username, status) VALUES ('李四', 2);
-- Error: CHECK OPTION failed 'mydb.v_active_user'
```

### CASCADED vs LOCAL

当视图基于另一个视图时：

- **CASCADED**（默认）：必须满足所有父视图的条件
- **LOCAL**：仅检查当前视图的条件（只要当前视图没有指定 CHECK OPTION，就不检查）

```sql
-- 基视图：无 CHECK OPTION
CREATE VIEW v1 AS SELECT id, name FROM t WHERE id > 10;

-- v2 基于 v1，加了 CHECK OPTION
CREATE VIEW v2 AS SELECT id, name FROM v1 WHERE id < 20
WITH CASCADED CHECK OPTION;  -- 同时检查 v1 和 v2 的条件
-- 等价于必须满足 id > 10 AND id < 20

CREATE VIEW v2 AS SELECT id, name FROM v1 WHERE id < 20
WITH LOCAL CHECK OPTION;  -- 仅检查 v2 的条件
-- 等价于必须满足 id < 20（因为 v1 没有 CHECK OPTION）
```

## 1.7视图的更新条件

视图可以进行 INSERT / UPDATE / DELETE，但有严格限制。要使视图可更新，视图中的行与基表中的行必须存在**一对一关系**。

**视图不可更新的情况**：视图定义中包含以下任一结构时不可更新（更新操作会失败）：

- 聚合函数：`SUM()`、`MIN()`、`MAX()`、`COUNT()`、`AVG()` 等
- `DISTINCT`
- `GROUP BY`
- `HAVING`
- `UNION` / `UNION ALL`
- 子查询在 SELECT 列表中
- 使用了多张表的 JOIN（某些情况可通过触发器实现）
- FROM 子句中有不可更新的视图

# 2.存储过程（Stored Procedure）

## 2.1概念与优缺点

存储过程是**事先经过编译并存储在数据库中的一段 SQL 语句集合**。调用存储过程可以简化应用开发，减少数据在数据库和应用服务器之间的传输。

**优点**：
- **封装复用**：复杂业务逻辑封装后多处调用
- **减少网络交互**：多条 SQL 在数据库端一次执行，只需一次网络往返
- **性能提升**：预编译，执行效率比逐条 SQL 高
- **安全控制**：应用程序不能直接访问表，只能通过存储过程操作（数据库层面权限最小化）
- **事务控制**：可在存储过程中精细控制事务

**缺点**：
- **调试困难**：缺乏调试工具，排查 BUG 麻烦
- **版本管理难**：存储过程是数据库对象，不像应用代码容易版本化和 CI/CD
- **移植性差**：存储过程语法各数据库差异大（MySQL 的存储过程与 Oracle/SQL Server 不兼容）
- **扩展困难**：数据库计算资源是系统中价格最高的，应将计算逻辑放在应用层更易扩展

> **工程实践**：现代互联网架构中，存储过程的使用逐渐减少。推荐将业务逻辑放在应用层（Java、Go），数据库只做存储和基础查询。存储过程适合金融、ERP 等传统系统中需要保持数据一致性的复杂计算。

## 2.2基本语法

### 创建存储过程

```sql
-- 注意：命令行中需要用 DELIMITER 修改语句结束符
DELIMITER //

CREATE PROCEDURE get_user_count(IN dept_id INT, OUT total INT)
BEGIN
  SELECT COUNT(*) INTO total FROM user WHERE dept_id = dept_id;
END //

DELIMITER ;
```

> **DELIMITER 的作用**：存储过程体中包含多条 SQL（以分号结尾），命令行默认以分号为语句结束符，会导致存储过程定义被截断。使用 `DELIMITER //` 将结束符改为 `//`，定义完成后再改回 `;`。**Navicat / DataGrip 等 GUI 工具通常不需要手动设置 DELIMITER**。

### 调用存储过程

```sql
-- 调用有 OUT 参数的存储过程
CALL get_user_count(1, @total);
SELECT @total;
```

## 2.3变量

### 系统变量

MySQL 内置的全局或会话级变量：

```sql
-- 查看系统变量
SHOW SESSION VARIABLES LIKE 'autocommit';
SHOW GLOBAL VARIABLES LIKE 'max_connections';
SELECT @@SESSION.autocommit;
SELECT @@GLOBAL.max_connections;

-- 设置系统变量（重启后失效，永久生效需改 my.cnf）
SET SESSION autocommit = 0;
SET GLOBAL max_connections = 500;
```

### 用户自定义变量

作用域为当前连接，无需声明，直接 `@` 使用：

```sql
-- 赋值
SET @name = '张三';
SET @total := 0;           -- := 也可用于非 SELECT 语句
SELECT COUNT(*) INTO @total FROM user;

-- 使用
SELECT @name, @total;
```

### 局部变量

在存储过程内部使用，需要 DECLARE 声明：

```sql
CREATE PROCEDURE demo()
BEGIN
  DECLARE total INT DEFAULT 0;              -- 声明并初始化
  DECLARE avg_age DECIMAL(4,2);
  
  SELECT COUNT(*) INTO total FROM user;
  SELECT AVG(age) INTO avg_age FROM user;
  
  SELECT total, avg_age;
END;
```

## 2.4参数

存储过程的参数有三种模式：

| 模式 | 说明 | 示例 |
|------|------|------|
| **IN**（默认） | 输入参数，调用时传入 | `IN dept_id INT` |
| **OUT** | 输出参数，作为返回值 | `OUT total INT` |
| **INOUT** | 既输入也输出 | `INOUT count INT` |

```sql
CREATE PROCEDURE update_salary(IN emp_id INT, IN raise_percent DECIMAL(5,2), OUT new_salary DECIMAL(10,2))
BEGIN
  UPDATE emp SET salary = salary * (1 + raise_percent / 100) WHERE id = emp_id;
  SELECT salary INTO new_salary FROM emp WHERE id = emp_id;
END;

-- 调用
CALL update_salary(1, 10, @new_sal);
SELECT @new_sal;
```

## 2.5分支结构

### IF ... ELSEIF ... ELSE

```sql
CREATE PROCEDURE grade_score(IN score INT, OUT result VARCHAR(10))
BEGIN
  IF score >= 90 THEN SET result = '优秀';
  ELSEIF score >= 80 THEN SET result = '良好';
  ELSEIF score >= 60 THEN SET result = '及格';
  ELSE SET result = '不及格';
  END IF;
END;
```

### CASE

两种写法：

```sql
-- 写法一：等值判断
CASE expr
  WHEN val1 THEN ... ;
  WHEN val2 THEN ... ;
  ELSE ... ;
END CASE;

-- 写法二：条件判断（类似 IF）
CASE
  WHEN condition1 THEN ... ;
  WHEN condition2 THEN ... ;
  ELSE ... ;
END CASE;
```

```sql
CREATE PROCEDURE print_season(IN month INT)
BEGIN
  CASE month
    WHEN 3 THEN SELECT '春季';
    WHEN 4 THEN SELECT '春季';
    WHEN 5 THEN SELECT '春季';
    ELSE SELECT '其他季节';
  END CASE;
END;
```

## 2.6循环结构

### WHILE（条件满足时循环）

```sql
-- 累加 1 到 n
CREATE PROCEDURE sum_n(IN n INT, OUT result INT)
BEGIN
  DECLARE i INT DEFAULT 1;
  SET result = 0;
  
  WHILE i <= n DO
    SET result = result + i;
    SET i = i + 1;
  END WHILE;
END;
```

### REPEAT（满足条件退出循环）

```sql
CREATE PROCEDURE repeat_demo(IN n INT)
BEGIN
  DECLARE i INT DEFAULT 1;
  
  REPEAT
    SELECT CONCAT('当前值: ', i);
    SET i = i + 1;
    UNTIL i > n       -- 满足条件时退出（注意：至少执行一次）
  END REPEAT;
END;
```

### LOOP（无限循环，需配合 LEAVE 退出）

```sql
CREATE PROCEDURE loop_demo()
BEGIN
  DECLARE i INT DEFAULT 0;
  
  label_loop: LOOP
    SET i = i + 1;
    IF i >= 10 THEN LEAVE label_loop; END IF;   -- LEAVE 退出循环
    IF i % 2 = 0 THEN ITERATE label_loop; END IF; -- ITERATE 跳过本次
    SELECT i;
  END LOOP label_loop;
END;
```

> **LEAVE**：退出指定标签的循环，类似 Java 的 `break`。
> **ITERATE**：跳过本次循环，进入下一次，类似 Java 的 `continue`。

## 2.7游标（Cursor）

游标用于逐行处理 SELECT 结果集：

```sql
CREATE PROCEDURE process_users()
BEGIN
  DECLARE done INT DEFAULT 0;
  DECLARE u_name VARCHAR(50);
  DECLARE u_email VARCHAR(100);
  
  -- 声明游标
  DECLARE user_cursor CURSOR FOR SELECT username, email FROM user WHERE status = 1;
  
  -- 声明条件处理程序：游标取完数据时设置 done = 1
  DECLARE CONTINUE HANDLER FOR NOT FOUND SET done = 1;
  
  OPEN user_cursor;
  
  read_loop: LOOP
    FETCH user_cursor INTO u_name, u_email;
    IF done = 1 THEN LEAVE read_loop; END IF;
    
    -- 处理每条记录（这里仅输出）
    SELECT CONCAT('处理: ', u_name, ' - ', u_email);
  END LOOP;
  
  CLOSE user_cursor;
END;
```

> **必须配合条件处理程序**：游标遍历完所有行后，继续 FETCH 会导致 NOT FOUND 异常，必须用 Handler 捕获。

## 2.8条件处理程序（Handler）

用于捕获和处理存储过程中的异常：

```sql
DECLARE action HANDLER FOR condition_value statement;
```

| action | 说明 |
|--------|------|
| CONTINUE | 继续执行当前程序 |
| EXIT | 终止执行当前程序 |

| condition_value | 说明 |
|----------------|------|
| SQLSTATE 'code' | 特定 SQL 状态码 |
| SQLWARNING | 所有 01 开头的 SQLSTATE |
| NOT FOUND | 所有 02 开头的 SQLSTATE |
| SQLEXCEPTION | 所有未被 SQLWARNING 或 NOT FOUND 捕获的 |

```sql
CREATE PROCEDURE safe_insert(IN u_name VARCHAR(50))
BEGIN
  -- 遇到唯一键冲突时，不报错，设置错误信息
  DECLARE EXIT HANDLER FOR SQLSTATE '23000'
    SELECT '用户名已存在，插入失败' AS msg;
  
  INSERT INTO user (username) VALUES (u_name);
  SELECT '插入成功' AS msg;
END;
```

## 2.9管理操作

```sql
-- 查看存储过程列表
SHOW PROCEDURE STATUS WHERE db = 'mydb';

-- 查看存储过程定义
SHOW CREATE PROCEDURE get_user_count;

-- 从 information_schema 查询
SELECT * FROM information_schema.ROUTINES
WHERE routine_schema = 'mydb' AND routine_type = 'PROCEDURE';

-- 删除存储过程
DROP PROCEDURE IF EXISTS get_user_count;
```

# 3.存储函数（Stored Function）

## 3.1概念

存储函数是有**返回值**的存储过程，参数只能是 IN 类型（隐式）。

**与存储过程的区别**：

| 特性 | 存储过程 | 存储函数 |
|------|---------|---------|
| 返回值 | 可以有多个 OUT 参数 | 必须有且仅有一个返回值 |
| 调用方式 | `CALL proc(params)` | 在 SQL 中直接调用 `func(params)` |
| 使用场景 | 封装复杂业务逻辑 | 计算并返回值，可用于 SELECT 列 |

## 3.2语法

```sql
CREATE FUNCTION calc_tax(salary DECIMAL(10,2))
RETURNS DECIMAL(10,2)
DETERMINISTIC        -- 确定性函数（相同输入一定相同输出）
READS SQL DATA       -- 读取 SQL 数据（不修改）
BEGIN
  DECLARE tax DECIMAL(10,2);
  
  IF salary <= 5000 THEN SET tax = 0;
  ELSEIF salary <= 8000 THEN SET tax = (salary - 5000) * 0.03;
  ELSEIF salary <= 17000 THEN SET tax = 90 + (salary - 8000) * 0.1;
  ELSE SET tax = 990 + (salary - 17000) * 0.2;
  END IF;
  
  RETURN tax;
END;

-- 使用（可在 SELECT 中调用）
SELECT username, salary, calc_tax(salary) AS tax FROM emp;
```

### 必须指定的特性

| 特性 | 说明 |
|------|------|
| DETERMINISTIC | 确定性：相同输入始终返回相同结果 |
| NO SQL | 函数体中无 SQL 语句 |
| READS SQL DATA | 有只读 SQL（SELECT） |
| MODIFIES SQL DATA | 有写入 SQL（INSERT/UPDATE/DELETE） |

> MySQL 8.0 默认开启 `log_bin_trust_function_creators`，要求必须有至少一个特性。生产环境主从复制时需确保函数是确定性的。

## 3.3管理操作

```sql
SHOW FUNCTION STATUS WHERE db = 'mydb';
SHOW CREATE FUNCTION calc_tax;
DROP FUNCTION IF EXISTS calc_tax;
```

# 4.触发器（Trigger）

## 4.1概念

触发器是与表有关的数据库对象，在 **INSERT / UPDATE / DELETE 之前或之后**，触发并执行触发器中定义的 SQL 语句集合。

**应用场景**：

- 数据校验：插入/更新前检查数据合法性
- 日志记录：自动记录数据变更历史
- 数据同步：主表变更时自动更新汇总表
- 审计：记录谁在什么时候修改了什么数据

> **MySQL 当前仅支持行级触发**（FOR EACH ROW），不支持语句级触发。Oracle 和 PostgreSQL 支持语句级触发。

## 4.2创建触发器

```sql
CREATE TRIGGER trigger_name
BEFORE | AFTER  INSERT | UPDATE | DELETE
ON table_name FOR EACH ROW
BEGIN
  -- 触发器体
END;
```

### NEW 和 OLD 引用

| 操作 | OLD | NEW |
|------|-----|-----|
| INSERT | 无 | 新插入的行 |
| UPDATE | 修改前的行 | 修改后的行 |
| DELETE | 被删除的行 | 无 |

## 4.3实战案例

### 日志记录触发器

```sql
-- 操作日志表
CREATE TABLE user_log (
  id BIGINT PRIMARY KEY AUTO_INCREMENT,
  operation VARCHAR(10) COMMENT 'INSERT/UPDATE/DELETE',
  user_id BIGINT,
  old_username VARCHAR(50),
  new_username VARCHAR(50),
  operate_time DATETIME DEFAULT CURRENT_TIMESTAMP
);

-- 触发器：用户表变更时写日志
CREATE TRIGGER tr_user_insert
AFTER INSERT ON user FOR EACH ROW
BEGIN
  INSERT INTO user_log (operation, user_id, new_username)
  VALUES ('INSERT', NEW.id, NEW.username);
END;

CREATE TRIGGER tr_user_update
AFTER UPDATE ON user FOR EACH ROW
BEGIN
  INSERT INTO user_log (operation, user_id, old_username, new_username)
  VALUES ('UPDATE', NEW.id, OLD.username, NEW.username);
END;

CREATE TRIGGER tr_user_delete
AFTER DELETE ON user FOR EACH ROW
BEGIN
  INSERT INTO user_log (operation, user_id, old_username)
  VALUES ('DELETE', OLD.id, OLD.username);
END;
```

### 数据校验触发器

```sql
-- 插入前检查年龄是否合法
CREATE TRIGGER tr_user_check_age
BEFORE INSERT ON user FOR EACH ROW
BEGIN
  IF NEW.age < 0 OR NEW.age > 150 THEN
    SIGNAL SQLSTATE '45000'
    SET MESSAGE_TEXT = '年龄必须在 0~150 之间';
  END IF;
END;
```

> `SIGNAL` 是 MySQL 5.5+ 提供的自定义异常抛出机制，`SQLSTATE '45000'` 是用户自定义异常的通用状态码。

## 4.4管理操作

```sql
-- 查看触发器
SHOW TRIGGERS;
SELECT * FROM information_schema.triggers WHERE trigger_schema = 'mydb';

-- 删除触发器
DROP TRIGGER IF EXISTS tr_user_insert;
```

## 4.5触发器的注意事项

- **触发器过于复杂会影响主操作性能**，因为触发器与主操作在同一事务中同步执行
- **避免循环触发**：A 表触发器修改 B 表，B 表触发器又修改 A 表
- **调试困难**：触发器在数据库后台静默执行，出问题较难排查
- **MySQL 不支持同一表上同一事件多个触发器**（Oracle 支持），需要合并逻辑

> **工程建议**：触发器适合简单的审计日志、自动更新时间等场景。复杂的业务逻辑更适合放在应用层处理，便于调试、测试和维护。
