表结构不是一次设计好就永不改动的。加字段、建索引、拆表、补数据，每一次变更都要可追溯、可重复执行、能在生产环境安全落地。SQLx 的迁移系统把这些需求收敛成"按序号排序的 SQL 文件 + 一张记录表"，简单，但用错会留下难以收拾的残留状态。

# 1. 为什么需要迁移

## 1.1 手工改表的问题

```text
开发：直接连数据库 ALTER TABLE
  ├─ 本地库改了，测试库忘了
  ├─ 同事拉代码后不知道要改什么
  ├─ 生产上线靠人肉执行 SQL
  └─ 出问题不知道改过哪些、怎么退
```

迁移要解决的是三件事：**版本可追溯**、**执行可重复**、**环境可对齐**。

## 1.2 SQLx 迁移的模型

```text
migrations/
├── 20240101090000_create_users.sql
├── 20240102103000_add_email_index.sql
└── 20240103080000_create_orders.sql

数据库：
┌──────────────────────────────────────────────┐
│ _sqlx_migrations                             │
│ version      │ description  │ checksum │ ... │
│ 20240101090000│ create users │ abc123    │ ... │
│ 20240102103000│ add email... │ def456    │ ... │
└──────────────────────────────────────────────┘
```

★ **关键认知**：SQLx 在数据库里维护一张 `_sqlx_migrations` 表，记录**已执行版本**和**文件校验和**。每次启动或手动执行迁移时，它会对比目录里的文件与表里的记录：没执行过的按版本号顺序执行，执行过的校验 checksum。⚠ 一旦某个已执行文件的**内容**被改动，checksum 不匹配，迁移会直接失败——这是保护，不是 bug。

# 2. 安装 sqlx-cli

## 2.1 安装

```bash
# 只装需要的数据库驱动，编译更快
cargo install sqlx-cli --no-default-features --features postgres,rustls

# 确认版本
cargo sqlx --version
```

## 2.2 常用 feature

| feature | 作用 |
|---------|------|
| `postgres` / `mysql` / `sqlite` | 支持对应数据库 |
| `rustls` | 纯 Rust TLS，不依赖系统 OpenSSL |
| `native-tls` | 用系统 TLS |
| `completions` | 生成 shell 补全 |

## 2.3 配置连接串

```bash
# Linux / macOS
export DATABASE_URL="postgres://postgres:postgres@localhost:5432/demo"

# Windows PowerShell
$env:DATABASE_URL = "postgres://postgres:postgres@localhost:5432/demo"
```

⚠ Windows 用户常踩的坑：`sqlx-cli` 读的是**当前进程**的环境变量，`setx` 设置后要重开终端；临时设置用 `$env:`，只对当前会话生效。

# 3. 迁移文件

## 3.1 创建

```bash
# 单向迁移：只生成一个文件（更常见）
cargo sqlx migrate add create_users

# 可逆迁移：同时生成 up 和 down 两个文件
cargo sqlx migrate add -r create_users
```

生成的文件名带时间戳前缀：

```text
migrations/20240101090000_create_users.sql
```

## 3.2 命名规则

| 部分 | 说明 |
|------|------|
| `20240101090000` | UTC 时间戳，决定执行顺序 |
| `create_users` | 描述，用 snake_case |
| `.sql` | 固定后缀 |
| `.up.sql` / `.down.sql` | 可逆迁移的两个方向 |

★ **关键认知**：版本号是**时间戳**，不是自增序号。多人协作时各自生成的文件不会撞号，合并分支后按时间先后执行。⚠ 不要手动改已提交迁移的时间戳，也不要删除中间版本，否则不同环境会走出不同的顺序。

## 3.3 迁移文件内容

```sql
-- migrations/20240101090000_create_users.sql
CREATE TABLE users (
    id         BIGSERIAL PRIMARY KEY,
    name       TEXT NOT NULL,
    email      TEXT NOT NULL UNIQUE,
    created_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_users_created_at ON users (created_at);
```

可逆版本：

```sql
-- migrations/20240101090100_add_age.up.sql
ALTER TABLE users ADD COLUMN age INT;

-- migrations/20240101090100_add_age.down.sql
ALTER TABLE users DROP COLUMN age;
```

