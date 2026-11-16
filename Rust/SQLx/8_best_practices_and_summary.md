前面七篇分别讲了连接、查询、迁移、事务、错误和测试。这一篇不再引入新 API，而是把散落的最佳实践收拢成一套可执行的项目规范：代码怎么分层、SQL 放在哪、类型怎么映射、哪些性能坑必须提前避、上线前检查什么。

# 1. 代码分层

## 1.1 推荐结构

```text
src/
├── db.rs            // 建池、跑迁移、关闭
├── models/user.rs   // FromRow 结构体与领域类型
├── repo/users.rs    // SQL、事务、错误映射
├── service/         // 业务规则与事务边界
├── handlers/        // 参数提取、响应
└── main.rs
```

| 层 | 职责 | 不该做 |
|----|------|--------|
| Handler | 参数提取、状态码、序列化 | 写 SQL、开事务 |
| Service | 业务规则、事务边界、补偿 | 拼 SQL |
| Repo | 查询、映射、错误分类 | 处理 HTTP |
| Model | 数据结构与类型映射 | 依赖 Web 框架 |

★ **关键认知**：分层的价值不在于"文件好看"，而在于**可测试性**。Repo 接受 `impl PgExecutor`，就能在事务里单测；Service 只依赖 Repo 的 trait，就能用 mock 测业务；Handler 只做转换，薄到不需要数据库。⚠ 如果 Handler 里直接 `pool.begin()`，事务边界会散落在各处，测试也绕不开真实数据库。

## 1.2 用 trait 隔离 Repo（可选）

做法是定义一个 `UserRepo` trait，声明 `find_by_id`、`create` 等方法；生产实现 `PgUserRepo` 持有一个 `PgPool`，测试实现可以用内存 map 或 mock。

★ **关键认知**：trait 不是必须的。项目小、只有一种数据库时，直接用具体 struct + `PgPool` 更简单；只有在"需要 mock"或"多数据源"时才引入 trait。⚠ 过早抽象会让每个查询都要写两遍（trait 定义 + 实现），收益却为零。

# 2. 查询组织

## 2.1 SQL 放哪

| 方式 | 优点 | 缺点 |
|------|------|------|
| 内联在函数里 | 就近、易读 | 复杂 SQL 撑爆函数 |
| 模块级常量 | 复用、易审阅 | 参数位置要人工对齐 |
| 独立 `.sql` 文件 + `include_str!` | 可格式化、可用 SQL 工具检查 | 失去 `query!` 的编译期校验 |

```rust
// 模块级常量：适合 10 行以上的查询
const LIST_ACTIVE_USERS: &str = r#"
    SELECT id, name, email FROM users
    WHERE deleted_at IS NULL
    ORDER BY created_at DESC
    LIMIT $1 OFFSET $2
"#;

let users = sqlx::query_as::<_, User>(LIST_ACTIVE_USERS)
    .bind(limit).bind(offset).fetch_all(pool).await?;
```

⚠ `query!` 宏的参数必须是**字符串字面量**，传常量会编译失败。想要编译期校验就只能内联 SQL，二者取其一。

## 2.2 统一分页参数

```rust
#[derive(Debug, Clone, Copy)]
pub struct Page {
    pub limit: i64,
    pub offset: i64,
}

impl Page {
    pub fn new(page: i64, size: i64) -> Self {
        let size = size.clamp(1, 100); // 上限 100，防拖库
        Self { limit: size, offset: (page.max(1) - 1) * size }
    }
}
```

★ **关键认知**：**分页大小必须设上限**。`?size=1000000` 让客户端直接拉全表，是性能事故的常见入口。⚠ `offset` 用 `i64` 而不是 `u32`，避免大页码溢出；同时深分页要换游标方案。

## 2.3 查询只取需要的列

```rust
// 不推荐：SELECT *，表结构一变映射就崩
sqlx::query_as::<_, User>("SELECT * FROM users WHERE id = $1");

// 推荐：显式列，和 FromRow 字段一一对应
sqlx::query_as::<_, User>("SELECT id, name, email FROM users WHERE id = $1");
```

# 3. 类型映射

## 3.1 领域新类型

```rust
// 原始新类型
#[derive(Debug, Clone, PartialEq, Eq, Hash)]
pub struct UserId(pub i64);

// SQLx 0.8：transparent 让新类型直接映射底层类型
#[derive(Debug, sqlx::Type)]
#[sqlx(transparent)]
pub struct OrderId(i64);
```

⚠ `transparent` 只适合"包装单一字段"的新类型。需要校验的 `Email` 不应该直接透明映射，应该在构造时校验，再提供 `as_str()` 给查询使用，否则绕过校验的非法值也能进数据库。

## 3.2 时间与时区

| 数据库类型 | 推荐 Rust 类型 | 说明 |
|------------|----------------|------|
| `TIMESTAMPTZ` | `DateTime<Utc>` | 带时区，存 UTC，显示时再转 |
| `TIMESTAMP` | `NaiveDateTime` | 不带时区，容易踩坑 |

