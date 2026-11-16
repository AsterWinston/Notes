表结构一定会变。加字段、改类型、建索引、拆表，如果靠手工执行 SQL，开发、测试、生产三套环境很快就会不一致。Alembic 把每一次结构变更记录成有版本号、可回滚、可审查的 Python 脚本。

# 1. 为什么需要迁移

## 1.1 create_all 的天花板

```python
Base.metadata.create_all(engine)
```

它只创建"当前 metadata 里不存在的新表"，**不会加列、改类型、删表，也不记录版本**。开发阶段删库重建很轻松，生产环境不能删库，所以只要数据有保留价值，就必须用迁移。

## 1.2 迁移系统在管什么

```text
模型代码（Base.metadata）
        │
        ▼
  autogenerate 对比
        │
        ▼
迁移脚本（versions/xxxx_add_status.py）
        │
        ├─ upgrade()   → 前进一个版本
        └─ downgrade() → 回退一个版本
        │
        ▼
数据库中的 alembic_version 表（记录当前版本号）
```

★ **关键认知**：迁移脚本是**代码资产**，要和业务代码一起提交、审查、发布。数据库里只有一行版本号，真正的变更历史在 `versions/` 目录。**不要手动改 `alembic_version` 表**，那会让版本链和实际结构脱节。

## 1.3 安装与初始化

```bash
pip install alembic
alembic init migrations
```

生成 `alembic.ini` 与 `migrations/` 目录，脚本放在 `migrations/versions/`，`env.py` 负责连接数据库并加载模型元数据。

# 2. 配置 env.py

## 2.1 数据库 URL

`alembic.ini` 可以直接写 `sqlalchemy.url`，但不要把生产密码提交进仓库。更常见的做法是从环境变量读取：

```python
import os
from logging.config import fileConfig

from alembic import context
from sqlalchemy import engine_from_config, pool

from app.models import Base          # 必须导入模型，否则 metadata 是空的

config = context.config
if config.config_file_name is not None:
    fileConfig(config.config_file_name)

config.set_main_option("sqlalchemy.url", os.environ["DATABASE_URL"])
target_metadata = Base.metadata
```

## 2.2 关键配置

```python
with engine.connect() as connection:
    context.configure(
        connection=connection,
        target_metadata=target_metadata,
        compare_type=True,       # 检测列类型变化
        compare_server_default=True,
        render_as_batch=True,    # SQLite 改列需要 batch 模式
    )
    with context.begin_transaction():
        context.run_migrations()
```

| 配置 | 作用 | 建议 |
|------|------|------|
| `target_metadata` | 对比的模型元数据 | 必填，且要导入所有模型 |
| `compare_type` | 检测类型变化 | 开启 |
| `compare_server_default` | 检测默认值变化 | 开启，注意噪声 |
| `render_as_batch` | 生成 batch 操作 | SQLite 必开 |

⚠ **模型没有被导入，`Base.metadata` 就是空的**，autogenerate 会生成一份"删除所有表"的迁移。确保 `env.py` 导入了包含全部模型的包。

## 2.3 首次基线

新库用 `alembic revision --autogenerate -m "init schema"` 再 `alembic upgrade head`。如果数据库表已经存在（历史项目接入），不要执行 upgrade，而是 `alembic stamp head`：它只写版本号、不执行 SQL，用于"结构已经和脚本一致，只差登记"的场景。

# 3. 生成迁移

## 3.1 autogenerate

```bash
alembic revision --autogenerate -m "add status to orders"
```

```python
"""add status to orders"""

from alembic import op
import sqlalchemy as sa

revision = "8f2c1a9d4b77"
down_revision = "3a1e5c7b9d20"


def upgrade() -> None:
    op.add_column(
        "orders",
        sa.Column("status", sa.String(20), nullable=False, server_default="pending"),
    )
    op.create_index("ix_orders_status", "orders", ["status"])


def downgrade() -> None:
    op.drop_index("ix_orders_status", table_name="orders")
    op.drop_column("orders", "status")
```

