事务解决的是"多条写操作要么全成功、要么全失败"的问题。SQLx 的事务 API 只有三个动作：`begin`、`commit`、`rollback`，但真正难的是想清楚**边界画在哪**、**错误路径有没有漏 commit**、**并发下锁拿得对不对**。

# 1. 事务的边界

## 1.1 转账为什么必须用事务

A 账户减 100、B 账户加 100，两条 UPDATE 必须同生共死：

```text
BEGIN
  ├─ UPDATE balance - 100  ← 成功
  ├─ UPDATE balance + 100  ← 失败
  └─ ROLLBACK：两条都撤销
```

★ **关键认知**：事务的边界应该包住**一个不可分割的业务动作**，而不是"整个请求"。一个请求里可能包含"创建订单"和"发通知"两件事，后者不在数据库事务里，也不该为了它把事务拉长。⚠ 事务里不要做 HTTP 请求、发消息、读大文件，长事务会持锁并占住连接，直接把池拖垮。

## 1.2 事务只覆盖同一个连接

事务里的多条语句必须跑在**同一条物理连接**上。调用支付接口、发 Kafka、写 Redis 都不在这条连接里，也就不在事务内。跨服务的一致性要靠本地消息表、事务消息或补偿方案。

# 2. 基本 API

## 2.1 begin / commit

```rust
use sqlx::PgPool;

async fn transfer(pool: &PgPool, from: i64, to: i64, amount: i64) -> Result<(), sqlx::Error> {
    let mut tx = pool.begin().await?;

    sqlx::query("UPDATE accounts SET balance = balance - $1 WHERE id = $2")
        .bind(amount).bind(from)
        .execute(&mut *tx).await?;

    sqlx::query("UPDATE accounts SET balance = balance + $1 WHERE id = $2")
        .bind(amount).bind(to)
        .execute(&mut *tx).await?;

    tx.commit().await?;
    Ok(())
}
```

## 2.2 为什么是 `&mut *tx`

`Transaction` 本身不是连接，它是对连接的包装。SQLx 的 `Executor` 实现给了 `&mut PgConnection`，所以执行时统一写 `.execute(&mut *tx)`：先解引用成连接，再取可变引用。

| 写法 | 结果 |
|------|------|
| `.execute(&mut *tx)` | 正确，语句在事务内 |
| `.execute(&mut tx)` | 编译错误，`Transaction` 未实现该 `Executor` |
| `.execute(pool)` | 语句在事务外，用的是另一条连接 |
| `.execute(&mut conn)` | 若 `conn` 不是事务来源的连接，也不在事务内 |

★ **关键认知**：事务的"边界"绑定在**那一条物理连接**上。任何绕过 `tx` 去用 `pool` 的语句，都会在另一条连接上自动提交，事务回滚时它不会撤销。⚠ 这类 bug 不会报错，只会表现为"明明回滚了，数据却还在"。

## 2.3 显式回滚

```rust
let mut tx = pool.begin().await?;

let result = sqlx::query("UPDATE accounts SET balance = balance - $1 WHERE id = $2")
    .bind(amount).bind(from).execute(&mut *tx).await?;

if result.rows_affected() == 0 {
    tx.rollback().await?; // 显式回滚，比依赖 drop 更可控
    return Err(sqlx::Error::RowNotFound);
}
tx.commit().await?;
```

★ **关键认知**：`commit` 和 `rollback` 都会消耗掉 `tx`。开了事务就一定要在每条返回路径上二选一：成功 `commit`，失败 `rollback`，或者交给 drop 兜底。

# 3. drop 会自动回滚吗

## 3.1 机制

假设执行了一条 INSERT 后没有 `commit` 就返回，`tx` 在离开作用域时被 drop。`Transaction` 的 `Drop` 实现会安排一次回滚：把连接放回池子的过程中执行 `ROLLBACK`。所以"忘记 commit"通常不会造成脏数据。

★ **关键认知**：drop 触发的回滚是**尽力而为**，不是保证。如果连接已经断开、网络已中断、进程被 `kill -9`，回滚语句发不出去，数据库会在连接超时后自行回滚。⚠ 对关键业务，宁可显式 `tx.rollback().await?` 并记录结果，也不要完全依赖 drop。

## 3.2 用 `?` 提前返回

```rust
let mut tx = pool.begin().await?;
sqlx::query("INSERT INTO orders (id) VALUES ($1)")
    .bind(1_i64)
    .execute(&mut *tx)
    .await?;

do_something_fallible().await?; // 返回 Err 时 tx drop，自动回滚
tx.commit().await?;
```

