CSV 没有类型，只有字符。解析做的第一件事是切分字段，第二件事是"猜"每个字段是什么类型——而猜错往往不会报错，只会让数据悄悄变样。本篇用 Python、Java、Go 三套主流做法，把读、写、类型推断和错误处理串起来。

# 1. 解析模型

## 1.1 三个层次

处理 CSV 可以分成三层，每层解决的问题不同：

```text
┌─────────────────────────────────────────┐
│ 词法层：按 RFC 4180 切分字段             │  引号、逗号、换行
├─────────────────────────────────────────┤
│ 结构层：把字段组织成行和列               │  字段数、表头
├─────────────────────────────────────────┤
│ 语义层：把字符串转换成目标类型           │  int、date、bool
└─────────────────────────────────────────┘
```

- **词法层**：决定 `"a,b"` 是一个字段还是两个，最容易写错。
- **结构层**：决定第一行是不是表头、每行字段数是否一致。
- **语义层**：决定 `"00123"` 是整数 `123` 还是字符串 `"00123"`。

★ **关键认知**：**这三层必须分开考虑**。很多人把 `split(",")` 和 `int()` 混在一起写，一旦遇到带引号的逗号就全盘崩溃。成熟库把词法层做对，把语义层交给你控制。

## 1.2 类型推断的本质

CSV 文件里没有 `int`、`float`、`date` 的标记，只有字节。所谓"类型推断"，是读取方**抽样一部分值，猜一个能装下它们的类型**，再应用到整列。

| 原因 | 例子 | 后果 |
|------|------|------|
| 样本不代表整体 | 前 1000 行是整数，第 1001 行是 `N/A` | 整列退化为字符串或报错 |
| 前导零丢失 | `00123` 被解析为 `123` | 邮编、工号被破坏 |
| 精度丢失 | 19 位订单号变成 `1.234e+18` | ID 对不上 |

★ **关键认知**：**推断只是便利，不是契约**。生产环境读取 CSV 时，应显式声明每列类型（`dtype`、struct 字段、schema），把"猜"变成"约定"。

# 2. Python 标准库 csv

## 2.1 reader 与 writer

`csv` 模块负责词法层，返回的每个字段都是 `str`，不做类型转换。

```python
import csv

with open("data.csv", newline="", encoding="utf-8") as f:
    reader = csv.reader(f)
    header = next(reader)          # 单独取出表头
    for row in reader:             # row 是 list[str]
        print(row)

rows = [
    ["id", "name", "note"],
    [1, "alice", "hello, world"],   # 含逗号，writer 会自动加引号
    [2, "bob", 'she said "hi"'],    # 含引号，writer 会自动转义
]
with open("out.csv", "w", newline="", encoding="utf-8") as f:
    csv.writer(f).writerows(rows)
```

★ **关键认知**：`open` 必须加 `newline=""`。否则 Python 的通用换行转换会把 `\r\n` 变成 `\r\r\n`，在 Windows 上生成带空行的 CSV。⚠ 也不要用 `f"{a},{b}"` 手拼 CSV，字段含逗号、引号、换行时必然出错。

## 2.2 DictReader 与 DictWriter

按列名访问比按位置可读性更好：

```python
import csv

with open("data.csv", newline="", encoding="utf-8") as f:
    for row in csv.DictReader(f):      # row 是 dict[str, str]
        print(row["name"], row["amount"])

fieldnames = ["id", "name", "amount"]
with open("out.csv", "w", newline="", encoding="utf-8") as f:
    writer = csv.DictWriter(f, fieldnames=fieldnames, extrasaction="ignore")
    writer.writeheader()               # 先写表头
    writer.writerow({"id": 1, "name": "alice", "amount": 100})
```

⚠ `DictWriter` 默认遇到 `fieldnames` 之外的键会抛 `ValueError`；要忽略多余键，传 `extrasaction="ignore"`。反过来，缺失的键会被写成空字符串。

## 2.3 方言与错误处理

`csv` 用"方言"（dialect）描述分隔符、引号等规则：

