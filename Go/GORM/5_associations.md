单表操作很少构成真实业务，用户有订单、订单有明细、文章有标签，这些关系才是数据访问的主体。GORM 的关联设计有一条清晰的主线：**结构体表达关系，外键负责落库，`Preload` 负责取数据**。把这条线理顺，N+1 这类问题自然就少了。

# 1. 四种关系

## 1.1 BelongsTo：子表持有外键

"一张信用卡属于一个用户"，外键在**子表**上：

```go
type User struct {
    ID   uint
    Name string
}

type CreditCard struct {
    ID     uint
    Number string
    UserID uint // 外键在这一侧
    User   User // 关联字段
}
```

查询时先取卡，再预加载持有人：

```go
var card CreditCard
db.Preload("User").First(&card, 1) // SELECT * FROM credit_cards WHERE id = 1
                                   // SELECT * FROM users WHERE id = ?
```

## 1.2 HasOne：父表被外键引用

"一个用户有一张卡"，外键在**对方表**上：

```go
type User struct {
    ID         uint
    Name       string
    CreditCard CreditCard `gorm:"foreignKey:UserID"` // UserID 在 CreditCard 上
}
```

★ **关键认知**：BelongsTo 和 HasOne 的代码写法几乎一样，区别在**外键落在哪张表**，以及删除时的语义。BelongsTo 是"我依赖别人"，HasOne 是"别人依赖我"。

## 1.3 HasMany：一对多

```go
type User struct {
    ID     uint
    Name   string
    Orders []Order `gorm:"foreignKey:UserID"` // 一个用户多个订单
}

type Order struct {
    ID      uint
    OrderNo string
    UserID  uint // 外键
    Amount  int64
}
```

```go
var user User
db.Preload("Orders").First(&user, 1)

// 只预加载符合条件的订单
db.Preload("Orders", "amount > ?", 1000).First(&user, 1)

// 只预加载指定列，务必带上外键列
db.Preload("Orders", func(db *gorm.DB) *gorm.DB {
    return db.Select("id", "user_id", "order_no")
}).First(&user, 1)
```

⚠ 条件预加载里如果用了 `Select` 却漏掉外键列（`user_id`），GORM 无法把订单匹配回用户，你会看到 `Orders` 全是空数组，且没有任何报错。

## 1.4 Many2Many：多对多

```go
type User struct {
    ID        uint
    Name      string
    Languages []Language `gorm:"many2many:user_languages;"`
}

type Language struct {
    ID   uint
    Name string
    Code string
}
```

GORM 会自动维护中间表 `user_languages`，默认包含 `user_id` 和 `language_id` 两个外键列，并组成联合主键。

```go
// 取出用户及其语言
var user User
db.Preload("Languages").First(&user, 1)

// 写入关联：先有 user，再挂语言
db.Model(&user).Association("Languages").Append(&lang)
```

## 1.5 四种关系对比

| 关系 | 外键位置 | 典型例子 | 结构体写法 |
|------|----------|----------|------------|
| BelongsTo | 当前表 | 订单 → 用户 | 当前表有 `UserID` + `User` |
| HasOne | 对方表 | 用户 → 身份证 | 当前表写 `CreditCard` |
| HasMany | 对方表 | 用户 → 订单 | 当前表写 `[]Order` |
| Many2Many | 中间表 | 用户 ↔ 角色 | 双方各写 `[]T` + `many2many` |

# 2. 外键与约束

## 2.1 默认约定

GORM 推断外键的规则是：`<关联类型名>ID`。所以 `User` 类型对应 `UserID` 字段。字段类型必须与被引用表主键一致（通常是 `uint`）。

## 2.2 自定义外键

```go
type Order struct {
    ID     uint
    BuyerID uint // 语义化的外键名
    Buyer   User `gorm:"foreignKey:BuyerID;references:ID"`
}
```

```go
type User struct {
    ID      uint
    Orders  []Order `gorm:"foreignKey:BuyerID;references:ID"`
}
```

