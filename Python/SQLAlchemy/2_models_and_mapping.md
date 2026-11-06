模型是 ORM 的地基。字段类型选错，迁移会反复改表；可空性写错，业务层就要处处判空；索引漏建，查询上线后才暴露。SQLAlchemy 2.x 用 `Mapped` 注解加 `mapped_column` 把"Python 类型"和"数据库列"绑在一起，让类型检查器和数据库约束尽量对齐。

# 1. 声明式映射的骨架

## 1.1 定义 Base

```python
from sqlalchemy.orm import DeclarativeBase


class Base(DeclarativeBase):
    pass
```

`DeclarativeBase` 是所有映射类的父类。它内部持有一个 `MetaData` 对象，负责收集所有子类声明的表。**一个项目通常只定义一个 Base**，多个 Base 会导致 `create_all` 和 Alembic 看不到彼此的表。

## 1.2 类是怎样变成表的

```python
class User(Base):
    __tablename__ = "users"

    id: Mapped[int] = mapped_column(primary_key=True)
    name: Mapped[str]
```

这段代码发生的事：

```text
1. 类体执行，收集 Mapped 注解与 mapped_column
2. 元类把类注册进 Base.metadata
3. 生成 Table 对象，列名默认取属性名
4. 生成 InstrumentedAttribute，支持 User.name == "alice" 这样的表达式
5. Base.metadata.tables["users"] 里出现完整的表定义
```

★ **关键认知**：`User.name == "alice"` 返回的不是布尔值，而是一个 SQL 表达式对象。这正是 ORM 能把 Python 比较运算翻译成 `WHERE name = :name` 的原因。**不要在普通 `if` 里直接用它判真假**，那会得到一个恒为真的表达式对象。

```python
# 错误：这个 if 永远为真
if User.name == "alice":
    ...

# 正确：交给数据库判断
stmt = select(User).where(User.name == "alice")
```

# 2. Mapped 与 mapped_column

## 2.1 注解决定列类型

```python
from datetime import datetime
from decimal import Decimal
from sqlalchemy import String, Numeric, DateTime
from sqlalchemy.orm import Mapped, mapped_column


class Product(Base):
    __tablename__ = "products"

    id: Mapped[int] = mapped_column(primary_key=True)
    name: Mapped[str] = mapped_column(String(100))
    price: Mapped[Decimal] = mapped_column(Numeric(10, 2))
    stock: Mapped[int] = mapped_column(default=0)
    created_at: Mapped[datetime] = mapped_column(DateTime(timezone=True))
```

常见映射关系：

| Python 注解 | 默认 SQL 类型 | 说明 |
|-------------|--------------|------|
| `int` | `Integer` | 主键默认自增 |
| `str` | `String` / `VARCHAR` | 建议显式给长度 |
| `bool` | `Boolean` | SQLite 存 0/1 |
| `float` | `Float` | 不适合金额 |
| `Decimal` | `Numeric` | 金额应使用它 |
| `datetime` | `DateTime` | 建议加 `timezone=True` |
| `date` | `Date` | 只有日期 |
| `bytes` | `LargeBinary` | 二进制 |
| `dict` / `list` | `JSON` | 需显式指定 `JSON` 类型 |

★ **关键认知**：**金额不要用 `float`**。二进制浮点无法精确表示 `0.1`，累加后会出现分位误差。用 `Numeric(10, 2)` 映射到 `Decimal`，数据库和 Python 两侧都保持定点语义。

## 2.2 可空性由注解推导

```python
class Article(Base):
    __tablename__ = "articles"

    id: Mapped[int] = mapped_column(primary_key=True)
    title: Mapped[str]                       # NOT NULL
    subtitle: Mapped[str | None]             # 允许 NULL
    views: Mapped[int] = mapped_column(default=0)
```

`Mapped[str]` 生成 `nullable=False`，`Mapped[str | None]` 生成 `nullable=True`。这是 2.x 最有价值的改进之一：**类型提示和数据库约束不再各说各话**。

⚠ 如果写 `Mapped[str]` 却传了 `None`，SQLAlchemy 不会在 Python 层拦截，而是等到 flush 时由数据库报 `NOT NULL constraint failed`。想让错误更早暴露，可以在应用层用 Pydantic 或显式校验。

## 2.3 mapped_column 的常用参数

