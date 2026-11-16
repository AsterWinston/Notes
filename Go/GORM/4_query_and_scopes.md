查询是 ORM 使用频率最高的部分，也是最容易写出"能跑但很慢"代码的地方。本篇讲清条件构造的优先级规则、字段裁剪、排序分页与聚合，再讲 `Scopes` 如何把重复条件沉淀成可组合的积木，最后落到 SQL 注入的防线。

# 1. 条件构造

## 1.1 Where 的三种写法

```go
// 写法一：占位符字符串，最常用、最可控
db.Where("name = ? AND age > ?", "alice", 18).Find(&users)

// 写法二：结构体，只取非零值字段
db.Where(&User{Name: "alice", Age: 0}).Find(&users) // 只生成 name = 'alice'

// 写法三：map，所有键都参与，包括零值
db.Where(map[string]any{"name": "alice", "age": 0}).Find(&users)

// 写法四：主键简写
db.First(&user, 1)
db.Find(&users, []int{1, 2, 3})
```

| 写法 | 零值参与 | 防注入 | 适用 |
|------|----------|--------|------|
| 字符串 + `?` | 是 | 是 | 绝大多数场景 |
| 结构体 | 否 | 是 | 精确匹配非零条件 |
| map | 是 | 是 | 动态条件集合 |
| 拼接字符串 | 是 | **否** | 禁止 |

## 1.2 条件组合的优先级

```go
db.Where("name = ?", "alice").
    Or("age > ?", 30).
    Find(&users)
```

生成：

```sql
SELECT * FROM users WHERE name = 'alice' OR age > 30;
```

★ **关键认知**：GORM 的条件是逐段拼接的，`Or` 会把它前面的**整段** `WHERE` 用 `OR` 接上新条件。当你有多个 `AND` 又想只让其中两个走 `OR` 时，必须用括号显式分组：

```go
// 错误：期望 (status = 1 AND age > 18) OR (vip = 1)，实际不是这个语义
db.Where("status = ?", 1).Where("age > ?", 18).Or("vip = ?", 1)

// 正确：把 OR 关系用一组括号包起来
db.Where("status = ?", 1).
    Where(db.Where("age > ?", 18).Or("vip = ?", 1)).
    Find(&users)
```

生成的 SQL 才是：

```sql
SELECT * FROM users WHERE status = 1 AND (age > 18 OR vip = 1);
```

## 1.3 Not 与嵌套

```go
db.Not("name = ?", "alice").Find(&users)
db.Not(map[string]any{"name": "alice", "age": 18}).Find(&users) // 生成 NOT (name AND age)
db.Where("age > ?", 18).Not("status = ?", "disabled").Find(&users)

// 子查询
sub := db.Model(&Order{}).Select("user_id").Where("amount > ?", 1000)
db.Where("id IN (?)", sub).Find(&users)
```

⚠ `Not` 与 `Or` 组合时优先级同样反直觉。任何包含 `OR` 的复杂条件，都用嵌套 `db.Where(...)` 显式分组，别依赖默认结合顺序。

# 2. 字段控制

## 2.1 Select 与 Omit

```go
// 只取需要的列，减少网络传输与内存
db.Select("id", "name").Find(&users)

// 排除大字段
db.Omit("content", "metadata").Find(&articles)

// 聚合
type Result struct {
    Status string
    Total  int64
}
var results []Result
db.Model(&Order{}).Select("status, COUNT(*) AS total").Group("status").Scan(&results)
```

★ **关键认知**：`Select` 不仅是性能优化，也是正确性问题。只查部分列时，结构体里没查的字段是零值；如果这个对象之后被 `Save`，零值会被写回数据库。**裁剪字段查出来的对象，只能读不能直接写**。

## 2.2 Count

```go
var count int64
db.Model(&User{}).Where("age > ?", 18).Count(&count)

// Count 会忽略 Select 里的列，但会保留 Where
```

⚠ `Count` 之后继续链式调用 `Find` 会复用同一条件（GORM 内部有语句缓存机制），但如果你把中间结果存成变量反复用，务必用 `Session(&gorm.Session{})` 隔离，避免条件污染：

```go
base := db.Model(&User{}).Where("age > ?", 18)
var total int64
base.Count(&total)
var users []User
base.Session(&gorm.Session{}).Find(&users) // 每次重新构建语句
```

