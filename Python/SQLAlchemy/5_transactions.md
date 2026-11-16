事务是数据库正确性的最后一道防线。SQLAlchemy 把事务藏在 Session 后面，用起来几乎无感，但"无感"也意味着边界不清：一次 commit 到底提交了什么、嵌套事务是不是真嵌套、并发扣库存为什么会少卖，这些问题不搞清就会在线上付出代价。

# 1. 事务边界

## 1.1 事务要保证什么

事务要保证原子性（全成功或全回滚）、一致性（约束始终成立）、隔离性（并发互不干扰到约定程度）和持久性（提交后不丢）。SQLAlchemy 不自己实现这些，它把 `BEGIN` / `COMMIT` / `ROLLBACK` 翻译给数据库。**隔离级别、锁、崩溃恢复最终都由数据库决定**，ORM 只负责边界和对象状态。

## 1.2 用 session.begin() 划定边界

```python
from sqlalchemy.orm import Session

with Session(engine) as session:
    with session.begin():
        session.add(User(name="alice"))
        session.add(Order(user_id=1, amount=100))
# 正常结束：commit
# 抛异常：rollback，异常继续向上传播
```

★ **关键认知**：`session.begin()` 的价值不是"开启事务"——SQLAlchemy 2.x 在第一次执行语句时会**自动开启（autobegin）**——而是**明确划出提交/回滚的边界**，并保证异常时一定回滚。没有它，你很容易在异常路径上漏掉 `rollback()`，把连接留在失败事务里。

## 1.3 autobegin 的实际行为

```python
session = Session(engine)
session.add(User(name="alice"))     # 此时还没有发 SQL
session.execute(select(User))       # 自动 BEGIN + flush + SELECT
session.commit()                    # COMMIT

session.execute(select(User))       # 又自动 BEGIN
session.close()                     # 未提交的事务被回滚，连接归还
```

`close()` 不会提交。**忘记 commit 就 close，数据会静默丢失**，而且不会报错。

# 2. 嵌套事务

## 2.1 begin_nested 是 SAVEPOINT

```python
with Session(engine) as session, session.begin():
    session.add(User(name="outer"))

    try:
        with session.begin_nested():        # SAVEPOINT sp_1
            session.add(User(name="inner"))
            raise ValueError("模拟失败")
    except ValueError:
        pass                                # 只回滚到 SAVEPOINT

    session.add(User(name="after"))
# 最终提交：outer 和 after 落库，inner 没有
```

生成的 SQL 顺序：

```sql
BEGIN;
INSERT INTO users (name) VALUES ('outer');
SAVEPOINT sp_1;
INSERT INTO users (name) VALUES ('inner');
ROLLBACK TO SAVEPOINT sp_1;
INSERT INTO users (name) VALUES ('after');
COMMIT;
```

★ **关键认知**：`begin_nested()` 不是"开启一个独立事务"，而是**在当前事务里打一个保存点**。保存点内的回滚只撤销它之后的操作，外层事务仍然可以继续。**如果外层最终回滚，保存点内已"成功"的部分同样会消失**。

## 2.2 用保存点隔离可恢复的错误

批量导入时，单条记录违反唯一约束不应导致整批失败：

```python
with Session(engine) as session, session.begin():
    for row in rows:
        try:
            with session.begin_nested():
                session.add(User(**row))
        except IntegrityError:
            continue          # 这一条回滚到保存点，循环继续
```

⚠ 发生 `IntegrityError` 后，**必须先回滚到保存点再继续使用 Session**。否则事务已处于失败状态，后续任何语句都会报 `current transaction is aborted`（PostgreSQL）或类似错误。

⚠ 不要在 `begin_nested()` 里调用 `session.commit()`。那会提交整个外层事务，保存点语义被破坏。`session.rollback()` 则回滚整个事务，包括所有保存点。

# 3. 隔离级别

## 3.1 四种并发异常

| 异常 | 现象 | 例子 |
|------|------|------|
| 脏读 | 读到别人未提交的数据 | 读到回滚前的余额 |
| 不可重复读 | 同一事务内两次读同一行结果不同 | 第一次 100，第二次 150 |
| 幻读 | 同一查询两次返回的行数不同 | 第一次 3 条，第二次 4 条 |
| 丢失更新 | 两个事务覆盖彼此的写 | 两人同时扣库存，只扣了一次 |

