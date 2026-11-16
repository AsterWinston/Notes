很多 Python 项目一开始用原生 `sqlite3` 或 `psycopg` 拼 SQL，等表多起来、字段一改、事务一嵌套，就会陷入"字符串拼接 + 手工映射"的泥潭。SQLAlchemy 解决的正是这一层问题：它把 SQL 的构造、执行、连接管理和对象映射拆成可组合的部件，让简单的查询保持简单，复杂的需求仍然能落到 SQL 上。

# 1. SQLAlchemy 的定位

## 1.1 它到底解决什么问题

SQLAlchemy 是 Python 生态里最成熟的关系数据库工具库，官方把它定位为"数据库工具包 + 对象关系映射器"。它同时提供两套能力：

| 能力 | 说明 | 典型使用者 |
|------|------|-----------|
| Core | 用 Python 表达式构造 SQL，管理引擎与连接 | 需要精细控制 SQL 的服务 |
| ORM | 把表映射成类，把行映射成对象，自动跟踪变更 | 业务代码、领域模型 |
| 连接池 | 复用数据库连接，处理断线重连 | 所有长期运行的服务 |
| 方言层 | 同一套代码适配 PostgreSQL、MySQL、SQLite | 多数据库部署 |

注意一个常见误解：**ORM 不是"不用写 SQL"，而是"把重复的 SQL 交给框架生成"**。复杂报表、批量更新、窗口函数，最终仍然要理解 SQL 本身。

## 1.2 Core 与 ORM 的分层

理解分层是学 SQLAlchemy 的第一道门槛。ORM 不是独立运行的，它建立在 Core 之上：

```text
┌──────────────────────────────────────────────┐
│  应用代码（业务逻辑、领域对象）               │
└───────────────────┬──────────────────────────┘
                    ▼
┌──────────────────────────────────────────────┐
│  ORM 层：Session、映射类、关系、身份映射      │
└───────────────────┬──────────────────────────┘
                    ▼
┌──────────────────────────────────────────────┐
│  Core 层：select()、Table、Column、类型系统   │
└───────────────────┬──────────────────────────┘
                    ▼
┌──────────────────────────────────────────────┐
│  Engine：连接池 + 方言 + 事务                 │
└───────────────────┬──────────────────────────┘
                    ▼
┌──────────────────────────────────────────────┐
│  数据库驱动：psycopg、pymysql、sqlite3        │
└──────────────────────────────────────────────┘
```

★ **关键认知**：`Session` 不会自己连接数据库，它向 `Engine` 借连接；`Engine` 也不直接执行 ORM 语句，它把 Core 编译出的 SQL 交给驱动。**整条链路只有一个连接池**，位置在 `Engine` 内部。理解这一点，后面的会话生命周期、事务边界、连接泄漏问题都会顺理成章。

# 2. 安装与环境

## 2.1 版本约定

本篇以 SQLAlchemy 2.x、Alembic 1.x、Python 3.11 为准。SQLAlchemy 2.x 与 1.4 的重大差别在于：

| 项目 | 1.x 风格 | 2.x 风格 |
|------|---------|---------|
| 查询 | `session.query(User)` | `session.execute(select(User))` |
| 模型字段 | `Column(Integer)` | `mapped_column()` + `Mapped[int]` |
| 基类 | `declarative_base()` | `DeclarativeBase` 子类 |
| 异步 | 需第三方 | 内置 `AsyncSession` |

新项目应直接使用 2.x 风格。网上大量旧教程仍是 1.x 写法，混用会得到"能跑但类型提示失效"的代码。

## 2.2 安装命令

```bash
# 创建虚拟环境（Windows PowerShell）
python -m venv .venv
.\.venv\Scripts\Activate.ps1

# 安装 SQLAlchemy 与 PostgreSQL 驱动
pip install "sqlalchemy>=2.0" "psycopg[binary]>=3.1" alembic

# 用 SQLite 练习时不需要额外驱动
pip install "sqlalchemy>=2.0"
```

驱动选择见下表：

| 数据库 | 推荐驱动 | URL 前缀 |
|--------|---------|----------|
| PostgreSQL | `psycopg`（v3） | `postgresql+psycopg://` |
| MySQL | `pymysql` | `mysql+pymysql://` |
| SQLite | 内置 `sqlite3` | `sqlite:///` |
| SQL Server | `pyodbc` | `mssql+pyodbc://` |

## 2.3 验证安装