# 3. 排序、分页与聚合

## 3.1 Order

```go
db.Order("age DESC").Find(&users)
db.Order("age DESC, id ASC").Find(&users)
db.Order(clause.OrderByColumn{Column: clause.Column{Name: "age"}, Desc: true})
```

⚠ `Order` 的字符串参数不会被参数化。如果排序字段来自用户输入（如 `?sort=created_at`），必须做白名单校验，否则是一个注入点。

## 3.2 分页

```go
const pageSize = 20
page := 1

var users []User
db.Order("id DESC").
    Limit(pageSize).
    Offset((page - 1) * pageSize).
    Find(&users)
```

★ **关键认知**：深分页（`OFFSET 1000000`）的性能问题不是 GORM 的锅，而是 SQL 的。数据库必须扫描并丢弃前 N 行。数据量大时应改成游标分页：

```go
// 基于上一页最后一条的 id 翻页，能走主键索引
db.Where("id < ?", lastID).Order("id DESC").Limit(pageSize).Find(&users)
```

## 3.3 Group 与 Having

```go
type Stat struct {
    UserID uint
    Total  int64
}

var stats []Stat
db.Model(&Order{}).
    Select("user_id, SUM(amount) AS total").
    Group("user_id").
    Having("SUM(amount) > ?", 1000).
    Order("total DESC").
    Scan(&stats)
```

| 阶段 | 关键字 | 作用对象 |
|------|--------|----------|
| 过滤行 | `Where` | 原始行 |
| 分组 | `Group` | 行 |
| 过滤组 | `Having` | 聚合结果 |
| 排序 | `Order` | 结果集 |
| 截断 | `Limit/Offset` | 结果集 |

# 4. 原生 SQL

## 4.1 Raw 与 Scan

```go
type UserStat struct {
    Name  string
    Total int64
}

var stats []UserStat
db.Raw(`
    SELECT u.name, COUNT(o.id) AS total
    FROM users u
    LEFT JOIN orders o ON o.user_id = u.id
    WHERE u.created_at > ?
    GROUP BY u.id
`, start).Scan(&stats)
```

## 4.2 Exec

```go
db.Exec("UPDATE users SET status = ? WHERE id = ?", "active", 1)

// 注意：Exec 不做软删除、不触发钩子、不维护 UpdatedAt
```

## 4.3 什么时候该用原生

| 场景 | 推荐 |
|------|------|
| 简单增删改查 | 链式 API |
| 复杂多表统计、窗口函数 | `Raw` |
| 批量更新/清理 | `Exec`（注意绕过钩子） |
| 特定数据库语法（如 `ON DUPLICATE KEY`） | `Raw` 或 `Clauses` |

★ **关键认知**：`Raw`/`Exec` 绕过了 GORM 的钩子、软删除过滤和自动时间戳。用它们写 `DELETE` 会真的物理删除，写 `UPDATE` 不会更新 `UpdatedAt`。知道代价再用。

# 5. Scopes

## 5.1 定义

Scope 的签名固定为 `func(*gorm.DB) *gorm.DB`，本质是"一段可复用的查询片段"：

```go
func AgeGreaterThan(age int) func(*gorm.DB) *gorm.DB {
    return func(db *gorm.DB) *gorm.DB {
        return db.Where("age > ?", age)
    }
}

func Paginate(page, size int) func(*gorm.DB) *gorm.DB {
    return func(db *gorm.DB) *gorm.DB {
        if page <= 0 {
            page = 1
        }
        if size <= 0 || size > 100 {
            size = 20 // 兜底，防止 size 传 100000 拖垮数据库
        }
        return db.Offset((page - 1) * size).Limit(size)
    }
}
```

## 5.2 组合使用

```go
db.Scopes(AgeGreaterThan(18), Paginate(2, 20)).
    Order("id DESC").
    Find(&users)
```

多个 Scope 按顺序执行，效果等价于依次调用。带参 Scope 返回闭包，是这个模式的关键——它把参数和执行时机分离了。

## 5.3 常用 Scope 清单

