真实业务很少只有一张表。用户有文章、文章有评论、文章和标签多对多，这些关系决定了查询怎么写、删除怎么级联、性能问题从哪里冒出来。SQLAlchemy 的关系映射有一条主线：**外键落在数据库，`relationship` 只负责对象导航，加载策略决定发几条 SQL**。

# 1. 关系的基本模型

## 1.1 一对多

外键放在"多"的一侧：

```python
from sqlalchemy import ForeignKey, String
from sqlalchemy.orm import Mapped, mapped_column, relationship


class User(Base):
    __tablename__ = "users"

    id: Mapped[int] = mapped_column(primary_key=True)
    name: Mapped[str] = mapped_column(String(50))
    posts: Mapped[list["Post"]] = relationship(back_populates="author")


class Post(Base):
    __tablename__ = "posts"

    id: Mapped[int] = mapped_column(primary_key=True)
    title: Mapped[str] = mapped_column(String(100))
    author_id: Mapped[int] = mapped_column(ForeignKey("users.id"))
    author: Mapped["User"] = relationship(back_populates="posts")
```

```text
users                          posts
┌────┬───────┐               ┌────┬────────┬───────────┐
│ id │ name  │               │ id │ title  │ author_id │
├────┼───────┤               ├────┼────────┼───────────┤
│ 1  │ alice │◀──────────────│ 1  │ 第一篇 │ 1         │
│ 2  │ bob   │◀──────────────│ 2  │ 第二篇 │ 1         │
└────┴───────┘               └────┴────────┴───────────┘
                                    外键在多的一方
```

★ **关键认知**：`Mapped[list["Post"]]` 告诉 ORM 这是集合，`Mapped["User"]` 告诉 ORM 这是单个对象。**关系的方向由注解的容器类型推断**，不需要再写 `uselist`。但外键列仍必须在数据库里显式声明，`relationship` 不会替你建外键。

## 1.2 双向关系与 back_populates

只写一侧也能工作，但两侧都写 `back_populates` 才能保证内存中的对象图同步：

```python
post = Post(title="新文章", author=user)
print(user.posts)          # 新文章已经出现在集合里
```

没有 `back_populates` 时，给 `post.author` 赋值不会自动维护 `user.posts`，同一会话里可能出现"关系一边有一边没有"的怪现象。

| 写法 | 双向同步 | 推荐 |
|------|---------|------|
| 单侧 `relationship` | 否 | 只在确实不需要反向导航时 |
| `back_populates` | 是 | 默认选择 |
| `backref` | 是 | 旧风格，2.x 仍可用但不推荐 |

## 1.3 一对一

把"多"的一侧改成标量注解，并给外键加唯一约束：

```python
class User(Base):
    __tablename__ = "users"

    id: Mapped[int] = mapped_column(primary_key=True)
    profile: Mapped["Profile | None"] = relationship(back_populates="user")


class Profile(Base):
    __tablename__ = "profiles"

    id: Mapped[int] = mapped_column(primary_key=True)
    user_id: Mapped[int] = mapped_column(ForeignKey("users.id"), unique=True)
    bio: Mapped[str | None]
    user: Mapped["User"] = relationship(back_populates="profile")
```

⚠ **唯一约束不能省**。没有 `unique=True`，数据库允许同一个用户挂多份资料，ORM 加载标量关系时会查到多行。

## 1.4 多对多

多对多需要一张中间表，通常用 Core 的 `Table` 声明：

```python
from sqlalchemy import Column, Table

post_tags = Table(
    "post_tags",
    Base.metadata,
    Column("post_id", ForeignKey("posts.id", ondelete="CASCADE"), primary_key=True),
    Column("tag_id", ForeignKey("tags.id", ondelete="CASCADE"), primary_key=True),
)


class Post(Base):
    __tablename__ = "posts"

    id: Mapped[int] = mapped_column(primary_key=True)
    tags: Mapped[list["Tag"]] = relationship(
        secondary=post_tags, back_populates="posts"
    )


class Tag(Base):
    __tablename__ = "tags"

    id: Mapped[int] = mapped_column(primary_key=True)
    name: Mapped[str] = mapped_column(String(30), unique=True)
    posts: Mapped[list["Post"]] = relationship(
        secondary=post_tags, back_populates="posts"
    )
```