## 3.2 设置隔离级别

```python
# 整个 Engine 统一设置
engine = create_engine(url, isolation_level="REPEATABLE READ")

# 单个连接
with engine.connect() as conn:
    conn = conn.execution_options(isolation_level="SERIALIZABLE")
    ...

# 单个 Session
session = Session(bind=engine.execution_options(isolation_level="SERIALIZABLE"))
```

## 3.3 主流数据库的默认值

| 数据库 | 默认级别 | 特点 |
|--------|---------|------|
| PostgreSQL | READ COMMITTED | RR 已能防幻读，SERIALIZABLE 用 SSI |
| MySQL InnoDB | REPEATABLE READ | 用间隙锁防幻读，容易死锁 |
| SQLite | SERIALIZABLE（实际串行写） | 单写者模型 |
| SQL Server | READ COMMITTED | 可选快照隔离 |

★ **关键认知**：**隔离级别不是越高越好**。SERIALIZABLE 会把冲突变成序列化失败，应用必须能重试；MySQL 的 RR 在高并发写入下更容易死锁。多数互联网业务用 READ COMMITTED + 应用层幂等/补偿，关键账务再单独提高级别。

# 4. 并发更新

## 4.1 丢失更新是怎么发生的

```text
库存初始 10，两个请求各买 1 件：

事务 A                          事务 B
BEGIN;                          BEGIN;
SELECT stock FROM p WHERE id=1;  -- 读到 10
                                SELECT stock FROM p WHERE id=1;  -- 也读到 10
UPDATE p SET stock=9 WHERE id=1;
COMMIT;
                                UPDATE p SET stock=9 WHERE id=1;  -- 覆盖成 9
                                COMMIT;
最终库存 9，卖出 2 件，少卖 1 件
```

## 4.2 悲观锁：SELECT FOR UPDATE

```python
from sqlalchemy import select

with Session(engine) as session, session.begin():
    product = session.scalars(
        select(Product).where(Product.id == 1).with_for_update()
    ).one()
    if product.stock < 1:
        raise ValueError("库存不足")
    product.stock -= 1
# commit 时释放行锁
```

`with_for_update()` 生成 `SELECT ... FOR UPDATE`，锁定选中行，直到事务结束。其他事务的同一行锁请求会阻塞。

常用选项：

```python
# 锁已被占用时立即失败，而不是等待
select(Product).where(...).with_for_update(nowait=True)

# 任务队列：跳过别人正在处理的行
select(Job).where(Job.status == "pending").limit(10).with_for_update(skip_locked=True)

# 只锁指定表
select(User).join(Order).with_for_update(of=User)
```

| 选项 | 作用 | 适用 |
|------|------|------|
| 默认 | 阻塞等待 | 普通扣减 |
| `nowait=True` | 拿不到锁立刻报错 | 快速失败、避免排队 |
| `skip_locked=True` | 跳过已锁行 | 多消费者抢任务 |
| `of=Model` | 只锁指定表 | 多表 JOIN |

⚠ `FOR UPDATE` 必须在事务内才有意义。自动提交模式下语句结束锁就释放，等于没锁。

## 4.3 乐观锁：版本号

悲观锁会阻塞，吞吐受限。乐观锁假设冲突很少，提交时再检查版本：

```python
class Product(Base):
    __tablename__ = "products"

    id: Mapped[int] = mapped_column(primary_key=True)
    stock: Mapped[int] = mapped_column(default=0)
    version: Mapped[int] = mapped_column(default=1, nullable=False)

    __mapper_args__ = {"version_id_col": version}
```

配置 `version_id_col` 后，ORM 自动维护版本号：

```sql
UPDATE products SET stock = 9, version = 2
WHERE id = 1 AND version = 1;
```

如果影响行数为 0，说明别人已经改过，SQLAlchemy 抛 `StaleDataError`：

```python
from sqlalchemy.orm.exc import StaleDataError

try:
    with Session(engine) as session, session.begin():
        product = session.get(Product, 1)
        product.stock -= 1
except StaleDataError:
    print("数据已被其他事务修改，请重试")
```

## 4.4 两种锁的取舍

