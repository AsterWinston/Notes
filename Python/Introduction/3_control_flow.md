控制流决定程序"走哪条路、走多少遍"。Python 的分支和循环语法很短，但 `for` 背后是可迭代协议、`else` 子句的语义、`match` 的模式匹配，这些才是写出地道代码的关键。本篇把这些机制讲透，并给出常见坑的对照写法。

# 1. 条件分支

## 1.1 if / elif / else

```python
score = 85

if score >= 90:
    grade = "A"
elif score >= 80:
    grade = "B"
elif score >= 60:
    grade = "C"
else:
    grade = "D"

print(grade)   # B
```

★ **关键认知**：`elif` 是**短路求值**——从上到下判断，**一旦某个条件为真，后面的分支全部跳过**。所以条件顺序决定结果：把 `score >= 60` 放最前面，85 分会先命中它，永远拿不到 B。多分支要从窄到宽、从特殊到一般。

## 1.2 条件表达式

```python
age = 20
label = "成年" if age >= 18 else "未成年"
```

只用于"根据条件取一个值"。嵌套超过一层就该改回 `if` 语句：

```python
# 不推荐
result = "A" if x > 90 else "B" if x > 80 else "C"

# 推荐
if x > 90:
    result = "A"
elif x > 80:
    result = "B"
else:
    result = "C"
```

## 1.3 真值陷阱

```python
# 错误：0 是合法值，却被当成"缺失"
def set_retry(n=None):
    if not n:
        n = 3
    return n

print(set_retry(0))   # 3，用户想要 0 次重试，却被改掉了

# 正确：只判断 None
def set_retry(n=None):
    if n is None:
        n = 3
    return n
```

⚠ **`if x:` 和 `if x is not None:` 语义完全不同**。前者对 `0`、`""`、`[]` 都为假；后者只判断"有没有值"。要清楚自己需要哪一种。

# 2. for 循环

## 2.1 迭代协议

`for` 不是计数循环，而是遍历可迭代对象：

```text
for item in obj:
    循环体

1. 调用 iter(obj) 得到迭代器
2. 反复调用 next(迭代器)
3. 每次拿到值赋给 item，执行循环体
4. 抛 StopIteration 时结束
```

```python
for ch in "abc":
    print(ch)          # a b c

for key in {"a": 1, "b": 2}:
    print(key)         # a b（遍历键）
```

★ **关键认知**：只要对象实现了 `__iter__` 或 `__getitem__`，就能被 `for` 遍历。这是**鸭子类型**的体现——不需要继承基类，行为对得上就能用。列表、字符串、字典、文件、生成器都走同一套协议。

## 2.2 range

```python
for i in range(5):          # 0 1 2 3 4
    print(i)

for i in range(2, 10, 3):   # 2 5 8
    print(i)

print(range(5))             # range(0, 5)，惰性对象
print(list(range(5)))       # [0, 1, 2, 3, 4]
```

⚠ **`range` 左闭右开**：`range(5)` 不含 5。想把 1 到 100 都包含，写 `range(1, 101)`。

## 2.3 enumerate 与 zip

```python
names = ["张三", "李四", "王五"]

for i, name in enumerate(names, start=1):
    print(i, name)
```

⚠ **`zip` 以最短序列为准，多余元素被静默丢弃**。Python 3.10+ 想严格等长，用 `zip(..., strict=True)`：

```python
list(zip([1, 2, 3], ["a", "b"]))              # [(1, 'a'), (2, 'b')]
list(zip([1, 2, 3], ["a", "b"], strict=True)) # ValueError
```

## 2.4 for-else

`else` 跟在 `for` 后，含义是"**循环正常走完、没有被 break 打断**"：

```python
target = 7
numbers = [1, 3, 5, 9]

for n in numbers:
    if n == target:
        print("找到了")
        break
else:
    print("没找到")   # 未被 break，执行这里
```

★ **关键认知**：`for-else` 不是"循环结束后总执行"，而是"**循环没有 break 才执行**"。很多人误以为 `else` 一定执行，这是最常见的误解。觉得反直觉就用标志变量，完全可接受。

# 3. while 循环

```python
count = 0
while count < 3:
    print(count)
    count += 1

while True:
    text = input("输入命令（q 退出）：")
    if text == "q":
        break
    print(f"你输入了：{text}")
```

⚠ **`while` 最常见的 bug 是忘了更新条件变量**，导致死循环。写之前先确认三件事：**初始条件、退出条件、每轮如何逼近退出条件**。

`while-else` 与 `for-else` 语义相同：