`?` 提前返回时，`tx` 离开作用域并 drop，回滚被安排。这是 SQLx 里最常见、也最容易读懂的写法。

## 3.3 常见误区

```rust
// 错误示范：开了事务却用 pool，insert_a 不在事务里
let mut tx = pool.begin().await?;
insert_a(pool).await?;
insert_b(&mut tx).await?;
tx.commit().await?;
```

⚠ 这类问题代码审查时很难看出来，建议在 repo 层约定：**开了事务之后，函数签名只接收 `&mut PgConnection`，不接收 `&PgPool`**，从类型上堵住误用。

# 4. 嵌套与 SavePoint

## 4.1 SQLx 没有自动嵌套事务

SQLx 0.8 没有"事务里再 begin 一个子事务"的 API。需要部分回滚时，直接写 SQL 的保存点：

```rust
let mut tx = pool.begin().await?;

sqlx::query("INSERT INTO orders (id) VALUES ($1)")
    .bind(order_id)
    .execute(&mut *tx)
    .await?;

sqlx::query("SAVEPOINT before_items").execute(&mut *tx).await?;

if insert_items(&mut tx, order_id).await.is_err() {
    // 只撤销明细，订单主记录保留
    sqlx::query("ROLLBACK TO SAVEPOINT before_items")
        .execute(&mut *tx)
        .await?;
} else {
    sqlx::query("RELEASE SAVEPOINT before_items")
        .execute(&mut *tx)
        .await?;
}

tx.commit().await?; // 外层仍然掌握最终提交权
```

## 4.2 SavePoint 语义

| 操作 | 作用 |
|------|------|
| `SAVEPOINT name` | 在当前事务里打标记 |
| `ROLLBACK TO SAVEPOINT name` | 回滚到标记，事务继续 |
| `RELEASE SAVEPOINT name` | 删除标记，释放资源 |
| `COMMIT` | 才真正落盘 |

★ **关键认知**：SavePoint 不是独立事务，它没有自己的提交能力。内层"回滚"只是撤销标记之后的语句，事务最终是否生效仍由最外层 `commit` 决定。⚠ SavePoint 用多了会让事务状态变复杂，且部分数据库对嵌套深度有限制，能用"拆分业务单元"解决就别嵌套。

## 4.3 更好的做法：把子步骤独立成事务

如果"明细失败也要保留订单"是业务允许的，往往说明这两件事本来就不该强绑在一个事务里。拆成两个事务，用状态字段表达中间态，比 SavePoint 更清晰。

# 5. 并发更新

## 5.1 先查后改的竞态

两个事务同时读到 `stock = 10`，各自扣 1 后写回 9，最终库存只少了 1，这就是丢失更新。三种常见解法：悲观锁、乐观锁、原子更新。

## 5.2 悲观锁：FOR UPDATE

```rust
let mut tx = pool.begin().await?;

// 锁住这一行，直到事务结束
let (stock,): (i64,) =
    sqlx::query_as("SELECT stock FROM products WHERE id = $1 FOR UPDATE")
        .bind(product_id)
        .fetch_one(&mut *tx)
        .await?;

if stock < qty {
    tx.rollback().await?;
    return Err(sqlx::Error::RowNotFound);
}

sqlx::query("UPDATE products SET stock = stock - $1 WHERE id = $2")
    .bind(qty)
    .bind(product_id)
    .execute(&mut *tx)
    .await?;

tx.commit().await?;
```

悲观锁的代价是**持锁时间长**：高并发下请求排队，事务里再有慢操作就容易超时甚至死锁。

## 5.3 乐观锁：版本号

```rust
let result = sqlx::query(
    "UPDATE products SET stock = stock - $1, version = version + 1
     WHERE id = $2 AND version = $3",
)
.bind(qty).bind(product_id).bind(expected_version)
.execute(pool).await?;

if result.rows_affected() == 0 {
    return Err(sqlx::Error::RowNotFound); // 版本冲突，交给上层重试
}
```

## 5.4 原子更新：首选

```rust
let result = sqlx::query(
    "UPDATE products SET stock = stock - $1 WHERE id = $2 AND stock >= $1",
)
.bind(qty).bind(product_id)
.execute(pool).await?;

if result.rows_affected() == 0 {
    return Err(sqlx::Error::RowNotFound); // 库存不足或商品不存在
}
```

## 5.5 三种方案对比

