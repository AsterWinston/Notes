会写 CRUD 只是起点。真正让项目可维护的，是边界清晰的分层、可控的会话生命周期、统一的异常转换，以及能在 CI 里跑起来的数据库测试。本篇把这些工程实践收拢成一份可落地的清单，作为整个模块的收尾。

# 1. 分层架构

## 1.1 三层职责

```text
┌─────────────────────────────────────────────┐
│  API 层（FastAPI / Flask / CLI）             │
│  参数校验、鉴权、HTTP 状态码、序列化          │
└───────────────────┬─────────────────────────┘
                    ▼
┌─────────────────────────────────────────────┐
│  Service 层（业务用例）                      │
│  事务边界、业务规则、跨仓储编排               │
└───────────────────┬─────────────────────────┘
                    ▼
┌─────────────────────────────────────────────┐
│  Repository 层（数据访问）                   │
│  select / add / delete，不 commit            │
└───────────────────┬─────────────────────────┘
                    ▼
┌─────────────────────────────────────────────┐
│  Model 层（SQLAlchemy 映射）                 │
└─────────────────────────────────────────────┘
```

★ **关键认知**：**事务边界放在 Service，而不是 Repository**。Repository 只负责"怎么查、怎么存"，Service 决定"这一组操作要么全成功要么全失败"。如果每个 Repository 方法自己 commit，跨仓储的业务就失去了原子性。

## 1.2 Repository 的写法

```python
from sqlalchemy import select
from sqlalchemy.orm import Session

class UserRepository:
    def __init__(self, session: Session) -> None:
        self.session = session

    def get(self, user_id: int) -> User | None:
        return self.session.get(User, user_id)

    def get_by_email(self, email: str) -> User | None:
        return self.session.scalar(select(User).where(User.email == email))

    def add(self, user: User) -> None:
        self.session.add(user)          # 只登记，不 commit
```

Repository 接收已经创建好的 Session，**自己不创建、不关闭、不提交**。这样它既能参与请求级事务，也能在测试里被替换。

## 1.3 Service 控制用例

```python
class UserService:
    def __init__(self, session: Session) -> None:
        self.session = session
        self.users = UserRepository(session)

    def register(self, email: str, name: str) -> User:
        if self.users.get_by_email(email) is not None:
            raise EmailAlreadyExists(email)

        user = User(email=email, name=name)
        self.users.add(user)
        self.session.commit()           # 用例成功，统一提交
        return user
```

真实项目里，Service 可能同时操作用户、订单、库存三个 Repository，提交点仍然只有一处。

## 1.4 不要跨层返回 ORM 对象

请求结束后 Session 关闭，ORM 对象可能处于 detached 状态，访问未加载属性会报错。API 层应返回 DTO（如 Pydantic 的 `UserOut`，配置 `from_attributes=True`），否则还会带来意外暴露敏感字段、序列化依赖 ORM 内部状态的问题。

# 2. Session 生命周期

## 2.1 Web 请求：一个请求一个 Session

```python
from collections.abc import Generator
from sqlalchemy.orm import Session, sessionmaker

SessionLocal = sessionmaker(bind=engine, autoflush=False, expire_on_commit=False)


def get_db() -> Generator[Session, None, None]:
    session = SessionLocal()
    try:
        yield session
    except Exception:
        session.rollback()       # 未提交的变更全部撤销
        raise
    finally:
        session.close()          # 归还连接
```

FastAPI 中通过 `Depends(get_db)` 注入，路由和 Service 共用同一个 Session。

★ **关键认知**：**Session 不是线程安全的，也不是协程安全的**。一个请求一个 Session，不要把它挂到全局变量，也不要在线程池任务之间共享。

## 2.2 后台任务与 CLI

```python
def send_report(user_id: int) -> None:
    with SessionLocal() as session:      # 后台任务自己创建
        user = session.get(User, user_id)
        ...
```

**不要把请求的 Session 传给后台任务**。请求结束后它已经关闭，后台任务再使用会报 detached 错误，甚至拿到已归还的连接。

CLI 与脚本用 `with SessionLocal() as session, session.begin()` 即可。长脚本要按批次提交，避免一个事务从头跑到尾。

## 2.3 生命周期原则

无论 Web、后台任务还是 CLI，原则都一样：**谁创建谁关闭，谁开始谁提交**。Web 用依赖注入，后台任务和 CLI 在任务内部新建，测试用 fixture 管理。