```python
n = 10
while n > 0:
    n -= 1
else:
    print("正常结束")   # 执行
```

# 4. break、continue 与 pass

| 关键字 | 作用 | 场景 |
|--------|------|------|
| `break` | 跳出**当前这一层**循环 | 找到目标、满足退出条件 |
| `continue` | 跳过本轮剩余代码 | 过滤不需要的元素 |
| `pass` | 什么都不做 | 语法占位 |

```python
for i in range(10):
    if i % 2 == 0:
        continue        # 偶数跳过
    if i > 7:
        break           # 大于 7 提前结束
    print(i)            # 1 3 5 7
```

⚠ **`break` 只跳出最内层循环**。嵌套想一次跳出两层，用标志变量，或把内层抽成函数用 `return`：

```python
def find_pair():
    for i in range(3):
        for j in range(3):
            if i * j == 4:
                return i, j
    return None
```

# 5. match 模式匹配

## 5.1 基本语法

```python
command = "start"

match command:
    case "start":
        print("启动")
    case "stop":
        print("停止")
    case _:
        print("未知命令")
```

★ **关键认知**：`match` 的 `case` **不会穿透**，第一个匹配的分支执行完就结束，不需要 `break`。它匹配的是**结构**而不只是值，能同时解构列表、字典、对象并绑定变量。

## 5.2 结构化匹配

```python
point = (1, 0)

match point:
    case (0, 0):
        print("原点")
    case (x, 0):
        print(f"x 轴上，x={x}")
    case (0, y):
        print(f"y 轴上，y={y}")
    case (x, y):
        print(f"普通点 ({x}, {y})")
```

```python
cmd = ["go", "north", "fast"]

match cmd:
    case ["go", direction]:
        print(f"向 {direction} 前进")
    case ["go", direction, *rest]:
        print(f"向 {direction} 前进，附加参数 {rest}")
```

```python
event = {"type": "click", "x": 10, "y": 20}

match event:
    case {"type": "click", "x": x, "y": y}:
        print(f"点击 ({x}, {y})")
    case {"type": "key"}:
        print("键盘事件")
```

## 5.3 守卫与或模式

```python
value = 15

match value:
    case n if n < 0:
        print("负数")
    case n if n == 0:
        print("零")
    case n if n % 2 == 0:
        print("正偶数")
    case _:
        print("正奇数")
```

⚠ **`match` 里的变量名是"捕获"，不是"比较"**。`case x:` 会匹配一切并把值绑定到 `x`，等同通配符。想比较已有变量，用守卫 `case n if n == x:`，或用点号常量（如 `Color.RED`）。

# 6. 常见错误

| 错误 | 现象 | 正确做法 |
|------|------|---------|
| 分支顺序从宽到窄 | 后面的分支永不命中 | 从窄到宽 |
| `if x:` 判断 None | `0`、`""` 被误判 | 用 `if x is None` |
| `range(1, 100)` 以为含 100 | 少一次 | 写 `range(1, 101)` |
| `zip` 长度不等 | 数据静默丢失 | `strict=True` |
| 嵌套循环 `break` | 只跳出一层 | 标志变量或函数 `return` |
| 以为 `for-else` 总执行 | 被 break 后不执行 | 记住"无 break 才执行" |

# 7. 练习

1. 写 `fizzbuzz(n)`：遍历 1 到 n，3 的倍数输出 `Fizz`，5 的倍数输出 `Buzz`，15 的倍数输出 `FizzBuzz`，其余输出数字。注意判断顺序。
2. 用 `enumerate` 和 `zip` 输出 `1. 张三: 90` 这样的排名行，并在两个列表长度不等时抛出明确错误。
3. 用 `match` 解析命令：`["move", x, y]` 输出坐标，`["say", text]` 输出文本，其他输出"未知命令"；给 `move` 加守卫，要求 `x`、`y` 非负。

# 8. 小结

| 概念 | 要点 |
|------|------|
| `if/elif/else` | 短路求值，条件从窄到宽 |
| 条件表达式 | 只用于简单二选一 |
| `for` | 遍历可迭代对象，走迭代协议 |
| `range` | 惰性序列，左闭右开 |
| `enumerate`/`zip` | 索引遍历、并行遍历首选 |
| `for-else` | 未被 break 才执行 |
| `break` | 只跳出最内层 |
| `match` | 结构化匹配，无穿透，变量是捕获 |

下一篇进入函数，讲清楚参数的四种形态、可变默认参数这个经典陷阱、LEGB 作用域与闭包的延迟绑定问题，以及类型注解在 3.11/3.12 的写法。
