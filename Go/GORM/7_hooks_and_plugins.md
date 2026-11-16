只要业务里出现"每条记录都要写创建人""每次更新都要留审计日志""所有查询都要带租户条件"这类重复诉求，就该考虑钩子和插件了。它们能把横切逻辑从业务代码里抽走，但抽错位置也会带来更难排查的隐式行为。

# 1. 钩子是什么

## 1.1 生命周期一览

GORM 在每条 SQL 执行前后提供了固定名字的钩子方法，只要你的模型实现了对应方法，就会被自动调用：

```text
Create:  BeforeSave → BeforeCreate → INSERT → AfterCreate → AfterSave
Update:  BeforeSave → BeforeUpdate → UPDATE → AfterUpdate → AfterSave
Delete:  BeforeDelete → DELETE → AfterDelete
Query:   SELECT → AfterFind
```

| 钩子 | 触发时机 | 典型用途 |
|------|----------|----------|
| `BeforeSave` | 写库前（Create/Update 都触发） | 统一做字段清洗 |
| `BeforeCreate` | 插入前 | 生成 UUID、加密密码 |
| `AfterCreate` | 插入后 | 写审计日志、发事件 |
| `BeforeUpdate` | 更新前 | 校验状态流转 |
| `AfterUpdate` | 更新后 | 清缓存 |
| `BeforeDelete` | 删除前 | 阻止删除、级联清理 |
| `AfterDelete` | 删除后 | 记录删除日志 |
| `AfterFind` | 查询后 | 解密字段、补算派生值 |

★ **关键认知**：`BeforeSave`/`AfterSave` 是 `Create` 和 `Update` 的公共入口，会在这两类操作中各触发一次。如果你在 `BeforeSave` 里做了计数，一次创建会算一次，一次更新又会算一次。

## 1.2 签名与返回值

```go
func (u *User) BeforeCreate(tx *gorm.DB) error {
    if u.Email == "" {
        return errors.New("email 不能为空")
    }
    return nil
}
```

规则：

- 方法必须定义在**指针接收者**上，GORM 需要通过指针回写字段。
- 返回值只有 `error`。**返回非 nil 会中断操作**，在事务中会导致回滚。
- 钩子里拿到的 `tx` 是当前语句的执行上下文，直接用它执行额外 SQL。

# 2. 常用钩子

## 2.1 Create 钩子

```go
type User struct {
    ID       uint   `gorm:"primaryKey"`
    Email    string `gorm:"uniqueIndex;size:128"`
    Password string `gorm:"size:128"`
    Salt     string `gorm:"size:32"`
}

func (u *User) BeforeCreate(tx *gorm.DB) error {
    // 生成盐并加密，避免明文落库
    u.Salt = randomString(16)
    u.Password = hashPassword(u.Password, u.Salt)
    return nil
}

func (u *User) AfterCreate(tx *gorm.DB) error {
    // 这里能拿到自增 ID，适合写关联审计记录
    return tx.Create(&AuditLog{
        TargetID: u.ID,
        Action:   "user.create",
    }).Error
}
```

⚠ 批量 `Create(&users)` 时，`BeforeCreate` / `AfterCreate` 会**为切片里的每条记录各调用一次**。如果钩子里有外部调用（发短信、调接口），批量 1000 条就是 1000 次，务必改成批量处理。

## 2.2 Update 钩子

```go
func (u *User) BeforeUpdate(tx *gorm.DB) error {
    // 注意：只有通过 Model/Updates 触发且携带了完整模型时，钩子才会被调用
    if u.Status == "deleted" && u.PreviousStatus != "" {
        return errors.New("已删除用户不允许再更新")
    }
    return nil
}

func (u *User) AfterUpdate(tx *gorm.DB) error {
    // 清缓存要在提交后做，钩子内清缓存可能因回滚而不一致
    return tx.Exec("DELETE FROM cache WHERE k = ?", "user:"+strconv.Itoa(int(u.ID))).Error
}
```

⚠ `db.Model(&User{}).Where(...).Update(...)` 这种**不带完整模型实例**的批量更新，`BeforeUpdate`/`AfterUpdate` 仍会被调用，但 `Statement.ReflectValue` 是零值，你在钩子里读不到被更新行的旧值。需要旧值时，必须在业务层先查。

## 2.3 Delete 钩子

```go
func (u *User) BeforeDelete(tx *gorm.DB) error {
    var count int64
    // 有未完成的订单就不允许删除
    if err := tx.Model(&Order{}).
        Where("user_id = ? AND status = ?", u.ID, "pending").
        Count(&count).Error; err != nil {
        return err
    }
    if count > 0 {
        return errors.New("存在未完成订单，不能删除")
    }
    return nil
}
```

注意：软删除时 `Delete` 实际是 `UPDATE`，`BeforeDelete`/`AfterDelete` 依然会触发，但执行的是更新语句。

