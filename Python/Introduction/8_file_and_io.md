文件读写是"程序与外部世界交换数据"的第一站。编码不对会乱码、模式写错会清空文件、忘记关闭会丢数据、路径拼接在 Windows 和 Linux 上行为不同。本篇从 `open` 的模式讲到 `pathlib`，再落到 JSON/CSV 的实战写法。

# 1. open 基础

## 1.1 路径

```python
open("data.txt")                              # 相对当前工作目录
open(r"C:\Users\name\data.txt")               # 绝对路径，原始字符串
```

★ **关键认知**：相对路径的基准是**当前工作目录（`os.getcwd()`）**，不是脚本文件所在目录。从不同目录运行同一脚本，相对路径会指向不同位置——这是"本地能跑、换个地方就找不到文件"的最常见原因。需要相对脚本自身时：

```python
from pathlib import Path

base = Path(__file__).resolve().parent
data_path = base / "data.txt"
```

## 1.2 模式

| 模式 | 含义 | 不存在 | 已存在 |
|------|------|--------|--------|
| `"r"` | 只读（默认） | 报错 | 从头读 |
| `"w"` | 只写 | 创建 | **清空** |
| `"a"` | 追加 | 创建 | 末尾追加 |
| `"x"` | 独占创建 | 创建 | 报错 |
| `"r+"` | 读写 | 报错 | 从头读写 |
| `"b"` | 二进制 | 配合上面 | — |

⚠ **`"w"` 会立即清空文件**，哪怕一个字都没写。追加用 `"a"`；要"存在就报错、避免覆盖"用 `"x"`。

## 1.3 读写

```python
# 一次性读全部
with open("data.txt", encoding="utf-8") as f:
    content = f.read()

# 按行读成列表
with open("data.txt", encoding="utf-8") as f:
    lines = f.readlines()

# 逐行迭代（大文件首选）
with open("data.txt", encoding="utf-8") as f:
    for line in f:
        print(line.rstrip("\n"))
```

```python
with open("output.txt", "w", encoding="utf-8") as f:
    f.write("第一行\n")
    f.write("第二行\n")

with open("output.txt", "w", encoding="utf-8") as f:
    f.writelines(["a\n", "b\n", "c\n"])
```

★ **关键认知**：文件对象本身**可迭代**，`for line in f` 每次只读一行。处理大文件时不要用 `read()` 或 `readlines()` 一次性载入，几个 GB 的文件会直接打满内存。

⚠ **`write()` 不会自动加换行**，要自己写 `\n`。

# 2. 文本与二进制

## 2.1 编码

```text
读：磁盘字节 ──解码（decode）──▶ str
写：str ──编码（encode）──▶ 磁盘字节
```

```python
# 文本模式：自动编解码
with open("data.txt", "w", encoding="utf-8") as f:
    f.write("你好")

# 二进制模式：拿原始 bytes，自己负责编解码
with open("data.bin", "wb") as f:
    f.write("你好".encode("utf-8"))
```

★ **关键认知**：**永远显式指定 `encoding="utf-8"`**。不指定时 Python 用平台默认编码——Windows 中文环境常是 `gbk`，Linux/macOS 是 `utf-8`。同一份代码在两边读写同一文件，可能一个正常、一个乱码。这是跨平台项目最经典的坑。

## 2.2 换行

文本模式下，Python 会自动把 `\r\n` 统一成 `\n`。处理 CSV 或需要精确控制时，反而要关掉：

```python
with open("data.csv", "w", encoding="utf-8", newline="") as f:
    ...
```

# 3. with 与资源管理

```python
with open("data.txt", encoding="utf-8") as f:
    content = f.read()
# 离开 with 自动 close()，即使读的过程中抛异常
```

不用 `with` 的等价写法：

```python
f = open("data.txt", encoding="utf-8")
try:
    content = f.read()
finally:
    f.close()
```

★ **关键认知**：`with` 的价值不是"少写两行"，而是**保证异常路径下也释放资源**。文件句柄是有限资源，泄漏多了会触发 `Too many open files`。任何需要成对操作的资源（文件、锁、连接、临时目录）都该用 `with`。

```python
with (
    open("in.txt", encoding="utf-8") as src,
    open("out.txt", "w", encoding="utf-8") as dst,
):
    for line in src:
        dst.write(line.upper())
```

# 4. pathlib

## 4.1 Path 基础

```python
from pathlib import Path

p = Path("data") / "raw" / "users.csv"   # 用 / 拼接，自动处理分隔符
print(p)              # data\raw\users.csv（Windows）
print(p.name)         # users.csv
print(p.stem)         # users
print(p.suffix)       # .csv
print(p.parent)       # data\raw
print(p.exists())     # 是否存在
print(p.is_file())    # 是否是文件
```

| 操作 | `pathlib` | `os.path` |
|------|-----------|-----------|
| 拼接 | `Path("a") / "b"` | `os.path.join` |
| 文件名 | `p.name` | `os.path.basename` |
| 后缀 | `p.suffix` | `os.path.splitext(p)[1]` |
| 父目录 | `p.parent` | `os.path.dirname` |
| 绝对路径 | `p.resolve()` | `os.path.abspath` |
| 是否存在 | `p.exists()` | `os.path.exists` |

