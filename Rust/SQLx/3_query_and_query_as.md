SQLx 的查询 API 不多，但每个函数都有明确分工：要不要映射结构体、要一行还是多行、是取标量还是只关心影响行数。用错函数不会崩溃，却会写出"多查一次""错误被吞掉"的代码。本篇把 `query` 系列讲透，并把参数绑定、`FromRow` 映射和动态 SQL 的边界一次说清。

# 1. 查询入口的分工

## 1.1 四个函数

| 函数 | 结果映射 | 典型用途 |
|------|----------|----------|
| `sqlx::query` | 不映射，返回 `PgRow` | 手写 `row.get()`、无需结构体 |
| `sqlx::query_as` | 映射到 `FromRow` 类型 | 绝大多数业务查询 |
| `sqlx::query_scalar` | 单列单值 | `COUNT(*)`、`EXISTS`、单字段 |
| `sqlx::query_as!` | 编译期校验 + 映射 | 想尽早发现 SQL 与 schema 不一致 |

## 1.2 执行方式

| 方法 | 返回 | 无结果时 |
|------|------|----------|
| `.fetch_all(executor)` | `Vec<T>` | 空 `Vec`，不报错 |
| `.fetch_one(executor)` | `T` | `Error::RowNotFound` |
| `.fetch_optional(executor)` | `Option<T>` | `None` |
| `.execute(executor)` | `PgQueryResult` | 不含行数据，含 `rows_affected()` |
| `.fetch(executor)` | `Stream<Item = Result<T, Error>>` | 需要逐行处理时用 |

★ **关键认知**：`fetch_one` 与 `fetch_optional` 的区别不是"性能"，而是**语义**。查询"至多一行"（按主键查）用 `fetch_optional`；查询"必须存在一行"（插入后 `RETURNING`）用 `fetch_one`。⚠ 用 `fetch_one` 查可能不存在的记录，会把"没找到"变成 `500`；用 `fetch_optional` 查必须存在的记录，则把"数据异常"藏进了 `None`。

## 1.3 最小示例

```rust
use sqlx::{PgPool, Row};

#[derive(Debug, sqlx::FromRow)]
struct User {
    id: i64,
    name: String,
    email: String,
}

async fn demo(pool: &PgPool, id: i64) -> Result<(), sqlx::Error> {
    // 1. query：自己按列名取值
    let row = sqlx::query("SELECT id, name FROM users WHERE id = $1")
        .bind(id)
        .fetch_one(pool)
        .await?;
    let name: String = row.get("name");

    // 2. query_as：映射到结构体
    let user = sqlx::query_as::<_, User>(
        "SELECT id, name, email FROM users WHERE id = $1",
    )
    .bind(id)
    .fetch_optional(pool)
    .await?;

    // 3. query_scalar：只取一个值
    let total: i64 = sqlx::query_scalar("SELECT COUNT(*) FROM users")
        .fetch_one(pool)
        .await?;

    println!("{} {} {}", name, user.is_some(), total);
    Ok(())
}
```

# 2. 参数绑定

## 2.1 bind 的规则

```rust
let user = sqlx::query_as::<_, User>(
    "SELECT id, name, email FROM users WHERE id = $1 AND name = $2",
)
.bind(id)        // 对应 $1
.bind(name)      // 对应 $2
.fetch_optional(pool)
.await?;
```

绑定顺序与占位符编号一一对应。PostgreSQL 用 `$1`、`$2`，MySQL 和 SQLite 用 `?`，方言不同但语义一致。

## 2.2 为什么 bind 能防注入

数据库的预处理协议分两步：先发送带占位符的 SQL 模板，数据库把它编译成执行计划；再单独发送参数值。参数在第二步才到达，此时 SQL 结构已经固定，值不可能再被解析成语法。

```text
拼字符串（危险）：
"SELECT * FROM users WHERE name = '" + input + "'"
     └─ 输入 ' OR '1'='1 会改变 SQL 结构

bind（安全）：
第一步：SELECT * FROM users WHERE name = $1   ← 结构已编译
第二步：$1 = "' OR '1'='1"                    ← 只是字符串值
```

★ **关键认知**：防注入的本质不是"转义特殊字符"，而是**让值永远不进入 SQL 语法层**。`.bind()` 走的正是预处理协议。⚠ 只要你用 `format!`、字符串拼接、`QueryBuilder::push` 拼用户输入，就等于把值送回了语法层，SQLx 帮不了你。

