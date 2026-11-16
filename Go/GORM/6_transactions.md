事务解决的是"多条写操作要么全成功、要么全失败"的问题。GORM 提供了自动事务和手动事务两套 API，但真正难的不是调用哪个方法，而是想清楚**事务边界画在哪**、**锁有没有拿对**、**回滚之后错误有没有被吞掉**。

# 1. 事务基础

## 1.1 为什么需要事务

转账是经典例子：A 账户减 100，B 账户加 100。这两条 UPDATE 必须同生共死：

```text
┌──────────────┐
│ BEGIN        │
└──────┬───────┘
       ▼
┌──────────────────────────┐
│ UPDATE accounts          │
│ SET balance = balance-100│  ← 第 1 步成功
│ WHERE id = 1             │
└──────┬───────────────────┘
       ▼
┌──────────────────────────┐
│ UPDATE accounts          │
│ SET balance = balance+100│  ← 第 2 步失败
│ WHERE id = 2             │
└──────┬───────────────────┘
       ▼
┌──────────────┐   ┌──────────────┐
│ ROLLBACK     │   │ COMMIT       │
│ 第 1 步也撤销 │   │ 全部生效      │
└──────────────┘   └──────────────┘
```

★ **关键认知**：事务只保证"在同一个数据库连接上"的原子性。跨库、跨服务、发消息、调第三方接口，都不在事务范围内——这些要靠本地消息表、最终一致性方案解决，不能指望 `db.Transaction`。

## 1.2 GORM 的默认事务

GORM 默认给**单条写操作**也包了一层事务（`SkipDefaultTransaction: false`）：

```go
db.Create(&user) // 内部：BEGIN → INSERT → COMMIT
```

这层事务对单条 INSERT 并无必要，因为单条语句本身就是原子的。生产环境常开：

```go
db, err := gorm.Open(dsn, &gorm.Config{
    SkipDefaultTransaction: true, // 关闭单条写的默认事务，提升写入吞吐
})
```

⚠ 关闭后，`Create` 关联对象（一次插入主表 + 多张关联表）就**不再自动原子**。有多个写语句时必须用显式事务。

# 2. Transaction 自动事务

## 2.1 基本用法

```go
err := db.Transaction(func(tx *gorm.DB) error {
    // 用 tx，不要用 db
    if err := tx.Create(&order).Error; err != nil {
        return err // 返回非 nil 即回滚
    }
    if err := tx.Model(&account).
        Where("id = ?", fromID).
        UpdateColumn("balance", gorm.Expr("balance - ?", amount)).Error; err != nil {
        return err
    }
    if err := tx.Model(&account).
        Where("id = ?", toID).
        UpdateColumn("balance", gorm.Expr("balance + ?", amount)).Error; err != nil {
        return err
    }
    return nil // 返回 nil 才提交
})
if err != nil {
    return fmt.Errorf("转账失败: %w", err)
}
```

★ **关键认知**：`Transaction` 的返回值是唯一的开关——**返回 `nil` 提交，返回任何非 nil 回滚**。回调内部对 `db` 的调用不属于这个事务，必须全程用参数里的 `tx`。

## 2.2 错误处理

```go
// 错误示范：吞掉了错误，事务被提交
db.Transaction(func(tx *gorm.DB) error {
    tx.Create(&order) // 没检查 Error
    return nil        // 即使 Create 失败也会提交
})

// 正确：每个可能失败的操作都检查
db.Transaction(func(tx *gorm.DB) error {
    if err := tx.Create(&order).Error; err != nil {
        return err
    }
    return nil
})
```

## 2.3 panic 也会回滚

```go
db.Transaction(func(tx *gorm.DB) error {
    defer func() {
        if r := recover(); r != nil {
            // 在这里记录堆栈，然后继续 panic 让 GORM 回滚
            log.Printf("事务 panic: %v", r)
            panic(r)
        }
    }()

    if err := tx.Create(&order).Error; err != nil {
        return err
    }
    doSomethingThatMayPanic() // 若 panic，GORM 会 Rollback 并继续抛
    return nil
})
```

⚠ 不要在事务回调里 `recover` 之后返回 `nil`。那样 GORM 会认为业务成功并提交一个"只执行了一半"的事务，这是最隐蔽的数据损坏来源。

# 3. 手动事务

## 3.1 Begin / Commit / Rollback