```python
import sqlalchemy
from sqlalchemy import create_engine, text

print(sqlalchemy.__version__)          # 应输出 2.x

engine = create_engine("sqlite:///:memory:", echo=False)
with engine.connect() as conn:         # 从连接池借出一条连接
    value = conn.execute(text("SELECT 1")).scalar_one()
    print(value)                       # 1
```

`with` 结束时连接会归还连接池，而不是关闭物理连接。这一点在排查"连接数打满"时非常关键。

# 3. Engine 与连接

## 3.1 create_engine 做了什么

`create_engine` 是惰性的：调用时只解析 URL、创建方言和连接池，**不会立刻建立网络连接**。真正的连接发生在第一次 `connect()` 或执行语句时。

```python
from sqlalchemy import create_engine

engine = create_engine(
    "postgresql+psycopg://app:secret@localhost:5432/shop",
    pool_size=10,              # 常驻连接数
    max_overflow=20,           # 高峰期最多再借 20 条
    pool_timeout=30,           # 借不到连接时最多等 30 秒
    pool_pre_ping=True,        # 借出前先 ping，避免用到被数据库断开的死连接
    echo=False,                # 生产环境不要开，SQL 日志量很大
)
```

## 3.2 URL 的组成

```text
postgresql+psycopg://app:secret@localhost:5432/shop?sslmode=require
└────┬───┘ └──┬──┘   └─┬─┘ └──┬──┘ └──┬──┘ └─┬─┘ └──────┬──────┘
   方言       驱动     用户    密码    主机    端口   数据库   查询参数
```

★ **关键认知**：URL 里的密码如果含有 `@`、`:`、`/` 等字符，必须做 URL 编码，否则解析会错位。更稳妥的做法是用 `URL.create()` 由程序拼接：

```python
from sqlalchemy import URL

url = URL.create(
    "postgresql+psycopg",
    username="app",
    password="p@ss:word/2024",     # 原样传入，不需要手工编码
    host="localhost",
    port=5432,
    database="shop",
)
engine = create_engine(url)
```

## 3.3 连接池的默认行为

不同数据库的默认池不同：

| 场景 | 默认池 | 说明 |
|------|--------|------|
| PostgreSQL / MySQL | `QueuePool` | 固定池 + 溢出，适合 Web 服务 |
| SQLite 文件 | `QueuePool`（2.x） | 单文件写入需要串行 |
| SQLite 内存 | `SingletonThreadPool` | 每个线程一个连接 |

⚠ **SQLite 的 `:memory:` 数据库是"每个连接一份"**。用默认池建表后再开新连接，很可能查不到表。练习时要么用文件数据库，要么用 `StaticPool` 让所有会话共用同一条内存连接。开发时打开 `echo=True` 可以看到真实 SQL，生产环境应改用 `logging` 按级别控制。

# 4. 第一个 Core 查询

## 4.1 用 text() 执行原生 SQL

```python
from sqlalchemy import text

with engine.connect() as conn:
    result = conn.execute(text("SELECT id, name FROM users WHERE age > :age"), {"age": 18})
    for row in result:
        print(row.id, row.name)        # 可以按列名访问
```

## 4.2 参数绑定不是字符串拼接

```python
# 错误示范：SQL 注入
user_input = "1 OR 1=1"
conn.execute(text(f"SELECT * FROM users WHERE id = {user_input}"))

# 正确做法：命名参数
conn.execute(text("SELECT * FROM users WHERE id = :uid"), {"uid": user_input})
```

★ **关键认知**：SQLAlchemy 的参数绑定会把值作为**驱动层参数**传递，而不是拼进 SQL 文本。这不仅防注入，还能让数据库复用执行计划。**任何把用户输入放进 f-string 再交给 `text()` 的写法都是错的**。

## 4.3 Result 的几种读取方式

```python
result = conn.execute(text("SELECT id, name FROM users"))

row = result.first()            # 第一行，没有则 None
rows = result.fetchall()        # 全部行，列表
one = result.scalar_one()       # 恰好一行一列，否则抛异常
maybe = result.scalar_one_or_none()  # 零或一行一列
mapping = result.mappings().first() # 返回字典式 RowMapping
```

`one()` 在零行或多行时都会抛异常，`one_or_none()` 允许零行，`scalar_one()` 用于聚合计数，`mappings()` 返回字典式结果。

# 5. 第一个 ORM 示例

## 5.1 定义模型

