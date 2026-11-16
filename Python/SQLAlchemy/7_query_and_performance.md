查询写错，功能可能还是对的，只是慢。慢查询很少是"数据库不行"，多数是 N+1、缺索引、取了不需要的列、或者连接池配置不合理。本篇从 `select` 的写法讲到 `EXPLAIN` 的读法，目标是把"感觉慢"变成"知道慢在哪"。

# 1. select 基础

## 1.1 查实体还是查列

```python
from sqlalchemy import select

# 查完整实体：返回 User 对象，进入身份映射
users = session.scalars(select(User)).all()

# 只查需要的列：返回 Row，更省内存
rows = session.execute(select(User.id, User.name)).all()
```

| 写法 | 返回 | 适用 |
|------|------|------|
| `select(User)` | 实体对象 | 需要修改对象、使用关系 |
| `select(User.id, User.name)` | Row | 列表展示、导出 |
| `select(func.count()).select_from(User)` | 单值 | 计数 |

★ **关键认知**：**列表页只查需要的列，往往比查完整实体快一个量级**。实体查询会加载所有列、构造对象、登记身份映射；数据量大时，这些开销比 SQL 本身还高。

## 1.2 常用过滤操作符

```python
from sqlalchemy import and_, or_

stmt = (
    select(User)
    .where(User.age >= 18)
    .where(User.status == "active")
    .where(User.name.like("a%"))
    .where(User.email.is_not(None))
    .where(User.id.in_([1, 2, 3]))
    .where(User.created_at.between(start, end))
)
```

| Python 写法 | 生成的 SQL |
|------------|-----------|
| `User.name.is_(None)` | `name IS NULL` |
| `User.name.like("a%")` | `name LIKE 'a%'` |
| `User.name.ilike("a%")` | 忽略大小写匹配 |
| `User.id.in_([1, 2])` | `id IN (1, 2)` |
| `or_(a, b)` / `and_(a, b)` | `a OR b` / `a AND b` |

⚠ `in_()` 不要传空列表。部分数据库会生成 `IN ()` 语法错误，或让 `NOT IN` 的语义与预期相反。空列表应直接跳过该条件。

## 1.3 排序、分页与 keyset

```python
stmt = (
    select(User)
    .order_by(User.created_at.desc(), User.id.desc())
    .limit(20)
    .offset(40)
)
```

**排序要稳定**：只按 `created_at` 排序时，同一秒的多条记录顺序不确定，翻页可能重复或遗漏。加上唯一的 `id` 作为第二排序键。

`OFFSET 100000` 会让数据库先扫描并丢弃十万行，越翻越慢。更好的方式是**基于上一页最后一条记录的值继续查**：

```python
# 第一页
page = session.scalars(select(Post).order_by(Post.id.desc()).limit(20)).all()

# 下一页：从上一页最后一个 id 继续
last_id = page[-1].id
next_page = session.scalars(
    select(Post).where(Post.id < last_id).order_by(Post.id.desc()).limit(20)
).all()
```

| 方案 | 优点 | 缺点 | 适用 |
|------|------|------|------|
| `OFFSET` | 实现简单、可跳页 | 深页慢 | 后台管理、小数据量 |
| keyset | 稳定、快 | 不能跳页 | 信息流、无限滚动 |

# 2. 连接查询

## 2.1 join 与 outerjoin

```python
# 内连接：只保留有关联的数据
stmt = (
    select(Post.title, User.name)
    .join(User, Post.author_id == User.id)
    .where(User.status == "active")
)

# 外连接：没有文章的用户也会出现，计数为 0
stmt = (
    select(User.name, func.count(Post.id))
    .outerjoin(Post, Post.author_id == User.id)
    .group_by(User.id, User.name)
)
```

也可以用关系名简化连接：`select(Post).join(Post.author)`。**多外键、复合外键或没有外键时推断会失败**，这时要显式写 `onclause`。

## 2.2 join 与预加载的分工

| 目的 | 写法 |
|------|------|
| 按关联条件过滤主表 | `join` |
| 需要关联对象参与展示 | `options(joinedload(...))` |
| 需要关联集合参与展示 | `options(selectinload(...))` |
| 只取关联表的几个列 | `join` + `select(具体列)` |