```go
tx := db.Begin()
if tx.Error != nil {
    return tx.Error
}

// defer 兜底：无论从哪里返回，只要没提交就回滚
defer func() {
    if r := recover(); r != nil {
        tx.Rollback()
        panic(r)
    }
}()

if err := tx.Create(&order).Error; err != nil {
    tx.Rollback()
    return err
}
if err := tx.Model(&Account{}).Where("id = ?", 1).
    UpdateColumn("balance", gorm.Expr("balance - ?", 100)).Error; err != nil {
    tx.Rollback()
    return err
}

return tx.Commit().Error
```

## 3.2 自动与手动的取舍

| 维度 | `db.Transaction` | 手动 `Begin` |
|------|------------------|--------------|
| 代码量 | 少 | 多 |
| panic 回滚 | 自动 | 需自己 `defer recover` |
| 控制粒度 | 只能整体提交/回滚 | 可在中途做复杂判断 |
| 易错点 | 忘记用 `tx` | 忘记 `Rollback`、重复提交 |

★ **关键认知**：优先用 `db.Transaction`。手动 `Begin` 只在"需要把 `tx` 传给多个函数、或需要在事务外做耗时操作再决定是否提交"时才有必要。用的时候务必配 `defer` 兜底。

# 4. 嵌套事务与 SavePoint

## 4.1 嵌套 Transaction

```go
db.Transaction(func(tx *gorm.DB) error {
    if err := tx.Create(&order).Error; err != nil {
        return err
    }

    // 内层 Transaction 不会真正再开一个连接，而是 SAVEPOINT
    err := tx.Transaction(func(inner *gorm.DB) error {
        return inner.Create(&orderItem).Error
    })
    if err != nil {
        // 内层已回滚到 SAVEPOINT，外层仍可继续，也可以选择整体回滚
        log.Printf("明细写入失败，忽略：%v", err)
    }

    return nil // 外层提交，内层失败部分不生效
})
```

实际 SQL：

```sql
BEGIN;
INSERT INTO orders ...;
SAVEPOINT sp1;
INSERT INTO order_items ...; -- 失败
ROLLBACK TO SAVEPOINT sp1;
COMMIT;
```

★ **关键认知**：GORM 的"嵌套事务"本质是 **SavePoint**，不是独立事务。内层失败只回滚到保存点，外层仍然掌握最终提交权。想禁用这一行为，用 `Session{DisableNestedTransaction: true}`，此时内层会直接复用外层事务。

## 4.2 显式 SavePoint

```go
tx := db.Begin()
tx.SavePoint("before_batch")

if err := tx.Create(&batch).Error; err != nil {
    tx.RollbackTo("before_batch") // 只撤销这一批
}

tx.Commit()
```

| 操作 | 作用 |
|------|------|
| `SavePoint(name)` | 打一个可回滚的标记 |
| `RollbackTo(name)` | 回滚到标记，事务继续 |
| `Rollback()` | 整个事务结束 |

# 5. 隔离级别

## 5.1 四种隔离级别与异常

| 级别 | 脏读 | 不可重复读 | 幻读 |
|------|------|------------|------|
| Read Uncommitted | 可能 | 可能 | 可能 |
| Read Committed | 不可能 | 可能 | 可能 |
| Repeatable Read | 不可能 | 不可能 | 可能（InnoDB 用间隙锁基本避免） |
| Serializable | 不可能 | 不可能 | 不可能 |

- **脏读**：读到别人未提交的数据。
- **不可重复读**：同一事务内两次读同一行，结果不同。
- **幻读**：同一事务内两次范围查询，行数不同。

## 5.2 在 GORM 中设置

```go
import "database/sql"

tx := db.Begin(&sql.TxOptions{
    Isolation: sql.LevelRepeatableRead, // 隔离级别
    ReadOnly:  false,                   // 只读事务可优化性能
})
defer func() {
    if r := recover(); r != nil {
        tx.Rollback()
        panic(r)
    }
}()
// ... 业务
tx.Commit()
```

| 数据库 | 默认隔离级别 |
|--------|--------------|
| MySQL InnoDB | Repeatable Read |
| PostgreSQL | Read Committed |
| SQL Server | Read Committed |
| Oracle | Read Committed |

⚠ 隔离级别越高，锁竞争与死锁概率越大。大多数业务用数据库默认级别即可，只有明确遇到"同一事务内读到不同值"时再上调，并且要压测。

# 6. 并发扣减

## 6.1 悲观锁

