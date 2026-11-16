测试是 FastAPI 项目最容易被跳过的一环。接口能跑通不等于可靠：参数边界、权限分支、异常路径、数据库事务，都需要测试来兜底。好在 FastAPI 的依赖注入给了测试一个天然切口——替换依赖即可隔离数据库、认证和外部服务。本篇从 `TestClient` 讲到异步测试、数据库隔离和契约测试。

# 1. 测试的分层

## 1.1 三层测试

| 层级 | 测什么 | 速度 | 依赖 |
|------|--------|------|------|
| 单元测试 | 纯函数、校验逻辑 | 毫秒 | 无 |
| 集成测试 | 路由 + 依赖 + 数据库 | 百毫秒 | 有 |
| 契约测试 | 响应结构、状态码、OpenAPI | 秒 | 有 |

★ **关键认知**：不要试图用测试覆盖一切。**单元测试保证逻辑正确，集成测试保证接线正确，契约测试保证对外承诺不破**。三者职责不同，混在一起会又慢又脆。测试文件放 `tests/`，共享 fixture 写进 `conftest.py`。

# 2. TestClient 入门

## 2.1 安装

```bash
pip install pytest httpx
```

FastAPI 的 `TestClient` 依赖 `httpx`。`fastapi[standard]` 已经包含。

## 2.2 第一个测试

```python
# tests/test_main.py
from fastapi.testclient import TestClient
from app.main import app

client = TestClient(app)

def test_read_root():
    response = client.get("/")
    assert response.status_code == 200
    assert response.json() == {"message": "Hello FastAPI"}
```

`TestClient` 不会真的监听端口，它直接把请求交给 ASGI 应用，所以速度快、无需启动服务。

## 2.3 使用上下文管理器

```python
with TestClient(app) as client:
    response = client.get("/")
```

带 `with` 会触发 `startup` / `shutdown` 事件。如果应用依赖启动时初始化连接池，测试必须用这种写法。

⚠ **模块级 `client = TestClient(app)` 不会触发 lifespan 事件**。需要生命周期初始化时，用 fixture + `with`。

# 3. pytest fixture

## 3.1 共享 client

```python
# tests/conftest.py
import pytest
from fastapi.testclient import TestClient
from app.main import app

@pytest.fixture
def client():
    with TestClient(app) as c:
        yield c
```

测试直接声明参数：

```python
def test_health(client):
    assert client.get("/health").status_code == 200
```

## 3.2 fixture 作用域

作用域从 `function` 到 `session`，越大隔离越差。连接池用 `session` 合理，会话对象必须 `function` 级。

★ **关键认知**：**作用域越大，隔离越差**。会话跨测试共享，会出现未提交/未回滚的状态污染。

## 3.3 参数化

```python
import pytest

@pytest.mark.parametrize(
    "payload,expected",
    [
        ({"name": "a"}, 422),                       # 名字太短
        ({"name": "alice", "age": -1}, 422),        # 年龄非法
        ({"name": "alice", "age": 18}, 201),        # 合法
    ],
)
def test_create_user_validation(client, payload, expected):
    response = client.post("/users", json=payload)
    assert response.status_code == expected
```

参数化把"同一逻辑、不同输入"压成一张表，失败时 pytest 会指出具体哪一组。

# 4. 依赖覆盖

## 4.1 替换数据库会话

```python
# tests/conftest.py
import pytest
from fastapi.testclient import TestClient
from app.main import app
from app.db import get_session

@pytest.fixture
def client():
    app.dependency_overrides[get_session] = fake_session
    with TestClient(app) as c:
        yield c
    app.dependency_overrides.clear()
```

`fake_session` 可以是内存实现，也可以是真数据库的测试会话。

## 4.2 替换当前用户

```python
def fake_user():
    return {"id": 1, "name": "tester", "roles": ["user"]}

@pytest.fixture
def auth_client(client):
    app.dependency_overrides[get_current_user] = fake_user
    yield client
    app.dependency_overrides.pop(get_current_user, None)
```

测试权限时，分别覆盖成普通用户和管理员，验证 403 / 200。

## 4.3 覆盖的边界

适合覆盖数据库会话、外部客户端、当前用户和配置；不要覆盖正在被测的路由函数与校验逻辑。

⚠ **覆盖必须清理**。忘记 `clear()` 会让后续测试继承假依赖，出现"单跑通过、全跑失败"的诡异现象。用 fixture 的 `yield` + 清理最稳。

# 5. 异步测试

## 5.1 为什么需要

`async def` 路由里的逻辑无法用同步测试直接 `await`。多数接口测试用 `TestClient` 就够；需要直接测异步函数或异步数据库时，才用 `httpx.AsyncClient` + `ASGITransport`。

## 5.2 异步客户端

```bash
pip install pytest-asyncio
```

```python
import pytest
import httpx
from httpx import ASGITransport
from app.main import app

@pytest.mark.asyncio
async def test_async_endpoint():
    transport = ASGITransport(app=app)
    async with httpx.AsyncClient(
        transport=transport,
        base_url="http://test",
    ) as client:
        response = await client.get("/async")
    assert response.status_code == 200
```

