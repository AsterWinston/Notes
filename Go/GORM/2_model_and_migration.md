模型是 GORM 的起点：一张表长什么样、主键怎么生成、时间字段谁维护、索引建在哪里，全部由结构体和标签决定。本篇把"结构体如何变成表"讲透，并说明 `AutoMigrate` 能做什么、绝对不能拿它当生产迁移工具的原因。

# 1. 模型即表结构

## 1.1 默认约定

GORM 用一套"约定优于配置"的规则把结构体映射成表：

| 结构体 | 默认表名 | 说明 |
|--------|----------|------|
| `User` | `users` | 驼峰转下划线并复数化 |
| `UserProfile` | `user_profiles` | 逐词转换 |
| `APIKey` | `api_keys` | 连续大写按一个词处理 |
| `Person` | `people` | 不规则复数 |

```go
type User struct {
    ID   uint
    Name string
    Age  int
}
// 对应：CREATE TABLE users (id bigint unsigned AUTO_INCREMENT, name longtext, age bigint, PRIMARY KEY (id))
```

★ **关键认知**：GORM 把所有字段都当成"要持久化的列"，只有以下情况例外——字段名是 `Model`、`ID` 之外被标签排除的、非导出字段（首字母小写）。非导出字段会被**静默忽略**，这是新手常见的"字段丢了"的原因。

## 1.2 字段标签总览

标签统一写在 `gorm:"..."` 里，多个选项用分号分隔：

```go
type Product struct {
    ID          uint      `gorm:"primaryKey;autoIncrement"`
    SKU         string    `gorm:"column:sku;type:varchar(32);not null;uniqueIndex:uk_sku"`
    Name        string    `gorm:"size:128;not null;comment:商品名称"`
    Price       int64     `gorm:"not null;default:0;comment:价格，单位分"`
    Status      int8      `gorm:"not null;default:1;index:idx_status_created,priority:1"`
    CreatedAt   time.Time `gorm:"index:idx_status_created,priority:2"`
    Description string    `gorm:"type:text"`
}
```

| 标签 | 作用 | 常见取值 |
|------|------|----------|
| `column` | 指定列名 | `column:user_name` |
| `type` | 指定数据库类型 | `type:varchar(64)`、`type:json` |
| `size` | 字符串长度/数值精度 | `size:255`、`size:10,2` |
| `primaryKey` | 声明主键 | 无值 |
| `autoIncrement` | 自增 | 无值 |
| `not null` | 非空 | 无值 |
| `default` | 默认值 | `default:0`、`default:active` |
| `uniqueIndex` | 唯一索引 | `uniqueIndex:uk_email` |
| `index` | 普通索引 | `index:idx_name,priority:2` |
| `comment` | 列注释 | `comment:创建时间` |
| `-` | 忽略该字段 | `gorm:"-"` |
| `->` / `<-` | 只读/只写权限 | `gorm:"->"` 表示只读取不写入 |

⚠ `size` 与 `type` 不要同时写。`size:64` 在 MySQL 下生成 `varchar(64)`，而 `type:varchar(64)` 会直接覆盖类型推导；两者冲突时以 `type` 为准，容易埋下不一致。

# 2. 主键与时间戳

## 2.1 主键策略

```go
// 方式一：约定名 ID，自动识别为主键
type User struct {
    ID uint
}

// 方式二：显式声明
type Order struct {
    OrderNo string `gorm:"primaryKey;size:32"`
}

// 方式三：联合主键
type UserRole struct {
    UserID uint `gorm:"primaryKey"`
    RoleID uint `gorm:"primaryKey"`
}
```

| 主键类型 | 适用场景 | 注意 |
|----------|----------|------|
| 自增 `uint`/`int64` | 单库、内部系统 | 暴露 ID 会泄露业务量；分库分表困难 |
| 雪花 ID `int64` | 分布式、需趋势递增 | 需要自己生成，不能用 `autoIncrement` |
| UUID/ULID 字符串 | 对外暴露、离线生成 | 随机 UUID 作聚簇索引会页分裂，MySQL 建议 ULID 或有序 UUID |

★ **关键认知**：`string` 类型的主键**不会**默认自增，必须自己赋值。如果忘了赋值，GORM 会插入空字符串，第二条就撞主键。

## 2.2 CreatedAt、UpdatedAt、DeletedAt