## 2.4 Find 钩子

```go
func (u *User) AfterFind(tx *gorm.DB) error {
    // 把库里存的手机号做脱敏，供接口直接返回
    if len(u.Phone) == 11 {
        u.Phone = u.Phone[:3] + "****" + u.Phone[7:]
    }
    return nil
}
```

⚠ `AfterFind` 会改变返回给业务的数据。如果同一个对象后续要用于 `Save`，脱敏后的值会被写回数据库。**展示层脱敏不要放在 `AfterFind`**，放在 DTO 转换层更安全。

# 3. 钩子的事务语义

## 3.1 钩子运行在同一事务里

```go
db.Transaction(func(tx *gorm.DB) error {
    // tx.Create 触发的 BeforeCreate/AfterCreate 都在这个事务中
    if err := tx.Create(&order).Error; err != nil {
        return err
    }
    return nil
})
```

★ **关键认知**：钩子拿到的 `tx` 与触发它的语句属于**同一个事务**。钩子里的写操作要么一起提交，要么一起回滚。这既是优点（一致性有保障），也是风险（钩子里的错误会连累主操作）。

## 3.2 返回错误即回滚

```go
func (o *Order) AfterCreate(tx *gorm.DB) error {
    if err := tx.Create(&Outbox{
        Event:   "order.created",
        Payload: o.OrderNo,
    }).Error; err != nil {
        return err // 主订单的插入也会回滚
    }
    return nil
}
```

这是"本地消息表"模式的常见实现：业务表与消息表在同一个事务里写入，保证"业务成功则消息一定发出"。

## 3.3 钩子里不要用 db

```go
// 错误示范：用了全局 db，脱离了事务
func (u *User) AfterCreate(tx *gorm.DB) error {
    return db.Create(&AuditLog{...}).Error // 事务回滚了，审计日志却留下了
}

// 正确：用参数里的 tx
func (u *User) AfterCreate(tx *gorm.DB) error {
    return tx.Create(&AuditLog{...}).Error
}
```

⚠ 另外不要在钩子里开启新事务或用 `tx.Begin()`，会和外层事务形成两个连接，直接死锁（外层持锁等内层，内层等外层释放）。

# 4. 插件机制

## 4.1 Plugin 接口

```go
type Plugin interface {
    Name() string
    Initialize(*gorm.DB) error
}
```

```go
type TenantPlugin struct {
    TenantID uint
}

func (p *TenantPlugin) Name() string {
    return "tenant"
}

func (p *TenantPlugin) Initialize(db *gorm.DB) error {
    // 在查询回调里注入租户条件
    return db.Callback().Query().Before("gorm:query").Register("tenant:query", func(tx *gorm.DB) {
        if tx.Statement.Table == "" {
            return
        }
        tx.Statement.AddClause(clause.Where{
            Exprs: []clause.Expression{
                clause.Eq{Column: clause.Column{Name: "tenant_id"}, Value: p.TenantID},
            },
        })
    })
}

db.Use(&TenantPlugin{TenantID: 42}) // 全局生效
```

⚠ 插件是**全局状态**。一个 `*gorm.DB` 实例上只适合挂固定租户。多租户服务应该为每个请求 `db.Session()` 并从 context 取租户，或用 `db.Scopes()` 显式传入，而不是把租户 ID 焊死在全局插件里。

## 4.2 回调注册

```go
// 在 create 之前执行
db.Callback().Create().Before("gorm:create").Register("audit:before_create", func(tx *gorm.DB) {
    if tx.Statement.Schema == nil {
        return
    }
    if field := tx.Statement.Schema.LookUpField("CreatedBy"); field != nil {
        if userID, ok := tx.Statement.Context.Value("user_id").(uint); ok {
            _ = field.Set(tx.Statement.Context, tx.Statement.ReflectValue, userID)
        }
    }
})

// 移除
db.Callback().Create().Remove("audit:before_create")
```

| 方式 | 粒度 | 适用 |
|------|------|------|
| 模型钩子 | 单个模型 | 业务规则、字段加工 |
| 回调 | 全局所有模型 | 审计字段、租户过滤、日志 |
| 插件 | 打包多个回调 | 可复用的通用能力 |

★ **关键认知**：钩子适合**写业务的人**，回调适合**写基础设施的人**。团队里如果每个人都往回调里塞逻辑，排查问题时将无法知道某条 WHERE 是谁加的。回调必须有明确的主人。

# 5. 日志

## 5.1 日志级别

```go
import (
    "gorm.io/gorm/logger"
)

db, err := gorm.Open(dsn, &gorm.Config{
    Logger: logger.Default.LogMode(logger.Info),
})
```

| 级别 | 输出内容 |
|------|----------|
| `Silent` | 不输出 |
| `Error` | 只输出错误 |
| `Warn` | 错误 + 慢查询 |
| `Info` | 所有 SQL |