★ **关键认知**：**autogenerate 是草稿，不是成品**。它无法知道数据语义、无法处理重命名、也无法判断某个 DDL 在生产大表上是否会锁表。**每一份自动生成的迁移都必须人工读一遍**。

## 3.2 能力边界

| 变更 | autogenerate | 说明 |
|------|-------------|------|
| 新建表、加列、删列 | 能 | 删列要确认数据 |
| 改列类型、可空性 | 能（开 `compare_type`） | 有数据时可能失败 |
| 新建索引、唯一约束 | 能 | 命名约定要提前配好 |
| 表重命名、列重命名 | **不能** | 会生成"删旧建新" |
| 数据搬运 | **不能** | 必须手写 `op.execute` |
| 触发器、视图、存储过程 | **不能** | 手写 SQL |

表重命名必须手写 `op.rename_table("old_orders", "orders")`，否则会丢数据。

## 3.3 常用 op 操作

常用 `op.add_column`、`op.alter_column`、`op.execute`。SQLite 不支持直接 `ALTER COLUMN`，要用 `op.batch_alter_table` 包一层，否则迁移会直接失败。

## 3.4 迁移脚本不要依赖 ORM 模型

```python
# 推荐：用轻量表定义，不 import 会演进的 ORM 模型
orders = sa.table("orders", sa.column("id", sa.Integer), sa.column("status", sa.String))
```

迁移是**历史快照**，必须能在两年后原样执行。引用当前模型会让旧脚本随模型演进而失效。

# 4. 升级与回滚

## 4.1 常用命令

```bash
alembic current                 # 查看数据库当前版本
alembic history                 # 查看版本链
alembic upgrade head            # 升级到最新
alembic upgrade +1              # 只前进一个版本
alembic downgrade -1            # 回退一个版本
alembic downgrade base          # 回退全部
```

## 4.2 版本链

```text
base ──▶ 3a1e5c7b9d20 ──▶ 8f2c1a9d4b77 ──▶ c4d9e2f1a630 ──▶ head
         init schema        add status        add coupons
```

每个脚本的 `down_revision` 指向上一版本，形成单向链表。**不要手动修改已经发布的版本号**，否则其他环境的版本链会对不上。

## 4.3 回滚脚本要真的能回滚

`downgrade()` 要按相反顺序撤销 `upgrade()` 的操作，比如先 `drop_index` 再 `drop_column`。如果 `upgrade()` 里做了数据转换，`downgrade()` 必须考虑能否还原。**删除列的回滚只能恢复结构，不能恢复数据**。对不可逆操作，要么提前备份，要么在脚本里明确注释"不可回滚"。

# 5. 生产流程

## 5.1 分环境执行

| 环境 | 执行方式 | 注意事项 |
|------|---------|---------|
| 开发 | 改模型后 `autogenerate` + `upgrade head` | 检查生成内容 |
| 测试 | CI 中对空库执行全部迁移 | 验证迁移链完整 |
| 预发 | 与生产同构，演练升级和回滚 | 记录耗时与锁等待 |
| 生产 | 发布流程中单独执行 | 备份、限流、可回滚 |

★ **关键认知**：**应用启动时自动执行迁移是危险习惯**。多实例同时启动会并发跑迁移，轻则报错，重则半完成。迁移应该是独立的一步，先于应用发布执行。

## 5.2 查看将要执行的 SQL

```bash
alembic upgrade head --sql > migration.sql
```

先生成 SQL 供 DBA 审查，确认没有意外的 `DROP`、没有对大表的全表重写，再执行。

## 5.3 零停机变更

大表加索引用 `CREATE INDEX CONCURRENTLY`（PostgreSQL）：

```python
def upgrade() -> None:
    op.execute("CREATE INDEX CONCURRENTLY IF NOT EXISTS ix_orders_status ON orders (status)")
```

它**不能在事务块内执行**，需要配置 `transaction_per_migration=False` 或使用 autocommit 块。

更通用的**扩展-收缩**流程：

