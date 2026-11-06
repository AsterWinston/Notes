FastAPI 的异步能力来自 ASGI，但"框架支持异步"不等于"你的代码真的异步"。只要在 `async def` 里混进一个阻塞调用，整个事件循环就会被卡住。数据库是最容易出事的地方：用错驱动、会话管理不当、忘记 `await`，都会让并发退化成串行。本篇把 async/await、SQLAlchemy 2 异步会话和阻塞调用的边界讲清楚。

# 1. async/await 在 FastAPI 里的位置

## 1.1 两种路由的执行模型

```python
@app.get("/sync")
def sync_route():
    return {"mode": "threadpool"}     # 跑在线程池

@app.get("/async")
async def async_route():
    return {"mode": "event_loop"}     # 跑在事件循环
```

| 写法 | 执行位置 | 阻塞后果 |
|------|---------|---------|
| `def` | 线程池 | 只占一个线程，其他请求仍可处理 |
| `async def` | 事件循环 | 卡住所有请求 |

★ **关键认知**：FastAPI 对 `def` 路由的处理是"丢进线程池"，对 `async def` 路由是"直接在事件循环里跑"。所以**同步代码写成 `def` 是安全的，写成 `async def` 反而危险**。选择依据不是"哪个更高级"，而是"这段代码是不是真异步"。

## 1.2 什么时候该用 async

| 场景 | 建议 |
|------|------|
| 数据库访问（异步驱动） | `async def` |
| 调用外部 HTTP API（httpx.AsyncClient） | `async def` |
| Redis / 消息队列异步客户端 | `async def` |
| 文件读写（同步库） | `def` 或 `asyncio.to_thread` |
| CPU 密集计算 | `def` + 进程池，不要放事件循环 |
| 同步 ORM / 同步 SDK | `def` |

⚠ **不要为了"统一风格"把所有路由都写成 `async def`**。一旦里面用了同步数据库驱动，性能会比全同步还差，因为线程池的并行能力也被放弃了。

# 2. 异步数据库选型

## 2.1 驱动对照

| 数据库 | 同步驱动 | 异步驱动 |
|--------|---------|---------|
| PostgreSQL | `psycopg2` | `asyncpg` / `psycopg` 3 |
| MySQL | `pymysql` | `aiomysql` / `asyncmy` |
| SQLite | `sqlite3` | `aiosqlite` |

SQLAlchemy 2 通过不同的 URL scheme 选择驱动：

```text
postgresql+asyncpg://user:pass@localhost/db
mysql+aiomysql://user:pass@localhost/db
sqlite+aiosqlite:///./app.db
```

安装依赖：`pip install "sqlalchemy[asyncio]" asyncpg`，SQLite 开发再装 `aiosqlite`。

★ **关键认知**：**异步 ORM 必须配异步驱动**。URL 里写 `postgresql://`（默认 psycopg2）却在 `async def` 里用 `AsyncSession`，会在运行时报 "greenlet_spawn has not been called" 之类的错误。驱动和会话类型必须成对。

# 3. SQLAlchemy 2 异步基础

## 3.1 定义模型

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
        DateTime, server_default=func.now()
    )
```

`Mapped[T]` + `mapped_column()` 是 SQLAlchemy 2 的声明式写法，类型即列类型，IDE 能推断。

## 3.2 创建引擎与会话工厂

```python
from sqlalchemy.ext.asyncio import (
    AsyncSession,
    async_sessionmaker,
    create_async_engine,
)

engine = create_async_engine(
    "sqlite+aiosqlite:///./app.db",
    echo=False,          # 生产环境关闭 SQL 日志
    pool_pre_ping=True,  # 取连接前先探活，避免拿到失效连接
)

SessionLocal = async_sessionmaker(
    bind=engine,
    class_=AsyncSession,
    expire_on_commit=False,   # 提交后仍可读取对象属性
)
```

| 参数 | 作用 |
|------|------|
| `echo` | 打印 SQL，调试用 |
| `pool_size` | 连接池大小 |
| `max_overflow` | 超出池大小的临时连接数 |
| `pool_pre_ping` | 使用前探活 |
| `expire_on_commit` | 提交后是否让对象属性过期 |

⚠ **异步场景建议 `expire_on_commit=False`**。默认 `True` 时，`commit()` 后访问对象属性会触发一次懒加载，而在异步里懒加载会报 `MissingGreenlet`。

## 3.3 会话依赖

```python
from collections.abc import AsyncGenerator
from typing import Annotated
from fastapi import Depends

async def get_session() -> AsyncGenerator[AsyncSession, None]:
    async with SessionLocal() as session:
        try:
            yield session
        except Exception:
            await session.rollback()   # 出错回滚
            raise

SessionDep = Annotated[AsyncSession, Depends(get_session)]
```

`async with` 保证会话关闭，`try/except` 保证异常回滚。路由里只声明 `SessionDep`，不关心创建与销毁。

## 3.4 增删改查

```python
from fastapi import FastAPI, HTTPException
from sqlalchemy import select

app = FastAPI()

@app.post("/users")
async def create_user(name: str, email: str, session: SessionDep):
    user = User(name=name, email=email)
    session.add(user)
    await session.commit()
    await session.refresh(user)     # 拿回自增 id 和默认值
    return {"id": user.id, "name": user.name}

@app.get("/users")
async def list_users(session: SessionDep):
    result = await session.execute(select(User).order_by(User.id))
    users = result.scalars().all()  # scalars() 取实体，不是 Row
    return [{"id": u.id, "name": u.name} for u in users]