```python
from datetime import datetime
from sqlalchemy import String, DateTime, func
from sqlalchemy.orm import DeclarativeBase, Mapped, mapped_column


class Base(DeclarativeBase):
    pass


class User(Base):
    __tablename__ = "users"

    id: Mapped[int] = mapped_column(primary_key=True)
    name: Mapped[str] = mapped_column(String(50), index=True)
    email: Mapped[str] = mapped_column(String(120), unique=True)
    created_at: Mapped[datetime] = mapped_column(
        DateTime(timezone=True), server_default=func.now()
    )
```

## 5.2 建表

```python
Base.metadata.create_all(engine)
```

`create_all` 只创建"不存在的表"，不会修改已有表结构。**生产环境的表结构变更必须交给 Alembic**，这一点在迁移篇会展开。

## 5.3 插入与查询

```python
from sqlalchemy.orm import Session
from sqlalchemy import select

with Session(engine) as session:
    session.add(User(name="alice", email="alice@example.com"))
    session.commit()                       # 提交事务，对象进入持久态

with Session(engine) as session:
    stmt = select(User).where(User.name == "alice")
    user = session.execute(stmt).scalar_one()
    print(user.id, user.email)
```

ORM 的完整数据流可以画成：

```text
定义类 User
   │
   ▼
Base.metadata 收集表结构
   │
   ▼
create_all() / Alembic ──▶ 数据库中的 users 表
   │
   ▼
Session.add(obj) ──▶ 身份映射（identity map）登记
   │
   ▼
flush ──▶ 生成 INSERT ──▶ 数据库
   │
   ▼
commit ──▶ 事务提交，对象变为 persistent
   │
   ▼
select(User) ──▶ 查回的行自动组装成 User 实例
```

★ **关键认知**：ORM 的"对象"和数据库的"行"之间隔着一个 **Session**。同一个 Session 内，主键相同的行只会对应一个 Python 对象，这叫**身份映射**。它避免了同一请求里出现两个互相覆盖的副本，也解释了为什么"改了对象不 commit 就不会落库"。

# 6. 常见错误

| 现象 | 原因 | 解决 |
|------|------|------|
| `ModuleNotFoundError: psycopg2` | URL 用了 `postgresql://` 但只装了 psycopg3 | 改写为 `postgresql+psycopg://` |
| 查询结果为空但表里有数据 | 连到了另一个数据库或 `:memory:` 的另一条连接 | 打印 `engine.url`，确认数据库文件 |
| `InvalidRequestError: Object is already attached to session` | 对象来自另一个 Session | 用 `session.merge()` 或重新查询 |
| 中文写入报编码错误 | 数据库/连接未设 UTF-8 | 建库时指定编码，URL 加 `charset=utf8mb4` |
| 服务运行一段时间后连接耗尽 | 手写了 `engine.connect()` 却忘记关闭 | 一律用 `with` 上下文管理器 |
| `create_all` 改了模型却没生效 | 它不会修改已有表 | 用 Alembic 生成迁移 |

# 7. 动手练习

1. 用 SQLite 文件数据库建一个 `notes` 表，包含 `id`、`title`、`body`、`created_at`，用 Core 的 `text()` 插入两条记录并按时间倒序查询。
2. 把上面的表改写成 ORM 模型，使用 `Mapped` 与 `mapped_column`，然后用 `Session` 完成同样的插入和查询。
3. 故意把 PostgreSQL URL 写成 `postgresql://`（不指定驱动），观察报错信息，再改成 `postgresql+psycopg://`，解释两者差异。

# 8. 小结

| 主题 | 要点 |
|------|------|
| 定位 | Core 管 SQL 与连接，ORM 管对象映射，二者共用 Engine |
| Engine | 惰性创建，内部持有连接池，是全局单例 |
| URL | `方言+驱动://用户:密码@主机:端口/库`，特殊字符用 `URL.create()` |
| Core 查询 | `text()` + 命名参数，永远不要拼接用户输入 |
| Result | 按预期行数选择 `one` / `one_or_none` / `scalar_one` |
| ORM 流程 | 定义模型 → 建表 → Session 增删改查 → commit |
| 建表 | 开发可用 `create_all`，生产必须用迁移工具 |

下一篇进入模型与映射，讲清楚 `DeclarativeBase` 如何收集元数据、`Mapped` 注解如何决定可空性、字段类型与主键索引怎么选，以及 `__tablename__`、命名约定这些"一开始定错、后面到处改"的细节。