`ASGITransport` 把 `httpx` 的请求直接送进 ASGI 应用，不经过网络。

## 5.3 异步 fixture

```python
import pytest_asyncio

@pytest_asyncio.fixture
async def async_client():
    transport = ASGITransport(app=app)
    async with httpx.AsyncClient(transport=transport, base_url="http://test") as c:
        yield c
```

⚠ **不要用普通 `@pytest.fixture` 定义异步 fixture**，会得到协程对象而不是值。要么用 `pytest_asyncio.fixture`，要么在 `pyproject.toml` 里开启 `asyncio_mode = "auto"`。

# 6. 数据库隔离

## 6.1 三种策略

隔离策略有重建库、事务回滚、独立 schema 三种；**推荐事务回滚**，快且干净。

## 6.2 事务回滚

`engine` 由 session 级 fixture 提供，测试库用独立 URL。

```python
import pytest
from sqlalchemy.ext.asyncio import AsyncSession, async_sessionmaker

@pytest.fixture
async def db_session(engine):
    connection = await engine.connect()
    transaction = await connection.begin()

    Session = async_sessionmaker(bind=connection, expire_on_commit=False)
    session = Session()

    try:
        yield session
    finally:
        await session.close()
        await transaction.rollback()   # 丢弃本测试的所有写入
        await connection.close()
```

业务代码里的 `commit()` 只提交到外层事务，测试结束统一回滚，数据库保持干净。

## 6.3 依赖覆盖接上测试会话

把 `get_session` 覆盖成返回 `db_session` 的函数，路由就会使用测试事务。

★ **关键认知**：**测试隔离的关键不是"删数据"，而是"让每个测试都在自己的事务里"**。删除容易漏表、受外键约束影响，回滚则天然干净。

测试用独立数据库（如 `test.db` 或临时 PostgreSQL），建表用 session 级 fixture，**绝不连开发库或生产库**。

# 7. 契约测试

## 7.1 什么是契约

契约是"接口对外承诺的形状"：路径、方法、状态码、响应字段、字段类型。契约测试验证这些承诺没被无意改掉。

```python
def test_user_response_contract(client):
    response = client.get("/users/1")
    assert response.status_code == 200

    body = response.json()
    assert set(body.keys()) == {"id", "name", "email"}
    assert isinstance(body["id"], int)
    assert isinstance(body["name"], str)
```

## 7.2 用 OpenAPI schema 校验

请求 `/openapi.json`，断言关键路径与方法存在；也可以进一步用 `schemathesis` 基于 schema 自动生成请求。初学阶段手写关键断言即可。

## 7.3 契约变更的意识

新增可选字段通常不破坏契约；删除字段、字段改名、改类型、新增必填请求字段都会破坏契约，必须走版本化。

# 8. 覆盖率与 CI

## 8.1 覆盖率

```bash
pip install pytest-cov
pytest --cov=app --cov-report=term-missing
```

⚠ **覆盖率是下限不是目标**。100% 覆盖也可能全是无效断言。优先覆盖分支、异常路径和边界值。

## 8.2 CI 中的测试

CI 里固定 Python 版本、装依赖、跑 `pytest -q`，并用独立数据库 service container，测试通过才允许合并。

# 9. 常见错误

| 问题 | 后果 | 正确做法 |
|------|------|---------|
| 测试连开发库 | 数据被污染 | 独立测试库 + 事务回滚 |
| 覆盖后不清理 | 测试互相污染 | fixture 里 `clear()` |
| 异步 fixture 用同步装饰器 | 拿到协程对象 | `pytest_asyncio.fixture` |
| 只测 200 分支 | 异常路径无覆盖 | 补 4xx/5xx 用例 |
| 断言整个 JSON 字符串 | 字段顺序/新增字段即失败 | 断言关键字段 |
| 用 `TestClient` 不触发 lifespan | 连接池未初始化 | `with TestClient(app)` |
| 覆盖率当 KPI | 测试变形式主义 | 关注分支与边界 |

# 10. 练习

1. 为 `/users` 的创建接口写参数化测试，覆盖 201、422（缺字段、类型错误、越界）三类情况。
2. 用 `dependency_overrides` 替换 `get_current_user`，分别以普通用户和管理员身份请求删除接口，断言 403 与 200。
3. 搭一个事务回滚的数据库 fixture，写一个会写库的测试，运行两次确认第二次不受第一次数据影响。

# 11. 小结

| 主题 | 要点 |
|------|------|
| TestClient | 直接调用 ASGI，无需起服务 |
| fixture | `conftest.py` 共享，作用域越大隔离越差 |
| 参数化 | 一张表覆盖多组输入 |
| 依赖覆盖 | `dependency_overrides` 按函数对象替换，记得清理 |
| 异步测试 | `httpx.AsyncClient` + `ASGITransport` |
| 数据库隔离 | 每测试一事务，结束回滚 |
| 契约 | 断言关键字段与 schema，改契约要评估兼容性 |

下一篇进入部署与总结，讲清 uvicorn/gunicorn 的分工、Docker 镜像、健康检查、优雅关闭、监控指标，以及整个模块的收束。
