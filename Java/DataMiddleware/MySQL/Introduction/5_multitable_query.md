# 1. 多表关系

表与表之间存在三种基本关系：

| 关系类型 | 实现方式 | 示例 |
|---------|---------|------|
| **一对多（1:N）** | 在"多"的一方添加外键，指向"一"的一方主键 | 部门 ↔ 员工：一个员工属于一个部门，一个部门有多个员工 |
| **多对多（M:N）** | 创建中间表，含两个外键分别指向两张表 | 学生 ↔ 课程：通过"选课表"关联 |
| **一对一（1:1）** | 在任意一方添加外键并设为 UNIQUE | 用户 ↔ 用户详情（表拆分） |

## 1.1 一对多

```sql
CREATE TABLE dept (
  id INT PRIMARY KEY AUTO_INCREMENT,
  name VARCHAR(50) NOT NULL
);

CREATE TABLE emp (
  id INT PRIMARY KEY AUTO_INCREMENT,
  name VARCHAR(50) NOT NULL,
  dept_id INT NOT NULL,
  CONSTRAINT fk_emp_dept FOREIGN KEY (dept_id) REFERENCES dept(id)
);
```

## 1.2 多对多

```sql
CREATE TABLE student (
  id INT PRIMARY KEY AUTO_INCREMENT,
  name VARCHAR(50) NOT NULL
);

CREATE TABLE course (
  id INT PRIMARY KEY AUTO_INCREMENT,
  name VARCHAR(50) NOT NULL
);

-- 中间表（关联表）
CREATE TABLE student_course (
  student_id INT NOT NULL,
  course_id INT NOT NULL,
  score DECIMAL(4, 1),
  PRIMARY KEY (student_id, course_id),
  FOREIGN KEY (student_id) REFERENCES student(id),
  FOREIGN KEY (course_id) REFERENCES course(id)
);
```

## 1.3 一对一

```sql
CREATE TABLE user (
  id INT PRIMARY KEY AUTO_INCREMENT,
  username VARCHAR(50) NOT NULL
);

CREATE TABLE user_profile (
  id INT PRIMARY KEY AUTO_INCREMENT,
  user_id INT UNIQUE NOT NULL,     -- UNIQUE 保证一对一
  avatar VARCHAR(255),
  bio TEXT,
  FOREIGN KEY (user_id) REFERENCES user(id)
);
```

> **为什么需要一对一表拆分？**
> - **垂直分表**：将常用字段与不常用字段分开，减少每行数据大小，提升缓存命中率
> - **敏感数据隔离**：如将手机号、身份证号存到独立表，单独做权限控制
> - **扩展性**：不同业务模块扩展自己的属性表，避免基础表字段膨胀

# 2. 多表查询概述

多表查询（JOIN）是从多张表中获取数据的手段。当查询涉及多张表时，如果不指定连接条件，会产生**笛卡尔积**：

```sql
-- 笛卡尔积：emp 有 M 行，dept 有 N 行，结果 = M × N 行（通常是错误的查询意图）
SELECT * FROM emp, dept;

-- 正确的多表查询：指定连接条件
SELECT * FROM emp, dept WHERE emp.dept_id = dept.id;
```

**多表查询的性能核心**：JOIN 的本质是表之间的匹配运算。MySQL 优化器会选择驱动表（小表驱动大表原则）和被驱动表，在被驱动表上通过索引快速匹配。

# 3. 连接查询（JOIN）

## 3.1 内连接（INNER JOIN）

内连接查询的是两张表的**交集**——只返回两个表中匹配的行。

```sql
-- 隐式内连接（WHERE 连接条件）
SELECT e.name, d.name AS dept_name
FROM emp e, dept d
WHERE e.dept_id = d.id;

-- 显式内连接（推荐：语义更清晰）
SELECT e.name, d.name AS dept_name
FROM emp e
INNER JOIN dept d ON e.dept_id = d.id;
```

> **推荐使用显式 JOIN 语法**：将连接条件（ON）和过滤条件（WHERE）分开，可读性更好，不易出错。

**多表内连接**：

```sql
-- 三表内连接：查询学生姓名、课程名、成绩
SELECT s.name, c.name, sc.score
FROM student s
INNER JOIN student_course sc ON s.id = sc.student_id
INNER JOIN course c ON sc.course_id = c.id
WHERE sc.score >= 60;
```

## 3.2 外连接（OUTER JOIN）

外连接除了返回匹配的行，还会保留**左表或右表中不匹配的行**（不匹配的部分用 NULL 填充）。

### 3.2.1 左外连接（LEFT JOIN）

```sql
-- 查询所有员工及其部门（包括没有部门的员工）
SELECT e.name, d.name AS dept_name
FROM emp e
LEFT JOIN dept d ON e.dept_id = d.id;
-- 结果：有部门的员工正常显示，没有部门的员工 dept_name 为 NULL
```

### 3.2.2 右外连接（RIGHT JOIN）

```sql
-- 查询所有部门及其员工（包括没有员工的部门）
SELECT e.name, d.name AS dept_name
FROM emp e
RIGHT JOIN dept d ON e.dept_id = d.id;
-- 等价于以下 LEFT JOIN
SELECT e.name, d.name AS dept_name
FROM dept d
LEFT JOIN emp e ON e.dept_id = d.id;
```

> **最佳实践**：绝大多数场景使用 LEFT JOIN 即可，RIGHT JOIN 可以通过交换表顺序转为 LEFT JOIN。统一使用 LEFT JOIN 有助于代码可读性和维护。

### 3.2.3 全外连接（FULL OUTER JOIN）

MySQL 不直接支持 FULL OUTER JOIN，可通过 **LEFT JOIN + UNION + RIGHT JOIN** 模拟：

