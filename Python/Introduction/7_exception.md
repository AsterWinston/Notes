异常处理决定程序出错时的行为：是静默失败、是给出清晰报错、还是带着上下文优雅退出。Python 的异常机制很轻，但 `except` 写得太宽、`finally` 里有 `return`、异常链被吞掉，都会让问题更难查。本篇把异常体系、执行顺序、自定义异常和上下文管理讲透。

# 1. 异常体系

## 1.1 异常与 traceback

异常是**运行时的错误信号**。它打断正常流程向上冒泡，直到被 `except` 捕获；没人捕获就终止程序并打印 traceback。

```python
def divide(a, b):
    return a / b

print(divide(1, 0))
```

★ **关键认知**：traceback 要**从下往上读**。最后一行是异常类型和消息（什么错了），往上是调用栈（在哪错的、怎么走到这的）。先看最后一行，再顺着栈找到自己代码的那一层。

## 1.2 继承结构

```text
BaseException
├─ SystemExit            # sys.exit()
├─ KeyboardInterrupt     # Ctrl+C
├─ GeneratorExit
└─ Exception            # ★ 绝大多数业务异常都在这
   ├─ ValueError
   ├─ TypeError
   ├─ KeyError
   ├─ IndexError
   ├─ OSError
   │  ├─ FileNotFoundError
   │  └─ PermissionError
   └─ RuntimeError
```

★ **关键认知**：**业务代码只该捕获 `Exception`，不要捕获 `BaseException`**。`KeyboardInterrupt`、`SystemExit` 是控制信号，不是错误——捕获它们会让 Ctrl+C 失效、程序无法正常退出。

```python
# 错误：Ctrl+C 也被吞掉
try:
    do_work()
except BaseException:
    pass

# 正确
try:
    do_work()
except Exception as err:
    print(f"处理失败：{err}")
```

## 1.3 常见内置异常

| 异常 | 触发场景 | 例子 |
|------|---------|------|
| `ValueError` | 类型对但值不合法 | `int("abc")` |
| `TypeError` | 类型不匹配 | `"1" + 1` |
| `KeyError` | 字典键不存在 | `{}["x"]` |
| `IndexError` | 索引越界 | `[][0]` |
| `AttributeError` | 对象没有该属性 | `"x".foo` |
| `FileNotFoundError` | 文件不存在 | `open("无此文件")` |
| `ZeroDivisionError` | 除零 | `1 / 0` |

# 2. try/except/else/finally

## 2.1 基本结构

```python
try:
    result = 10 / int(user_input)    # 可能出错的代码
except ValueError:
    print("输入不是数字")
except ZeroDivisionError:
    print("除数不能为 0")
else:
    print(f"结果：{result}")          # 没异常才执行
finally:
    print("清理工作")                 # 无论如何都执行
```

## 2.2 执行顺序

```text
进入 try
  ├─ 没有异常 ──▶ 执行 else ──▶ 执行 finally
  ├─ 异常被捕获 ──▶ 执行 except ──▶ 执行 finally
  └─ 异常未捕获 ──▶ 执行 finally ──▶ 继续冒泡
```

★ **关键认知**：`else` 的语义是"**try 块没有异常才执行**"。它的价值是区分"可能出错的代码"和"依赖成功结果的代码"，避免把 `else` 的逻辑放进 `try` 从而误捕获不该捕获的异常：

```python
# 不推荐：print 出错也会被当成除法错误
try:
    result = 10 / x
    print(result)
except ZeroDivisionError:
    print("除零")

# 推荐：try 只包可能出错的那句
try:
    result = 10 / x
except ZeroDivisionError:
    print("除零")
else:
    print(result)
```

## 2.3 捕获多个异常

```python
try:
    data = load_config(path)
except (FileNotFoundError, PermissionError) as err:
    print(f"读取配置失败：{err}")
```

多个 `except` 要**从具体到宽泛**排列，例如先捕获 `KeyError`，再用 `Exception` 兜底。

⚠ **`except Exception: pass` 是反模式**。它把错误彻底吞掉，留下"程序行为不对但没有任何日志"的黑洞。真要兜底，至少记录日志：

```python
import logging

try:
    process()
except Exception:
    logging.exception("处理失败")   # 自动带 traceback
    raise                            # 或重新抛出
```

## 2.4 异常链

```python
try:
    config = load_config(path)
except OSError as err:
    raise RuntimeError("配置加载失败") from err
```

★ **关键认知**：`raise ... from err` 建立**显式异常链**，traceback 会显示"The above exception was the direct cause of..."，让底层原因和上层语义都保留。在 `except` 里单独写 `raise` 表示原样重新抛出，**不要写 `raise err`**，那会重置 traceback。

# 3. 自定义异常

```python
class AppError(Exception):
    """应用异常的基类。"""


class ValidationError(AppError):
    """输入校验失败。"""

    def __init__(self, field: str, message: str):
        self.field = field
        self.message = message
        super().__init__(f"{field}: {message}")


def validate_age(age: int) -> None:
    if age < 0:
        raise ValidationError("age", "不能为负数")

try:
    validate_age(-1)
except ValidationError as err:
    print(err.field, err.message)   # age 不能为负数
```

