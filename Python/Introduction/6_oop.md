Python 的面向对象比 Java 轻得多：没有接口、没有访问修饰符、支持多继承。灵活的另一面是更多约定和陷阱——类属性被实例共享、继承顺序靠 MRO、魔术方法写错就出诡异行为。本篇从"类与实例"讲到 `dataclass`，把 OOP 的核心机制讲清楚。

# 1. 类与实例

## 1.1 定义类与 self

```python
class User:
    def __init__(self, name: str, age: int):
        self.name = name
        self.age = age

    def greet(self) -> str:
        return f"你好，我是 {self.name}"

u = User("张三", 20)
print(u.greet())    # 你好，我是 张三
```

★ **关键认知**：`self` 不是关键字，只是**约定俗成的第一个参数名**。`u.greet()` 在底层等价于 `User.greet(u)`——实例被自动作为第一个参数传入。这解释了为什么定义要写 `self`，调用时却不用传。

`__init__` 不是构造函数（对象已由 `__new__` 创建），而是**初始化方法**，负责设置属性。

## 1.2 类属性与实例属性

```python
class User:
    species = "人类"        # 类属性，所有实例共享

    def __init__(self, name):
        self.name = name    # 实例属性，每个实例独有

a = User("张三")
b = User("李四")
a.species = "外星人"       # 给 a 新建实例属性，遮蔽类属性
print(a.species)           # 外星人
print(b.species)           # 人类，b 不受影响
print(User.species)        # 人类，类属性本身没变
```

★ **关键认知**：读属性时先找实例字典，再找类；**赋值时只要实例上不存在，就会在实例上新建**。所以 `a.species = ...` 只影响 `a`。想改所有实例共享的值，必须写 `User.species = ...`。

⚠ **可变类属性是最隐蔽的共享陷阱**：

```python
class Team:
    members = []            # 错误：所有实例共享同一个列表

    def add(self, name):
        self.members.append(name)

t1, t2 = Team(), Team()
t1.add("张三")
print(t2.members)           # ['张三']，t2 也"有了"张三

class Team:
    def __init__(self):
        self.members = []   # 正确：每个实例独立

    def add(self, name):
        self.members.append(name)
```

# 2. 三种方法

```python
class User:
    def __init__(self, name, age):
        self.name = name
        self.age = age

    def greet(self):                    # 实例方法：第一个参数 self
        return f"我是 {self.name}"

    @classmethod
    def from_string(cls, text):         # 类方法：第一个参数 cls
        name, age = text.split(",")
        return cls(name, int(age))      # 用 cls 让子类也能正确构造

    @staticmethod
    def is_adult(age):                  # 静态方法：无隐式参数
        return age >= 18

u = User.from_string("张三,20")
print(u.age, User.is_adult(20))         # 20 True
```

| 方法类型 | 第一个参数 | 访问实例 | 访问类 | 典型用途 |
|---------|-----------|---------|--------|---------|
| 实例方法 | `self` | 是 | 是 | 绝大多数行为 |
| 类方法 | `cls` | 否 | 是 | 工厂方法 |
| 静态方法 | 无 | 否 | 否 | 与类相关的工具函数 |

★ **关键认知**：类方法里用 `cls(...)` 而不是硬编码 `User(...)`，这样**子类调用时会构造子类实例**，是工厂方法能正确支持继承的关键。

# 3. 继承与组合

## 3.1 继承与 super

```python
class Animal:
    def __init__(self, name):
        self.name = name

    def speak(self):
        return "..."


class Dog(Animal):
    def __init__(self, name, breed):
        super().__init__(name)      # 调用父类初始化
        self.breed = breed

    def speak(self):
        return f"{super().speak()} 汪汪"

print(Dog("旺财", "柴犬").speak())   # ... 汪汪
```

⚠ **不要写 `super(Animal, self)` 这种旧式写法**，直接 `super()` 即可。也不要忘记 `super().__init__()`，否则父类属性不会设置，报错往往出现在很远的地方。

## 3.2 MRO

多继承时，Python 用 **C3 线性化**确定查找顺序，结果存在 `__mro__`：

```python
class A:
    def hello(self): return "A"

class B(A):
    def hello(self): return "B"

class C(A):
    def hello(self): return "C"

class D(B, C):
    pass

print(D.__mro__)     # (D, B, C, A, object)
print(D().hello())   # B
```

★ **关键认知**：`super()` 在 MRO 里不是"调用父类"，而是"**调用 MRO 中的下一个类**"。MRO 保证子类先于父类、同层按声明顺序，也不会重复调用菱形继承的公共祖先。

## 3.3 组合优于继承

继承表达 "is-a"，组合表达 "has-a"。只是复用功能时，用组合更清晰：

```python
class Engine:
    def start(self):
        return "引擎启动"

class Car:
    def __init__(self):
        self.engine = Engine()      # 组合

    def start(self):
        return self.engine.start()
```

| 维度 | 继承 | 组合 |
|------|------|------|
| 关系 | is-a | has-a |
| 耦合 | 强 | 弱 |
| 灵活 | 父类改动影响子类 | 运行时可替换组件 |
| 建议 | 层级浅、语义明确时用 | 默认优先 |