中间表的主键由两个外键组成，天然防止重复关联。**如果中间表还要存额外字段**（如排序、创建时间），就应该把它定义成独立模型，用两个一对多替代 `secondary`。

# 2. 加载策略

## 2.1 lazy 的几种取值

```python
posts: Mapped[list["Post"]] = relationship(
    back_populates="author", lazy="select"
)
```

| 取值 | 行为 | 发 SQL 时机 |
|------|------|------------|
| `select`（默认） | 访问属性时单独查询 | 首次访问 |
| `selectin` | 用 `IN` 批量加载 | 首次访问，但一次查完 |
| `joined` | 主查询 LEFT JOIN | 主查询时 |
| `subquery` | 子查询批量加载 | 主查询后 |
| `raise` | 禁止懒加载 | 访问即报错 |
| `noload` | 永远返回空 | 不查询 |

## 2.2 N+1 是怎么发生的

```python
users = session.scalars(select(User)).all()      # 1 条
for user in users:
    print(user.posts)                            # 每个用户 1 条
```

```text
N+1 的执行过程（N=100）：
1 条   SELECT * FROM users;
100 条 SELECT * FROM posts WHERE author_id = 1;
                            WHERE author_id = 2;
                            ...
                            WHERE author_id = 100;
```

★ **关键认知**：N+1 不会报错，只会在数据量上来后突然变慢。**看到循环里访问关系属性，就要警觉**。最有效的修复不是改循环，而是把加载策略提前到查询上。

## 2.3 selectinload：集合的首选

```python
from sqlalchemy.orm import selectinload

stmt = select(User).options(selectinload(User.posts))
users = session.scalars(stmt).all()
```

实际执行两条 SQL：

```sql
SELECT * FROM users;
SELECT * FROM posts WHERE author_id IN (1, 2, 3, ...);
```

优点是不会因为一对多 JOIN 导致主表行膨胀，分页安全。**集合关系优先用它**。

## 2.4 joinedload：多对一的首选

```python
from sqlalchemy.orm import joinedload

stmt = select(Post).options(joinedload(Post.author))
posts = session.scalars(stmt).all()
```

只发一条带 LEFT JOIN 的 SQL，适合"多对一"或"一对一"。⚠ 对**集合**使用 `joinedload` 时，主表行会被关联行放大，再叠加 `LIMIT` 就会得到错误的条数。

## 2.5 选择策略

| 场景 | 推荐 | 原因 |
|------|------|------|
| 查列表并展示集合 | `selectinload` | 两条 SQL，分页安全 |
| 多对一展示父对象 | `joinedload` | 一条 SQL，无行放大 |
| 只需要过滤、不取关联 | 显式 `join` | 不加载对象，最省内存 |
| 深层嵌套 | `selectinload(A).selectinload(A.B)` | 每层一条 `IN` |
| 防止意外懒加载 | `raiseload("*")` | 开发期暴露 N+1 |

```python
stmt = select(User).options(
    selectinload(User.posts).selectinload(Post.comments)
)
```

测试环境开启 `raiseload("*")` 后，任何未预加载的关系访问都会直接抛错，能在 CI 阶段拦住新写的 N+1。

# 3. 级联

## 3.1 cascade 的常见取值

```python
posts: Mapped[list["Post"]] = relationship(
    back_populates="author",
    cascade="all, delete-orphan",
)
```

| 取值 | 含义 |
|------|------|
| `save-update` | 父对象加入会话时，子对象一起加入（默认） |
| `delete` | 删除父对象时删除子对象 |
| `delete-orphan` | 子对象脱离父集合时删除 |
| `merge` | 合并父对象时递归合并子对象 |
| `all` | 上述除 `delete-orphan` 外的组合 |

★ **关键认知**：`all` **不包含** `delete-orphan`。想让"从集合里移除即删除"生效，必须显式写 `cascade="all, delete-orphan"`。