# 3. 异常处理

## 3.1 数据库异常翻译成业务语义

```python
from sqlalchemy.exc import IntegrityError


def create_user(session: Session, email: str) -> User:
    user = User(email=email)
    session.add(user)
    try:
        session.commit()
    except IntegrityError as err:
        session.rollback()
        raise EmailAlreadyExists(email) from err    # 保留原始因果链
    return user
```

| 数据库异常 | 常见原因 | 对外语义 |
|-----------|---------|---------|
| `IntegrityError`（唯一约束） | 重复邮箱、重复订单号 | 409 Conflict |
| `IntegrityError`（外键） | 引用的父记录不存在 | 400 / 409 |
| `NoResultFound` | 期望恰好一条却没有 | 404 |
| `OperationalError`（连接） | 数据库不可用、超时 | 503 |
| `DBAPIError`（序列化失败） | 隔离级别冲突 | 重试或 409 |
| `StaleDataError` | 乐观锁版本不匹配 | 409 + 提示重试 |

## 3.2 异常链不能丢

用 `raise EmailAlreadyExists(email) from err` 保留因果链，排查时能看到底层是哪个约束失败。直接抛新异常会丢掉最有价值的数据库错误。

## 3.3 不要吞异常

`except Exception: pass` 会让事务失败无人知晓。至少 `session.rollback()`、记录 `logger.exception(...)`，再把异常抛出去。

重试只用于可重试错误，例如 PostgreSQL 的 `40001`（序列化失败）和 `40P01`（死锁）。重试前确认操作幂等，且事务内没有不可撤销的外部副作用。

# 4. 异步边界

## 4.1 同步驱动会阻塞事件循环

在 `async` 路由里使用同步 Session，数据库调用会阻塞整个事件循环，其他请求只能排队。要么改用 `def` 路由（FastAPI 会放到线程池），要么使用异步驱动。

## 4.2 AsyncSession 的正确用法

```python
from sqlalchemy.ext.asyncio import AsyncSession, async_sessionmaker, create_async_engine

engine = create_async_engine(
    "postgresql+psycopg://app:secret@localhost/shop",
    pool_size=10,
)
AsyncSessionLocal = async_sessionmaker(engine, expire_on_commit=False)

async def get_user(user_id: int) -> User | None:
    async with AsyncSessionLocal() as session:
        return await session.get(User, user_id)
```

常见异步 URL：PostgreSQL 用 `postgresql+psycopg://` 或 `postgresql+asyncpg://`，MySQL 用 `mysql+aiomysql://`，SQLite 用 `sqlite+aiosqlite://`。

## 4.3 异步下的懒加载与并发

```python
user = await session.get(User, 1)
print(user.posts)      # MissingGreenlet：异步环境不允许隐式 IO
```

异步环境必须显式预加载：`select(User).options(selectinload(User.posts))`，否则会在访问关系属性时抛 `MissingGreenlet`。

★ **关键认知**：**异步 Session 不能跨任务共享**。`asyncio.gather` 里多个协程同时用一个 AsyncSession 会产生未定义行为。正确做法是每个任务创建自己的 Session，或串行执行。

# 5. 测试

## 5.1 单元测试与集成测试分开

| 层次 | 数据库 | 目的 | 速度 |
|------|--------|------|------|
| 单元测试 | SQLite 内存 | 验证业务逻辑 | 快 |
| 集成测试 | PostgreSQL（容器） | 验证 SQL、约束、并发 | 慢 |
| 迁移测试 | 空库跑 `upgrade head` | 验证版本链 | 中 |

⚠ SQLite 与 PostgreSQL 行为差异明显：类型检查宽松、`ALTER TABLE` 能力弱、锁与并发语义不同、JSON/数组支持有限。**核心查询和迁移必须在真实数据库上验证**。

## 5.2 事务回滚式 fixture

```python
import pytest
from sqlalchemy import create_engine
from sqlalchemy.orm import Session
from sqlalchemy.pool import StaticPool
from app.models import Base


@pytest.fixture
def session():
    engine = create_engine("sqlite:///:memory:", poolclass=StaticPool)
    Base.metadata.create_all(engine)

    connection = engine.connect()
    trans = connection.begin()
    session = Session(bind=connection, join_transaction_mode="create_savepoint")

    try:
        yield session
    finally:
        session.close()
        trans.rollback()          # 每个测试结束自动清理
        connection.close()
```

