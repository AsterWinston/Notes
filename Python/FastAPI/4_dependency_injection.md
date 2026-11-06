依赖注入（Dependency Injection）是 FastAPI 里被低估的能力。多数人只把它当"取当前用户"的工具，实际上它是一套**声明式的前置逻辑复用机制**：参数校验、数据库会话、权限、分页、配置，都能写成依赖，在路由函数执行前自动完成。本篇把 `Depends` 的解析、复用、类依赖、`yield` 清理和测试替换讲透。

# 1. Depends 的基本用法

## 1.1 最小示例

```python
from typing import Annotated
from fastapi import Depends, FastAPI

app = FastAPI()

def common_params(q: str | None = None, skip: int = 0, limit: int = 10):
    return {"q": q, "skip": skip, "limit": limit}

@app.get("/items")
def list_items(params: Annotated[dict, Depends(common_params)]):
    return params
```

FastAPI 在执行 `list_items` 之前，先调用 `common_params`，把它解析到的查询参数打包，再注入 `params`。`common_params` 里同样可以有类型注解和 `Query` 约束。

`Annotated[dict, Depends(...)]` 是推荐写法，等价于 `dict = Depends(...)`。

★ **关键认知**：`Depends` 的返回值就是被注入的值，但**依赖函数的参数也会被 FastAPI 解析**。也就是说，依赖本身可以声明路径参数、查询参数、请求体、其他依赖——它和路由函数享有同一套解析能力。

## 1.2 依赖也能取路径参数

```python
def get_item_or_404(item_id: int):
    if item_id > 100:
        raise HTTPException(status_code=404, detail="商品不存在")
    return {"id": item_id}

@app.get("/items/{item_id}")
def read_item(item: Annotated[dict, Depends(get_item_or_404)]):
    return item
```

依赖里声明的 `item_id` 会从路径中取，无需在路由函数里重复声明。

# 2. 依赖复用与层级

## 2.1 一处定义，多处使用

```python
def verify_token(token: str | None = None):
    if token != "secret":
        raise HTTPException(status_code=401, detail="无效令牌")
    return token

@app.get("/profile")
def profile(token: Annotated[str, Depends(verify_token)]):
    return {"token": token}

@app.get("/orders")
def orders(token: Annotated[str, Depends(verify_token)]):
    return {"token": token}
```

同一依赖在多个路由里声明，FastAPI 会按需调用，不必手写重复代码。

## 2.2 子依赖

依赖可以再依赖别的依赖，形成一棵树：

```python
def get_settings():
    return {"db_url": "sqlite:///./app.db"}

def get_db(settings: Annotated[dict, Depends(get_settings)]):
    # 实际项目里这里创建会话
    return {"conn": "db", "url": settings["db_url"]}

@app.get("/users")
def list_users(db: Annotated[dict, Depends(get_db)]):
    return {"db": db}
```

```text
list_users
   └── get_db
         └── get_settings
```

FastAPI 会自底向上解析，先 `get_settings`，再 `get_db`，最后 `list_users`。

★ **关键认知**：同一次请求里，**同一个依赖默认只执行一次**，结果被缓存复用。即使 `get_db` 被多个子依赖引用，也只调用一遍。这个缓存按"请求"隔离，不会跨请求共享。

## 2.3 关闭缓存

```python
@app.get("/a")
def a(
    x: Annotated[int, Depends(get_value)],
    y: Annotated[int, Depends(get_value, use_cache=False)],
):
    return {"x": x, "y": y}
```

`use_cache=False` 会让该依赖每次都重新执行。只在依赖有副作用且必须重算时使用。

# 3. 类依赖

## 3.1 用类封装带状态的逻辑

```python
class Pagination:
    def __init__(self, skip: int = 0, limit: int = 10):
        self.skip = skip
        self.limit = limit

@app.get("/items")
def list_items(page: Annotated[Pagination, Depends()]):
    return {"skip": page.skip, "limit": page.limit}
```

`Depends()` 不传参数时，FastAPI 会**用类本身作为依赖**：实例化时，构造函数的参数会被当作查询参数解析。

## 3.2 类的可调用实例

```python
class TokenChecker:
    def __init__(self, required_role: str):
        self.required_role = required_role

    def __call__(self, role: str = "user"):
        if role != self.required_role:
            raise HTTPException(status_code=403, detail="权限不足")
        return role

admin_only = TokenChecker("admin")

@app.get("/admin", dependencies=[Depends(admin_only)])
def admin_panel():
    return {"ok": True}
```

类依赖适合"带配置的检查器"：`__init__` 收配置，`__call__` 收请求参数。

# 4. yield 依赖

## 4.1 资源管理

数据库会话、文件句柄、锁这类资源，需要"用完关闭"。`yield` 依赖把"准备"和"清理"写在同一个函数里：

```python
from collections.abc import Generator

def get_db() -> Generator[dict, None, None]:
    db = {"connected": True}       # 准备：创建连接
    try:
        yield db                   # 交给路由使用
    finally:
        db["connected"] = False    # 清理：无论如何都执行
        print("数据库连接已关闭")

@app.get("/users")
def list_users(db: Annotated[dict, Depends(get_db)]):
    return {"connected": db["connected"]}
```