★ **关键认知**：跨平台路径拼接**不要用字符串 `+`**。`Path("data") / "a.txt"` 会自动用当前平台分隔符，是唯一推荐写法。

## 4.2 遍历与匹配

```python
from pathlib import Path

base = Path("data")

for path in base.glob("*.csv"):    # 当前目录
    print(path.name)

for path in base.rglob("*.py"):    # 递归
    print(path)

(base / "output").mkdir(parents=True, exist_ok=True)

text = Path("data.txt").read_text(encoding="utf-8")
Path("out.txt").write_text("你好", encoding="utf-8")
```

⚠ **`glob("*.csv")` 只匹配当前目录**，递归用 `rglob` 或 `glob("**/*.csv")`。`read_text`/`write_text` 适合小文件，大文件仍要 `open` 逐行处理。

# 5. 结构化数据读写

## 5.1 JSON

```python
import json

data = {"name": "张三", "age": 20, "tags": ["python", "backend"]}

# 写：对象 → 文件
with open("user.json", "w", encoding="utf-8") as f:
    json.dump(data, f, ensure_ascii=False, indent=2)

# 读：文件 → 对象
with open("user.json", encoding="utf-8") as f:
    loaded = json.load(f)

print(loaded["name"])   # 张三
```

`dumps`/`loads` 处理字符串，`dump`/`load` 处理文件对象——**结尾有没有 `s` 就是"字符串"和"流"的区别**。

⚠ **`ensure_ascii=False` 必须加**，否则中文变成 `\u5f20\u4e09`，能解析但不可读。`indent=2` 让输出便于查看。

★ **关键认知**：`json.load` 遇到格式错误抛 `json.JSONDecodeError`。读取外部 JSON 应捕获它，并打印 `err.msg` 和 `err.pos`——位置信息能快速定位是哪个字符写错（常见是多余逗号或引号）。

```python
try:
    with open("user.json", encoding="utf-8") as f:
        data = json.load(f)
except json.JSONDecodeError as err:
    print(f"JSON 解析失败：{err.msg}，位置 {err.pos}")
```

## 5.2 CSV

```python
import csv

rows = [["name", "age"], ["张三", 20], ["李四", 30]]

with open("users.csv", "w", encoding="utf-8", newline="") as f:
    csv.writer(f).writerows(rows)

with open("users.csv", encoding="utf-8", newline="") as f:
    reader = csv.DictReader(f)
    for row in reader:
        print(row["name"], row["age"])
```

```python
with open("out.csv", "w", encoding="utf-8", newline="") as f:
    writer = csv.DictWriter(f, fieldnames=["name", "age"])
    writer.writeheader()
    writer.writerow({"name": "王五", "age": 25})
```

⚠ **CSV 读出来的值全是字符串**，`"20"` 要自己转 `int`；空字符串和缺失字段也难以区分。结构化要求高时优先 JSON 或数据库。

★ **关键认知**：`csv` 模块负责引号、逗号转义等细节，**不要自己用 `split(",")` 解析 CSV**——字段里含逗号或换行就会错位。写 CSV 时 `newline=""` 是必须的，否则 Windows 上每行之间会多一个空行。

# 6. 常见错误

| 错误 | 现象 | 正确做法 |
|------|------|---------|
| 不写 `encoding` | Windows 乱码 | 显式 `utf-8` |
| 用 `"w"` 追加 | 原内容被清空 | 用 `"a"` |
| 相对路径 | 换目录就找不到 | `Path(__file__).parent` |
| 大文件 `read()` | 内存爆掉 | 逐行迭代 |
| CSV 用 `split` | 字段错位 | 用 `csv` 模块 |
| CSV 不写 `newline` | Windows 多空行 | `newline=""` |
| JSON `ensure_ascii` | 中文变 `\uXXXX` | `ensure_ascii=False` |

# 7. 练习

1. 用 `pathlib` 遍历当前目录所有 `.py` 文件，统计每个文件的行数，按行数从多到少打印文件名和行数。
2. 把含中文的字典写入 `config.json`，要求中文可读、缩进 2 空格；读回并断言内容一致。故意删掉一个逗号，捕获 `JSONDecodeError` 并打印错误位置。
3. 用 `csv.DictReader` 读取用户表，过滤年龄大于 25 的记录写入新 CSV，要求表头保持为 `name,age` 且 Windows 下不出现空行。

# 8. 小结

| 概念 | 要点 |
|------|------|
| `open` | 模式决定读/写/追加/清空 |
| 编码 | 永远显式 `utf-8` |
| 文本 vs 二进制 | 文本自动编解码，二进制拿 `bytes` |
| `with` | 保证异常路径也释放资源 |
| `pathlib` | `/` 拼接、`glob` 匹配、跨平台 |
| JSON | `dump/load` 对文件，`dumps/loads` 对字符串 |
| CSV | 用 `csv` 模块，`newline=""` |

下一篇进入标准库，讲清楚 `collections`、`itertools`、`functools`、`datetime`、`re`、`argparse` 这六个高频模块的典型用法与坑。