| 参数 | 作用 | 示例 |
|------|------|------|
| `primary_key` | 声明主键 | `mapped_column(primary_key=True)` |
| `nullable` | 覆盖注解推导 | `mapped_column(nullable=False)` |
| `default` | Python 侧默认值 | `default=0`、`default=list` |
| `server_default` | DDL 默认值 | `server_default=func.now()` |
| `onupdate` | ORM 更新时赋值 | `onupdate=func.now()` |
| `index` | 建单列索引 | `index=True` |
| `unique` | 建唯一约束 | `unique=True` |
| `comment` | 列注释 | `comment="用户邮箱"` |
| `name` | 数据库列名 | `name="user_name"` |

# 3. 主键设计

## 3.1 自增整数

```python
id: Mapped[int] = mapped_column(primary_key=True, autoincrement=True)
```

优点是短、索引友好、插入顺序与主键顺序一致。缺点是**暴露业务规模**，且分库分表时难以保证全局唯一。

## 3.2 UUID

```python
import uuid
from sqlalchemy import Uuid

id: Mapped[uuid.UUID] = mapped_column(
    Uuid, primary_key=True, default=uuid.uuid4
)
```

UUID 适合分布式生成、需要隐藏自增量的场景。代价是占用空间更大、随机写入会让 B-tree 索引页分裂更频繁。PostgreSQL 可用 `uuid_generate_v4()` 作为 `server_default`，但要注意扩展是否已安装。

## 3.3 复合主键与代理键

多个字段同时 `primary_key=True` 即组成复合主键，中间表、明细表常用这种设计。

★ **关键认知**：**主键应选择"永不改变"的值**。邮箱、手机号、用户名都可能变，一旦作为主键，所有外键都要跟着改。惯例是用自增或 UUID 做代理主键，把业务唯一性放到 `unique=True` 的列上。

# 4. 约束与索引

## 4.1 单列索引与唯一约束

```python
class User(Base):
    __tablename__ = "users"

    id: Mapped[int] = mapped_column(primary_key=True)
    email: Mapped[str] = mapped_column(String(120), unique=True, index=True)
    name: Mapped[str] = mapped_column(String(50), index=True)
```

`unique=True` 会创建唯一约束（多数数据库同时创建唯一索引）；`index=True` 只创建普通索引。**不要给低选择性的列单独建索引**，例如性别、状态只有几个值时，全表扫描往往更快。

## 4.2 复合索引与列顺序

```python
from sqlalchemy import Index

class Order(Base):
    __tablename__ = "orders"

    id: Mapped[int] = mapped_column(primary_key=True)
    user_id: Mapped[int]
    status: Mapped[str] = mapped_column(String(20))
    created_at: Mapped[datetime]

    __table_args__ = (
        Index("ix_orders_user_status_created", "user_id", "status", "created_at"),
    )
```

复合索引遵循**最左前缀**原则：`(user_id, status, created_at)` 能服务 `WHERE user_id = ?`、`WHERE user_id = ? AND status = ?`，但**不能**高效服务只查 `status` 的语句。

⚠ 把区分度最高的列放在最左边通常更有利，但最终要看真实查询。索引不是越多越好：每个索引都会增加写入成本和存储占用。

## 4.3 检查约束

```python
from sqlalchemy import CheckConstraint

class Product(Base):
    __tablename__ = "products"

    id: Mapped[int] = mapped_column(primary_key=True)
    price: Mapped[Decimal] = mapped_column(Numeric(10, 2))

    __table_args__ = (
        CheckConstraint("price >= 0", name="price_non_negative"),
    )
```

约束在数据库层生效，能挡住绕过 ORM 的写入。**业务规则如果能用约束表达，就尽量下沉到数据库**。

## 4.4 外键

```python
from sqlalchemy import ForeignKey

class Order(Base):
    __tablename__ = "orders"

    id: Mapped[int] = mapped_column(primary_key=True)
    user_id: Mapped[int] = mapped_column(
        ForeignKey("users.id", ondelete="CASCADE"), index=True
    )
```

外键列**建议手动加索引**。很多数据库不会自动为外键创建索引，而删除主表记录、按外键查询都会用到它。

## 4.5 命名约定

自动生成的约束名在不同数据库下不一致，迁移回滚时容易找不到目标。建议在 Base 上统一命名：

```python
from sqlalchemy import MetaData

convention = {
    "ix": "ix_%(column_0_label)s",
    "uq": "uq_%(table_name)s_%(column_0_name)s",
    "ck": "ck_%(table_name)s_%(constraint_name)s",
    "fk": "fk_%(table_name)s_%(column_0_name)s_%(referred_table_name)s",
    "pk": "pk_%(table_name)s",
}


class Base(DeclarativeBase):
    metadata = MetaData(naming_convention=convention)
```