```sql
SELECT e.name, d.name FROM emp e LEFT JOIN dept d ON e.dept_id = d.id
UNION
SELECT e.name, d.name FROM emp e RIGHT JOIN dept d ON e.dept_id = d.id;
```

## 3.3 自连接（SELF JOIN）

自连接是指自己连接自己，用于处理**表中数据存在层级关系**的场景。

```sql
-- 表结构：employee(id, name, manager_id)
-- manager_id 指向本表的 id

-- 查询员工及其上级领导姓名
SELECT e.name AS employee, m.name AS manager
FROM employee e
LEFT JOIN employee m ON e.manager_id = m.id;
```

> 自连接本质还是 JOIN，只是连接的两张表是同一张表，必须使用**别名**区分。自连接常用于：
> - 组织架构（员工 → 经理 → 总监）
> - 树形结构（菜单 → 父菜单）
> - 地域层级（省 → 市 → 区）

## 3.4 交叉连接（CROSS JOIN）

返回笛卡尔积：

```sql
SELECT * FROM emp CROSS JOIN dept;  -- 等价于 SELECT * FROM emp, dept;
```

# 4. 子查询（Subquery）

子查询是指嵌套在另一个 SQL 语句（SELECT、INSERT、UPDATE、DELETE）内部的 SELECT 查询。子查询的结果作为外层查询的条件或数据源。

## 4.1 按结果分类

### 4.1.1 标量子查询（结果为一格——单个值）

```sql
-- 查询工资高于平均工资的员工
SELECT name, salary FROM emp
WHERE salary > (SELECT AVG(salary) FROM emp);
```

### 4.1.2 列子查询（结果为一列，可能多行）

配合 `IN`、`NOT IN`、`ANY`、`SOME`、`ALL` 使用：

```sql
-- 查询"研发部"和"市场部"的所有员工
SELECT name FROM emp
WHERE dept_id IN (SELECT id FROM dept WHERE name IN ('研发部', '市场部'));

-- 查询比"研发部所有人工资都高"的员工
SELECT name FROM emp
WHERE salary > ALL (SELECT salary FROM emp WHERE dept_id = 1);

-- 查询比"研发部任意一人工资高"的员工
SELECT name FROM emp
WHERE salary > ANY (SELECT salary FROM emp WHERE dept_id = 1);
```

### 4.1.3 行子查询（结果为一行多列）

```sql
-- 查询与"张三"同部门且同职级的员工
SELECT name FROM emp
WHERE (dept_id, level) = (SELECT dept_id, level FROM emp WHERE name = '张三')
  AND name != '张三';
```

### 4.1.4 表子查询（结果为多行多列）

```sql
-- 子查询作为虚拟表（必须取别名）
SELECT t.* FROM (
  SELECT dept_id, COUNT(*) AS cnt, AVG(salary) AS avg_sal
  FROM emp
  GROUP BY dept_id
) t
WHERE t.cnt > 5;
```

## 4.2 按位置分类

| 位置 | 示例 | 注意 |
|------|------|------|
| WHERE 之后 | `WHERE id = (SELECT ...)` | 标量/列/行子查询 |
| FROM 之后 | `FROM (SELECT ...) AS t` | 必须取别名 |
| SELECT 之后 | `SELECT (SELECT ...) AS col` | 必须是标量子查询 |
| HAVING 之后 | `HAVING AVG(salary) > (SELECT ...)` | 必须是标量子查询 |
| INSERT ... SELECT | `INSERT INTO t SELECT ...` | 表子查询 |

# 5. 联合查询（UNION）

联合查询将多个 SELECT 的结果合并为一个结果集。

```sql
SELECT name, phone FROM customer
UNION
SELECT name, phone FROM supplier;

-- UNION ALL：保留重复行（性能更好，不需要去重）
SELECT name FROM customer
UNION ALL
SELECT name FROM supplier;
```

**UNION vs UNION ALL**：

| 特性 | UNION | UNION ALL |
|------|-------|-----------|
| 去重 | 是（排序 + 去重） | 否 |
| 性能 | 较慢 | 较快 |
| 列要求 | 列数、列顺序、列类型必须一致 | 同左 |
| 排序 | 默认按第一列升序（可能被忽略） | 不排序 |

> **使用 UNION 的场景**：多张同构表（如按年份分表 `order_2024`、`order_2025`）的数据汇总查询。

# 6. 多表查询优化要点

- **用小表驱动大表**：LEFT JOIN 时，左表（驱动表）应尽量小；INNER JOIN 时，优化器会自动选择
- **连接字段必须有索引**：被驱动表的连接字段（ON 后面的字段）必须有索引，否则每次匹配都做全表扫描
- **避免 SELECT \***：明确需要的列，减少网络传输，也有利于覆盖索引
- **子查询尽量改写为 JOIN**：MySQL 优化器对 JOIN 的优化更成熟，大部分子查询可以优化为 JOIN

```sql
-- 子查询写法（较慢）
SELECT name FROM emp WHERE dept_id IN (SELECT id FROM dept WHERE name = '研发部');

-- 等价 JOIN 写法（通常更快）
SELECT e.name FROM emp e INNER JOIN dept d ON e.dept_id = d.id WHERE d.name = '研发部';
```

> **EXISTS vs IN**：
> - `IN`：先执行子查询，再与外部查询匹配。适合子查询结果集小的场景
> - `EXISTS`：遍历外部查询每一行，代入子查询判断是否存在。适合外部表较小的场景。`EXISTS` 子查询中如果有索引，一旦找到匹配就立即返回，不会继续扫描
> ```sql
> SELECT name FROM emp e
> WHERE EXISTS (SELECT 1 FROM dept d WHERE d.id = e.dept_id AND d.name = '研发部');
> ```
