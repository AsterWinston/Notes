GORM 本身几乎不是性能瓶颈，真正拖慢系统的是它替你生成的那些 SQL：缺索引的全表扫描、一次查回所有列、循环里发 N 条查询、连接池配得太小。本篇从索引讲到执行计划，再讲到连接池与压测，目标是让你能**用数据而不是感觉**判断一次优化是否有效。

# 1. 性能问题的来源

## 1.1 三类瓶颈

```text
一次请求的耗时去向
┌────────────────────────────────────────────┐
│ 网络往返  │ SQL 执行（索引 / 扫描 / 排序）  │ 结果映射 │
└───────────┴─────────────────────────────────┴──────────┘
     ▲                    ▲                       ▲
  连接池太小         缺索引、全表扫描          SELECT * 取回
  跨机房部署         回表、文件排序           大字段无谓传输
```

| 瓶颈类型 | 典型症状 | 优先动作 |
|----------|----------|----------|
| SQL 层 | 单条变慢，数据量涨后突然劣化 | 加索引、改查询 |
| 交互层 | 请求数随数据量线性增长 | 消除 N+1、批量操作 |
| 连接层 | 高并发下排队、超时 | 调连接池、缩短事务 |

★ **关键认知**：优化顺序应该是"先减少 SQL 条数，再优化单条 SQL，最后才调参数"。把 100 条查询合成 2 条，收益远大于把每条查询优化 10%。

# 2. 索引设计

## 2.1 最左前缀

复合索引 `(tenant_id, status, created_at)` 的可用方式：

| 查询条件 | 是否走索引 |
|----------|------------|
| `tenant_id = ?` | 是 |
| `tenant_id = ? AND status = ?` | 是 |
| `tenant_id = ? AND status = ? AND created_at > ?` | 是 |
| `status = ?` | 否（跳过最左列） |
| `tenant_id = ? AND created_at > ?` | 部分用（只能用 `tenant_id`） |
| `tenant_id = ? AND status > ? AND created_at > ?` | 只能用前两列 |

★ **关键认知**：范围条件会**终止**后续列在索引中的使用。所以复合索引的列顺序应遵循"等值列在前，范围列在后"。把 `created_at` 放在 `status` 前面，会让 `status` 的等值过滤失效。

## 2.2 区分度

```sql
-- 区分度 = 不同值数量 / 总行数，越接近 1 越适合建索引
SELECT COUNT(DISTINCT status) / COUNT(*) FROM orders;      -- 例如 0.0001，不适合单列索引
SELECT COUNT(DISTINCT user_id) / COUNT(*) FROM orders;     -- 例如 0.8，适合
```

低区分度列（性别、状态、是否删除）单独建索引通常没用，优化器会判断"走索引再回表"比全表扫描更贵。它们适合作为**复合索引的前缀**，配合高区分度列使用。

## 2.3 覆盖索引

如果查询需要的列全部在索引里，数据库不必回表：

```sql
-- 索引 (user_id, created_at, amount)
SELECT user_id, created_at, amount FROM orders WHERE user_id = ?;
-- Extra: Using index  ← 覆盖索引，不回表
```

对应 GORM：

```go
type Order struct {
    ID        uint      `gorm:"primaryKey"`
    UserID    uint      `gorm:"index:idx_user_created_amount,priority:1"`
    CreatedAt time.Time `gorm:"index:idx_user_created_amount,priority:2"`
    Amount    int64     `gorm:"index:idx_user_created_amount,priority:3"`
}

// 只 Select 索引内的列，才有机会走覆盖索引
db.Select("user_id", "created_at", "amount").
    Where("user_id = ?", uid).
    Find(&orders)
```

⚠ 覆盖索引的收益要用 `EXPLAIN` 的 `Extra` 里是否出现 `Using index` 验证，不要假设写了 `Select` 就一定走覆盖。

## 2.4 常见的索引失效

| 写法 | 为什么失效 | 改法 |
|------|------------|------|
| `WHERE DATE(created_at) = '2026-01-01'` | 列上套函数 | 改范围：`>= 当天 AND < 次日` |
| `WHERE name LIKE '%abc'` | 前导通配 | 改后缀匹配或全文索引 |
| `WHERE phone = 13800000000`（列是字符串） | 隐式类型转换 | 参数用字符串 |
| `WHERE status != 1` | 不等号通常不划算 | 改 `IN` 或拆查询 |
| `WHERE a = 1 OR b = 2` | 任一侧无索引即全表 | 建复合索引或改 `UNION` |
| `ORDER BY name` 但索引是 `(id, name)` | 排序字段非最左 | 调整索引顺序 |

## 2.5 用标签建索引

```go
type Order struct {
    ID         uint      `gorm:"primaryKey"`
    TenantID   uint      `gorm:"index:idx_tenant_status_created,priority:1"`
    Status     string    `gorm:"size:16;index:idx_tenant_status_created,priority:2"`
    CreatedAt  time.Time `gorm:"index:idx_tenant_status_created,priority:3"`
    OrderNo    string    `gorm:"size:32;uniqueIndex:uk_order_no"`
    UserID     uint      `gorm:"index:idx_user"`
}
```