@app.get("/users/{user_id}")
async def get_user(user_id: int, session: SessionDep):
    user = await session.get(User, user_id)   # 按主键取，最省事
    if user is None:
        raise HTTPException(status_code=404, detail="用户不存在")
    return {"id": user.id, "name": user.name}
```

★ **关键认知**：异步会话的每个数据库操作都要 `await`。`session.add()` 是例外——它只是把对象放进待提交集合，不访问数据库，所以不 `await`。`commit`、`execute`、`refresh`、`get`、`delete` 都是协程。

## 3.5 事务边界

```python
@app.post("/transfer")
async def transfer(from_id: int, to_id: int, amount: int, session: SessionDep):
    async with session.begin():        # 显式事务，异常自动回滚
        a = await session.get(User, from_id)
        b = await session.get(User, to_id)
        if a is None or b is None:
            raise HTTPException(status_code=404, detail="账户不存在")
        # 这里省略余额字段的增减，仅示意事务写法
    return {"from": from_id, "to": to_id, "amount": amount}
```

`async with session.begin()` 在块结束时提交，异常时回滚。适合"多条写操作必须一起成功"的场景。简单单条写入用 `commit()` 即可。

# 4. 阻塞调用怎么办

## 4.1 识别阻塞

`time.sleep`、`requests.get`、大文件同步读写、同步数据库驱动都会阻塞；`await asyncio.sleep`、`httpx.AsyncClient`、异步会话不会。

## 4.2 三种处理方式

```python
import asyncio
import httpx

# 方式一：路由改成 def，交给线程池
@app.get("/sync-io")
def sync_io():
    return {"data": "ok"}

# 方式二：异步路由里用 to_thread 包住阻塞调用
@app.get("/thread-io")
async def thread_io():
    result = await asyncio.to_thread(blocking_function)
    return {"result": result}

# 方式三：换成真正的异步库
@app.get("/async-io")
async def async_io():
    async with httpx.AsyncClient() as client:
        resp = await client.get("https://example.com")
        return {"status": resp.status_code}
```

★ **关键认知**：`asyncio.to_thread` 是"把阻塞函数挪到线程池"的逃生通道，适合无法替换的同步库。但它有线程池上限，不能当成无限并发方案。高频路径应换成原生异步客户端。

⚠ **`to_thread` 里不能碰 `AsyncSession`**。数据库会话不是线程安全的，跨线程使用会出诡异错误。会话只能在创建它的事件循环里用。

# 5. 常见陷阱

## 5.1 MissingGreenlet

```text
sqlalchemy.exc.MissingGreenlet: greenlet_spawn has not been called;
can't call await_only() here.
```

常见原因：

1. `expire_on_commit=True` 且提交后访问属性。
2. 在异步会话里触发了同步懒加载（如 `user.orders`）。
3. 在 `async def` 里用了同步驱动。

解决：设 `expire_on_commit=False`；用 `selectinload` 预加载关联；确认驱动是异步的。

```python
from sqlalchemy.orm import selectinload

result = await session.execute(
    select(User).options(selectinload(User.orders))
)
```

## 5.2 会话跨请求复用

```python
# 错误示范：全局会话
session = SessionLocal()

@app.get("/bad")
async def bad():
    await session.execute(select(User))   # 并发请求共享，事务互相污染
```

会话必须**每请求一个**，用 `yield` 依赖创建与关闭。

## 5.3 在依赖里捕获异常后忘记 re-raise

```python
async def get_session():
    async with SessionLocal() as s:
        try:
            yield s
        except Exception:
            await s.rollback()
            # 漏了 raise，异常被吞，客户端收到 200
```

回滚后必须重新抛出，否则 FastAPI 认为一切正常。

# 6. 常见错误

| 问题 | 后果 | 正确做法 |
|------|------|---------|
| `async def` 里用同步驱动 | 卡住事件循环或报错 | 换异步驱动 |
| 忘记 `await session.execute` | 拿到协程对象 | 所有 DB 操作都 `await` |
| `commit` 后访问属性报错 | `MissingGreenlet` | `expire_on_commit=False` |
| 会话全局复用 | 并发事务污染 | 每请求一个会话 |
| 依赖里吞异常 | 错误被掩盖 | 回滚后 `raise` |
| 关联查询懒加载 | 异步下报错 | `selectinload` 预加载 |

# 7. 练习

1. 用 `sqlite+aiosqlite` 建一张 `Article` 表（`id`、`title`、`content`、`created_at`），实现创建与列表两个异步接口，列表按 `id` 倒序。
2. 给列表接口加分页参数 `skip`、`limit`，用 `select(...).offset(...).limit(...)` 实现，并验证 `await` 链路完整。
3. 写一个 `async def` 路由，里面先 `await asyncio.sleep(1)`，再调用 `asyncio.to_thread(time.sleep, 1)`，用并发请求测总耗时，解释为什么总耗时约 1 秒而不是 2 秒。

# 8. 小结

| 主题 | 要点 |
|------|------|
| 执行模型 | `def` 走线程池，`async def` 走事件循环 |
| 选型 | 真异步用 `async def`，同步库用 `def` |
| 驱动 | 异步 ORM 必须配异步驱动 |
| 会话 | 每请求一个，`yield` 创建与关闭 |
| 事务 | `commit` / `async with session.begin()` |
| 阻塞 | `to_thread` 兜底，不能碰会话 |
| 陷阱 | `MissingGreenlet`、懒加载、异常吞掉 |

下一篇进入认证与安全，讲清 OAuth2 密码流、JWT 签发与校验、密码哈希、权限控制、CORS、限流和密钥管理。