| 方案 | 加锁方式 | 冲突处理 | 适合 |
|------|----------|----------|------|
| 原子更新 | 数据库行锁（隐式） | `rows_affected == 0` | 简单计数、库存扣减 |
| 悲观锁 | `FOR UPDATE` | 排队等待 | 事务内多步依赖同一行 |
| 乐观锁 | 版本号 CAS | 失败重试 | 读多写少、冲突不频繁 |

★ **关键认知**：能用一条原子 SQL 解决，就不要开事务加锁。`UPDATE ... WHERE stock >= $1` 既短又不会死锁，是扣减场景的首选。⚠ 原子更新只适合"单行、单条件"的判断；一旦业务要先读多行再算总价，就得上事务加锁或乐观锁。

# 6. 隔离级别

## 6.1 四种级别与异常

| 级别 | 脏读 | 不可重复读 | 幻读 |
|------|------|------------|------|
| Read Uncommitted | 可能 | 可能 | 可能 |
| Read Committed | 不可能 | 可能 | 可能 |
| Repeatable Read | 不可能 | 不可能 | 可能（InnoDB 用间隙锁基本避免） |
| Serializable | 不可能 | 不可能 | 不可能 |

三个异常分别是：脏读（读到未提交数据）、不可重复读（同一事务内两次读同一行结果不同）、幻读（两次范围查询行数不同）。

## 6.2 在 SQLx 中设置

```rust
let mut tx = pool.begin().await?;

// PostgreSQL：在事务内、任何查询之前设置
sqlx::query("SET TRANSACTION ISOLATION LEVEL REPEATABLE READ")
    .execute(&mut *tx)
    .await?;

// ... 业务查询

tx.commit().await?;
```

MySQL 的隔离级别必须在事务开始**之前**设置，例如先执行 `SET SESSION TRANSACTION ISOLATION LEVEL REPEATABLE READ`，再 `begin`。

| 数据库 | 默认隔离级别 |
|--------|--------------|
| PostgreSQL | Read Committed |
| MySQL InnoDB | Repeatable Read |
| SQL Server | Read Committed |

★ **关键认知**：隔离级别是**数据库连接级**的状态，不是你调用一次就全局生效。用 `SET SESSION` 改的是当前连接，连接归还池子后可能被下一个请求复用。⚠ 需要固定隔离级别时，在事务内用 `SET TRANSACTION`（仅对当前事务生效），或用 `after_connect` 统一设置并接受全局影响。Serializable 下冲突会以序列化失败（PostgreSQL `40001`）暴露，应用必须重试。

# 7. 常见错误

| 现象 | 原因 | 解决 |
|------|------|------|
| 回滚了数据还在 | 语句用了 `pool` 而非 `tx` | 事务内只传 `&mut PgConnection` |
| 忘记提交，数据没写入 | 某条路径直接 return | 检查所有分支，必要时显式 rollback |
| `PoolTimedOut` 增多 | 事务里有慢查询或外部调用 | 缩短事务，设语句超时 |
| 死锁 `40P01` | 多事务加锁顺序不一致 | 统一按主键顺序更新 |
| 嵌套事务没生效 | SQLx 没有嵌套 API | 用 `SAVEPOINT` 或拆分事务 |
| 隔离级别设置无效 | 用了 `SET SESSION` 后被复用 | 改 `SET TRANSACTION` |

# 8. 动手练习

1. 实现转账，要求余额不足返回业务错误、任何一步失败整体回滚，并写一个并发测试验证总金额不变。
2. 用 `SAVEPOINT` 实现"批量插入明细，某条失败只跳过该条"，最后统计成功与失败数量。
3. 用 1000 个并发任务抢 100 件库存，分别用"先查后改""FOR UPDATE""原子更新"三种方式实现，记录最终库存与耗时。

验收标准：第 1 题总金额守恒；第 3 题三种方案最终库存都应为 0，且你能解释哪种最快、为什么。

# 9. 小结

| 主题 | 要点 |
|------|------|
| 边界 | 包住一个不可分割的业务动作，不放外部调用 |
| 执行 | 事务内一律 `&mut *tx`，不要用 `pool` |
| 提交 | `tx.commit().await?` 才落盘 |
| 回滚 | drop 尽力回滚，关键路径显式 rollback |
| 嵌套 | 无自动嵌套 API，用 `SAVEPOINT` |
| 并发 | 优先原子更新，其次乐观锁，最后悲观锁 |
| 隔离级别 | 连接级状态，事务内用 `SET TRANSACTION` |

下一篇进入错误处理，讲 `sqlx::Error` 的各个变体、如何识别唯一键冲突与外键冲突、`RowNotFound` 与 `Option` 的取舍、池错误与连接错误的重试边界，以及怎么把它们映射成业务错误。
