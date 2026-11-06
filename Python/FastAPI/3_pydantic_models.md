Pydantic 是 FastAPI 的"数据关口"：进来的请求体先过它，出去的数据也先过它。写 FastAPI 有一半时间在建模型，模型写得好，接口就稳。本篇以 Pydantic 2.x 为准，讲清 `BaseModel`、字段校验、嵌套、响应模型和序列化，顺带把 1.x 到 2.x 的常见迁移坑点掉。

# 1. BaseModel 基础

## 1.1 定义与实例化

```python
from pydantic import BaseModel

class User(BaseModel):
    id: int
    name: str
    age: int = 18          # 有默认值即可选

user = User(id=1, name="alice")
print(user.name)           # alice
print(user.model_dump())   # {'id': 1, 'name': 'alice', 'age': 18}
```

`BaseModel` 在实例化时做三件事：**类型转换、校验、填充默认值**。传 `id="1"` 会被转成 `1`（宽松模式），传 `id="abc"` 抛 `ValidationError`。

★ **关键认知**：Pydantic 的默认行为是**转换优先于拒绝**。字符串 `"42"` 能变成 `42`，这在接表单和 JSON 时很方便，但也意味着"类型对了"不等于"值可信"。业务约束仍要靠 `Field` 和自定义校验。

## 1.2 常用字段类型

| 类型 | 示例 | 说明 |
|------|------|------|
| `str` / `int` / `float` / `bool` | `name: str` | 基础类型 |
| `list[str]` | `tags: list[str]` | 列表元素也会校验 |
| `dict[str, int]` | `scores: dict[str, int]` | 键值都校验 |
| `str \| None` | `nick: str \| None = None` | 可为空 |
| `datetime` / `date` / `time` | `created_at: datetime` | 自动解析 ISO 字符串 |
| `UUID` | `id: UUID` | 自动解析 UUID 字符串 |
| `Enum` | `status: Status` | 限定取值 |
| `EmailStr` | `email: EmailStr` | 需安装 `email-validator` |

```bash
pip install "pydantic[email]"   # 使用 EmailStr 时
```

# 2. Field 与字段约束

## 2.1 Field 常用参数

```python
from typing import Annotated
from pydantic import BaseModel, Field

class Item(BaseModel):
    name: Annotated[str, Field(min_length=1, max_length=50)]
    price: Annotated[float, Field(gt=0, description="单价，必须大于 0")]
    stock: Annotated[int, Field(ge=0, default=0)]
    tags: Annotated[list[str], Field(max_length=10)] = []
```

| 参数 | 作用 |
|------|------|
| `min_length` / `max_length` | 字符串或列表长度 |
| `gt` / `ge` / `lt` / `le` | 数值比较 |
| `pattern` | 正则匹配 |
| `default` | 默认值 |
| `default_factory` | 用工厂函数生成默认值 |
| `alias` | 对外字段名 |
| `description` | 文档描述 |
| `examples` | 文档示例 |

用 `Annotated` 而不是"默认值 + Field"是老少皆宜的写法，可读性更好，也便于复用。

## 2.2 可变默认值要用 default_factory

```python
from pydantic import BaseModel, Field

class Cart(BaseModel):
    items: list[str] = Field(default_factory=list)   # 正确
    # items: list[str] = []                          # 错误示范
```

⚠ **Pydantic 会深拷贝默认值，所以 `= []` 在这里其实不会共享**，但 `default_factory` 仍是更明确的写法，尤其是默认值来自函数调用（如 `datetime.now`）时，必须用工厂，否则所有实例拿到同一个时间。

## 2.3 别名字段

前后端命名风格不一致时：

```python
class User(BaseModel):
    user_id: int = Field(alias="userId")
    model_config = {"populate_by_name": True}    # 允许用字段名或别名填充

u = User(userId=1)          # 用别名
u2 = User(user_id=2)        # populate_by_name=True 时才允许
```

序列化时控制别名：

```python
u.model_dump(by_alias=True)   # {'userId': 1}
```

# 3. 自定义校验

## 3.1 field_validator

```python
from pydantic import BaseModel, field_validator

class User(BaseModel):
    name: str
    email: str

    @field_validator("name")
    @classmethod
    def name_must_not_be_blank(cls, v: str) -> str:
        if not v.strip():
            raise ValueError("名字不能为空白")
        return v.strip()
```

`@field_validator` 可以一次校验多个字段：`@field_validator("name", "email")`。

## 3.2 model_validator

跨字段校验用 `model_validator`：

```python
from typing import Self
from pydantic import BaseModel, model_validator

class DateRange(BaseModel):
    start: int
    end: int

    @model_validator(mode="after")
    def check_order(self) -> Self:
        if self.start >= self.end:
            raise ValueError("开始必须小于结束")
        return self
```

- `mode="before"`：拿到原始输入（dict 等），可做结构改写。
- `mode="after"`：拿到已校验的模型实例，适合跨字段比较。

★ **关键认知**：校验函数里抛 `ValueError` 或 `AssertionError`，Pydantic 会包装成 `ValidationError`；抛其他异常会直接冒泡。所以自定义校验统一用 `ValueError`，FastAPI 才能把它转成 422。

## 3.3 枚举约束

```python
from enum import Enum

class Role(str, Enum):
    admin = "admin"
    user = "user"

class User(BaseModel):
    role: Role = Role.user
```

传入 `"guest"` 会得到 422，文档里自动列出 `admin`、`user`。

# 4. 嵌套模型