# 4. 魔术方法

## 4.1 __str__ 与 __repr__

```python
class User:
    def __init__(self, name, age):
        self.name = name
        self.age = age

    def __str__(self):
        return f"{self.name}（{self.age} 岁）"       # 给人看

    def __repr__(self):
        return f"User(name={self.name!r}, age={self.age!r})"  # 给开发者看

u = User("张三", 20)
print(u)          # 张三（20 岁）
print(repr(u))    # User(name='张三', age=20)
```

★ **关键认知**：`print` 和 f-string 优先用 `__str__`，容器打印和调试器用 `__repr__`。**只定义 `__repr__` 时，`__str__` 会自动复用它**，反过来不成立。所以至少实现 `__repr__`。

## 4.2 __eq__ 与 __hash__

```python
class Point:
    def __init__(self, x, y):
        self.x = x
        self.y = y

    def __eq__(self, other):
        if not isinstance(other, Point):
            return NotImplemented
        return self.x == other.x and self.y == other.y

    def __hash__(self):
        return hash((self.x, self.y))

print(Point(1, 2) == Point(1, 2))        # True
print(len({Point(1, 2), Point(1, 2)}))   # 1
```

⚠ **定义了 `__eq__` 却忘了 `__hash__`，实例会变成不可哈希**，不能放进 `set` 或当字典键。反过来，可变对象不要同时定义值相等和哈希——修改后哈希值变化会导致字典查找失败。

## 4.3 运算符重载

常用魔术方法：`__len__`（`len()`）、`__getitem__`（`obj[key]`）、`__iter__`（`for`）、`__enter__`/`__exit__`（`with`）。

# 5. 属性与封装

```python
class Circle:
    def __init__(self, radius):
        self._radius = radius

    @property
    def radius(self):
        return self._radius

    @radius.setter
    def radius(self, value):
        if value < 0:
            raise ValueError("半径不能为负")
        self._radius = value

    @property
    def area(self):
        return 3.14159 * self._radius ** 2
```

★ **关键认知**：`property` 让"属性访问"和"方法调用"外观统一——**外部写 `obj.x`，内部可随时从普通属性升级为带校验的计算属性，而不用改调用方**。这是 Python 式封装的精髓：靠约定和按需升级，而不是访问修饰符。

| 写法 | 含义 |
|------|------|
| `name` | 公开 |
| `_name` | 受保护，"请不要直接访问" |
| `__name` | 名称改写为 `_ClassName__name`，避免子类冲突 |

⚠ **`__name` 不是真正的私有**，只是名称改写，`a._A__secret` 照样能访问。

# 6. dataclass

```python
from dataclasses import dataclass, field

@dataclass
class User:
    name: str
    age: int = 0
    tags: list[str] = field(default_factory=list)   # 可变默认值必须用 factory
```

常用参数：`frozen=True`（不可变、可哈希）、`order=True`（可排序）、`slots=True`（省内存）、`kw_only=True`（仅关键字传参）。

⚠ **`dataclass` 的可变默认值必须用 `field(default_factory=...)`**，直接写 `tags: list = []` 会抛 `ValueError`。原因和可变默认参数一样：默认值只求值一次，会被所有实例共享。

★ **关键认知**：`dataclass` 适合"主要是数据载体"的类。如果类里有大量行为、复杂状态机，手写 `__init__` 或普通类更合适。

# 7. 常见错误

| 错误 | 现象 | 正确做法 |
|------|------|---------|
| 可变类属性 | 实例间共享列表 | 在 `__init__` 里创建 |
| 忘了 `super().__init__()` | 父类属性缺失 | 显式调用 |
| `__eq__` 无 `__hash__` | 不可哈希 | 补 `__hash__` 或用 `frozen` |
| 只写 `__str__` | 调试信息不清晰 | 至少写 `__repr__` |
| 多继承过深 | MRO 难懂 | 优先组合 |
| `dataclass` 可变默认 | `ValueError` | `default_factory` |

# 8. 练习

1. 定义 `Shape` 基类含 `area()`；派生 `Circle` 和 `Rectangle`，用 `property` 暴露 `area`，并让 `__repr__` 输出可读信息。
2. 用 `@dataclass(frozen=True)` 实现 `Point`，支持 `==`、排序和哈希；放进 `set` 去重，验证相同坐标只保留一个。
3. 用组合改写"`Car` 继承 `Engine`"的错误设计：`Engine` 提供 `start()`，`Car` 持有 `Engine` 并委托调用；说明为什么组合更合适。

# 9. 小结

| 概念 | 要点 |
|------|------|
| `self` | 实例自动传入的第一个参数 |
| 类属性 | 所有实例共享，赋值会遮蔽 |
| 三种方法 | 实例 / 类 / 静态，看第一个参数 |
| 继承 | is-a，MRO 决定查找顺序 |
| 组合 | has-a，默认优先 |
| 魔术方法 | 接入 Python 语法体系 |
| `property` | 属性访问 + 校验/计算 |
| `dataclass` | 自动生成样板，可变默认用 factory |

下一篇进入异常处理，讲清楚异常体系的结构、`try/except/else/finally` 的执行顺序、自定义异常的设计，以及 `with` 背后的上下文管理协议。
