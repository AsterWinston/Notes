Session 是 SQLAlchemy ORM 的核心入口，也是最容易被误用的部分。很多人把它当成"数据库连接"随手创建、随手关闭，结果遇到对象过期、连接泄漏、批量插入慢几十倍的问题。理解 Session 的"工作单元"语义，CRUD 才会写得既正确又高效。

# 1. Session 是什么

## 1.1 工作单元与身份映射

Session 不是连接，而是一个**工作单元（Unit of Work）**：它记录你新增、修改、删除了哪些对象，在 flush 时统一翻译成 SQL，在 commit 时统一提交。

它内部维护两张关键的表：

| 结构 | 作用 |
|------|------|
| Identity Map | 主键 → 对象实例，保证同一会话内同一行只有一个对象 |
| Dirty Set | 被修改过的持久态对象，flush 时据此生成 UPDATE |

```python
with Session(engine) as session:
    a = session.get(User, 1)
    b = session.get(User, 1)
    print(a is b)          # True，第二次直接命中身份映射，不发 SQL
```

★ **关键认知**：身份映射带来两个直接后果。第一，同一个 Session 里修改对象，不需要 `session.update()`，flush 时会自动比较快照生成 UPDATE。第二，**跨 Session 的对象不能直接复用**，因为它绑定在另一个身份映射上。

## 1.2 用 sessionmaker 管理配置

```python
from sqlalchemy.orm import sessionmaker

SessionLocal = sessionmaker(
    bind=engine,
    autoflush=False,          # 手动控制 flush 时机，行为更可预测
    expire_on_commit=False,   # commit 后对象属性仍可读，适合 API 返回
)
```

`sessionmaker` 是一个工厂。全局只创建一次，每次请求调用 `SessionLocal()` 得到新会话。

```python
def get_db():
    db = SessionLocal()
    try:
        yield db
    finally:
        db.close()             # 归还连接，不是关闭物理连接
```

## 1.3 Session 与 Connection 的区别

Connection 属于 Core，手动管理事务，返回 Row；Session 属于 ORM，内部管理事务，返回映射对象并维护身份映射。一个 Session 在需要时会向 Engine 借连接，commit 或 close 后归还。**Session 可以比连接活得久**，但不应跨请求、跨线程共享。

# 2. 对象状态

## 2.1 五种状态

```text
                 add()                flush()              commit()
transient ─────────────▶ pending ─────────────▶ persistent ─────────▶ persistent
    │                       │                       │                    │
    │                       │                       │ delete()           │ close()/expunge()
    │                       │                       ▼                    ▼
    │                       │                    deleted              detached
    │                       │                       │
    └───────────────────────┴───────────────────────┴──▶ 未入库即回滚，回到 transient
```

| 状态 | 有主键 | 在 Session 中 | 有数据库行 | 典型操作 |
|------|--------|--------------|-----------|---------|
| transient | 可能没有 | 否 | 否 | `User(name="a")` |
| pending | 可能没有 | 是 | 否 | `session.add(obj)` |
| persistent | 有 | 是 | 是 | 查询或 commit 后 |
| deleted | 有 | 是 | 待删除 | `session.delete(obj)` |
| detached | 有 | 否 | 是 | `session.close()` 后 |

## 2.2 状态转换的实际影响

`add()` 让对象进入 pending，`flush()` 后变 persistent 并拿到自增 id，`commit()` 提交事务，`close()` 后变 detached。

⚠ `expire_on_commit=True`（默认）下，commit 会让对象属性过期。关闭会话后再访问未加载的属性，会抛 `DetachedInstanceError`。两种解法：在会话内读完需要的字段，或设置 `expire_on_commit=False`。

# 3. 增删改查

## 3.1 新增

```python
with Session(engine) as session, session.begin():
    user = User(name="alice", email="alice@example.com")
    session.add(user)            # 单个
    session.add_all([            # 多个
        User(name="bob", email="bob@example.com"),
        User(name="carol", email="carol@example.com"),
    ])
# 离开 with 时自动 commit；异常则自动 rollback
```