⚠ 对一对多关系，`join` 会让主表行重复。要计数用 `count(distinct ...)` 或子查询，不要直接 `len(rows)`。

# 3. 聚合

## 3.1 常用聚合函数

```python
from sqlalchemy import func

total = session.scalar(select(func.count()).select_from(Order))
amount = session.scalar(select(func.sum(Order.amount)))
avg = session.scalar(select(func.avg(Order.amount)))
```

## 3.2 分组与 HAVING

```python
stmt = (
    select(User.id, User.name, func.count(Order.id).label("order_count"))
    .join(Order, Order.user_id == User.id)
    .group_by(User.id, User.name)
    .having(func.count(Order.id) > 5)
    .order_by(func.count(Order.id).desc())
)
```

SQL 的规则：`SELECT` 里出现的非聚合列，必须出现在 `GROUP BY` 中。PostgreSQL 会严格检查，MySQL 在宽松模式下可能放过，但结果不可靠。

# 4. N+1 的定位与修复

## 4.1 先让它可见

```python
import logging

logging.basicConfig()
logging.getLogger("sqlalchemy.engine").setLevel(logging.INFO)
```

观察日志中是否有大量结构相同、只有参数不同的 SQL：

```text
SELECT ... FROM users;
SELECT ... FROM posts WHERE author_id = 1;
SELECT ... FROM posts WHERE author_id = 2;
SELECT ... FROM posts WHERE author_id = 3;
```

## 4.2 用预加载修复

```python
from sqlalchemy.orm import selectinload

stmt = select(User).options(selectinload(User.posts))
users = session.scalars(stmt).all()
```

测试环境可以加 `raiseload("*")`，任何未预加载的关系访问都会直接抛错，能在 CI 阶段拦住新写的 N+1。

★ **关键认知**：**N+1 的根因不是"用了循环"，而是"在循环里触发查询"**。修复方式要么把查询提前（预加载），要么把计算下推到数据库（JOIN + 聚合）。应用层循环只适合处理已经取回内存的数据。

# 5. 索引与 EXPLAIN

## 5.1 什么时候该建索引

| 条件 | 说明 |
|------|------|
| 列出现在 `WHERE` | 高选择性列收益最大 |
| 列用于 `JOIN` | 外键列通常需要 |
| 列用于 `ORDER BY` / `GROUP BY` | 可避免排序或加速分组 |
| 列选择性高 | 邮箱、订单号适合；性别不适合 |

## 5.2 复合索引的顺序

```sql
CREATE INDEX ix_orders_user_status_created
ON orders (user_id, status, created_at);
```

它能服务 `WHERE user_id = ?`、`WHERE user_id = ? AND status = ?`，以及带 `ORDER BY created_at` 的查询；**不能**高效服务只查 `status` 的语句，因为跳过了最左列。

## 5.3 读 EXPLAIN

```python
from sqlalchemy import text

plan = session.execute(
    text("EXPLAIN (ANALYZE, BUFFERS) SELECT * FROM orders WHERE user_id = :uid"),
    {"uid": 1},
).all()
for line in plan:
    print(line[0])
```

```text
Index Scan using ix_orders_user_status_created on orders
  Index Cond: (user_id = 1)
  Buffers: shared hit=3 read=1
Planning Time: 0.1 ms
Execution Time: 0.05 ms
```

| 节点 | 含义 | 是否正常 |
|------|------|---------|
| `Seq Scan` | 全表扫描 | 小表正常，大表要查索引 |
| `Index Scan` | 走索引后回表 | 常见且正常 |
| `Index Only Scan` | 只读索引，不回表 | 最优 |
| `Nested Loop` | 嵌套循环连接 | 小结果集常见 |
| `Hash Join` | 哈希连接 | 大表连接常见 |
| `Sort` | 显式排序 | 数据量大时可考虑索引 |

⚠ `EXPLAIN ANALYZE` 会**真正执行**语句。对 `UPDATE` / `DELETE` 做分析时务必放在事务里并回滚，否则会改数据。

## 5.4 常见慢查询模式