## 4.1 模型套模型

```python
class Address(BaseModel):
    city: str
    street: str

class User(BaseModel):
    name: str
    address: Address
    tags: list[str] = []

user = User(
    name="alice",
    address={"city": "上海", "street": "南京路"},   # dict 会自动转模型
)
print(user.address.city)        # 上海
print(type(user.address))       # <class 'Address'>
```

## 4.2 嵌套的校验传递

```python
class Order(BaseModel):
    id: int
    items: list[Item]           # Item 来自上文

# 只要有一个元素不合法，整个请求 422
order = Order(id=1, items=[{"name": "笔", "price": 0}])   # price 必须 > 0，报错
```

错误信息会带上**字段路径**，例如 `items.0.price`，这正是前端定位问题所需。

# 5. 模型配置

## 5.1 ConfigDict

Pydantic 2 用 `model_config` 替代了 1.x 的内部 `class Config`：

```python
from pydantic import BaseModel, ConfigDict

class User(BaseModel):
    model_config = ConfigDict(
        extra="forbid",            # 拒绝未知字段
        str_strip_whitespace=True, # 自动去首尾空格
        frozen=True,               # 不可变
        from_attributes=True,      # 支持从对象属性读取（ORM 常用）
    )

    id: int
    name: str
```

| 配置 | 作用 |
|------|------|
| `extra="forbid"` | 出现未声明字段直接报错 |
| `extra="ignore"` | 忽略未知字段（默认） |
| `str_strip_whitespace` | 自动裁剪字符串空白 |
| `frozen` | 实例不可修改 |
| `from_attributes` | 允许 `model_validate(obj)` 读对象属性 |
| `populate_by_name` | 允许用字段名填充别名字段 |

⚠ **`extra` 的默认值是 `"ignore"`**。客户端把 `emial` 拼错成 `e-mail`，服务端会静默丢弃，字段变空。对契约要求高的接口，建议显式设 `extra="forbid"`。

# 6. 响应模型与序列化

## 6.1 response_model 的作用

```python
class UserOut(BaseModel):
    id: int
    name: str

@app.get("/users/{user_id}", response_model=UserOut)
def get_user(user_id: int):
    return {"id": user_id, "name": "alice", "password_hash": "xxx"}
```

响应里**不会出现 `password_hash`**。`response_model` 同时承担三个职责：

1. **过滤**：只输出声明的字段。
2. **校验**：返回数据不符合模型会报错（开发期能及早发现）。
3. **文档**：OpenAPI 里描述响应结构。

★ **关键认知**：`response_model` 是**输出侧的安全边界**。内部模型可以有很多字段，对外模型只暴露必要字段。不要图省事直接把数据库模型当响应模型返回。

## 6.2 序列化方法

| 方法 | 作用 |
|------|------|
| `model_dump()` | 转成 dict |
| `model_dump_json()` | 转成 JSON 字符串 |
| `model_validate(obj)` | 从 dict/对象构造模型 |
| `model_validate_json(s)` | 从 JSON 字符串构造 |
| `model_copy(update={...})` | 复制并覆盖部分字段 |

```python
user = User(id=1, name="alice")
data = user.model_dump()                  # dict
data_exclude = user.model_dump(exclude={"id"})
json_str = user.model_dump_json()
```

`model_dump` 支持 `exclude`、`include`、`by_alias`、`exclude_none` 等参数，做响应裁剪时很实用。

# 7. 常见错误

| 问题 | 后果 | 正确做法 |
|------|------|---------|
| 用 Pydantic 1 的 `@validator` | 2.x 下报错或失效 | 改用 `@field_validator` |
| 用 `class Config` | 2.x 仍可用但已过时 | 改用 `model_config = ConfigDict(...)` |
| `.dict()` / `.json()` | 2.x 弃用告警 | `model_dump()` / `model_dump_json()` |
| 返回 ORM 对象无 `from_attributes` | 校验失败 | 开启 `from_attributes=True` |
| 默认值写 `datetime.now()` | 所有实例同一时间 | 用 `default_factory=datetime.now` |
| 忘记 `extra="forbid"` | 拼错字段被静默忽略 | 契约严格时显式设置 |

# 8. 练习

1. 定义 `Product` 模型：`name`（1-50 字）、`price`（大于 0）、`tags`（最多 5 个）、`created_at`（默认当前时间）。用 `field_validator` 保证 `name` 不为空白。
2. 定义 `Order` 模型，包含 `items: list[Product]`，用 `model_validator` 校验总价不超过 10000，超出时抛出中文错误。
3. 把 `Product` 拆成 `ProductCreate`（输入）与 `ProductOut`（输出，含 `id`），在路由里用 `response_model=ProductOut`，确认内部字段不会泄露。

# 9. 小结

| 主题 | 要点 |
|------|------|
| BaseModel | 实例化即校验，默认宽松转换 |
| Field | 长度、范围、正则、别名、默认值 |
| 校验 | `field_validator` 管单字段，`model_validator` 管跨字段 |
| 嵌套 | 模型可嵌套，错误带字段路径 |
| 配置 | `ConfigDict` 控制 extra、frozen、from_attributes |
| 响应模型 | 过滤 + 校验 + 文档，是输出安全边界 |
| 序列化 | `model_dump` / `model_dump_json`，支持裁剪 |

下一篇进入依赖注入，讲清 `Depends` 的解析机制、类依赖、子依赖、`yield` 依赖的资源管理，以及如何在测试里替换依赖。