⚠ PostgreSQL 的 DDL 支持事务，整份迁移失败会整体回滚；MySQL 的 DDL 会隐式提交，失败可能留下"做了一半"的结构。写 MySQL 迁移时尽量把 DDL 拆小，并准备好修复脚本。一个文件可以包含多条语句，但要把它们看成一个整体来设计失败后的恢复路径。

# 4. 常用命令

| 命令 | 作用 |
|------|------|
| `cargo sqlx migrate add <name>` | 新建迁移文件 |
| `cargo sqlx migrate run` | 执行所有未应用的迁移 |
| `cargo sqlx migrate revert` | 回滚最近一个迁移 |
| `cargo sqlx migrate info` | 列出各版本状态 |
| `cargo sqlx migrate add -r <name>` | 新建可逆迁移 |
| `cargo sqlx database create` | 创建数据库 |
| `cargo sqlx database drop` | 删除数据库 |

## 4.1 执行与查看

`migrate run` 会按顺序执行并打印 `Applied <版本>/<描述>`；`migrate info` 则列出每个版本是 `applied` 还是 `pending`，用来确认环境是否对齐。

## 4.2 回滚

```bash
# 只回滚最近一个，依赖 .down.sql
cargo sqlx migrate revert
```

★ **关键认知**：`revert` 只能回滚**最近一个**迁移，而且必须有对应的 `.down.sql`。⚠ 回滚不是"撤销历史"，它是在当前状态上再执行一段 `down` SQL。如果后面已经叠加了依赖该结构的迁移，回滚会失败；生产环境更常见的做法是**写一个前向修复迁移**，而不是 revert。

# 5. 版本管理原理

## 5.1 checksum 校验

```text
迁移文件内容 ──计算──► checksum ──写入──► _sqlx_migrations
                                      │
下次执行时重新计算 ────────────────────┘
                 │
        不一致 ──► 报错：migration was previously applied but has been modified
```

这意味着：**已执行过的迁移文件不可修改**。发现写错了，正确做法是新增一个迁移去修复：

```sql
-- migrations/20240104090000_fix_users_email_length.sql
ALTER TABLE users ALTER COLUMN email TYPE VARCHAR(320);
```

## 5.2 多人协作与冲突

```text
同事 A：20240101120000_add_status.sql
同事 B：20240101120000_add_avatar.sql   ← 同一秒生成，版本号相同
```

SQLx 对同版本号会报冲突。解决办法是重新生成 B 的文件（改时间戳），或在合并时手动调整到不同版本号。⚠ 只调文件名不够，还要确认内容顺序符合依赖关系。

# 6. 在代码里跑迁移

## 6.1 migrate! 宏

```rust
use sqlx::PgPool;

let pool = PgPool::connect(&url).await?;

// 编译时把 migrations/ 目录嵌入二进制
sqlx::migrate!("./migrations")
    .run(&pool)
    .await?;
```

## 6.2 运行时加载

```rust
// 不嵌入，从磁盘读取，适合需要热更新迁移文件的场景
sqlx::migrate::Migrator::new(std::path::Path::new("./migrations"))
    .await?
    .run(&pool)
    .await?;
```

| 方式 | 优点 | 缺点 |
|------|------|------|
| `migrate!` 宏 | 迁移随二进制发布，不会漏文件 | 改迁移要重新编译 |
| `Migrator::new` | 可外部提供文件 | 部署要带目录，路径易错 |

★ **关键认知**：生产环境推荐 `migrate!` 宏，因为它把迁移文件**编译进二进制**，版本和代码严格绑定，不会出现"代码是新的、迁移文件是旧的"这种错配。⚠ 如果采用"部署时单独跑迁移"的流程，务必在应用启动前完成，不要让多个副本并发执行迁移。

# 7. 生产迁移流程

## 7.1 展开-收缩（expand-contract）

直接 `ALTER TABLE ... RENAME` 或 `DROP COLUMN` 会让新旧代码同时失效。安全做法分三步：

```text
目标：把 users.name 拆成 first_name + last_name

步骤 1（展开）：新增列，允许 NULL，双写
  ALTER TABLE users ADD COLUMN first_name TEXT;
  ALTER TABLE users ADD COLUMN last_name  TEXT;
  应用：写 name 时同时写 first_name/last_name

步骤 2（回填）：分批补齐历史数据
  UPDATE users SET first_name = split_part(name, ' ', 1)
  WHERE first_name IS NULL;

步骤 3（收缩）：代码只读新列，之后删除旧列
  ALTER TABLE users DROP COLUMN name;
```

