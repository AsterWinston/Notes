函数是组织代码的第一层抽象。Python 的函数看似只是 `def` 加缩进，但参数传递、默认值、作用域、闭包、类型注解每一块都有容易踩的坑。本篇按"定义 → 参数 → 作用域 → 闭包 → 注解"的顺序，把函数机制一次讲清楚。

# 1. 定义与调用

## 1.1 def 与返回值

```python
def add(a, b):
    """返回两个数之和。"""
    return a + b

print(add(1, 2))    # 3
```

★ **关键认知**：Python 的函数**永远返回一个对象**。没有 `return` 就返回 `None`；`return a, b` 实际返回元组。所谓"多返回值"就是"返回元组再解包"。

## 1.2 文档字符串

```python
def divide(a: float, b: float) -> float:
    """计算 a 除以 b。

    参数:
        a: 被除数。
        b: 除数，不能为 0。
    """
    if b == 0:
        raise ValueError("除数不能为 0")
    return a / b

help(divide)   # IDE 和 help() 都会读取 docstring
```

# 2. 参数

## 2.1 位置参数与关键字参数

```python
def create_user(name, age, city="北京"):
    print(name, age, city)

create_user("张三", 20)                    # 位置参数
create_user(name="李四", age=30)            # 关键字参数
create_user("王五", city="上海", age=25)     # 混用：位置在前
```

⚠ **位置参数必须写在关键字参数之前**。`create_user(name="张三", 20)` 直接 `SyntaxError`，Python 不允许"关键字在前、位置在后"。

## 2.2 默认参数与可变默认陷阱

```python
def greet(name, greeting="你好"):
    return f"{greeting}，{name}"
```

★ **关键认知**：**默认参数只在函数定义时求值一次**，所有调用共享同一个默认对象。所以默认值绝对不能用可变对象：

```python
# 错误示范：所有调用共享同一个列表
def add_item(item, bucket=[]):
    bucket.append(item)
    return bucket

print(add_item(1))   # [1]
print(add_item(2))   # [1, 2]，不是 [2]！

# 正确：用 None 占位，函数内再创建
def add_item(item, bucket=None):
    if bucket is None:
        bucket = []
    bucket.append(item)
    return bucket
```

⚠ 这个坑隐蔽在于：单次调用看不出问题，第二次才暴露。**只要默认值是可变对象，就默认它是 bug。**

## 2.3 *args 与 **kwargs

```python
def total(*args):
    print(type(args))    # <class 'tuple'>
    return sum(args)

def build(**kwargs):
    print(type(kwargs))  # <class 'dict'>
    return kwargs

print(total(1, 2, 3))             # 6
print(build(name="张三", age=20))  # {'name': '张三', 'age': 20}
```

组合顺序固定：**普通位置 → `*args` → 仅关键字 → `**kwargs`**。

```python
def mixed(a, b, *args, sep="-", **kwargs):
    print(a, b, args, sep, kwargs)

mixed(1, 2, 3, 4, sep="|", debug=True)
# 1 2 (3, 4) | {'debug': True}
```

★ **关键认知**：`*args`/`**kwargs` 的名字可任意取，`*` 和 `**` 才是语法。它们在定义处**收集**参数，在调用处则**解包**：

```python
nums = [1, 2, 3]
print(total(*nums))            # 解包成位置参数

config = {"name": "张三", "age": 20}
print(build(**config))         # 解包成关键字参数
```

## 2.4 仅位置与仅关键字参数

```python
def strict(a, b, /, c, *, d):
    print(a, b, c, d)

strict(1, 2, 3, d=4)         # 正确
strict(a=1, b=2, c=3, d=4)   # TypeError：a、b 只能位置传
strict(1, 2, 3, 4)           # TypeError：d 只能关键字传
```

| 写法 | 含义 |
|------|------|
| `def f(a, /, b)` | `a` 只能位置传 |
| `def f(a, *, b)` | `b` 只能关键字传 |
| `def f(a, /, *, b)` | `a` 仅位置，`b` 仅关键字 |

这常用于公共 API：仅位置参数让参数名可随时改而不破坏调用方，仅关键字参数让调用意图更清晰。

# 3. 作用域

## 3.1 LEGB

```text
L（Local）       当前函数内部
E（Enclosing）   外层嵌套函数
G（Global）      模块全局
B（Built-in）    内置命名空间
```

```python
x = "global"

def outer():
    x = "enclosing"

    def inner():
        x = "local"
        print(x)      # local

    inner()
    print(x)          # enclosing

outer()
print(x)              # global
```

★ **关键认知**：Python 是**词法作用域**，作用域在编译时就确定。函数能访问哪些名字，看它**定义在哪里**，而不是被谁调用。这是理解闭包的前提。

## 3.2 global 与 nonlocal

```python
count = 0

def increment():
    global count     # 声明使用全局变量
    count += 1

increment()
print(count)         # 1
```

```python
def make_counter():
    count = 0

    def counter():
        nonlocal count   # 改外层函数变量
        count += 1
        return count

    return counter

c = make_counter()
print(c())   # 1
print(c())   # 2
```

