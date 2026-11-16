GORM 的 CRUD 看着只有几个方法，真正容易翻车的点集中在三处：零值到底更不更新、`Save` 到底插还是改、`Delete` 到底是删还是改。本篇把这三件事拆开讲，并给出 `RowsAffected` 的正确用法。

# 1. Create

## 1.1 单条写入

```go
user := User{Name: "alice", Email: "alice@example.com"}
result := db.Create(&user) // 必须传指针，否则主键无法回填
if result.Error != nil {
    log.Printf("插入失败: %v", result.Error)
}
fmt.Println(user.ID, result.RowsAffected) // 1 1
```

★ **关键认知**：`Create` 必须传**指针**。传值也能执行，但自增主键写不回你的变量，后续 `user.ID` 永远是 0，用它做关联或更新就会静默失败。

## 1.2 批量写入

```go
users := []User{
    {Name: "a", Email: "a@example.com"},
    {Name: "b", Email: "b@example.com"},
}

// 一次 INSERT 多行，默认整批提交
if err := db.Create(&users).Error; err != nil {
    return err
}

// 数据量大时按批切分，避免单条 SQL 过长
db.CreateInBatches(&users, 500) // 每 500 条一批
```

⚠ 批量 `Create` 在默认配置下会包在一个事务里（除非开了 `SkipDefaultTransaction`），任何一条失败整批回滚。如果你需要"部分成功"，得自己按条处理，不要指望 GORM 帮你跳过坏数据。

## 1.3 冲突处理与 upsert

```go
import "gorm.io/gorm/clause"

// 唯一键冲突时更新指定列（MySQL）
db.Clauses(clause.OnConflict{
    Columns:   []clause.Column{{Name: "email"}},              // 冲突判定列
    DoUpdates: clause.AssignmentColumns([]string{"name", "age"}), // 冲突时更新
}).Create(&user)

// 冲突时什么也不做
db.Clauses(clause.OnConflict{DoNothing: true}).Create(&user)
```

| 需求 | 写法 | 说明 |
|------|------|------|
| 冲突即失败 | 默认 `Create` | 返回 `Error 1062` |
| 冲突则忽略 | `OnConflict{DoNothing: true}` | 不发错误，`RowsAffected` 为 0 |
| 冲突则更新 | `OnConflict{DoUpdates: ...}` | 依赖唯一索引存在 |

## 1.4 创建时的字段行为

```go
type User struct {
    ID        uint      `gorm:"primaryKey"`
    Name      string
    Age       int       `gorm:"default:18"`
    CreatedAt time.Time
    UpdatedAt time.Time
}
```

★ **关键认知**：对于带 `default` 标签的字段，Go 的零值（`Age == 0`）**会被 GORM 跳过**，交给数据库默认值。所以你想显式插入 `age = 0`，要么用指针 `*int`，要么用 `map[string]any`，要么去掉 `default` 标签。这个行为和后面的零值更新是同一套逻辑。

# 2. 查询单条

## 2.1 三个取一条的方法

```go
var u User

db.First(&u)                  // 主键升序第一条
db.Last(&u)                   // 主键降序第一条
db.Take(&u)                   // 不带 ORDER BY，取到哪条看数据库
db.First(&u, 10)              // 主键 = 10
db.First(&u, "name = ?", "alice")
db.First(&u, map[string]any{"name": "alice", "age": 18})
```

| 方法 | 排序 | 未找到 |
|------|------|--------|
| `First` | 主键 ASC | `ErrRecordNotFound` |
| `Last` | 主键 DESC | `ErrRecordNotFound` |
| `Take` | 无 | `ErrRecordNotFound` |
| `Find` | 无 | 不报错，空切片 |

## 2.2 列表查询

```go
var users []User

db.Find(&users)                            // 全表，危险
db.Where("age > ?", 18).Find(&users)       // 条件
db.Where("name LIKE ?", "a%").Limit(20).Find(&users)

// 按主键批量取
db.Find(&users, []int{1, 2, 3})
```