执行顺序：

```text
1. 调用 get_db，创建资源
2. 执行到 yield，把值注入路由函数
3. 路由函数执行完毕，响应生成
4. 回到 get_db，执行 finally 清理
```

★ **关键认知**：`yield` 之后的代码在**响应发送之后**执行。所以清理逻辑不能依赖"修改响应内容"——响应已经发出去了。要影响响应，必须在 `yield` 之前完成。

⚠ **`yield` 依赖里必须用 `try/finally` 保证清理**。如果路由抛异常，`yield` 之后的普通代码可能被跳过，只有 `finally` 一定执行。

## 4.2 异常传播

```python
def get_db():
    db = create_session()
    try:
        yield db
    except Exception:
        db.rollback()      # 路由抛异常时回滚
        raise
    finally:
        db.close()
```

路由里抛出的 `HTTPException` 也会经过这里，所以事务回滚、连接释放都能统一处理。

## 4.3 依赖的退出顺序

嵌套的 `yield` 依赖按**后进先出**清理：

```text
get_settings  →  get_db  →  get_tx  →  路由
                   ↑          ↑
                先关闭      先回滚
```

理解成栈即可：最后进入的最先退出。

# 5. 依赖在路由组上的应用

## 5.1 路由级依赖

```python
from fastapi import APIRouter

router = APIRouter(
    prefix="/admin",
    dependencies=[Depends(verify_token)],   # 不注入返回值，只做检查
)

@router.get("/stats")
def stats():
    return {"visits": 100}
```

`dependencies=[...]` 里的依赖**不把返回值传给路由函数**，只执行副作用（校验、记录日志）。需要返回值时，仍在函数参数里声明。

# 6. 测试中替换依赖

## 6.1 dependency_overrides

FastAPI 提供官方替换机制，测试时不必改业务代码：

```python
from fastapi.testclient import TestClient

def fake_db():
    return {"connected": True, "fake": True}

app.dependency_overrides[get_db] = fake_db
client = TestClient(app)

def test_users():
    resp = client.get("/users")
    assert resp.json()["connected"] is True

app.dependency_overrides.clear()      # 测试后清理
```

★ **关键认知**：`dependency_overrides` 的键是**原始依赖函数对象**，不是它的名字。替换会递归生效——覆盖了 `get_db`，所有依赖 `get_db` 的路由都会用假实现，包括嵌套依赖。

## 6.2 用 pytest fixture 管理

```python
import pytest
from fastapi.testclient import TestClient

@pytest.fixture
def client():
    app.dependency_overrides[get_db] = fake_db
    with TestClient(app) as c:
        yield c
    app.dependency_overrides.clear()
```

这样每个测试自动获得干净的替换，不会互相污染。第 8 篇会展开完整的测试结构。

## 6.3 替换的边界

- 适合替换：数据库会话、外部 HTTP 客户端、当前用户、配置。
- 不适合替换：正在被测的纯函数本身，那等于没测。
- 注意：覆盖只影响 FastAPI 的依赖解析，不影响直接 `import` 调用的函数。

# 7. 常见错误

| 问题 | 后果 | 正确做法 |
|------|------|---------|
| `yield` 依赖不用 `try/finally` | 异常时资源泄漏 | 用 `try/finally` 包裹 |
| 以为依赖结果跨请求缓存 | 误共享状态 | 缓存只在单次请求内 |
| 在 `dependencies=[]` 里指望拿返回值 | 拿不到 | 需要返回值就写进函数参数 |
| 覆盖依赖时传函数名字符串 | 不生效 | 传函数对象 |
| 测试后不清空覆盖 | 污染后续测试 | fixture 里清理 |
| 依赖里写阻塞 IO 且用 async | 卡住事件循环 | 同步依赖用 `def` |

# 8. 练习

1. 写一个 `get_current_user` 依赖，从请求头 `X-Token` 读取令牌，令牌不等于 `"secret"` 时抛 401，等于时返回 `{"name": "alice"}`。在两个路由里复用它。
2. 写一个 `yield` 依赖模拟数据库会话，进入时打印"打开"，退出时打印"关闭"；在路由里故意抛 `HTTPException`，确认 `finally` 仍然执行。
3. 用 `dependency_overrides` 把第 2 题的会话依赖替换成假对象，写一个 `TestClient` 测试，断言路由拿到的是假数据，并在测试结束后清理覆盖。

# 9. 小结

| 主题 | 要点 |
|------|------|
| Depends | 路由前执行，返回值注入参数 |
| 子依赖 | 依赖可嵌套，自底向上解析 |
| 缓存 | 单次请求内同一依赖只执行一次，可 `use_cache=False` 关闭 |
| 类依赖 | `Depends()` 自动实例化，适合带配置的检查器 |
| yield | 响应后清理，必须 `try/finally` |
| 路由组 | `dependencies=[...]` 只做检查，不注入 |
| 测试 | `dependency_overrides` 按函数对象替换 |

下一篇进入异步与数据库，讲清 `async/await` 在 FastAPI 里的正确用法、SQLAlchemy 2 的异步会话管理，以及阻塞调用如何避免拖垮事件循环。