`session.begin()` 让"提交/回滚"由上下文管理器负责，比手写 `try/except/rollback` 更不容易漏。

## 3.2 查询

```python
from sqlalchemy import select

# 按主键取，优先命中身份映射
user = session.get(User, 1)

# 条件查询
stmt = select(User).where(User.email == "alice@example.com")
user = session.execute(stmt).scalar_one_or_none()

# 取多个标量对象
users = session.scalars(select(User).where(User.name.like("a%"))).all()

# 只取部分列
rows = session.execute(select(User.id, User.name)).all()
```

| 写法 | 返回 | 适用 |
|------|------|------|
| `session.get(User, pk)` | 实例或 `None` | 主键查询，最快 |
| `session.scalars(stmt).all()` | 实例列表 | 查实体 |
| `session.execute(stmt).all()` | Row 列表 | 查部分列或多表 |
| `session.scalar(stmt)` | 单值 | `count`、`max` |

## 3.3 修改

```python
with Session(engine) as session, session.begin():
    user = session.get(User, 1)
    user.name = "alice2"          # 直接改属性，无需 update()
# commit 时自动生成 UPDATE users SET name = ? WHERE id = ?
```

ORM 的更新是**脏检查**：flush 时把对象当前值与加载时的快照比较，只更新变化的列。

批量更新则用 Core 表达式，避免逐个加载对象：

```python
from sqlalchemy import update

stmt = (
    update(User)
    .where(User.created_at < cutoff)
    .values(status="inactive")
)
session.execute(stmt)
```

⚠ 批量 `update()` 默认绕过身份映射。如果会话里已经加载了这些对象，它们的属性仍是旧值。需要同步时传 `execution_options(synchronize_session="fetch")`，或更新后 `expire_all()`。

## 3.4 删除

```python
with Session(engine) as session, session.begin():
    user = session.get(User, 1)
    if user is not None:
        session.delete(user)      # 标记为 deleted，flush 时 DELETE
```

批量删除：

```python
from sqlalchemy import delete

session.execute(delete(User).where(User.status == "inactive"))
```

★ **关键认知**：`session.delete()` 会先加载对象、再触发 ORM 级联；`delete()` 语句直接发 SQL，**不会触发 ORM 级联和事件**。需要清理关联数据时，要么让数据库外键 `ON DELETE CASCADE` 负责，要么先手动处理。

# 4. flush 与 commit

## 4.1 两者的分工

| 操作 | 发送 SQL | 结束事务 | 对象状态 | 可回滚 |
|------|---------|---------|---------|--------|
| `flush()` | 是 | 否 | pending → persistent | 是 |
| `commit()` | 先自动 flush | 是 | 保持 persistent | 否 |
| `rollback()` | 否 | 是 | 撤销未提交变更 | —— |

```python
with Session(engine) as session:
    user = User(name="alice")
    session.add(user)
    session.flush()               # 已 INSERT，user.id 可读
    print(user.id)                # 例如 1
    session.rollback()            # 撤销，数据库里没有这条记录
```

## 4.2 autoflush 的陷阱

默认 `autoflush=True`：每次查询前会自动 flush，保证查询能看到未提交的变更。

```python
session.add(User(name="alice"))
# 下面这次查询会先触发 INSERT
count = session.scalar(select(func.count()).select_from(User))
```

这通常符合直觉，但会在你没想到的位置发 SQL，导致：

- 查询前出现意外的 `NOT NULL` 报错；
- 批量循环中频繁 flush，性能骤降；
- 调试时 SQL 顺序难以理解。

需要精确控制时设 `autoflush=False`，在合适的位置显式 `flush()`。

## 4.3 expire_on_commit

```python
SessionLocal = sessionmaker(bind=engine, expire_on_commit=False)
```