## 2.3 错误示范与正确做法

```rust
// 错误示范：把用户输入拼进 SQL，注入风险
let sql = format!("SELECT * FROM users WHERE name = '{}'", name);
sqlx::query(&sql).fetch_all(pool).await?;

// 正确做法：占位符 + bind
let users = sqlx::query_as::<_, User>(
    "SELECT id, name, email FROM users WHERE name = $1",
)
.bind(name)
.fetch_all(pool)
.await?;
```

LIKE 查询同理：把 `%` 放进绑定值里，SQL 中只保留占位符，例如 `.bind(format!("%{}%", keyword))`。

⚠ 表名、列名、`ORDER BY` 字段**不能用占位符**，因为它们在 SQL 解析阶段就要确定。这类动态标识符必须走白名单：用 `match` 把用户输入映射到固定的静态字符串，再拼进 SQL，绝不直接拼原文。

# 3. FromRow 映射

## 3.1 derive

```rust
#[derive(Debug, sqlx::FromRow)]
struct User {
    id: i64,
    name: String,
    email: String,
}
```

默认按**列名与字段名完全一致**映射。字段顺序无关，但类型必须匹配：

| SQL 类型 | Rust 类型 |
|----------|-----------|
| `BIGINT` | `i64` |
| `INT` | `i32` |
| `TEXT` / `VARCHAR` | `String` |
| `BOOLEAN` | `bool` |
| `TIMESTAMPTZ` | `chrono::DateTime<Utc>` |
| `UUID` | `uuid::Uuid` |
| `JSONB` | `serde_json::Value` |
| `NUMERIC` | `rust_decimal::Decimal`（需 feature） |

★ **关键认知**：`FromRow` 只做"按名字取列"，**不做类型转换**。数据库是 `INT` 而字段写 `i64`，运行时可能报 `ColumnDecode`；数据库是 `TEXT` 而字段写 `Option<String>`，`NULL` 才会得到 `None`，非空字符串不会自动变 `None`。⚠ 把 `NOT NULL` 列映射成 `Option<T>` 编译能过，但你会永远拿到 `Some`，属于掩盖约束。

## 3.2 rename 与 default

```rust
#[derive(Debug, sqlx::FromRow)]
struct UserRow {
    id: i64,

    // 数据库列叫 user_name，Rust 里叫 name
    #[sqlx(rename = "user_name")]
    name: String,

    // 列可能为 NULL 时用 Option
    #[sqlx(default)]
    nickname: Option<String>,
}
```

## 3.3 flatten 与嵌套

```rust
#[derive(sqlx::FromRow)]
struct Address {
    city: String,
    street: String,
}

#[derive(sqlx::FromRow)]
struct Customer {
    id: i64,
    #[sqlx(flatten)]
    address: Address, // 把 city、street 两列摊进嵌套结构
}
```

⚠ `flatten` 只适合"平铺列"，不能用来做关联查询。一对多关系请分别查询再在内存里组装，或使用 `json_agg` 一次查回。

# 4. 动态 SQL

## 4.1 什么时候需要动态 SQL

条件数量不固定时，静态 SQL 很难写。典型场景是"三个筛选条件都可能为空"，需要按实际传入的参数拼出不同的 WHERE。

## 4.2 QueryBuilder

沿用前面的 `User` 结构体，条件按传入参数动态拼接：
```rust
use sqlx::{PgPool, QueryBuilder};

async fn search(
    pool: &PgPool,
    name: Option<&str>,
    email: Option<&str>,
    limit: i64,
) -> Result<Vec<User>, sqlx::Error> {
    let mut builder: QueryBuilder<sqlx::Postgres> =
        QueryBuilder::new("SELECT id, name, email FROM users WHERE 1 = 1");

    // push 的是固定 SQL 片段，bind 的是值
    if let Some(name) = name {
        builder.push(" AND name = ").push_bind(name);
    }
    if let Some(email) = email {
        builder.push(" AND email = ").push_bind(email);
    }

    builder.push(" ORDER BY id LIMIT ").push_bind(limit);

    builder
        .build_query_as::<User>()
        .fetch_all(pool)
        .await
}

// 用法：只传部分条件
let users = search(&pool, Some("张三"), None, 20).await?;
```

