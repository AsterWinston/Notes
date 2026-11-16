路由决定"哪个 URL 交给哪个函数"，参数决定"函数能拿到什么"。这两件事看起来简单，实际最容易写错：参数放错了位置，框架不会报错，只会给你一个空值或 422。本篇把路径、查询、请求体三者的边界划清，再讲状态码、`APIRouter` 分组和依赖的初步用法。

# 1. 路由基础

## 1.1 声明一个路由

```python
from fastapi import FastAPI

app = FastAPI()

@app.get("/users")
def list_users():
    return [{"id": 1, "name": "alice"}]
```

装饰器 `@app.get(...)` 做了两件事：把路径注册进路由表，并把函数标记为 GET 处理函数。常用的方法装饰器：

| 装饰器 | HTTP 方法 | 典型语义 |
|--------|----------|---------|
| `@app.get` | GET | 读取资源，不应有副作用 |
| `@app.post` | POST | 创建资源 |
| `@app.put` | PUT | 整体替换资源 |
| `@app.patch` | PATCH | 局部更新 |
| `@app.delete` | DELETE | 删除资源 |

★ **关键认知**：路由匹配是**按注册顺序**进行的，先匹配到的先赢。所以固定路径要写在动态路径前面，否则 `/users/me` 会被 `/users/{user_id}` 抢先匹配。

## 1.2 路径顺序陷阱

```python
@app.get("/users/{user_id}")
def get_user(user_id: str):
    return {"user_id": user_id}

@app.get("/users/me")
def get_me():
    return {"user": "current"}
```

访问 `/users/me` 会命中第一个函数，`user_id` 变成字符串 `"me"`。正确做法：

```python
@app.get("/users/me")          # 固定路径在前
def get_me():
    return {"user": "current"}

@app.get("/users/{user_id}")   # 动态路径在后
def get_user(user_id: str):
    return {"user_id": user_id}
```

# 2. 路径参数

## 2.1 类型转换与校验

路径参数写在 `{}` 里，函数参数名必须一致，类型注解会自动转换：

```python
@app.get("/items/{item_id}")
def read_item(item_id: int):
    return {"item_id": item_id}
```

访问 `/items/42` 得到 `{"item_id": 42}`；访问 `/items/abc` 会返回 **422 Unprocessable Entity**，而不是 404。

想加约束，用 `Path`：

```python
from typing import Annotated
from fastapi import Path

@app.get("/items/{item_id}")
def read_item(
    item_id: Annotated[int, Path(ge=1, le=1000, description="商品 ID")],
):
    return {"item_id": item_id}
```

| 参数 | 含义 |
|------|------|
| `ge` / `le` | 大于等于 / 小于等于 |
| `gt` / `lt` | 大于 / 小于 |
| `min_length` / `max_length` | 字符串长度 |
| `pattern` | 正则约束 |
| `description` | 出现在文档里 |

★ **关键认知**：**路径参数永远是必填的**。它参与路由匹配，不可能"缺省"。所以不要给路径参数设默认值，那在语义上不成立。

# 3. 查询参数

## 3.1 默认值与可选

函数里**没有出现在路径中**的普通参数，会被当作查询参数：

```python
@app.get("/items")
def list_items(skip: int = 0, limit: int = 10):
    return {"skip": skip, "limit": limit}
```

访问 `/items?skip=20&limit=5`。带默认值即为可选，不带默认值即为必填：

```python
@app.get("/search")
def search(q: str):            # 必填查询参数
    return {"q": q}
```

缺少 `q` 时返回 422。

## 3.2 可选与默认 None

```python
@app.get("/items")
def list_items(q: str | None = None):
    if q:
        return {"q": q}
    return {"q": "未提供"}
```

`str | None = None` 表示"可以不传"，文档里会标成可选。

⚠ **不要用可变对象做默认值**（如 `tags: list[str] = []`）。Python 的默认参数只求值一次，多个请求会共享同一个列表。正确写法是 `tags: list[str] | None = None`，函数内再初始化。

## 3.3 查询参数校验

```python
from typing import Annotated
from fastapi import Query

@app.get("/items")
def list_items(
    q: Annotated[str | None, Query(min_length=3, max_length=50)] = None,
    limit: Annotated[int, Query(ge=1, le=100)] = 10,
):
    return {"q": q, "limit": limit}
```

`Query` 还支持 `alias`（对外参数名与内部变量名不同）、`deprecated`、`description` 等。

# 4. 请求体

## 4.1 Pydantic 模型作为请求体

```python
from pydantic import BaseModel

class ItemCreate(BaseModel):
    name: str
    price: float
    tags: list[str] = []

@app.post("/items")
def create_item(item: ItemCreate):
    return {"name": item.name, "price": item.price}
```

参数类型是 `BaseModel` 子类时，FastAPI 自动把它当作 **JSON 请求体**解析并校验。

## 4.2 三种参数混用

```python
@app.put("/items/{item_id}")
def update_item(
    item_id: int,                    # 路径
    item: ItemCreate,                # 请求体
    notify: bool = False,            # 查询
):
    return {"item_id": item_id, "item": item, "notify": notify}
```