设为 `False` 后，commit 不会让对象属性过期，适合"提交后立刻序列化返回"的接口。代价是对象可能持有**提交前的旧值**，在长会话里容易读到过期数据。Web 请求级别的短会话通常可以安全关闭它。

# 5. 批量操作

## 5.1 add_all 仍然是逐条 INSERT

```python
session.add_all([User(name=f"u{i}") for i in range(1000)])
session.commit()
```

默认情况下 ORM 会为每个对象生成一条 INSERT。虽然有参数批处理优化，但对象构造、状态跟踪的开销仍然存在。**上万行导入不要用 `add_all`**。

## 5.2 executemany 批量插入

```python
from sqlalchemy import insert

with Session(engine) as session, session.begin():
    session.execute(
        insert(User),
        [
            {"name": "u1", "email": "u1@example.com"},
            {"name": "u2", "email": "u2@example.com"},
            {"name": "u3", "email": "u3@example.com"},
        ],
    )
```

第二条参数是字典列表，驱动会使用 executemany，显著减少往返次数。

## 5.3 批量更新与删除

```python
# 批量更新
session.execute(
    update(Product).where(Product.stock == 0).values(status="sold_out")
)

# 批量删除
session.execute(delete(Log).where(Log.created_at < cutoff))
```

## 5.4 用 RETURNING 拿回生成值

```python
stmt = insert(User).returning(User.id, User.created_at)
rows = session.execute(
    stmt,
    [{"name": "u1", "email": "u1@example.com"}],
).all()
```

PostgreSQL 与较新的 SQLite 支持 `RETURNING`，可以一次拿到自增主键和数据库默认值，避免插入后再逐条查询。

★ **关键认知**：判断该用哪种批量方式的简单规则——**需要 ORM 事件、级联、对象后续操作时用 `add_all`；纯数据导入用 `insert().values()` 或 executemany；大规模清洗用 Core 语句或数据库原生工具**。三者性能可能相差一个数量级。

# 6. 常见错误

| 现象 | 原因 | 解决 |
|------|------|------|
| `DetachedInstanceError` | 会话已关闭，属性又过期 | 会话内取完数据，或 `expire_on_commit=False` |
| 改了对象没落库 | 忘记 commit，或会话提前关闭 | 用 `with session.begin()` |
| 批量插入极慢 | 循环里逐条 `add` + `commit` | 一次事务 + executemany |
| 批量更新后对象还是旧值 | 绕过了身份映射 | `synchronize_session` 或 `expire_all()` |
| 同一行出现两个对象 | 跨 Session 或用了 `merge` 不当 | 一个请求一个 Session |
| 查询前意外报 NOT NULL | autoflush 提前写入半成品对象 | 先补齐字段，或 `autoflush=False` |

# 7. 动手练习

1. 用同一个 Session 连续 `get` 两次同一个主键，打开 `echo=True` 观察 SQL 条数，解释身份映射的作用。
2. 写一个导入脚本，分别用 `add_all` 和 `insert().values()` 插入 5000 行，记录耗时并解释差异。
3. 复现一次 `DetachedInstanceError`：在会话内查询对象，关闭会话后访问一个未加载的关联属性，然后用 `selectinload` 预加载解决它。

# 8. 小结

| 主题 | 要点 |
|------|------|
| Session | 工作单元，持有身份映射与脏数据集合，不是连接 |
| 状态 | transient → pending → persistent → detached，删除走 deleted |
| 查询 | 主键用 `get`，条件用 `select` + `scalars` |
| 修改 | 改属性即可，flush 时脏检查生成 UPDATE |
| flush/commit | flush 发 SQL 不结束事务，commit 提交并默认过期对象 |
| 批量 | 数据导入用 executemany，业务对象用 `add_all` |
| 生命周期 | 一个请求一个 Session，`with` 保证归还连接 |

下一篇进入关系映射，讲清楚一对多、多对多和一对一的外键落点、`back_populates` 的双向维护、`lazy` 策略的选择，以及 `selectinload` 如何把 N+1 压成两条 SQL。