★ **关键认知**：生产迁移的原则是**任何时刻，新旧两版代码都能同时跑在同一份表结构上**。所以先加后删、先松后紧，绝不在一步里做"重命名 + 改类型 + 加约束"。⚠ 加 `NOT NULL` 约束前必须确保历史数据已回填，否则迁移会直接失败并锁表。

## 7.2 锁与超时

```sql
-- PostgreSQL：设置锁等待上限，避免迁移无限阻塞
SET lock_timeout = '5s';

-- 建索引不锁写（PostgreSQL）
CREATE INDEX CONCURRENTLY idx_users_email ON users (email);
```

| 操作 | 风险 | 建议 |
|------|------|------|
| `ADD COLUMN` 带默认值 | 旧版本 PG 会重写全表 | PG 11+ 可安全加常量默认值 |
| `ALTER COLUMN TYPE` | 重写全表、长时间锁 | 用新列 + 回填 + 切换 |
| `CREATE INDEX` | 阻塞写入 | 用 `CONCURRENTLY` |
| `ADD CONSTRAINT` | 全表校验 | 先 `NOT VALID` 再加 `VALIDATE` |
| `DROP COLUMN` | 依赖它的旧代码报错 | 确认旧版本已全部下线 |

⚠ `CREATE INDEX CONCURRENTLY` 不能在事务块里执行。如果你的迁移工具默认包事务，需要单独处理，或把它拆成独立迁移并关闭事务包裹。

## 7.3 CI 中的迁移检查

```bash
# 在临时库上完整跑一遍迁移，确认可重复执行
cargo sqlx database create
cargo sqlx migrate run
cargo sqlx migrate info

# 再用 query! 宏生成离线元数据，确保代码与 schema 对得上
cargo sqlx prepare --check
```

# 8. 常见错误

| 现象 | 原因 | 解决 |
|------|------|------|
| `migration was previously applied but has been modified` | 改了已执行的迁移文件 | 新增迁移修复，不要改历史 |
| `relation already exists` | 手工建过表，但没记录版本 | 用 `migrate run` 统一管理，或先 `migrate info` 核对 |
| `revert` 报找不到 down 文件 | 建的是单向迁移 | 用 `-r` 重新生成，或手写前向修复 |
| 多副本同时迁移冲突 | 应用启动时各副本都执行 | 部署流程里单独跑一次迁移 |
| 迁移卡住不返回 | 等表锁 | 设 `lock_timeout`，错峰执行 |
| MySQL 迁移失败留下半张表 | DDL 隐式提交 | 拆小 DDL，准备修复脚本 |

# 9. 动手练习

1. 用 `cargo sqlx migrate add -r` 创建可逆迁移，给 `users` 加 `age` 列，执行 `run` 后再 `revert`，用 `migrate info` 观察状态变化。
2. 故意修改一个已执行迁移文件的内容，再跑 `migrate run`，记录报错信息；然后用新增迁移的方式修复。
3. 模拟展开-收缩：给 `users` 加 `full_name` 列，写迁移回填 `name` 的数据，再写代码验证新旧字段一致，最后删除旧列。

验收标准：迁移在空库上能从零跑通；`migrate info` 全部为 `applied`；第 2 题能解释 checksum 的作用。

# 10. 小结

| 主题 | 要点 |
|------|------|
| 记录表 | `_sqlx_migrations` 存版本与 checksum |
| 命名 | 时间戳 + 描述，按时间排序执行 |
| 已执行文件 | 不可修改，只能新增修复迁移 |
| `migrate!` | 编译期嵌入，迁移与代码绑定 |
| 回滚 | 只回滚最近一个，生产优先前向修复 |
| 生产流程 | 展开 → 回填 → 收缩，新旧代码共存 |
| 锁 | 设 `lock_timeout`，大表用 `CONCURRENTLY` |

下一篇进入事务，讲 `begin/commit/rollback` 的正确写法、`&mut *tx` 为什么必须解引用、drop 触发的自动回滚靠不靠得住、嵌套 SavePoint 的语义，以及并发更新时的锁与隔离级别选择。