这三个字段名是 GORM 的保留约定，只要类型匹配就自动生效：

```go
type Article struct {
    ID        uint           `gorm:"primaryKey"`
    Title     string
    CreatedAt time.Time      // Create 时自动填充
    UpdatedAt time.Time      // Create 和每次 Update 自动刷新
    DeletedAt gorm.DeletedAt `gorm:"index"` // 有它即启用软删除
}
```

- 想让 `UpdatedAt` 在批量更新时也刷新，必须用 `Model(&Article{})` 或带主键的结构体作为更新目标。
- `DeletedAt` 用 `gorm.DeletedAt`（不是 `*time.Time`）才会接管查询过滤与 `Delete` 行为。

## 2.3 时间精度与零值

```go
type Event struct {
    ID        uint      `gorm:"primaryKey"`
    HappenedAt time.Time `gorm:"precision:3"` // MySQL datetime(3)，毫秒
    CreatedAt  time.Time
}
```

⚠ MySQL 的 `datetime` 默认精度是秒。如果你的业务需要毫秒且用 `time.Time` 比较，必须显式 `precision:3`，否则写进去的值会被截断，导致"刚创建就查不到"。

# 3. 表名与命名策略

## 3.1 TableName 方法

```go
type User struct {
    ID uint
}

// 单表覆盖：实现 Tabler 接口
func (User) TableName() string {
    return "sys_user" // 对接历史库时最常用
}
```

## 3.2 全局命名策略

```go
import "gorm.io/gorm/schema"

db, err := gorm.Open(mysql.Open(dsn), &gorm.Config{
    NamingStrategy: schema.NamingStrategy{
        TablePrefix:   "t_",  // 表前缀
        SingularTable: true,  // 关闭复数，User -> user
        NameReplacer:  strings.NewReplacer("ID", "Id"), // 处理 ID 命名
    },
})
```

| 配置 | 效果 |
|------|------|
| 默认 | `UserProfile` → `user_profiles` |
| `TablePrefix: "t_"` | → `t_user_profiles` |
| `SingularTable: true` | → `user_profile` |

★ **关键认知**：`NamingStrategy` 只影响**默认推导**。一旦某个模型实现了 `TableName()`，全局策略对它失效。历史库改造时，优先用 `TableName()` 精确控制，别指望全局规则覆盖所有特例。

# 4. 字段类型细节

## 4.1 数值与字符串

```go
type Account struct {
    ID       uint64 `gorm:"primaryKey"`
    Balance  int64  `gorm:"not null;default:0;comment:余额，单位分"` // 金额永远用整数分
    Rate     int32  `gorm:"not null;default:100"`               // 百分比放大 100 倍
    Nickname string `gorm:"size:32;not null;default:''"`
    Bio      string `gorm:"type:text"`
    Extra    []byte `gorm:"type:json"` // 或 datatypes.JSON
}
```

⚠ 金额不要用 `float64`。数据库的 `FLOAT/DOUBLE` 与 Go 的浮点都存在精度误差，累加后会出现 `0.30000000000000004` 这类结果。统一用"分"存整数是最稳的做法。

## 4.2 默认值与 NULL

```go
type Config struct {
    ID     uint    `gorm:"primaryKey"`
    Key    string  `gorm:"size:64;not null"`
    Value  *string `gorm:"type:text"`          // 指针：nil 表示 NULL
    Enable bool    `gorm:"not null;default:true"`
}
```

★ **关键认知**：Go 的零值无法表达 NULL。`int` 的 0 和"数据库里的 NULL"在结构体里长得一样。需要区分时用指针或 `sql.NullInt64`，尤其在 `WHERE value = ?` 与 `WHERE value IS NULL` 语义不同的场景。

## 4.3 复合索引与优先级

```go
type Log struct {
    ID        uint      `gorm:"primaryKey"`
    TenantID  uint      `gorm:"index:idx_tenant_time,priority:1"`
    CreatedAt time.Time `gorm:"index:idx_tenant_time,priority:2"`
    Level     string    `gorm:"size:16;index:idx_tenant_time,priority:3"`
}
```

同一个索引名 + 不同 `priority` 会合并成一个复合索引 `(tenant_id, created_at, level)`。顺序就是最左前缀，必须按查询条件的高频顺序排。

# 5. AutoMigrate

## 5.1 基本用法