| 参数 | 默认值 | 作用 |
|------|--------|------|
| `delimiter` | `,` | 字段分隔符 |
| `quotechar` | `"` | 引号字符 |
| `quoting` | `QUOTE_MINIMAL` | 何时加引号 |
| `doublequote` | `True` | 用 `""` 转义引号 |
| `escapechar` | `None` | 反斜杠转义（非 RFC 风格） |

```python
import csv

try:
    with open("data.csv", newline="", encoding="utf-8") as f:
        for row in csv.reader(f):
            if len(row) != 3:
                raise ValueError(f"字段数不对：期望 3，实际 {len(row)}")
except UnicodeDecodeError as err:
    print("编码不对，尝试 utf-8-sig 或 gbk：", err)
except csv.Error as err:
    print("CSV 格式错误：", err)
```

⚠ 标准库对"未闭合引号"的处理比较宽松，可能把后面所有内容吞成一个字段而不是报错。遇到字段数突然变少，优先怀疑引号不配对。

# 3. pandas

## 3.1 读取与类型控制

pandas 在词法层之上做了结构化和向量化，适合分析场景。

```python
import pandas as pd

df = pd.read_csv(
    "data.csv",
    encoding="utf-8-sig",                # 自动剥离 BOM
    usecols=["id", "name", "amount", "created_at"],  # 只读需要的列
    dtype={"id": "string", "amount": "float64"},  # ID 按字符串，防前导零丢失
    parse_dates=["created_at"],
    na_values=["", "NULL", "-", "N/A"],  # 追加缺失标记
    nrows=1000,                          # 先读一小部分做探查
)
print(df.info())                         # 列名、非空数、dtype、内存
```

★ **关键认知**：**读入时指定 `dtype` 比读进来再 `astype` 更好**——既省一次全表转换的内存，又避免推断阶段就把 `00123` 变成 `123`。一旦前导零在读取时丢了，后面再怎么转字符串也补不回来。

## 3.2 写出与坏行处理

```python
df.to_csv(
    "out.csv",
    index=False,              # 不写行号，否则会多出 Unnamed: 0
    encoding="utf-8-sig",     # 带 BOM，Excel 打开中文不乱码
    na_rep="",                # 缺失值写成空字符串
    columns=["id", "name", "amount"],
)

# 跳过字段数不对的行，并打印告警
df = pd.read_csv("dirty.csv", on_bad_lines="warn")
```

⚠ 不要默认用 `skip` 静默丢数据。至少先用 `warn` 看一眼坏行长什么样，确认是脏数据而不是分隔符选错了。

# 4. Java

## 4.1 OpenCSV 读写

Java 标准库没有 CSV 解析器，很多人第一反应是 `line.split(",")`，但它在字段含逗号、换行、引号时全部失效。生产代码应使用 OpenCSV、Apache Commons CSV 或 Jackson CSV。

```java
import com.opencsv.CSVReader;
import com.opencsv.CSVWriter;
import java.io.Reader;
import java.nio.file.Files;
import java.nio.file.Path;

try (Reader reader = Files.newBufferedReader(Path.of("data.csv"));
     CSVReader csv = new CSVReader(reader)) {

    String[] header = csv.readNext();          // 第一行是表头
    String[] row;
    while ((row = csv.readNext()) != null) {   // 逐行读取，内存友好
        System.out.println(row[0] + " " + row[1]);
    }
}

try (CSVWriter writer = new CSVWriter(
        Files.newBufferedWriter(Path.of("out.csv")))) {
    writer.writeNext(new String[]{"id", "name", "amount"});
    writer.writeNext(new String[]{"1", "alice", "100"});
    writer.writeNext(new String[]{"2", "bob", "he said \"hi\""});
}
```

⚠ OpenCSV 的 `CsvToBean` 能把行映射成对象，但字段类型自动转换遇到空字符串会抛 `NumberFormatException`，需提前处理缺失值或改用包装类型。

# 5. Go

## 5.1 读取

Go 标准库的 `encoding/csv` 严格、可靠，默认要求每行字段数一致。