## 5.2 慢查询日志

```go
newLogger := logger.New(
    log.New(os.Stdout, "\r\n", log.LstdFlags),
    logger.Config{
        SlowThreshold:             200 * time.Millisecond, // 超过即标记慢查询
        LogLevel:                  logger.Warn,
        IgnoreRecordNotFoundError: true,  // 不把"记录不存在"当错误刷屏
        Colorful:                  false, // 生产环境关掉颜色
    },
)

db, err := gorm.Open(dsn, &gorm.Config{Logger: newLogger})
```

⚠ 生产环境长期开 `Info` 会把每条 SQL 和参数打进日志，既有性能开销也有泄露敏感数据的风险（密码、身份证号会出现在日志里）。标准做法是默认 `Warn`，按需临时调高。

# 6. 横切逻辑实践

## 6.1 审计字段

```go
type BaseModel struct {
    CreatedBy uint `gorm:"not null;default:0"`
    UpdatedBy uint `gorm:"not null;default:0"`
}

func RegisterAudit(db *gorm.DB) {
    db.Callback().Create().Before("gorm:create").Register("audit:create", func(tx *gorm.DB) {
        uid := currentUserID(tx.Statement.Context)
        setField(tx, "CreatedBy", uid)
        setField(tx, "UpdatedBy", uid)
    })

    db.Callback().Update().Before("gorm:update").Register("audit:update", func(tx *gorm.DB) {
        setField(tx, "UpdatedBy", currentUserID(tx.Statement.Context))
    })
}

func setField(tx *gorm.DB, name string, value any) {
    if tx.Statement.Schema == nil {
        return
    }
    if f := tx.Statement.Schema.LookUpField(name); f != nil {
        _ = f.Set(tx.Statement.Context, tx.Statement.ReflectValue, value)
    }
}
```

调用方只需要把用户 ID 放进 context：

```go
ctx := context.WithValue(r.Context(), "user_id", uint(7))
db.WithContext(ctx).Create(&order)
```

## 6.2 多租户

```go
const tenantKey = "tenant_id"

func TenantScope(ctx context.Context) func(*gorm.DB) *gorm.DB {
    return func(db *gorm.DB) *gorm.DB {
        tenantID, ok := ctx.Value(tenantKey).(uint)
        if !ok || tenantID == 0 {
            // 没有租户上下文时直接返回一个必然为空的条件，避免越权
            return db.Where("1 = 0")
        }
        return db.Where("tenant_id = ?", tenantID)
    }
}

db.WithContext(ctx).Scopes(TenantScope(ctx)).Find(&orders)
```

⚠ 多租户最危险的失败模式是"忘了加条件"，而不是"加错条件"。因此**漏租户时应该返回空结果（`1 = 0`），而不是返回全量**。这个默认值选择比任何注释都有效。

# 7. 钩子误用

| 误用 | 后果 | 正确做法 |
|------|------|----------|
| 钩子里调 `db` 而非 `tx` | 脱离事务，回滚后留下脏数据 | 一律用 `tx` |
| 钩子里发 HTTP/消息 | 事务被拉长，超时、重复投递 | 用 outbox 表，事务提交后再投递 |
| 钩子里再开事务 | 连接耗尽或死锁 | 复用 `tx` |
| `AfterFind` 里改业务值 | 值被写回库 | 用 DTO 转换 |
| 批量操作里逐条发通知 | N 次外部调用 | 汇总后批量处理 |
| 回调里静默加条件 | 排查困难 | 集中注册、统一日志 |

# 8. 动手练习

1. 给 `User` 实现 `BeforeCreate`，要求自动生成 UUID 主键、对密码加盐哈希；再实现 `BeforeDelete`，要求存在未完成订单时拒绝删除。
2. 用回调实现 `CreatedBy`/`UpdatedBy` 自动填充，要求从 `context` 取当前用户，取不到时写入 0 而不是报错。
3. 写一个多租户 Scope，要求：无租户上下文时返回空集；并解释为什么不应该把它写成全局插件。

# 9. 小结

| 主题 | 要点 |
|------|------|
| 钩子 | 模型实现固定方法名，指针接收者，返回 error 即中断 |
| 公共钩子 | `BeforeSave`/`AfterSave` 在 Create 和 Update 都会触发 |
| 事务 | 钩子与主语句同事务，必须用 `tx` 而非 `db` |
| 批量 | Create 钩子按记录调用，注意外部调用次数 |
| 插件 | `Name` + `Initialize`，底层是回调注册 |
| 日志 | 生产用 `Warn` + 慢查询阈值，别长期 `Info` |
| 审计/租户 | 用回调统一注入，漏条件时返回空集 |

下一篇进入性能与索引，讲清索引的最左前缀与区分度、`EXPLAIN` 该怎么看、批量插入与字段裁剪的收益、连接池参数怎么配，以及如何用基准测试量化优化效果。