**判定规则很简单**：名字在路径里 → 路径参数；类型是 Pydantic 模型 → 请求体；其余 → 查询参数（或表单，取决于声明方式）。

★ **关键认知**：一个函数**最多只有一个请求体参数**，因为 HTTP 请求体只有一份。如果需要合并多个模型，用一个外层模型包裹，或者用多个依赖。

# 5. 状态码与响应

## 5.1 设置 status_code

```python
from fastapi import status

@app.post("/items", status_code=status.HTTP_201_CREATED)
def create_item(item: ItemCreate):
    return item
```

| 状态码 | 语义 |
|--------|------|
| 200 | 成功 |
| 201 | 已创建 |
| 204 | 成功但无内容 |
| 400 | 请求错误 |
| 401 | 未认证 |
| 403 | 已认证但无权限 |
| 404 | 资源不存在 |
| 409 | 冲突（如重复创建） |
| 422 | 校验失败 |
| 500 | 服务器内部错误 |

⚠ **`204` 不能返回响应体**。如果声明了 `status_code=204` 却返回了 dict，某些客户端会报错或行为异常。无内容时返回 `None`。

## 5.2 响应模型

```python
class ItemOut(BaseModel):
    id: int
    name: str
    price: float

@app.post("/items", response_model=ItemOut, status_code=201)
def create_item(item: ItemCreate):
    return {"id": 1, "name": item.name, "price": item.price, "secret": "x"}
```

`response_model` 会**过滤掉模型之外的字段**，`secret` 不会出现在响应里。这既是文档，也是安全边界。

# 6. APIRouter 分组

## 6.1 为什么要分组

当接口超过十几个，全部堆在 `main.py` 会难以维护。`APIRouter` 就像"子应用"，可以把一组接口拆到独立文件：

```python
# routers/users.py
from fastapi import APIRouter

router = APIRouter(prefix="/users", tags=["users"])

@router.get("")
def list_users():
    return []

@router.get("/{user_id}")
def get_user(user_id: int):
    return {"user_id": user_id}
```

```python
# main.py
from fastapi import FastAPI
from routers import users

app = FastAPI()
app.include_router(users.router)
```

`prefix` 统一加前缀，`tags` 让文档按组折叠。

★ **关键认知**：`APIRouter` 不是运行时"子服务"，它只是**路由声明的收集器**。`include_router` 时才真正注册到主应用，所以路由顺序以 `include_router` 的调用顺序为准。

# 7. 依赖的第一次登场

依赖（dependency）是"路由函数之前要执行的准备工作"，比如取当前用户、开数据库会话：

```python
from fastapi import Depends

def pagination(skip: int = 0, limit: int = 10):
    return {"skip": skip, "limit": limit}

@app.get("/items")
def list_items(page: dict = Depends(pagination)):
    return page
```

`Depends(pagination)` 会让 FastAPI 先调用 `pagination`，把它解析到的查询参数注入，再把返回值传给 `page`。依赖也可以复用、嵌套、做校验，第 4 篇会专门展开。

# 8. 常见错误

| 问题 | 后果 | 正确做法 |
|------|------|---------|
| 固定路径写在动态路径后 | 被动态路由截获 | 固定路径前置 |
| 查询参数用可变默认值 | 请求间共享状态 | 用 `None` 再初始化 |
| 一个函数写两个请求体参数 | 启动或解析报错 | 合并成一个模型 |
| 路径参数设默认值 | 语义矛盾 | 路径参数永远必填 |
| 认为 404 会覆盖校验错误 | 实际返回 422 | 区分"路径不存在"与"参数不合法" |
| `204` 返回了内容 | 客户端异常 | 无内容返回 `None` |

# 9. 练习

1. 实现 `/articles/{article_id}` 的 GET 与 PUT，GET 返回单篇，PUT 接收 `ArticleUpdate` 请求体并返回更新后的对象，状态码为 200。
2. 给列表接口加 `page`、`size` 两个查询参数，要求 `page >= 1`、`1 <= size <= 100`，用 `Query` 做约束，并在 `/docs` 里验证。
3. 把用户相关接口拆到 `routers/users.py`，用 `APIRouter(prefix="/users", tags=["users"])` 分组，再在 `main.py` 中 `include_router`，确认文档里出现独立分组。

# 10. 小结

| 主题 | 要点 |
|------|------|
| 路由 | 按注册顺序匹配，固定路径在前 |
| 路径参数 | 必填，参与匹配，用 `Path` 约束 |
| 查询参数 | 非路径的普通参数，默认值决定是否可选 |
| 请求体 | Pydantic 模型，一个函数最多一个 |
| 状态码 | 用 `status` 常量，`204` 不返回内容 |
| 分组 | `APIRouter` 收集声明，`include_router` 注册 |
| 依赖 | `Depends` 在函数前执行，结果注入参数 |

下一篇进入 Pydantic 模型，讲清字段校验、嵌套结构、响应模型与序列化，把本篇里"请求体长什么样"这件事彻底讲透。