```go
package main

import (
	"encoding/csv"
	"fmt"
	"io"
	"os"
)

func main() {
	f, err := os.Open("data.csv")
	if err != nil {
		panic(err)
	}
	defer f.Close()

	r := csv.NewReader(f)
	r.FieldsPerRecord = -1     // 允许字段数不一致；默认 0 表示以首行为准

	header, err := r.Read()    // 读表头
	if err != nil {
		panic(err)
	}
	fmt.Println(header)

	for {
		record, err := r.Read()
		if err == io.EOF {
			break              // 正常结束
		}
		if err != nil {
			panic(err)         // 格式错误，含行号和列号
		}
		fmt.Println(record[0], record[1])
	}
}
```

★ **关键认知**：Go 的 `csv.Reader` 默认 `FieldsPerRecord = 0`，意思是"以第一条记录的字段数为标准"。后续行多一列或少一列都会返回 `ErrFieldCount`。这比静默补齐安全，但也意味着遇到脏文件要显式放宽。

## 5.2 写入与类型转换

```go
f, _ := os.Create("out.csv")
defer f.Close()

w := csv.NewWriter(f)
defer w.Flush()                // 必须 Flush，否则缓冲区内容不会落盘

records := [][]string{
	{"id", "name", "note"},
	{"1", "alice", "hello, world"},
	{"2", "bob", `she said "hi"`},
}
if err := w.WriteAll(records); err != nil {
	panic(err)
}
if err := w.Error(); err != nil {   // Flush 之后要检查错误
	panic(err)
}

id, err := strconv.ParseInt(record[0], 10, 64)   // Read() 返回 []string
if err != nil {
	return fmt.Errorf("第 %d 行 id 非法: %w", lineNo, err)
}
```

⚠ `strconv.Atoi("00123")` 会得到 `123`，和 pandas 推断一样会丢前导零。ID 类字段应原样保留字符串，或使用带格式化的编号类型。

# 6. 跨语言对照

| 能力 | Python csv | pandas | Java OpenCSV | Go encoding/csv |
|------|-----------|--------|--------------|-----------------|
| 词法合规 | 好 | 好 | 好 | 严格 |
| 类型转换 | 手动 | 自动推断 | Bean 自动 | 手动 |
| 逐行流式 | 是 | 分块 | 是 | 是 |
| 字段数校验 | 手动 | 补齐 | 可配置 | 默认严格 |
| 错误定位 | 弱 | 行号 | 异常信息 | 行号 + 列号 |
| 适合场景 | 脚本、小文件 | 分析 | 企业应用 | 服务、管道 |

★ **关键认知**：**不要在不同语言间假设相同行为**。同样是 `00123`，pandas 可能推断为整数，Go 原样保留字符串，Java Bean 映射成 `long` 又丢掉前导零。跨系统交换时，类型约定必须写在文档里。

# 7. 常见错误

| 错误 | 原因 | 解决 |
|------|------|------|
| `open` 没加 `newline=""` | 换行被二次转换 | 写 CSV 时总是加 |
| 忘记 `index=False` | 多出索引列 | 写出时显式关闭 |
| 坏行被静默跳过 | `on_bad_lines="skip"` | 先 `warn` 再决定 |
| 用 `split` 解析 | 忽略引号 | 使用标准库或专业库 |

# 8. 练习

1. 用 `csv.DictReader` 读取一份含 `id`、`name`、`amount` 的 CSV，统计 `amount` 的总和，并跳过 `amount` 为空的行。
2. 用 pandas 读取同一份文件，先不指定 `dtype` 看 `id` 的类型，再指定 `dtype={"id": "string"}` 对比结果，解释差异。
3. 用 Go 写一个程序，读 CSV 并统计每行字段数，把字段数不等于表头的行号打印出来。

# 9. 小结

| 主题 | 要点 |
|------|------|
| 解析分层 | 词法、结构、语义分开处理 |
| 类型推断 | 只是猜测，生产环境显式声明类型 |
| Python csv | `newline=""`、`DictReader`、`csv.Error` |
| pandas | `dtype`、`na_values`、`on_bad_lines`、`index=False` |
| Java | 用 OpenCSV/Commons CSV，别 `split`；Bean 注意空值 |
| Go | `FieldsPerRecord`、`Flush`、手动 `ParseInt` |
| 跨语言 | 类型行为不一致，必须显式约定 |

下一篇进入编码与引号，讲 UTF-8/BOM、Excel 兼容、字段转义细节，以及常被忽视的 CSV 注入风险。