```go
import "gorm.io/gorm/clause"

err := db.Transaction(func(tx *gorm.DB) error {
    var product Product
    // SELECT ... FOR UPDATE，锁住这一行直到事务结束
    if err := tx.Clauses(clause.Locking{Strength: "UPDATE"}).
        First(&product, productID).Error; err != nil {
        return err
    }

    if product.Stock < 1 {
        return errors.New("库存不足")
    }

    // 这里可以直接用内存里的值，因为行已被锁住
    return tx.Model(&product).Update("stock", product.Stock-1).Error
})
```

悲观锁的代价：**持锁时间长**，高并发下请求排队，容易超时甚至死锁。务必保证事务内没有慢操作（HTTP 调用、文件 IO）。

## 6.2 乐观锁

```go
type Product struct {
    ID      uint `gorm:"primaryKey"`
    Stock   int
    Version int `gorm:"not null;default:0"`
}

// 更新时带上版本号，版本不匹配则 RowsAffected 为 0
res := db.Model(&Product{}).
    Where("id = ? AND version = ?", id, oldVersion).
    UpdateColumns(map[string]any{
        "stock":   gorm.Expr("stock - ?", 1),
        "version": oldVersion + 1,
    })
if res.Error != nil {
    return res.Error
}
if res.RowsAffected == 0 {
    return errors.New("并发冲突，请重试") // 由上层重试
}
```

## 6.3 原子更新：最轻的方案

```go
// 库存足够才减，一条 SQL 完成判断与扣减
res := db.Model(&Product{}).
    Where("id = ? AND stock >= ?", id, qty).
    UpdateColumn("stock", gorm.Expr("stock - ?", qty))
if res.RowsAffected == 0 {
    return errors.New("库存不足")
}
```

## 6.4 三种方案对比

| 方案 | 加锁方式 | 冲突处理 | 适用 |
|------|----------|----------|------|
| 原子更新 | 数据库行锁（隐式） | `RowsAffected == 0` | 简单计数、库存扣减 |
| 悲观锁 | `FOR UPDATE` | 排队等待 | 事务内多步依赖同一行 |
| 乐观锁 | 版本号 CAS | 失败重试 | 读多写少、冲突不频繁 |

★ **关键认知**：能用一条原子 SQL 解决，就不要开事务加锁。`UPDATE ... WHERE stock >= ?` 既短又不会死锁，是扣减场景的首选。

# 7. 常见错误

| 现象 | 原因 | 解决 |
|------|------|------|
| 事务里改了数据却没生效 | 用了 `db` 而不是 `tx` | 回调内全程用 `tx` |
| 出错但数据已写入 | 没检查 `.Error`，直接 `return nil` | 每个写操作都判错 |
| `ERROR 1213: Deadlock` | 加锁顺序不一致 | 统一按主键顺序更新，缩短事务 |
| `context deadline exceeded` | 事务内有慢查询或外部调用 | 事务内只做数据库操作 |
| 嵌套事务没回滚 | 误解 SavePoint 语义 | 内层返回错误后外层也要处理 |
| 乐观锁频繁失败 | 冲突率高 | 改悲观锁或原子更新 |

# 8. 动手练习

1. 用 `db.Transaction` 实现转账，要求：余额不足返回业务错误、任何一步失败整体回滚、记录失败原因。再故意在中间 `panic`，验证是否回滚。
2. 用 1000 个 goroutine 抢 100 件库存，分别用"先查后改" "悲观锁" "原子更新"三种方案实现，记录最终库存与耗时。
3. 给商品表加 `version` 字段，实现乐观锁扣减，并写一个最多重试 3 次的循环。

# 9. 小结

| 主题 | 要点 |
|------|------|
| 自动事务 | `db.Transaction`，返回 `nil` 提交、非 nil 回滚 |
| 手动事务 | `Begin/Commit/Rollback`，必须 `defer` 兜底 |
| 嵌套 | 本质是 SavePoint，内层失败外层仍可提交 |
| 隔离级别 | 数据库默认通常够用，上调要压测 |
| 悲观锁 | `FOR UPDATE`，持锁久、易排队 |
| 乐观锁 | 版本号 CAS，失败重试 |
| 原子更新 | `WHERE stock >= ?` 一条 SQL，优先选它 |

下一篇进入钩子与插件，讲清 `BeforeCreate`/`AfterFind` 等钩子的触发时机、事务中的钩子行为、插件与回调的注册方式，以及多租户、审计字段这类横切逻辑如何优雅落地。