| 场景 | 标签 |
|------|------|
| 唯一约束 | `uniqueIndex:uk_xxx` |
| 复合索引顺序 | 同名索引 + `priority:N` |
| 前缀索引（MySQL） | `index:idx_name,length:16` |
| 索引注释 | `index:idx_x,comment:订单查询` |

# 3. 读懂执行计划

## 3.1 先拿到真实 SQL

```go
sql := db.ToSQL(func(tx *gorm.DB) *gorm.DB {
    var orders []Order
    return tx.Model(&Order{}).
        Where("tenant_id = ? AND status = ?", 1, "paid").
        Order("created_at DESC").
        Limit(20).
        Find(&orders)
})
fmt.Println(sql)
```

输出类似：

```sql
SELECT * FROM `orders` WHERE tenant_id = 1 AND status = 'paid' ORDER BY created_at DESC LIMIT 20
```

## 3.2 EXPLAIN 关键列（MySQL）

```sql
EXPLAIN SELECT * FROM orders WHERE tenant_id = 1 AND status = 'paid' ORDER BY created_at DESC LIMIT 20;
```

| 列 | 关注点 |
|----|--------|
| `type` | 访问类型，`const`/`eq_ref`/`ref`/`range` 较好，`ALL` 是全表扫描 |
| `key` | 实际使用的索引，`NULL` 表示没走索引 |
| `key_len` | 用了索引的多少字节，可判断复合索引用到第几列 |
| `rows` | 预估扫描行数，越小越好 |
| `filtered` | 扫描后剩余比例 |
| `Extra` | `Using index` 覆盖索引；`Using filesort` 额外排序；`Using temporary` 临时表 |

```go
// 在 GORM 里直接看执行计划
type ExplainRow struct {
    ID    int
    Type  string
    Key   string
    Rows  int
    Extra string
}
var rows []ExplainRow
db.Raw("EXPLAIN SELECT * FROM orders WHERE tenant_id = ? AND status = ?", 1, "paid").Scan(&rows)
```

★ **关键认知**：判断慢查询只看"有没有走索引"是不够的。`type=range` + `rows=100000` 依然可能很慢。真正要盯的是**预估扫描行数**与**是否产生 filesort / temporary**。

# 4. 批量插入

## 4.1 逐条 vs 批量

```go
// 慢：1000 条 = 1000 次网络往返 + 1000 次解析
for _, u := range users {
    db.Create(&u)
}

// 快：一次 INSERT 多行，通常快一个数量级
db.Create(&users)
```

| 方式 | 网络往返 | 事务开销 | 适用量级 |
|------|----------|----------|----------|
| 逐条 `Create` | N | N | < 50 |
| `Create(&slice)` | 1 | 1 | 数百 |
| `CreateInBatches(&slice, 500)` | N/500 | N/500 | 上万 |

## 4.2 分批

```go
db.CreateInBatches(&users, 500) // 每 500 条一批，平衡 SQL 长度与往返次数
```

⚠ 批大小不是越大越好。MySQL 的 `max_allowed_packet` 默认 64MB，但一条 SQL 里几百个占位符已经会让解析变慢，同时长事务会放大锁范围。500 到 1000 是常见起点，最终要压测。

## 4.3 Upsert 批量

```go
db.Clauses(clause.OnConflict{
    Columns:   []clause.Column{{Name: "email"}},
    DoUpdates: clause.AssignmentColumns([]string{"name", "updated_at"}),
}).CreateInBatches(&users, 500)
```

# 5. 查询优化

## 5.1 字段裁剪

```go
// 差：把 TEXT 大字段也拉回来
db.Find(&articles)

// 好：列表页只取摘要所需字段
db.Select("id", "title", "author_id", "created_at").Find(&articles)
```

## 5.2 Count 的正确用法

```go
// 只需要"有没有"，用 EXISTS 比 COUNT 快
var exists bool
db.Raw("SELECT EXISTS(SELECT 1 FROM orders WHERE user_id = ?)", uid).Scan(&exists)

// 需要精确总数才用 COUNT，并尽量带上能走索引的条件
var total int64
db.Model(&Order{}).Where("tenant_id = ?", tid).Count(&total)
```

⚠ 分页接口常见的"每页都 `COUNT(*)` 全表"是慢查询大户。数据量大时改用"是否还有下一页"判断（多取一条），或维护计数表。

## 5.3 分页优化

```go
// 深分页：OFFSET 越大越慢，因为要扫描并丢弃前 N 行
db.Order("id DESC").Limit(20).Offset(100000).Find(&orders)

// 游标分页：用上一页最后一条的 id，直接定位
db.Where("id < ?", lastID).Order("id DESC").Limit(20).Find(&orders)
```

## 5.4 消除 N+1

```go
// 差：循环查询
for _, u := range users {
    db.Where("user_id = ?", u.ID).Find(&u.Orders)
}

// 好：一次 IN 查询
db.Preload("Orders").Find(&users)
```

# 6. 连接池