## 3.2 delete-orphan 的语义

```python
user.posts.remove(post)     # post 成为孤儿
session.commit()            # 生成 DELETE FROM posts WHERE id = ?
```

它只适用于一对多和一对一，且子对象必须只有一个父对象。多对多的中间表关系不能用它。

## 3.3 passive_deletes 与数据库级联

```python
class User(Base):
    __tablename__ = "users"

    id: Mapped[int] = mapped_column(primary_key=True)
    posts: Mapped[list["Post"]] = relationship(
        back_populates="author",
        cascade="all, delete-orphan",
        passive_deletes=True,
    )


class Post(Base):
    __tablename__ = "posts"

    id: Mapped[int] = mapped_column(primary_key=True)
    author_id: Mapped[int] = mapped_column(
        ForeignKey("users.id", ondelete="CASCADE")
    )
```

`passive_deletes=True` 表示"删除父对象时不要先把子对象全部加载出来"，交给数据库的 `ON DELETE CASCADE` 处理。**这是大表删除的正确姿势**：否则 ORM 会先 SELECT 出几万条子记录，再逐条 DELETE。

| 方式 | 触发条件 | 优点 | 缺点 |
|------|---------|------|------|
| ORM `cascade="delete"` | 通过 Session 删除 | 可触发 ORM 事件 | 需要加载子对象，慢 |
| 数据库 `ON DELETE CASCADE` | 任何 DELETE | 快、原子、绕过 ORM 也生效 | 不在 ORM 事件体系内 |
| 应用层手动删除 | 业务代码 | 完全可控 | 容易漏 |

★ **关键认知**：两者**不要盲目叠加**。同时开启 ORM 级联和数据库级联通常不会出错，但会产生多余的 SELECT。大表场景推荐：**数据库负责 `ON DELETE CASCADE`，ORM 侧用 `passive_deletes=True` 避免预加载**。

# 4. 常见错误

| 现象 | 原因 | 解决 |
|------|------|------|
| 关联属性全是空 | 忘了预加载，或会话已关闭 | `selectinload` / `joinedload` |
| 列表页越翻越慢 | N+1 | 查询时统一预加载 |
| 分页条数不对 | 对集合用了 `joinedload` + `LIMIT` | 改 `selectinload` 或子查询分页 |
| 删除父对象报外键错误 | 没有级联策略 | 配 `ondelete` 或 ORM cascade |
| 删除大对象极慢 | ORM 逐条删除子对象 | `passive_deletes=True` |
| 一对一加载到多行 | 外键缺唯一约束 | 加 `unique=True` |
| 多外键关系配置报错 | 未指定 `foreign_keys` | 每个关系显式指明列 |

# 5. 动手练习

1. 建立 `User`、`Post`、`Comment` 三层关系，用 `selectinload` 一次查出"用户 + 文章 + 每篇文章的评论"，打印实际 SQL 条数。
2. 给 `Post` 和 `Tag` 建多对多，完成"给文章加标签""移除标签"，观察中间表在 flush 前后的变化。
3. 把关系默认加载改成 `lazy="raise"`，运行现有测试，列出所有触发懒加载的位置，再逐个补上预加载。

# 6. 小结

| 主题 | 要点 |
|------|------|
| 一对多 | 外键在多的一侧，集合用 `Mapped[list[T]]` |
| 双向 | `back_populates` 保证内存对象图一致 |
| 多对多 | `secondary` 指向中间表，中间表用复合主键防重 |
| 加载 | 集合用 `selectinload`，多对一用 `joinedload` |
| N+1 | 循环里访问关系属性就是信号 |
| 级联 | `all` 不含 `delete-orphan`，大表用数据库级联 + `passive_deletes` |
| 分页 | 集合关系不要用 `joinedload` 配 `LIMIT` |

下一篇进入事务，讲清楚 `session.begin()` 的边界、嵌套事务与 SAVEPOINT 的真实语义、隔离级别的选择，以及并发扣库存时悲观锁与乐观锁分别怎么写。
