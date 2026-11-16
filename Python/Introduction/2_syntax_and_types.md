Python 的语法看起来简单，但"变量是什么""类型什么时候确定""为什么 `0.1 + 0.2 != 0.3`"这几个问题，恰恰是初学者最容易含糊带过的地方。本篇从名字绑定讲起，把数字、字符串、布尔、`None` 和类型转换串成一条完整的类型认知链。

# 1. 变量与对象

## 1.1 变量是名字，不是盒子

```python
a = [1, 2, 3]
b = a          # b 和 a 指向同一个列表对象
b.append(4)
print(a)       # [1, 2, 3, 4]，a 也变了
```

```text
a ──┐
    ├──▶ [1, 2, 3, 4]   （同一个 list 对象）
b ──┘
```

★ **关键认知**：赋值 `=` **从不复制对象**，只让左边的名字指向右边的对象。判断两个变量会不会互相影响，只需问：**它们是不是同一个对象？** 用 `a is b` 验证，需要副本用 `a.copy()`。

## 1.2 动态类型与强类型

```python
x = 42
x = "hello"        # 类型随对象改变，变量本身没有类型
print("1" + 1)     # TypeError！不做不合理的隐式转换
```

| 类型系统 | 含义 | 例子 |
|---------|------|------|
| 动态类型 | 类型属于对象，运行时确定 | Python、Ruby |
| 静态类型 | 类型属于变量，编译时确定 | Java、Go |
| 强类型 | 不允许隐式跨类型运算 | Python、Java |
| 弱类型 | 允许隐式转换 | JavaScript、C |

★ **关键认知**：动态类型带来灵活，代价是**错误推迟到运行时**。`x.upper()` 类型不对，只有执行到那行才报 `AttributeError`。这是类型注解和测试格外重要的原因。

## 1.3 is 与 ==

```python
a = 1000
b = 1000
print(a is b)     # 可能 False，不要依赖
print(a == b)     # True，这才是正确比较
print(a is None)  # False
```

| 运算符 | 比较 | 用途 |
|--------|------|------|
| `is` | 对象身份（`id`） | `None`、哨兵对象 |
| `==` | 值相等 | 绝大多数内容比较 |

⚠ **小整数和短字符串有驻留优化**，`is` 有时会意外为真，但这是实现细节。**值比较永远用 `==`，`None` 判断永远用 `is`。**

# 2. 数字

## 2.1 int 与 float

```python
print(2 ** 100)    # 任意精度整数，不会溢出
print(1 + 2.5)     # 3.5，int 自动提升为 float
print(7 / 2)       # 3.5，真除法
print(7 // 2)      # 3，向下取整除法
print(7 % 2)       # 1
```

⚠ **负数整除是"向下"取整**：

```python
print(-7 // 2)     # -4，不是 -3
print(-7 % 2)      # 1，不是 -1
```

这保证 `a == (a // b) * b + a % b` 恒成立。分页、取模哈希要注意负数。

## 2.2 浮点误差与 Decimal

```python
print(0.1 + 0.2)          # 0.30000000000000004
print(0.1 + 0.2 == 0.3)   # False
```

★ **关键认知**：这不是 Python 的 bug，而是**二进制浮点无法精确表示 0.1 和 0.2**，几乎所有语言都如此。比较浮点要用容差：

```python
import math
print(math.isclose(0.1 + 0.2, 0.3))   # True

from decimal import Decimal
print(Decimal("0.1") + Decimal("0.2") == Decimal("0.3"))   # True
```

⚠ **`Decimal` 必须用字符串构造**。`Decimal(0.1)` 会把浮点误差原样带进去，等于白用。

## 2.3 可读性写法

```python
count = 1_000_000    # 下划线只是分隔符
print(0b1010)        # 二进制 → 10
print(0o17)          # 八进制 → 15
print(0xFF)          # 十六进制 → 255
```

# 3. 字符串

## 3.1 不可变

```python
s = "hello"
s[0] = "H"       # TypeError！字符串不可变
s = "H" + s[1:]  # 正确：创建新字符串
```

★ **关键认知**：`str` 不可变，所有"修改"都返回新对象。好处是可以安全作字典键、可以共享；代价是循环里 `s += x` 会不断新建对象，复杂度 O(n²)，应改用 `"".join(parts)`。

```python
# 错误：循环内反复拼接
result = ""
for word in words:
    result += word

# 正确：先收集再 join
result = "".join(words)
```

## 3.2 切片与 f-string