⚠ `Find` 必须传切片指针。传 `[]User` 值不会报错，但结果写不回去，你会得到一个空切片还以为数据库没数据。

# 3. Update

## 3.1 Update 与 Updates

```go
// 单列
db.Model(&user).Update("age", 20)

// 多列：结构体会跳过零值
db.Model(&user).Updates(User{Name: "bob", Age: 0}) // Age=0 被忽略！

// 多列：map 会更新所有给出的键，包括零值
db.Model(&user).Updates(map[string]any{
    "name": "bob",
    "age":  0,
})
```

★ **关键认知**：`Updates` 传结构体时**只更新非零值字段**，这是为了防止"局部更新把没传的字段覆盖成零值"。传 `map` 时没有这个保护——键出现就更新，哪怕值是零值。

## 3.2 零值更新的三种正确姿势

```go
// 姿势一：map，最直观
db.Model(&user).Updates(map[string]any{"age": 0})

// 姿势二：Select 显式指定字段，强制更新零值
db.Model(&user).Select("Name", "Age").Updates(User{Name: "bob", Age: 0})

// 姿势三：指针字段，nil 表示不更新，指向 0 表示更新为 0
type UserPatch struct {
    Name *string
    Age  *int
}
age := 0
db.Model(&user).Updates(UserPatch{Age: &age})
```

## 3.3 Save 到底做什么

```go
db.Save(&user)
```

`Save` 的语义分两种情况：

| 主键 | 行为 |
|------|------|
| 非零 | 执行 `UPDATE`，**更新所有字段**（含零值） |
| 零值 | 执行 `INSERT` |

⚠ `Save` 会把它认为的所有字段一起写回。危险场景：你先 `First` 出一个完整对象，改了 `Name`，然后 `Save`。如果这个对象里有 `Balance`、`Status` 这类字段，中间被别的请求改过，你会用旧值把它们覆盖掉——典型的丢失更新。局部更新优先用 `Updates(map)`。

## 3.4 表达式与原子更新

```go
import "gorm.io/gorm/clause"

// 库存自减，交给数据库做减法，避免读改写竞态
db.Model(&Product{}).
    Where("id = ? AND stock >= ?", id, 1).
    UpdateColumn("stock", gorm.Expr("stock - ?", 1))

// 用 column 表达式保证列名被正确引用
db.Model(&User{}).Where("id = ?", id).
    UpdateColumn("login_count", gorm.Expr("login_count + 1"))
```

★ **关键认知**：`gorm.Expr("stock - ?", 1)` 生成的是 `stock = stock - 1`，减法在数据库内完成，天然并发安全。先 `First` 再 `Update("stock", p.Stock-1)` 则存在竞态窗口，高并发下会超卖。

## 3.5 全局更新必须带条件

```go
// 危险：更新全表，GORM 会直接拒绝无 WHERE 的更新
db.Model(&User{}).Update("status", 0) // Error: WHERE conditions required

// 确实要更新全表，显式放行
db.Session(&gorm.Session{AllowGlobalUpdate: true}).Model(&User{}).Update("status", 0)
```

⚠ 不要在业务代码里为了"省事"打开 `AllowGlobalUpdate`。这个保护存在的意义就是拦住漏写 `Where` 的事故。

# 4. Delete

## 4.1 软删除机制

模型里有 `gorm.DeletedAt` 字段时，`Delete` 变成 `UPDATE`：

```go
type User struct {
    ID        uint           `gorm:"primaryKey"`
    DeletedAt gorm.DeletedAt `gorm:"index"`
}

db.Delete(&user, 1)
```

实际执行的 SQL 是：

```sql
UPDATE users SET deleted_at = '2026-11-05 10:00:00' WHERE id = 1 AND deleted_at IS NULL;
```

之后所有查询都会自动追加 `deleted_at IS NULL`：

```sql
SELECT * FROM users WHERE deleted_at IS NULL;
```