⚠ **不加 `global`/`nonlocal` 直接赋值，会创建局部变量**，并在读取时报 `UnboundLocalError`。但两者都应少用——它们让函数产生副作用、难以测试。更好的是把状态作为参数传入、结果作为返回值传出。

## 3.3 闭包与延迟绑定

闭包是"**函数 + 它捕获的外层变量**"：

```python
def make_multiplier(factor):
    def multiply(x):
        return x * factor   # factor 被闭包捕获
    return multiply

double = make_multiplier(2)
print(double(10))   # 20
```

★ **关键认知**：闭包捕获的是**变量本身，不是变量当时的值**。循环里创建的多个函数，最终都指向循环结束时的那个变量：

```python
# 错误示范：所有函数都返回 2
funcs = [lambda: i for i in range(3)]
print([f() for f in funcs])   # [2, 2, 2]，不是 [0, 1, 2]

# 正确一：默认参数在定义时绑定当前值
funcs = [lambda i=i: i for i in range(3)]

# 正确二：工厂函数制造独立作用域
def make(i):
    return lambda: i

funcs = [make(i) for i in range(3)]
```

⚠ 这个坑在写回调、事件处理器时非常常见。**闭包看到的是变量的最终值，想固定当前值就用默认参数或工厂函数。**

# 4. lambda 与高阶函数

它的价值在于"作为参数传给高阶函数"：

```python
pairs = [(1, "b"), (3, "a"), (2, "c")]
pairs.sort(key=lambda p: p[1])
print(pairs)        # [(3, 'a'), (1, 'b'), (2, 'c')]

from functools import reduce
nums = [1, 2, 3, 4]
print(list(map(lambda x: x * 2, nums)))          # [2, 4, 6, 8]
print(list(filter(lambda x: x % 2 == 0, nums)))  # [2, 4]
print(reduce(lambda a, b: a + b, nums))          # 10
```

⚠ **不要把 lambda 赋给变量名**。PEP 8 建议：需要命名就用 `def`，它有名字、有 docstring、报错栈更清晰。`lambda` 只该出现在"用完即弃"的传参处。

# 5. 类型注解

```python
def greet(name: str, times: int = 1) -> str:
    return f"你好，{name}！" * times

print(greet(123))   # 运行时照样能跑，注解不校验
```

★ **关键认知**：类型注解是**给工具看的文档**，不是运行时约束。要真正检查，需要运行 `mypy` 或 `pyright`。它的价值是 IDE 补全更准、重构更安全、接口意图更清晰。

```python
def find(name: str) -> str | None:    # 3.10+ 用 | 代替 Optional
    ...
```

Python 3.12 引入 `type` 语句定义类型别名：

```python
type Point = tuple[float, float]

def distance(a: Point, b: Point) -> float:
    return ((a[0] - b[0]) ** 2 + (a[1] - b[1]) ** 2) ** 0.5
```

⚠ **注解里的类型名必须"运行时可求值"**。前向引用（类引用自身）要么加引号，要么在文件顶部写 `from __future__ import annotations`。养成加这一行的习惯，能省掉很多麻烦。

# 6. 常见错误

| 错误 | 现象 | 正确做法 |
|------|------|---------|
| 可变默认参数 | 多次调用共享状态 | 用 `None` 占位 |
| 闭包延迟绑定 | 所有函数返回同一个值 | 默认参数或工厂函数 |
| 忘记 `return` | 得到 `None` | 显式返回 |
| `global` 滥用 | 副作用难测 | 传参 + 返回 |
| 注解当运行时检查 | 传错类型不报错 | 配合 `mypy`/`pyright` |

# 7. 练习

1. 写 `make_accumulator()`，返回累加函数：每次传入一个数，返回累计和；能多次创建互不干扰的累加器。
2. 修复并解释：`def append_all(items, target=[]): target.extend(items); return target`，要求两次调用分别得到 `[1]` 和 `[2]`。
3. 给 `parse_users(rows: list[dict]) -> dict[str, int]` 补全类型注解并实现：把 `[{"name": "张三", "age": 20}]` 转成 `{"张三": 20}`，`age` 缺失时跳过。

# 8. 小结

| 概念 | 要点 |
|------|------|
| 返回值 | 永远返回一个对象，多值即元组 |
| 默认参数 | 定义时求值一次，禁用可变对象 |
| `*args`/`**kwargs` | 收集参数，调用处可解包 |
| `/` 与 `*` | 强制仅位置、仅关键字 |
| LEGB | 词法作用域，编译时确定 |
| `nonlocal` | 改外层函数变量，少用 |
| 闭包 | 捕获变量本身，注意延迟绑定 |
| lambda | 单表达式，只用于传参 |
| 类型注解 | 工具用，运行时不校验 |

下一篇进入模块与包，讲清楚 `import` 到底做了什么、模块搜索路径的顺序、`__init__.py` 的作用、相对导入的约束，以及如何避免循环导入。