| 标签 | 作用 |
|------|------|
| `foreignKey` | 指定本结构体或对方结构体上的外键字段 |
| `references` | 指定被引用的列，默认是主键 |
| `constraint:OnUpdate:CASCADE,OnDelete:SET NULL` | 生成数据库级外键约束与级联行为 |
| `polymorphic` | 多态关联 |

## 2.3 约束与迁移

```go
type Order struct {
    ID      uint
    UserID  uint
    User    User `gorm:"constraint:OnUpdate:CASCADE,OnDelete:RESTRICT"`
}
```

★ **关键认知**：`constraint` 只影响 `AutoMigrate` 时是否生成外键约束，**不影响 GORM 的运行时代码行为**。也就是说，即使数据库有 `ON DELETE CASCADE`，你用 `db.Delete(&user)` 软删除时也不会级联，因为那只是一条 `UPDATE`。

⚠ 生产环境常常**不建物理外键**：分库分表后外键跨库不可用，大规模写入时外键检查也是负担。业界更常见的做法是应用层保证一致性 + 异步对账。是否建外键要作为架构决策明确写下来，而不是默认打开。

# 3. 预加载

## 3.1 Preload 的基本行为

```go
var users []User
db.Preload("Orders").Find(&users)
```

实际执行两条 SQL：

```sql
SELECT * FROM users;
SELECT * FROM orders WHERE user_id IN (1, 2, 3); -- 一次捞回所有关联
```

★ **关键认知**：`Preload` 是"额外一条 `IN` 查询"，不是 JOIN。它能避免 N+1，但也会把关联数据整份取回内存。关联数据量很大时，仍要配合条件或分页。

## 3.2 条件与排序

```go
db.Preload("Orders", "status = ?", "paid").
    Preload("Orders", func(db *gorm.DB) *gorm.DB {
        return db.Order("created_at DESC").Limit(5) // 注意：Limit 在部分数据库下语义不精确
    }).
    Find(&users)
```

⚠ 给 HasMany 的 `Preload` 加 `Limit` 时，GORM 的 `Limit` 作用于"全部关联行的总集合"，而不是"每个用户取 5 条"。想要"每个用户最新 5 条"，要用窗口函数写原生 SQL。

## 3.3 嵌套预加载

```go
db.Preload("Orders.OrderItems").
    Preload("Orders.OrderItems.Product").
    Find(&users)
```

点号表示层级，每一层都会单独发一条 `IN` 查询。三层嵌套就是三条额外查询，要留意总量。

## 3.4 Joins 预加载

```go
// 只 JOIN 不取关联字段，用于过滤
var users []User
db.Joins("JOIN orders ON orders.user_id = users.id").
    Where("orders.amount > ?", 1000).
    Group("users.id").
    Find(&users)

// 带上关联字段：JOIN 一个别名
type UserWithCompany struct {
    User
    CompanyName string
}
db.Model(&User{}).
    Select("users.*, companies.name AS company_name").
    Joins("LEFT JOIN companies ON companies.id = users.company_id").
    Scan(&users)
```

## 3.5 Preload 与 Joins 的选择

| 维度 | Preload | Joins |
|------|---------|-------|
| SQL 条数 | 主表 1 条 + 每个关联 1 条 | 1 条 |
| 是否取回关联字段 | 是 | 需要手动 `Select` |
| 主表行数 | 不受关联影响 | 可能因一对多被放大 |
| 能否用关联条件过滤主表 | 不能直接过滤 | 能 |
| 分页正确性 | 安全 | 一对多时 `LIMIT` 会错 |
| 适用 | 展示关联数据 | 按关联条件筛选 |

★ **关键认知**：一对多场景下用 `Joins` + `Limit` 分页是经典错误——JOIN 后行数膨胀，`LIMIT 10` 取到的是 10 行**连接结果**，不是 10 个用户。分页查询主表时，要么用 `Preload`，要么用子查询先分页。

# 4. 关联写入

## 4.1 Create 时顺带保存

```go
user := User{
    Name: "alice",
    Orders: []Order{
        {OrderNo: "A001", Amount: 100},
        {OrderNo: "A002", Amount: 200},
    },
}
db.Create(&user) // 插入 user 后自动插入两条 order，并回填 UserID
```