| 建议 | 理由 |
|------|------|
| 为每个领域建一个基类 | 调用方可一次捕获整类错误 |
| 携带结构化字段 | 比解析字符串可靠 |
| 消息面向人类，字段面向代码 | 职责分开 |
| 不要继承 `BaseException` | 会破坏控制信号 |

★ **关键认知**：自定义异常的价值不只是"报错更清楚"，更是**让调用方按类型分流处理**。`except ValidationError` 可以提示用户，`except AppError` 可以统一记日志。如果全抛 `Exception`，调用方只能字符串匹配，非常脆弱。

⚠ **异常类要尽量小**，不要为每个错误码建一个类。"一个领域基类 + 少量具体子类"通常就够了。

# 4. 上下文管理器

## 4.1 with 解决了什么

文件、锁、连接都需要"用完必须释放"，手写 `try/finally` 很啰嗦。

`with` 把模板封装起来：

```python
with open("data.txt", encoding="utf-8") as f:
    content = f.read()
# 离开 with 自动关闭，即使中途抛异常
```

## 4.2 上下文管理协议

实现 `__enter__` 和 `__exit__` 即可用于 `with`：

```python
class Timer:
    def __enter__(self):
        import time
        self.start = time.perf_counter()
        return self              # as 后的变量拿到这个返回值

    def __exit__(self, exc_type, exc_value, traceback):
        import time
        print(f"耗时 {time.perf_counter() - self.start:.4f}s")
        return False             # False 表示不吞异常


with Timer():
    sum(range(1_000_000))
```

```text
with expr as var:
    代码块

等价于：
mgr = expr
var = mgr.__enter__()
try:
    代码块
finally:
    mgr.__exit__(异常信息)
```

★ **关键认知**：`__exit__` **返回 `True` 会吞掉异常，返回 `False` 或 `None` 会让异常继续传播**。这是自定义上下文管理器最容易写错的地方。

## 4.3 contextlib

```python
from contextlib import contextmanager

@contextmanager
def timer():
    import time
    start = time.perf_counter()
    try:
        yield                    # yield 前相当于 __enter__
    finally:
        print(f"耗时 {time.perf_counter() - start:.4f}s")  # 相当于 __exit__


with timer():
    sum(range(1_000_000))
```

⚠ **`@contextmanager` 里 `yield` 必须放在 `try/finally` 中**，否则代码块抛异常时清理逻辑不会执行。

常用工具还有 `contextlib.suppress`（忽略指定异常）和 `closing`（退出时调用 `close()`）。

# 5. 常见错误

| 错误 | 后果 | 正确做法 |
|------|------|---------|
| `except: pass` | 静默吞错，难排查 | 记录日志并重新抛出 |
| 捕获 `BaseException` | Ctrl+C 失效 | 只捕获 `Exception` |
| `except` 顺序从宽到窄 | 具体分支不执行 | 从具体到宽泛 |
| `raise err` | 重置 traceback | 单独 `raise` |
| `finally` 里 `return` | 吞掉异常和返回值 | 清理逻辑不返回 |
| `@contextmanager` 无 `finally` | 出错不清理 | `yield` 放进 `try/finally` |

⚠ **`finally` 里有 `return` 会覆盖返回值，甚至吞掉异常**：

```python
def f():
    try:
        raise ValueError("出错")
    finally:
        return 1        # 异常被吞掉，返回 1

print(f())   # 1，错误消失
```

这属于"能跑但极度危险"的写法，代码评审应直接拒绝。

# 6. 练习

1. 写 `safe_divide(a, b)`：`b` 为 0 时抛自定义 `DivisionByZeroError`（继承 `ValueError`），调用方捕获后打印友好提示，同时保留原始异常链。
2. 用类实现 `Timer` 上下文管理器，要求代码块抛异常时也打印耗时，且**不吞掉**异常；再用 `@contextmanager` 重写一遍。
3. 修复 `try: risky() except Exception: pass`：要求记录日志、区分 `ValueError` 和 `OSError`、其余异常继续抛出。

# 7. 小结

| 概念 | 要点 |
|------|------|
| 异常体系 | 业务异常继承 `Exception`，别碰 `BaseException` |
| `try/except` | 只包可能出错的代码，从具体到宽泛 |
| `else` | 无异常才执行 |
| `finally` | 一定执行，别写 `return` |
| `raise from` | 保留异常链 |
| 自定义异常 | 领域基类 + 结构化字段 |
| `with` | `__enter__` / `__exit__` 自动清理 |
| `contextlib` | `@contextmanager`、`suppress`、`closing` |

下一篇进入文件与 IO，讲清楚 `open` 的模式与编码、文本与二进制的边界、`pathlib` 的现代写法，以及 JSON/CSV 的读写陷阱。