| 模式 | 问题 | 改法 |
|------|------|------|
| `LIKE '%关键词%'` | 前置通配符无法走 B-tree | 全文索引、`pg_trgm` |
| 对列做函数运算 | `WHERE date(created_at) = ?` 索引失效 | 改范围查询 |
| 隐式类型转换 | `WHERE varchar_col = 123` | 类型对齐 |
| `OR` 跨多列 | 优化器可能放弃索引 | 改 `UNION ALL` 或复合索引 |
| 大偏移分页 | 扫描并丢弃大量行 | keyset 分页 |

# 6. 连接池

## 6.1 关键参数

```python
engine = create_engine(
    url,
    pool_size=10,           # 常驻连接
    max_overflow=20,        # 峰值额外连接
    pool_timeout=30,        # 借连接最长等待
    pool_recycle=1800,      # 连接存活上限，防数据库主动断开
    pool_pre_ping=True,     # 借出前探活
)
```

## 6.2 容量估算

```text
单进程连接数 ≈ pool_size + max_overflow
总连接数 = 单进程连接数 × 进程/实例数
总连接数 < 数据库 max_connections × 0.8
```

例如 4 个 worker，每个 `pool_size=5`、`max_overflow=5`，总连接上限是 40；数据库 `max_connections=100` 时留有足够余量。

★ **关键认知**：**连接池不是越大越好**。数据库每条连接都有内存和调度成本，连接数超过 CPU 并行能力后，吞吐不会上升，只会增加上下文切换和锁竞争。先优化慢查询和长事务，再考虑加连接。

## 6.3 连接泄漏与异步

请求量不大却频繁 `pool_timeout`、数据库连接数持续上涨、`idle in transaction` 很多，通常意味着有连接没关闭、有长事务，或在事务里调用了外部服务。

异步项目要用异步驱动与 `AsyncSession`（`create_async_engine` + `async_sessionmaker`）。**不要在异步代码里调用同步 Session**，那会阻塞事件循环。

# 7. 常见错误

| 现象 | 原因 | 解决 |
|------|------|------|
| 列表页 SQL 几百条 | N+1 | `selectinload` / `joinedload` |
| 加了索引还是慢 | 条件用了函数或隐式转换 | 改写条件、对齐类型 |
| 深分页越来越慢 | 大 `OFFSET` | keyset 分页 |
| 计数结果偏大 | 一对多 JOIN 导致行重复 | `count(distinct ...)` |
| 连接池频繁超时 | 长事务或连接泄漏 | 查 `idle in transaction` |
| `EXPLAIN` 估算行数离谱 | 统计信息过期 | `ANALYZE` 表 |
| 排序很慢 | 排序字段无索引 | 建匹配顺序的索引 |

# 8. 动手练习

1. 造 1 万个用户、每人 10 篇文章，分别用懒加载和 `selectinload` 查询用户及其文章，记录 SQL 条数与耗时。
2. 对 `orders(user_id, status, created_at)` 建复合索引，用 `EXPLAIN ANALYZE` 对比 `WHERE user_id=?`、`WHERE status=?`、`WHERE user_id=? AND status=?` 三种查询的执行计划。
3. 实现一个 keyset 分页接口，与 `OFFSET` 分页在第 1 页、第 1000 页的耗时做对比，解释差异。

# 9. 小结

| 主题 | 要点 |
|------|------|
| 查询列 | 列表页只取需要的列，别总查实体 |
| 过滤 | 用 `is_(None)`、`in_()`、`and_`/`or_`，注意空列表 |
| 分页 | 深分页用 keyset，排序加唯一键 |
| 连接 | `join` 过滤，预加载取对象，注意一对多行放大 |
| 聚合 | 非聚合列必须进 `GROUP BY`，Top N 用窗口函数 |
| N+1 | 循环里触发查询就是信号，预加载或下推数据库 |
| 索引 | 复合索引看最左前缀，避免函数运算和前置通配符 |
| 连接池 | 按数据库上限估算，先治慢查询再加连接 |

下一篇是收尾，讲清楚项目分层、Session 生命周期、异常转换、异步边界、测试策略，以及一份可以直接照着落地的 SQLAlchemy 工程清单。