默认只在关联字段是**零值**时 upsert；关联对象已存在且有主键时，GORM 默认只建关联不更新对方数据。要整份覆盖，用：

```go
db.Session(&gorm.Session{FullSaveAssociations: true}).Create(&user)
```

## 4.2 Association 模式

```go
// 追加
db.Model(&user).Association("Orders").Append(&Order{OrderNo: "A003"})

// 替换全部
db.Model(&user).Association("Orders").Replace(&order1, &order2)

// 删除指定关联（只解除关系或删除关联行，取决于关系类型）
db.Model(&user).Association("Orders").Delete(&order1)

// 清空
db.Model(&user).Association("Orders").Clear()

// 计数
count := db.Model(&user).Association("Orders").Count()
```

★ **关键认知**：Association 的 `Delete` 对 HasMany 是**物理删除关联行**，即使 `Order` 有 `DeletedAt` 也不会走软删除。要软删除请直接操作 `Order` 模型，别用 Association。

# 5. N+1 问题

```go
// 错误示范：1 次查用户 + N 次查订单
var users []User
db.Find(&users)
for i := range users {
    db.Where("user_id = ?", users[i].ID).Find(&users[i].Orders) // N 条 SQL
}
```

```text
N+1 的执行过程（N=100）：
1 条  SELECT * FROM users;
100 条 SELECT * FROM orders WHERE user_id = 1;
                          WHERE user_id = 2;
                          ...
                          WHERE user_id = 100;
```

```go
// 正确做法：Preload，固定 2 条 SQL
db.Preload("Orders").Find(&users)
```

| 方案 | SQL 条数 | 适用 |
|------|----------|------|
| 循环查询 | N+1 | 禁止 |
| Preload | 1 + 关联层数 | 常规列表 |
| Joins + Scan | 1 | 只取部分字段、需要过滤 |
| 批量手动查询 | 1 + 1 | 需要自定义映射时 |

★ **关键认知**：N+1 不会报错，只会在数据量上来后突然变慢。**看到循环里有 `db.` 调用就要警觉**，这是最有效的代码审查规则。

# 6. 常见错误

| 现象 | 原因 | 解决 |
|------|------|------|
| 关联字段始终为空 | 忘了 `Preload` | 显式预加载 |
| 预加载结果错乱 | `Preload` 的 `Select` 漏了外键列 | 补上 `user_id` |
| 创建时关联没保存 | 关联字段零值被跳过 | 用 `FullSaveAssociations` |
| 外键约束报错 | 主表记录不存在或类型不一致 | 先建主表，核对字段类型 |
| 中间表重复 | 未设联合主键/唯一索引 | 手动定义中间表模型 |
| 分页数量不对 | `Joins` 一对多导致行放大 | 改 `Preload` 或子查询 |

# 7. 动手练习

1. 建立 `User`、`Order`、`OrderItem`、`Product` 四张表，要求：用户有多个订单，订单有多个明细，明细关联商品。用 `Preload` 一次查出"用户 + 其订单 + 每单明细 + 商品"。
2. 用 `Joins` 查出"买过金额大于 1000 的商品的用户"，要求去重且支持分页，说明为什么不能直接 `LIMIT`。
3. 给 `User` 和 `Role` 建多对多，用 Association 完成"给用户追加角色""移除角色""清空角色"，并观察中间表数据变化。

# 8. 小结

| 主题 | 要点 |
|------|------|
| 关系判定 | 看外键在哪张表：当前表 = BelongsTo，对方表 = HasOne/HasMany |
| 外键 | 默认 `<Type>ID`，可用 `foreignKey`/`references` 改 |
| 约束 | `constraint` 只影响迁移，不影响运行时 |
| Preload | 额外 `IN` 查询，避免 N+1，一对多分页安全 |
| Joins | 单条 SQL，适合按关联过滤，注意行放大 |
| 关联写入 | `Create` 自动保存关联；Association 操作用来增删关系 |
| N+1 | 循环里出现 `db.` 就是信号 |

下一篇进入事务，讲清 `db.Transaction` 的自动回滚机制、手动事务与 `SavePoint` 的嵌套语义、隔离级别的选择，以及并发扣减场景下乐观锁与悲观锁的落地写法。