```go
sqlDB, err := db.DB()
if err != nil {
    return err
}

sqlDB.SetMaxOpenConns(50)                  // 最大打开连接数
sqlDB.SetMaxIdleConns(25)                  // 空闲连接数，建议为 MaxOpenConns 的一半
sqlDB.SetConnMaxLifetime(30 * time.Minute) // 小于数据库 wait_timeout
sqlDB.SetConnMaxIdleTime(5 * time.Minute)  // 空闲多久后真正关闭
```

| 参数 | 太小 | 太大 |
|------|------|------|
| `MaxOpenConns` | 请求排队、超时 | 数据库连接数耗尽，拖垮实例 |
| `MaxIdleConns` | 频繁建连 | 空闲连接占资源 |
| `ConnMaxLifetime` | 频繁重建连接 | 被数据库 `wait_timeout` 断开，出现 `invalid connection` |

★ **关键认知**：连接池大小要按**数据库能承载的总连接数**倒推，而不是拍脑袋。10 个服务实例 × 每个 100 连接 = 1000 连接，很可能直接超过 MySQL 的 `max_connections`。用 `MaxOpenConns × 实例数 < 数据库上限 × 0.8` 这个约束来定。

# 7. 慢查询治理

```go
slowLogger := logger.New(
    log.New(os.Stdout, "", log.LstdFlags),
    logger.Config{
        SlowThreshold:             200 * time.Millisecond,
        LogLevel:                  logger.Warn,
        IgnoreRecordNotFoundError: true,
    },
)
db, err := gorm.Open(dsn, &gorm.Config{Logger: slowLogger})
```

治理流程：

```text
发现慢查询
   ▼
抓出 SQL（日志 / 慢查询日志 / APM）
   ▼
EXPLAIN 看 type、key、rows、Extra
   ▼
补索引 / 改写法 / 减少数据量
   ▼
压测对比，确认 P99 下降
```

⚠ 不要看到慢查询就无脑加索引。索引会拖慢写入、占用磁盘，并可能让优化器选错执行计划。每个新索引都应该能对应到一个具体查询。

# 8. 基准测试

```go
func BenchmarkListOrders(b *testing.B) {
    db := setupTestDB(b)

    b.ResetTimer()
    for i := 0; i < b.N; i++ {
        var orders []Order
        if err := db.Where("tenant_id = ?", 1).
            Order("created_at DESC").
            Limit(20).
            Find(&orders).Error; err != nil {
            b.Fatal(err)
        }
    }
}
```

```bash
go test -bench=BenchmarkListOrders -benchmem -count=5
```

| 指标 | 含义 |
|------|------|
| `ns/op` | 单次操作耗时 |
| `B/op` | 单次分配字节数 |
| `allocs/op` | 单次分配次数 |
| `-count=5` | 多跑几轮，看波动 |

★ **关键认知**：基准测试必须连**真实规模的数据库**。在 100 行的空表上测出的"优化"，到 1000 万行时会得出相反结论——优化器会因为区分度太低而放弃索引。

# 9. 常见错误

| 现象 | 原因 | 解决 |
|------|------|------|
| 加了索引还是慢 | 索引顺序不符合最左前缀 | 按等值→范围重排 |
| 查询计划 `type=ALL` | 条件列无索引或隐式转换 | 补索引，核对字段类型 |
| `Using filesort` | 排序字段不在索引顺序里 | 建 `(where..., order...)` 复合索引 |
| 批量插入慢 | 逐条 Create | `CreateInBatches` |
| 高并发超时 | 连接池太小 | 调 `MaxOpenConns` 并压测 |
| 每页都慢 | 深分页 + 每次 COUNT | 游标分页 + 是否还有下一页 |

# 10. 动手练习

1. 造一张 100 万行的订单表，对比 `WHERE tenant_id = ? AND status = ?` 在无索引、单列索引、复合索引下的 `EXPLAIN` 与耗时。
2. 用 `db.ToSQL` 打印一个分页查询，改写成游标分页，比较两者的执行计划和 `rows` 预估值。
3. 写一个基准测试，对比 `Create` 逐条 1000 条与 `CreateInBatches(500)` 的耗时，并解释差异来自哪里。

# 11. 小结

| 主题 | 要点 |
|------|------|
| 优化顺序 | 先减 SQL 条数，再优化单条，最后调参数 |
| 索引 | 等值列在前、范围列在后；区分度低的列配高区分度列用 |
| 执行计划 | 看 `type`、`key`、`rows`、`Extra`，不只看有没有索引 |
| 批量 | `CreateInBatches`，批大小 500-1000 起压测 |
| 分页 | 深分页改游标；总数用 EXISTS 或额外维护 |
| 连接池 | `MaxOpenConns × 实例数` 不能超过数据库上限 |
| 验证 | 基准测试要连真实规模数据，看 `ns/op` 与 `allocs/op` |

下一篇是模块收尾，把 GORM 与 sqlx、原生 `database/sql` 放在一起对比，梳理知识地图、高频面试题与生产检查清单，帮助你把前面八篇串成一套可复用的判断标准。