```text
1. 加新列（可空），应用双写
2. 回填历史数据（分批）
3. 应用改读新列
4. 加非空约束 / 删除旧列
```

每一步都可独立发布、独立回滚，避免一次 DDL 同时改结构和数据。

## 5.4 回滚策略

刚执行且数据未变时可以 `alembic downgrade -1`；已经写入新数据时先评估能否丢弃，不要盲目回滚；不可逆迁移应写新迁移前滚修复，而不是硬退。生产故障优先恢复服务，再处理版本一致性。

# 6. 数据迁移

## 6.1 结构与数据分开

一次迁移只做一件事。把"加列"和"回填数据"拆成两个脚本，回滚时更容易定位。

第一个脚本只加列：

```python
def upgrade() -> None:
    op.add_column("users", sa.Column("full_name", sa.String(100)))
```

第二个脚本只回填数据：

```python
def upgrade() -> None:
    op.execute("UPDATE users SET full_name = first_name || ' ' || last_name "
               "WHERE full_name IS NULL")
```

## 6.2 大批量数据分批处理

一次性 `UPDATE` 百万行会长时间持锁、撑大事务日志。用主键区间循环：

```python
def upgrade() -> None:
    conn = op.get_bind()
    while True:
        result = conn.execute(
            sa.text(
                "UPDATE users SET full_name = first_name || ' ' || last_name "
                "WHERE id IN ("
                "  SELECT id FROM users WHERE full_name IS NULL "
                "  ORDER BY id LIMIT :limit"
                ")"
            ),
            {"limit": 5000},
        )
        if result.rowcount == 0:
            break
```

每批之间可以提交，避免单个超长事务。生产环境还要考虑主从延迟和业务低峰期。

# 7. 常见错误

| 现象 | 原因 | 解决 |
|------|------|------|
| autogenerate 生成删表脚本 | 模型未导入，metadata 为空 | 在 `env.py` 导入全部模型 |
| 生成的迁移重复建表 | 数据库已有表但没 stamp | 先 `alembic stamp head` 建基线 |
| 多实例启动报版本冲突 | 应用启动时自动迁移 | 迁移独立执行 |
| 大表加索引卡住业务 | 普通 `CREATE INDEX` 持锁 | 用 `CONCURRENTLY` 或低峰执行 |
| 回滚报约束不存在 | 约束名未统一 | 配置命名约定，检查 downgrade |
| 迁移在 SQLite 上失败 | 不支持 `ALTER COLUMN` | 使用 `batch_alter_table` |
| 旧迁移突然报错 | 迁移里 import 了会变的模型 | 用 `sa.table` 轻量表定义 |

# 8. 动手练习

1. 从空库初始化 Alembic，创建 `users` 表并升级到 head，再回退到 base，用 `alembic history` 观察版本链。
2. 给 `users` 加一个非空 `status` 列，分别写出"直接加非空列"和"先加可空列再回填再加约束"两版脚本，比较风险。
3. 写一个分批回填迁移，把 10 万行数据的 `full_name` 由 `first_name + last_name` 计算出来，要求每批 5000 行且可重复执行。

# 9. 小结

| 主题 | 要点 |
|------|------|
| 定位 | 迁移是版本化的结构变更脚本，不是 `create_all` 的替代品 |
| 初始化 | `alembic init`、配置 `target_metadata`、从环境变量读 URL |
| autogenerate | 只是草稿，重命名和数据迁移必须手写 |
| 回滚 | `downgrade()` 必须认真写，不可逆操作要备份 |
| 生产 | 迁移独立执行，先出 SQL 审查，大表用在线 DDL |
| 数据迁移 | 与结构变更分开，大批量分批提交 |
| 稳定性 | 迁移脚本不要依赖当前 ORM 模型 |

下一篇进入查询与性能，讲清楚 `select` 的过滤、连接、聚合怎么写，N+1 如何定位，索引与 `EXPLAIN` 怎么读，以及连接池参数在生产环境如何估算。