`join_transaction_mode="create_savepoint"` 让测试代码里的 commit 只提交到保存点，外层事务统一回滚，测试之间互不影响。

## 5.3 真实数据库集成测试

用 Testcontainers 启动 PostgreSQL，验证约束、索引、并发和迁移。CI 里可以只对关键用例启用，避免每次都拉镜像。**迁移测试尤其重要**：从空库跑 `upgrade head`，再跑 `downgrade base`，确认版本链完整。

# 6. 安全与运维

## 6.1 SQL 注入

```python
# 错误
session.execute(text(f"SELECT * FROM users WHERE email = '{email}'"))

# 正确
session.execute(text("SELECT * FROM users WHERE email = :email"), {"email": email})
```

ORM 的 `where(User.email == email)` 本身是参数化的，安全。**风险集中在 `text()` 拼接、动态表名、动态排序字段**。排序字段要用白名单映射，不能直接把用户输入拼进 `order_by`。

## 6.2 最小权限与密钥

| 账号 | 权限 | 用途 |
|------|------|------|
| 应用账号 | 仅业务表的 SELECT/INSERT/UPDATE/DELETE | 运行服务 |
| 迁移账号 | 额外 DDL 权限 | 执行 Alembic |
| 只读账号 | 仅 SELECT | 报表、排查 |

应用账号不应有 `DROP`、`ALTER` 权限。数据库 URL 从环境变量或密钥管理服务读取，不提交仓库。

## 6.3 可观测性

生产环境关注：慢查询数量与 P99、连接池使用率与等待时间、`idle in transaction` 数量、迁移执行耗时与锁等待、死锁和序列化失败次数。对超过阈值（如 200ms）的语句记录结构化日志，但不要打印完整连接串和参数中的敏感字段。

# 7. 工程清单

| 检查项 | 建议 |
|--------|------|
| Base | 全项目唯一，配置命名约定 |
| 模型 | 用 `Mapped` 注解，金额用 `Numeric`，时间带时区 |
| 关系 | 显式预加载，禁止隐式 N+1 |
| Session | 一个请求一个，`with` 或依赖注入管理 |
| 事务 | 边界在 Service，Repository 不 commit |
| 异常 | 数据库异常翻译为业务异常，保留异常链 |
| 批量 | 数据导入用 executemany，不用逐条 add |
| 迁移 | 每次变更都有脚本，生产独立执行 |
| 测试 | SQLite 跑单测，真实库跑集成与迁移 |
| 异步 | AsyncSession 不跨任务，禁止隐式懒加载 |
| 安全 | 参数绑定、最小权限、密钥不入库 |
| 监控 | 慢查询、连接池、长事务、迁移耗时 |

# 8. 动手练习

1. 把一个"路由里直接写 Session 和 SQL"的小项目重构成 API / Service / Repository 三层，要求 Repository 不出现 `commit()`，并补一个 Service 层单元测试。
2. 为你的项目写一套 pytest fixture：SQLite 内存库跑快速测试，Postgres 容器跑约束与迁移测试，统计两类测试的耗时差异。
3. 写一个异步查询接口，故意访问未预加载的关系复现 `MissingGreenlet`，再用 `selectinload` 修复，并解释为什么异步环境不能隐式发 SQL。

# 9. 小结

| 主题 | 核心结论 |
|------|---------|
| 分层 | API 管协议，Service 管事务与规则，Repository 管查询 |
| Session | 一个请求一个，不跨线程、不跨协程、不跨后台任务 |
| 异常 | 数据库错误翻译成业务语义，保留 `from err` 因果链 |
| 异步 | 用 AsyncSession 与异步驱动，显式预加载，禁止混用 |
| 测试 | 单测求快，集成测试求真，迁移必须单独验证 |
| 安全 | 参数绑定、最小权限、密钥外置 |
| 运维 | 慢查询、连接池、长事务、迁移耗时都要监控 |

回看整个模块：第 1 篇建立 Engine / Core / ORM 的分层认知，第 2 到 4 篇把模型、会话、关系的地基打牢，第 5 到 7 篇处理事务、迁移和性能这些生产问题，本篇收束为工程规范。下一步可以把这个数据访问层接到 FastAPI 上，用依赖注入管理 Session，用 Pydantic 做输入输出边界，形成一条完整的 Web 服务链路。