★ **关键认知**：命名约定必须在**第一张表创建之前**定好。等到线上已有几十张表再改，Alembic 会认为所有约束都要删除重建，生成一份无法安全执行的迁移。

# 5. 默认值与时间戳
## 5.1 default 与 server_default

```python
from sqlalchemy import func, text

class Post(Base):
    __tablename__ = "posts"

    id: Mapped[int] = mapped_column(primary_key=True)
    status: Mapped[str] = mapped_column(default="draft")             # Python 侧
    created_at: Mapped[datetime] = mapped_column(server_default=func.now())  # 数据库侧
    slug: Mapped[str] = mapped_column(server_default=text("''"))    # DDL 常量
```

| 方式 | 何时生效 | 能否被原生 SQL 使用 | 迁移里是否出现 |
|------|---------|-------------------|---------------|
| `default` | ORM 插入时 | 不能 | 不出现 |
| `server_default` | 数据库插入时 | 能 | 出现 |

★ **关键认知**：如果数据可能由其他服务、批处理或手工 SQL 写入，**关键默认值必须用 `server_default`**。只写 `default` 时，绕过 ORM 的 INSERT 会得到 NULL 或直接失败。

⚠ `default=list`、`default=dict` 这类可调用对象要传**函数本身**，不要写成 `default=[]`。后者会在所有实例间共享同一个可变对象。`onupdate` 则是 ORM 层行为，只有通过 ORM 更新对象时才触发，原生 SQL 的 `UPDATE` 不会自动改它。

# 6. 表名与 schema
## 6.1 __tablename__

```python
class UserProfile(Base):
    __tablename__ = "user_profiles"
```

不写 `__tablename__` 时，2.x 不会自动推导，必须显式声明。命名建议：**小写、下划线、复数**，与多数 Python 项目的表命名习惯一致。

PostgreSQL 中可以用 `__table_args__ = {"schema": "audit"}` 把表放进不同 schema。注意设置后，所有查询和迁移都会带上 schema 前缀，连接用户的搜索路径也要配置正确。

# 7. 常见错误
| 现象 | 原因 | 解决 |
|------|------|------|
| `Mapped[str]` 插入 NULL 才报错 | 注解只生成 DDL，不做运行时校验 | 应用层校验或改注解为 `str | None` |
| 迁移反复删除重建约束 | 未设命名约定 | 尽早配置 `naming_convention` |
| 金额出现 0.01 误差 | 用了 `Float` | 改 `Numeric` + `Decimal` |
| 时间少了 8 小时 | 未使用带时区类型 | `DateTime(timezone=True)`，统一存 UTC |
| 外键删除报错 | 未设 `ondelete` 或数据库未建约束 | 明确级联策略并生成迁移 |
| 默认值在原生插入时失效 | 只写了 `default` | 关键字段改用 `server_default` |

# 8. 动手练习

1. 设计 `User`、`Article`、`Comment` 三张表：用户有唯一邮箱，文章有状态与发布时间，评论记录作者与所属文章。为常用查询设计至少两个复合索引，并说明列顺序的理由。
2. 把 `Article` 的金额字段改成 `Numeric(12, 2)`，再故意用 `float` 写入 `0.1 + 0.2`，对比读回的结果，解释差异来源。
3. 给 Base 配置命名约定，然后用 Alembic 生成一次初始迁移，检查 `upgrade()` 里的约束名是否可读、可回滚。

# 9. 小结

| 主题 | 要点 |
|------|------|
| Base | 全项目唯一，持有 MetaData，收集所有表 |
| Mapped | 注解决定类型与可空性，`str | None` 才允许 NULL |
| 主键 | 选永不变化的值，业务唯一性放 unique 列 |
| 索引 | 复合索引看最左前缀，低选择性列不要单独建 |
| 默认值 | 关键字段用 `server_default`，`onupdate` 只对 ORM 生效 |
| 命名约定 | 必须在建表前定好，否则迁移不可控 |

下一篇进入 Session 与 CRUD，讲清楚会话为什么是"工作单元"、`flush` 与 `commit` 的分工、对象在 transient / pending / persistent / detached 之间怎么流转，以及批量写入时怎样避免逐条 INSERT 的性能陷阱。