★ **关键认知**：**优先用 `TIMESTAMPTZ` + `DateTime<Utc>`**。数据库存绝对时刻，应用层统一 UTC，只在返回给用户时按本地时区格式化。⚠ `TIMESTAMP` 不带时区，服务器时区一变，同一份数据会被解释成不同时刻，跨时区服务几乎必踩。

## 3.3 JSON 与数值

```rust
#[derive(Debug, sqlx::FromRow)]
struct Event {
    id: i64,
    payload: serde_json::Value, // 需要 json feature
}

#[derive(Debug, sqlx::FromRow)]
struct Order {
    id: i64,
    total: rust_decimal::Decimal, // 金额用 NUMERIC，不要用 f64
}
```

| 场景 | 不要用 | 推荐 |
|------|--------|------|
| 金额 | `f64` | `Decimal` 或整数分 |
| 大整数 id | `i32` | `i64` |
| 时间 | 字符串 | `DateTime<Utc>` |
| 枚举 | 裸字符串 | Rust enum + `#[sqlx(type_name)]` 或显式转换 |
| 可空列 | 默认值填充 | `Option<T>` |

⚠ 用 `f64` 存金额，`0.1 + 0.2 != 0.3` 会在对账时变成灾难。哪怕数据库是 `NUMERIC`，映射成 `f64` 也会丢精度。

# 4. 性能

## 4.1 N+1 查询

```rust
// 错误示范：先查 100 个用户，再逐个查订单，共 101 次往返
let users = list_users(pool).await?;
for user in &users {
    let orders = list_orders_by_user(pool, user.id).await?;
}

// 方案 1：一次 JOIN 查回，再在内存里按 user_id 分组
let rows = sqlx::query_as::<_, UserWithOrder>(
    "SELECT u.id, u.name, o.id AS order_id, o.total
     FROM users u LEFT JOIN orders o ON o.user_id = u.id
     WHERE u.deleted_at IS NULL",
)
.fetch_all(pool).await?;

// 方案 2：两次查询 + IN，注意参数上限
let ids: Vec<i64> = users.iter().map(|u| u.id).collect();
let orders = sqlx::query_as::<_, Order>(
    "SELECT id, user_id, total FROM orders WHERE user_id = ANY($1)",
)
.bind(&ids).fetch_all(pool).await?;
```

★ **关键认知**：N+1 的危害是**往返次数**，不是数据量。每次往返哪怕只有 1 毫秒，1000 个用户也会多出 1 秒。⚠ 用 `IN` 批量查时注意参数上限：PostgreSQL 单条语句最多 65535 个参数，超了要分批。

## 4.2 批量插入

```rust
// 慢：循环单条插入，每次一个往返
for user in &users {
    sqlx::query("INSERT INTO users (name, email) VALUES ($1, $2)")
        .bind(&user.name).bind(&user.email)
        .execute(pool).await?;
}

// 快：QueryBuilder 拼多值 INSERT，一次往返
use sqlx::QueryBuilder;

let mut builder: QueryBuilder<sqlx::Postgres> =
    QueryBuilder::new("INSERT INTO users (name, email) ");
builder.push_values(users, |mut b, user| {
    b.push_bind(&user.name).push_bind(&user.email);
});
builder.build().execute(pool).await?;
```

| 方式 | 往返次数 | 适合 |
|------|----------|------|
| 循环单条 | N | 数据量小、需要逐条错误处理 |
| 多值 INSERT | 1 | 批量导入、每批 500-1000 条 |
| `COPY` | 1 | 超大导入（PostgreSQL 专用） |

⚠ 单条 SQL 太大也会撑爆内存和日志，批量按 500-1000 条切块。MySQL 还有 `max_allowed_packet` 限制。

## 4.3 索引与执行计划

```sql
-- 看查询有没有走索引
EXPLAIN ANALYZE
SELECT id, name FROM users WHERE email = 'a@example.com';
```

```text
Seq Scan on users  (cost=0.00..1550.00 rows=1 width=32)
  Rows Removed by Filter: 99999   ← 全表扫了 10 万行
```

| 信号 | 含义 | 处理 |
|------|------|------|
| `Seq Scan` 大表 | 没走索引 | 检查 WHERE/JOIN 列是否有索引 |
| `Rows Removed by Filter` 很大 | 索引选择性差 | 考虑复合索引或调整条件 |

★ **关键认知**：索引不是越多越好。每个索引都会拖慢写入、占用空间，还会让优化器选择变多。**为实际查询建索引**，上线后用 `pg_stat_statements` 找最慢的语句，而不是凭感觉加。⚠ 复合索引遵循最左前缀，`(user_id, created_at)` 能服务 `WHERE user_id = ? ORDER BY created_at`，但服务不了单独按 `created_at` 查。

## 4.4 连接与并发