生成的 SQL 是 `SELECT ... WHERE 1 = 1 AND name = $1 ORDER BY id LIMIT $2`，参数依然走绑定。

★ **关键认知**：`push` 拼的是**代码里写死的 SQL 片段**，`push_bind` 绑的是**运行时值**。两者分工不能混：把用户输入交给 `push`，等于自己拆掉了注入防护。⚠ `push_bind` 的占位符编号由 QueryBuilder 自动递增，不要自己手写 `$1` 再 `push_bind`，会错位。

## 4.3 分页

```rust
// 偏移分页：简单，但大偏移会越翻越慢
let users = sqlx::query_as::<_, User>(
    "SELECT id, name, email FROM users ORDER BY id LIMIT $1 OFFSET $2",
)
.bind(limit)
.bind(offset)
.fetch_all(pool)
.await?;

// 游标分页：按上一页最后一个 id 往后取，性能稳定
let users = sqlx::query_as::<_, User>(
    "SELECT id, name, email FROM users WHERE id > $1 ORDER BY id LIMIT $2",
)
.bind(last_id)
.bind(limit)
.fetch_all(pool)
.await?;
```

| 方式 | 优点 | 缺点 | 适合 |
|------|------|------|------|
| `LIMIT/OFFSET` | 能跳页、实现简单 | 深分页扫描成本高 | 后台表格、小数据量 |
| 游标（`WHERE id > ?`） | 每页成本稳定 | 不能随机跳页 | 信息流、无限滚动 |

# 5. 执行结果与 rows_affected

```rust
let result = sqlx::query("UPDATE users SET name = $1 WHERE id = $2")
    .bind("新名字")
    .bind(id)
    .execute(pool)
    .await?;

if result.rows_affected() == 0 {
    println!("目标不存在或无需更新");
}
```

⚠ `rows_affected` 的语义因数据库而异：MySQL 默认返回"真正改变的行数"，值没变就算 0；PostgreSQL 返回"匹配的行数"。依赖它做存在性判断时，要清楚自己用的是哪种库。

# 6. 常见错误

| 现象 | 原因 | 解决 |
|------|------|------|
| `RowNotFound` 变成 500 | 该用 `fetch_optional` 却用了 `fetch_one` | 按语义选择 |
| `column "xxx" does not exist` | `FromRow` 字段名与列名不一致 | `#[sqlx(rename)]` 或写别名 |
| `ColumnDecode` 类型不匹配 | Rust 类型与 SQL 类型不符 | 核对类型映射表 |
| 编译期报 `no column found` | `query_as!` 的 SELECT 列与结构体不符 | 补列或改类型 |
| 查询结果为空但不是 NULL | `Option` 不能表达"没查到行" | 用 `fetch_optional` |
| 动态 SQL 报占位符数量不符 | 自己写了 `$1` 又用 `push_bind` | 交给 QueryBuilder 编号 |

# 7. 动手练习

1. 给 `users` 表写一个 `search(name, email, min_id)`，三个参数都可选，用 `QueryBuilder` 拼出动态条件，并打印最终 SQL 与绑定值确认没有拼接用户输入。
2. 用 `query_scalar` 实现 `exists_by_email(email) -> Result<bool, sqlx::Error>`，要求只查一列且不取回整行。
3. 把 `LIMIT/OFFSET` 分页改成游标分页，用 10 万条数据对比两种方式在深页时的耗时。

# 8. 小结

| 主题 | 要点 |
|------|------|
| `query` | 返回原始行，自己 `get` |
| `query_as` | 映射 `FromRow`，业务首选 |
| `query_scalar` | 单列单值，用于计数与存在性判断 |
| 取行方式 | `fetch_all` / `fetch_one` / `fetch_optional` 语义不同 |
| 绑定 | PostgreSQL `$1`，MySQL/SQLite `?` |
| 注入 | 值只用 `bind`，标识符走白名单 |
| FromRow | 按列名匹配，`rename`/`default`/`flatten` 辅助 |
| 动态 SQL | `QueryBuilder`：`push` 拼片段，`push_bind` 绑值 |

下一篇进入迁移，讲 `sqlx-cli` 的安装与命令、迁移文件的命名规则、`_sqlx_migrations` 表如何记录版本、回滚的边界，以及生产环境"展开-收缩"式的安全迁移流程。