```go
func Migrate(db *gorm.DB) error {
    return db.AutoMigrate(
        &User{},
        &Product{},
        &Order{},
        &OrderItem{},
    )
}
```

## 5.2 它做了什么，不做什么

| 会做 | 不会做 |
|------|--------|
| 建表、建索引 | 删除多余的列 |
| 新增列、新增索引 | 修改列类型（部分数据库有限支持） |
| 创建缺失的外键约束 | 重命名列、数据迁移 |
| 部分场景下加长字段 | 回滚、版本记录 |

★ **关键认知**：`AutoMigrate` 的定位是**开发期辅助**，不是迁移系统。它的核心假设是"结构只会增加，不会减少或变形"，因此对删列、改类型、重命名一律保守处理甚至静默跳过。

⚠ 生产环境直接 `AutoMigrate` 的三个典型事故：

1. 大表加索引触发长时间锁表，业务写入全部阻塞。
2. 改字段长度时某些数据库会重建表，数据量大时停机数十分钟。
3. 多实例同时启动并发迁移，出现表结构竞争。

## 5.3 生产迁移实践

```text
开发阶段                    生产阶段
┌──────────────┐           ┌──────────────────────┐
│ AutoMigrate  │           │ 版本化 SQL 迁移脚本   │
│ 快速迭代      │  ──────►  │ golang-migrate/atlas  │
└──────────────┘           │ 评审 + 灰度 + 回滚    │
                           └──────────────────────┘
```

推荐的落地方式：

```bash
# 用 golang-migrate 管理版本，脚本入库
migrate create -ext sql -dir migrations -seq add_user_phone
```

```sql
-- migrations/000002_add_user_phone.up.sql
ALTER TABLE users ADD COLUMN phone VARCHAR(20) NULL COMMENT '手机号';
CREATE INDEX idx_users_phone ON users (phone);

-- migrations/000002_add_user_phone.down.sql
DROP INDEX idx_users_phone ON users;
ALTER TABLE users DROP COLUMN phone;
```

大表变更的安全顺序应是"先加可空列 → 双写 → 回填 → 加约束 → 删旧列"，每一步都能独立回滚。

# 6. 常见错误

| 现象 | 原因 | 解决 |
|------|------|------|
| 某个字段没进表 | 字段非导出（小写开头） | 改首字母大写 |
| 表名不是预期的 | 复数化 + 驼峰转换 | 实现 `TableName()` 或配 `NamingStrategy` |
| `Error 1067: Invalid default value` | `time.Time` 零值写入非空无默认列 | 允许 NULL 或给默认值 |
| 时间差 8 小时 | DSN 与 `NowFunc` 时区不一致 | DSN 加 `loc=Local`，`NowFunc` 统一时区 |
| 主键重复 | 字符串主键忘了赋值 | 用 `BeforeCreate` 钩子或调用方显式赋值 |
| 唯一索引建不上 | 历史数据有重复值 | 先清洗数据再加索引 |

# 7. 动手练习

1. 定义 `Article` 模型，要求：`ID` 自增、`Slug` 唯一索引、`Status` 与 `CreatedAt` 组成复合索引、正文用 `text`、有软删除。用 `AutoMigrate` 建表后，用 `SHOW CREATE TABLE` 验证。
2. 把全局命名策略改成 `SingularTable: true` + 前缀 `t_`，观察 `User` 的表名变化，然后给其中一个模型实现 `TableName()`，验证"局部覆盖全局"。
3. 设计一张金额表，要求金额用整数分存储，同时保留一个 `*int64` 的可空折扣字段，说明为什么用指针。

# 8. 小结

| 主题 | 要点 |
|------|------|
| 映射规则 | 驼峰转下划线、复数表名，非导出字段被忽略 |
| 主键 | `ID` 约定或显式 `primaryKey`，字符串主键不自增 |
| 时间戳 | `CreatedAt`/`UpdatedAt`/`gorm.DeletedAt` 按字段名生效 |
| 命名 | `TableName()` 精确控制，`NamingStrategy` 管默认 |
| 标签 | `type`/`size` 二选一，复合索引用同名 + priority |
| 迁移 | `AutoMigrate` 只适合开发，生产用版本化 SQL |

下一篇进入 CRUD，重点讲清 `Save` 与 `Updates` 的零值陷阱、批量写入的行为差异、软删除的真实 SQL，以及 `RowsAffected` 如何影响"更新成功"的判断。