池大小按副本数与数据库上限倒推，`after_connect` 统一设 `statement_timeout`，事务只包数据库操作；读多写少时做读写分离，热点数据先查 Redis。

# 5. 安全

## 5.1 注入防线

```text
第一层：值一律 .bind() / .push_bind()
第二层：标识符走白名单，绝不拼用户输入
第三层：数据库账号最小权限，只给需要的表
第四层：应用层校验，但把约束也写进数据库
```

```sql
-- 最小权限：应用账号只能读写业务表，不给 DROP、TRUNCATE、CREATE
GRANT SELECT, INSERT, UPDATE, DELETE ON users, orders TO app_user;
```

★ **关键认知**：应用层校验和数据库约束不是二选一，而是**两道防线**。应用层给友好错误，数据库约束保证并发下也不出错。⚠ 只靠应用层"先查再插"，并发时仍会插入重复数据；只靠数据库约束，用户会收到难懂的错误。两边都要有。

## 5.2 敏感数据

| 风险 | 做法 |
|------|------|
| 连接串泄漏 | `.env` 不入库，生产用密钥管理 |
| 密码明文存储 | Argon2/bcrypt 哈希 |
| 日志与错误回显 | 参数脱敏，对外固定文案，详情只进日志 |

## 5.3 SQL 日志

开发环境可以用 `tracing_subscriber` 打印 SQL，生产环境降级到 `INFO`。⚠ 慢查询日志里可能带参数值，采集前要评估合规要求。

# 6. 常见坑汇总

| 现象 | 根因 | 修复 |
|------|------|------|
| 池被打满 | 慢查询、长事务、连接泄漏 | 设语句超时，缩短事务 |
| 回滚没生效 | 语句用了 `pool` 而非 `tx` | 事务内只传连接 |
| 唯一冲突 500 | 没分类错误 | `is_unique_violation` → 409 |
| 测试互相污染 | 共享库、没清理 | 独立库或事务回滚 |
| 编译期检查失效 | 改了 schema 没 `prepare` | CI 跑 `prepare --check` |
| 深分页越来越慢 | `OFFSET` 扫描 | 改游标分页 |
| 金额对不上 | `f64` 精度 | 用 `Decimal` 或整数分 |
| 时间差 8 小时 | `TIMESTAMP` 无时区 | 改 `TIMESTAMPTZ` + UTC |
| 迁移锁表 | 大表 DDL | `CONCURRENTLY` + `lock_timeout` |
| 动态 SQL 注入 | `format!` 拼用户输入 | `QueryBuilder::push_bind` |

# 7. 上线检查清单

```text
[ ] 连接池上限按副本数倒推，留 20% 余量
[ ] 设置了 acquire_timeout 与 statement_timeout
[ ] 所有查询使用 bind，无字符串拼接用户输入
[ ] 迁移能在空库从零跑通，且已 prepare
[ ] 事务内没有 HTTP/消息/文件 IO
[ ] 唯一冲突、外键冲突、池超时都有明确状态码
[ ] 分页有大小上限
[ ] 金额用 Decimal，时间用 TIMESTAMPTZ
[ ] 应用账号最小权限，连接串不入库
[ ] 关键路径有集成测试，CI 里真实跑数据库
```

# 8. 动手练习

1. 把前七篇的示例整合成一个小项目：用户列表（分页 + 搜索）、创建用户（唯一冲突 409）、转账（事务 + 原子更新），按 Handler/Service/Repo 分层。
2. 用 `EXPLAIN ANALYZE` 对比 `email` 列有无索引时查询 10 万行数据的耗时，并解释 `Seq Scan` 与 `Index Scan` 的差异。
3. 给项目加一份上线检查脚本：跑迁移、`prepare --check`、跑测试、检查 `.env` 是否被 git 跟踪，任何一步失败就退出非零。

验收标准：项目能在一个空数据库上从零启动；并发测试下余额守恒；检查脚本在故意改坏 schema 后能报错。

# 9. 小结

| 主题 | 要点 |
|------|------|
| 分层 | Handler 薄、Service 管事务、Repo 管 SQL |
| SQL 组织 | 短查询内联，长查询常量，`query!` 需字面量 |
| 类型 | 金额 `Decimal`，时间 `TIMESTAMPTZ`，可空用 `Option` |
| 性能 | 消 N+1、批量写、按查询建索引 |
| 安全 | bind、白名单、最小权限、错误脱敏 |
| 协作 | 迁移可重复、`prepare` 入 CI、测试隔离 |
| 上线 | 池、超时、错误映射、检查清单逐项过 |

SQLx 的八篇到这里就完整了：从建立连接池，到查询、迁移、事务、错误、测试，再到工程规范。下一步可以进入 `../Axum/7_database_integration.md`，看这套数据库能力如何注入 Web 服务的 `AppState`、如何把 `AppError` 变成统一的 HTTP 响应，以及鉴权、分页、审计这些横切逻辑如何与 Repo 层配合。