```python
s = "Python"
print(s[0])      # P
print(s[-1])     # n
print(s[1:4])    # yth，左闭右开
print(s[::-1])   # nohtyP，反转
print(s[1:100])  # ython，切片越界不报错
```

```python
name, age = "张三", 20
print(f"{name} 今年 {age} 岁")
print(f"{3.14159:.2f}")    # 3.14
print(f"{1234567:,}")      # 1,234,567
print(f"{42:05d}")         # 00042
x = 42
print(f"{x=}")             # x=42，调试神器
```

⚠ **索引越界抛 `IndexError`，切片越界只截断**。另外正则和 Windows 路径用原始字符串 `r"..."`，避免转义。

# 4. 布尔与 None

## 4.1 真值测试

```python
print(bool(0), bool(""), bool([]), bool(None))   # False False False False
print(bool("0"), bool([0]))                      # True True
```

| 为假（falsy） | 其余为真 |
|--------------|---------|
| `False`、`None` | 非空字符串 |
| `0`、`0.0`、`0j` | 非零数字 |
| `""`、`[]`、`()`、`{}`、`set()` | 非空容器 |

★ **关键认知**：判断"空"直接 `if not x`，但 `if x` 的语义是"调用 `bool(x)`"。如果只想判断"是不是 None"，必须写 `if x is None`——否则 `0`、`""`、`[]` 都会被误伤。

```python
def process(x=None):
    if x is None:      # 正确：只判断 None
        x = []
    # 若写 if not x，传入 0 也会被替换
```

## 4.2 None 的本质

`None` 是单例对象，类型 `NoneType`：

```python
print(type(None))     # <class 'NoneType'>
print(None is None)   # True
```

⚠ **判断 None 一律用 `is None`**。自定义对象可能重写 `__eq__`，让 `== None` 偏离预期；`is` 比较身份，永远准确。

# 5. 类型转换

```python
print(int("42"))        # 42
print(int(3.9))         # 3，向零截断
print(int(-3.9))        # -3
print(float("3.14"))    # 3.14
print(str(42))          # "42"
print(list("abc"))      # ['a', 'b', 'c']
```

⚠ **`int()` 不四舍五入，是向零截断**；`round()` 用的是"银行家舍入"：

```python
print(round(3.5))   # 4
print(round(2.5))   # 2，不是 3
print(int("3.14"))  # ValueError，需先 float 再 int
```

★ **关键认知**：Python 只在安全场景做隐式提升（`int` + `float`），**字符串与数字之间绝不隐式转换**。用户输入必须包在 `try/except` 里，因为 `int("abc")` 会抛 `ValueError`。

# 6. 常见错误

| 错误 | 现象 | 正确做法 |
|------|------|---------|
| 用 `is` 比较值 | `1000 is 1000` 可能 False | 用 `==` |
| 用 `== None` | 被自定义 `__eq__` 干扰 | 用 `is None` |
| 浮点直接相等 | `0.1+0.2 == 0.3` 为 False | `math.isclose` |
| 循环内字符串拼接 | 大数据量极慢 | `"".join()` |
| 假设赋值会复制 | 改副本影响原对象 | `copy()` / `deepcopy()` |
| `int("3.14")` | `ValueError` | 先 `float()` |

# 7. 练习

1. 先预测再运行：`0.1 + 0.2 == 0.3`、`-7 // 2`、`int(-3.9)`、`round(2.5)`，并解释每个结果。
2. 写 `safe_int(text)`：能处理 `"42"`、`"3.14"`、`"abc"`，分别返回 `42`、`3`、`None`。
3. 给定 `words = ["Py", "thon", "3", ".12"]`，分别用循环拼接和 `"".join()` 得到 `"Python3.12"`，用 `timeit` 比较大量拼接的耗时差异。

# 8. 小结

| 概念 | 要点 |
|------|------|
| 变量 | 名字绑定对象，赋值不复制 |
| 动态类型 | 类型属于对象，运行时确定 |
| 强类型 | 不做不合理的隐式转换 |
| `is` vs `==` | 身份 vs 值 |
| int | 任意精度，无溢出 |
| float | IEEE 754，有误差，用 `isclose` |
| str | 不可变，大量拼接用 `join` |
| None | 单例，判断用 `is None` |
| 转换 | 显式、可能抛异常，用户输入要捕获 |

下一篇进入控制流，讲清楚 `if/elif/else` 的求值顺序、`for` 与可迭代协议的关系、`range` 的惰性、`break/continue` 的控制边界，以及 Python 3.10+ 的 `match` 模式匹配。