```go
// 多租户隔离：所有查询强制带 tenant_id
func TenantScope(tenantID uint) func(*gorm.DB) *gorm.DB {
    return func(db *gorm.DB) *gorm.DB {
        return db.Where("tenant_id = ?", tenantID)
    }
}

// 时间范围
func CreatedBetween(start, end time.Time) func(*gorm.DB) *gorm.DB {
    return func(db *gorm.DB) *gorm.DB {
        return db.Where("created_at BETWEEN ? AND ?", start, end)
    }
}

// 状态筛选
func StatusIs(status string) func(*gorm.DB) *gorm.DB {
    return func(db *gorm.DB) *gorm.DB {
        if status == "" {
            return db // 空值不加条件
        }
        return db.Where("status = ?", status)
    }
}
```

⚠ Scope 无法自动应用到所有查询。想做到"多租户条件全局生效"，需要配合 GORM 的 `Callback` 或 `db.Scopes(...)` 包装，不能指望定义了就自动挂上。

# 6. SQL 注入防范

```go
// 错误示范：字符串拼接，用户输入直接进 SQL
name := c.Query("name")
db.Where("name = '" + name + "'").Find(&users)
// 输入 ' OR '1'='1 即全表泄露
```

```go
// 正确做法：占位符，参数由驱动转义
db.Where("name = ?", name).Find(&users)
```

★ **关键认知**：GORM 的 `?` 占位符走的是 `database/sql` 的预处理参数，不是字符串替换。**唯一不安全的是你把用户输入拼进了 SQL 字符串本身**。以下位置必须额外小心：

| 位置 | 能否参数化 | 做法 |
|------|------------|------|
| `WHERE x = ?` | 能 | 直接用 `?` |
| `ORDER BY ?` | 不能 | 字段名白名单 |
| `LIMIT ?` | 能（多数驱动） | 用 `?` 或强转整数 |
| 表名/列名 | 不能 | 代码内常量，绝不接收外部输入 |

```go
// 排序字段的白名单写法
allowed := map[string]bool{"created_at": true, "age": true}
sort := c.Query("sort")
if !allowed[sort] {
    sort = "created_at"
}
db.Order(sort + " DESC") // 此时 sort 一定来自白名单
```

# 7. 常见排错

| 现象 | 原因 | 排查 |
|------|------|------|
| 查询条件没生效 | `Or` 打乱了优先级 | 打开日志看最终 SQL |
| 结果比预期多 | 忘了软删除/租户条件 | 检查是否用了 `Unscoped` 或漏 Scope |
| `Scan` 后字段全空 | 列名与字段名不匹配 | 加 `gorm:"column:..."` 或起别名 |
| `Order` 报语法错误 | 字段名拼错或注入被拦 | 打印 SQL，检查排序字段 |
| 分页越翻越慢 | `OFFSET` 深分页 | 改游标分页 |

# 8. 动手练习

1. 写一个 `Search(keyword, status string, page, size int)` 函数，要求：`keyword` 模糊匹配 `name`，`status` 为空时不加条件，分页参数有兜底，最终只生成一条 SQL。用 `db.ToSQL` 打印验证。
2. 实现 `(status = 'paid' AND amount > 100) OR (status = 'refunded' AND amount > 1000)`，要求用嵌套 `Where` 保证括号正确。
3. 把"分页 + 租户隔离 + 时间范围"抽成三个 Scope，组合后用于一个列表查询，并说明为什么 Scope 要返回闭包而不是直接返回 `*gorm.DB`。

# 9. 小结

| 主题 | 要点 |
|------|------|
| 条件 | 字符串用 `?`，结构体跳零值，map 不跳 |
| 优先级 | 含 `OR` 一律用嵌套 `Where` 显式分组 |
| 字段 | `Select`/`Omit` 裁剪；裁剪出的对象只能读 |
| 分页 | 深分页改游标，`LIMIT/OFFSET` 有性能上限 |
| 原生 SQL | `Raw`/`Exec` 绕过钩子与软删除 |
| Scope | 签名 `func(*gorm.DB) *gorm.DB`，带参返回闭包 |
| 注入 | 只有拼进 SQL 字符串才危险，排序字段用白名单 |

下一篇进入关联，讲清 BelongsTo、HasOne、HasMany、Many2Many 的建表与外键规则，`Preload` 与 `Joins` 的取舍，以及如何识别并消除 N+1 查询。