## 4.2 查回与永久删除

```go
// 查询包含已软删除的数据
db.Unscoped().Where("id = ?", 1).Find(&user)

// 物理删除
db.Unscoped().Delete(&user, 1)

// 恢复
db.Unscoped().Model(&user).Update("deleted_at", nil)
```

## 4.3 批量删除

```go
// 按条件软删除，RowsAffected 是实际影响行数
res := db.Where("age < ?", 18).Delete(&User{})
fmt.Println(res.RowsAffected)

// 没有 WHERE 的批量删除会被拒绝（同更新）
db.Session(&gorm.Session{AllowGlobalUpdate: true}).Delete(&User{})
```

⚠ 软删除的唯一索引会失效：`email` 上建了唯一索引，用户被软删后同一个 email 无法再注册。解决方案是把 `deleted_at` 纳入唯一索引，或改用"删除时把 email 改写为 `email_deleted_时间戳`"。

# 5. RowsAffected

```go
res := db.Model(&User{}).Where("id = ?", id).Update("age", 20)
if res.Error != nil {
    return res.Error
}
if res.RowsAffected == 0 {
    // 要么记录不存在，要么新值和旧值完全一样（MySQL 默认返回 0）
}
```

| 场景 | RowsAffected |
|------|--------------|
| 更新命中且值有变化 | 1 |
| 更新命中但值没变 | MySQL 默认 0（取决于 `CLIENT_FOUND_ROWS`） |
| 记录不存在 | 0 |
| 软删除命中 | 1 |

★ **关键认知**：`RowsAffected == 0` **不能**直接推断"记录不存在"。MySQL 在"值未变化"时也返回 0。要区分两者，要么先查一次，要么在 DSN 加 `clientFoundRows=true`，要么用 `SELECT ... FOR UPDATE` 锁住再判断。

# 6. 常见错误

| 现象 | 原因 | 解决 |
|------|------|------|
| `Create` 后 `ID` 为 0 | 传了值而非指针 | 传 `&user` |
| `Updates` 零值没生效 | 结构体更新跳过零值 | 用 map 或 `Select` |
| `Save` 覆盖了别的字段 | `Save` 更新全部字段 | 局部更新用 `Updates` |
| 更新全表报错 | GORM 的全局更新保护 | 补 `Where`，别直接关保护 |
| 软删后数据还在 | 软删除就是 UPDATE | 要物理删用 `Unscoped()` |
| 并发扣减超卖 | 读改写存在竞态 | 用 `gorm.Expr` 原子更新 |

# 7. 动手练习

1. 建一张 `Product` 表（库存 `stock`），用 100 个 goroutine 各卖 1 件，先写"先查后改"版本，再用 `gorm.Expr` 写原子版本，比较最终库存。
2. 给 `User` 加 `DeletedAt`，验证：软删除后 `First` 查不到、`Unscoped().First` 能查到、唯一邮箱在软删后仍冲突，并给出至少一种修复方案。
3. 用 `Updates` 把一个用户的 `age` 改成 0、`name` 改成空字符串，要求两个字段都真正写入，至少写出两种写法。

# 8. 小结

| 操作 | 要点 |
|------|------|
| `Create` | 传指针；批量用 `CreateInBatches`；冲突用 `clause.OnConflict` |
| 查询 | `First` 未找到报错，`Find` 返回空切片；集合必须传切片指针 |
| `Update/Updates` | 结构体跳过零值，map 不跳；强制更新用 `Select` |
| `Save` | 有主键就全字段 UPDATE，易造成覆盖 |
| `Delete` | 有 `DeletedAt` 即软删；物理删用 `Unscoped()` |
| `RowsAffected` | 为 0 不代表不存在，MySQL 值未变也返回 0 |

下一篇进入查询与作用域，讲清 `Where`/`Or`/`Not` 的组合优先级、`Select`/`Omit` 的字段控制、分组聚合，以及如何用 `Scopes` 把分页、软删、多租户这类条件复用起来。