| 维度 | 悲观锁 | 乐观锁 |
|------|--------|--------|
| 加锁时机 | 读取时 | 提交时 |
| 冲突处理 | 阻塞等待 | 抛异常后重试 |
| 吞吐 | 低冲突时较差 | 低冲突时很好 |
| 高冲突 | 排队但确定成功 | 大量重试 |
| 实现 | `with_for_update()` | `version_id_col` |
| 适用 | 库存扣减、余额 | 内容编辑、配置更新 |

★ **关键认知**：**乐观锁不是"不加锁"，而是把冲突检测推到写语句**。它依赖 `WHERE version = ?` 的原子性，因此必须由数据库执行，不能用"先查版本再比较"的应用代码代替。

⚠ 重试前必须确认操作**幂等**，或确保失败的事务没有产生外部副作用（发短信、扣第三方余额）。在事务里调用外部接口，是重试逻辑最常见的破坏源。

# 5. 事务中的常见陷阱

## 5.1 长事务

事务从 BEGIN 到 COMMIT 之间持有锁和快照。中间做网络请求、等待用户输入、跑大数据计算，都会让锁持有时间失控。

```python
# 错误：HTTP 请求放在事务里
with Session(engine) as session, session.begin():
    order = create_order(session)
    requests.post("https://payment.example.com/charge", json=...)  # 可能耗时数秒
    order.status = "paid"
```

正确做法是**先提交本地状态，再调用外部服务，最后用幂等键回写结果**。

## 5.2 吞掉异常导致事务悬挂

```python
# 错误：异常被吞，事务仍处于失败状态
try:
    session.execute(text("INSERT INTO users (id) VALUES (1)"))
except Exception:
    pass
session.add(User(id=2))       # 可能报事务已中止
```

捕获异常后要么回滚整个事务，要么回滚到保存点，不能原地继续。

## 5.3 连接池与事务

连接在事务期间被占用。长事务会迅速耗尽 `pool_size + max_overflow`，表现为请求排队、`pool_timeout` 超时。监控连接池时，**先看有没有长事务，再看池大小是否真的不够**。

# 6. 常见错误

| 现象 | 原因 | 解决 |
|------|------|------|
| 数据没保存但没报错 | 忘记 commit 就 close | 用 `with session.begin()` |
| `current transaction is aborted` | 异常后未回滚就继续用 Session | 回滚到保存点或整个事务 |
| 扣库存少卖 | 读后写没有锁 | `FOR UPDATE` 或版本号 |
| `StaleDataError` | 乐观锁版本冲突 | 捕获并重试 |
| 保存点回滚后外层也丢了 | 外层最终 rollback | 理解保存点从属于外层事务 |
| 高峰期连接池耗尽 | 长事务占连接 | 缩短事务、外呼移出事务 |
| 死锁 | 多事务加锁顺序不一致 | 固定顺序、缩短事务、重试 |

# 7. 动手练习

1. 开两个 Session 复现丢失更新：都读到 `stock=10`，各自写回 9。再用 `with_for_update()` 改写，观察第二个事务是否阻塞。
2. 用 `begin_nested()` 实现"批量导入，坏数据跳过"，故意插入一条重复邮箱，确认其余记录仍能提交。
3. 给 `Product` 加 `version_id_col`，用两个 Session 同时修改同一行，捕获 `StaleDataError` 并实现三次指数退避重试。

# 8. 小结

| 主题 | 要点 |
|------|------|
| 边界 | `with session.begin()` 负责提交/回滚，close 不提交 |
| autobegin | 首次执行语句自动开启事务，忘记 commit 会静默丢数据 |
| 嵌套 | `begin_nested()` 是 SAVEPOINT，从属于外层事务 |
| 隔离 | 越高越安全但冲突越多，按业务选择并配套重试 |
| 悲观锁 | `with_for_update()`，事务结束释放 |
| 乐观锁 | `version_id_col`，冲突抛 `StaleDataError` |
| 长事务 | 持锁、占连接、阻碍清理，外部调用必须移出事务 |

下一篇进入 Alembic 迁移，讲清楚 `alembic init` 生成了什么、`autogenerate` 能识别哪些变更、生产环境为什么不能直接 `upgrade head`，以及数据迁移和回滚脚本该怎么写。
