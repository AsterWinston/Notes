GB 级 CSV 最容易出的事故不是"算得慢"，而是"内存直接爆"。`pd.read_csv("huge.csv")` 一行代码，可能瞬间吃掉几十 GB，然后进程被 OOM Killer 杀掉。本篇讲清楚内存为什么会被放大、如何逐行与分块、压缩文件的限制，以及并行处理到底能带来什么。

# 1. 为什么大文件会炸

## 1.1 内存放大

CSV 在磁盘上是一串字节，读进内存后要变成对象。这个过程会**放大数倍**：

```text
磁盘上的 CSV          内存中的 DataFrame
┌──────────────┐      ┌─────────────────────────────┐
│ 1 GB 文本     │  →   │ 字符串对象 + 索引 + dtype   │
└──────────────┘      │ 实测常为 3~10 倍            │
                      └─────────────────────────────┘
```

放大来源：

| 来源 | 说明 |
|------|------|
| 字符串对象开销 | Python `str` 对象头 + 引用，远大于字节长度 |
| 列式存储 | 每列单独分配数组，空值还要额外标记 |
| 中间副本 | 读取、转换、筛选中途产生临时对象 |
| 类型推断 | 为推断类型先按 object 存一遍 |

★ **关键认知**：**CSV 的文件大小不能直接当作内存需求**。至少预留 3 倍，字符串列多、缺失值多的文件可达 10 倍。先看文件大小，再决定"全量读"还是"流式读"。

## 1.2 一次性读入的代价

```python
import pandas as pd

df = pd.read_csv("huge.csv")   # 危险：可能直接 OOM
```

它会依次读入全部字节、切分字段、推断类型并构建完整 DataFrame，每一步都要在内存里持有全量数据。文件 20 GB、放大 4 倍，就需要 80 GB 内存，再快的机器也扛不住。

# 2. 逐行读取

## 2.1 流式模型

流式处理的核心是：**同一时刻内存里只有一小块数据**。

```text
磁盘 ──► 缓冲区(几 KB) ──► 处理/聚合 ──► 丢弃
                 ▲
           只保留当前行
```

## 2.2 Python 生成器

```python
import csv

def iter_rows(path: str):
    with open(path, newline="", encoding="utf-8-sig") as f:
        reader = csv.DictReader(f)
        for row in reader:
            yield row          # 一次只产出一行，调用方消费完即可释放

total = 0
count = 0
for row in iter_rows("huge.csv"):
    amount = row.get("amount") or "0"
    try:
        total += float(amount)
    except ValueError:
        continue               # 脏值跳过，记录日志更好
    count += 1

print(total, count)
```

★ **关键认知**：`for row in reader` 本身就是流式的，**不需要额外写生成器也已经是逐行**。生成器的价值在于把"读"和"处理"解耦，方便组合过滤、映射等多步逻辑。

⚠ 不要在循环里把每行 append 到列表。`rows.append(row)` 会让内存重新回到全量，前面的流式就白做了。要保留结果就做聚合（求和、计数、Top-N），或分批落盘。

## 2.3 Go 的流式读取

Go 的 `encoding/csv` 天然流式，不需要把整个文件读进内存：

```go
package main

import (
	"encoding/csv"
	"fmt"
	"io"
	"os"
)

func main() {
	f, err := os.Open("huge.csv")
	if err != nil {
		panic(err)
	}
	defer f.Close()

	r := csv.NewReader(f)
	r.ReuseRecord = true       // 复用底层切片，减少分配
	r.FieldsPerRecord = -1     // 允许列数不一致

	var total float64
	for {
		record, err := r.Read()
		if err == io.EOF {
			break
		}
		if err != nil {
			fmt.Println("坏行：", err)
			continue
		}
		var v float64
		if _, err := fmt.Sscanf(record[2], "%f", &v); err == nil {
			total += v
		}
	}
	fmt.Println(total)
}
```

⚠ `ReuseRecord = true` 会复用同一块内存，**上一行的切片内容会被下一行覆盖**。如果你要保留行数据，必须自己拷贝，否则会拿到错乱的结果。

# 3. 分块处理

## 3.1 pandas chunksize

`read_csv` 的 `chunksize` 返回一个迭代器，每次给出一个 DataFrame：

```python
import pandas as pd

total = 0
for chunk in pd.read_csv(
    "huge.csv",
    chunksize=100_000,          # 每块 10 万行
    usecols=["id", "amount"],   # 只读需要的列
    dtype={"id": "string", "amount": "float64"},
    encoding="utf-8-sig",
):
    total += chunk["amount"].sum()

print(total)
```

★ **关键认知**：`chunksize` 是**行的数量**，不是字节数。每块实际占多少内存取决于列数和类型，10 万行宽表可能比 100 万行窄表更吃内存。要按实测调整，不要照抄别人的数字。

## 3.2 分块聚合与落盘

分块读取后，常见两种去向：

| 去向 | 做法 | 适用 |
|------|------|------|
| 聚合 | 累加、计数、分组后合并 | 求和、去重、统计 |
| 落盘 | 处理后追加写入新文件 | 清洗、转换、过滤 |

```python
# 过滤后分批写出，避免一次性持有结果
first = True
for chunk in pd.read_csv("huge.csv", chunksize=100_000):
    filtered = chunk[chunk["amount"] > 100]
    filtered.to_csv(
        "clean.csv",
        mode="a",               # 追加模式
        header=first,           # 只有第一块写表头
        index=False,
        encoding="utf-8-sig",
    )
    first = False
```

⚠ 追加写要注意三件事：第一块写表头、后续不写；`mode="a"` 依赖上一块以换行结尾，否则会粘连；中途失败会留下半个文件。生产环境应先写临时文件，成功后再原子重命名。

# 4. 压缩

## 4.1 透明读取

pandas 能根据扩展名自动识别压缩格式：

```python
df = pd.read_csv("huge.csv.gz", chunksize=100_000)      # gzip
df = pd.read_csv("huge.csv.bz2")                        # bzip2
df = pd.read_csv("huge.csv.zip")                        # zip
df = pd.read_csv("huge.csv.xz")                         # xz
```

也可以显式指定：`df = pd.read_csv("data.bin", compression="gzip")`。

## 4.2 压缩文件的限制

★ **关键认知**：**gzip 是流式压缩，不支持随机访问**。你不能"跳到第 500 万行"直接读，必须先解压前面的全部内容。这直接影响并行切分：**压缩文件不能按字节切片并行**。

| 格式 | 可随机访问 | 压缩率 | 适合 |
|------|-----------|--------|------|
| gzip | 否 | 高 | 归档、传输 |
| bzip2 | 否 | 更高 | 冷数据 |
| xz | 否 | 最高 | 长期存储 |

Go 读取 gzip：

```go
import (
	"compress/gzip"
	"encoding/csv"
	"os"
)

f, _ := os.Open("huge.csv.gz")
defer f.Close()

gz, err := gzip.NewReader(f)   // 解压流
if err != nil {
	panic(err)
}
defer gz.Close()

r := csv.NewReader(gz)         // 直接对接解压流
```

# 5. 并行处理边界

## 5.1 为什么不能按字节切

很多人想"把文件切成 8 段，8 个进程各读一段"。对普通文本可以，对 CSV 不行，原因有两个：

1. **字段内可能有换行**（引号包裹），按行切也会切断记录。
2. **按字节切会切在字段中间**，每段开头都是半条记录。

```text
原始文件：
id,note
1,"line1
line2"
2,"a,b"

按固定字节切成两段：
段 A: id,note\n1,"line1
段 B: line2"\n2,"a,b"     ← 两条记录都残了
```

## 5.2 可行的切分方式

| 方式 | 做法 | 代价 |
|------|------|------|
| 预扫描边界 | 先流式扫一遍，记录每条记录起始偏移 | 多读一遍文件 |
| 先转 Parquet | 一次性转换，后续按 row group 并行 | 首次转换成本 |
| 按文件切 | 输入本身就是多个小文件 | 依赖上游分片 |
| 用支持并行的引擎 | DuckDB、Polars、Spark | 引入依赖 |

★ **关键认知**：**CSV 不是可切分（splittable）格式**。它的记录边界依赖引号状态，必须顺序扫描才能确定。所谓"并行读 CSV"，要么先建立索引，要么换格式，没有免费的午餐。

## 5.3 什么时候值得并行

先判断瓶颈在哪：

| 瓶颈 | 表现 | 并行的收益 |
|------|------|-----------|
| 磁盘 I/O | CPU 空闲、磁盘打满 | 很小，甚至更慢 |
| 单核解析 | 一个核 100%，其他空闲 | 明显 |
| 后续计算 | 解析快、计算慢 | 明显，且应并行计算而非解析 |
| 内存带宽 | 大对象频繁拷贝 | 视情况 |

对于 Python，还要考虑 GIL：`csv` 解析与字符串处理是 CPU 密集的，多线程难以提速，通常用多进程；而 pandas 的向量化计算在底层释放 GIL，效果更好。

⚠ 并行会带来顺序丢失、错误定位困难、临时文件管理复杂等成本。**先用流式把内存压下来，再考虑并行**；多数"大文件慢"的问题，其实靠 `usecols`、`dtype`、分块就能解决。

# 6. 常见错误

| 错误 | 后果 | 解决 |
|------|------|------|
| 直接 `read_csv` 全量 | OOM | `chunksize` 或逐行 |
| 循环里 `append` 所有行 | 内存回到全量 | 只做聚合或分批落盘 |
| 用 `readlines()` 切行 | 引号内换行被切断 | 用 CSV 解析器 |
| 按字节切分并行 | 记录错位 | 先扫描边界或转 Parquet |
| 对 gzip 做随机 seek | 解压后偏移不对 | 顺序流式读 |
| 追加写重复表头 | 中间混入表头行 | 仅第一块写 header |

# 7. 练习

1. 用 `csv.DictReader` 流式读取一个百万行 CSV，统计某数值列的总和与行数，要求内存占用不随文件增大而增长。
2. 用 `chunksize=50000` 分块读取同一文件，按城市分组求和，比较两种写法的代码复杂度与内存。
3. 把一个 CSV 用 gzip 压缩，分别测试"直接读 gzip"和"先解压再读"的耗时，并解释差异。

# 8. 小结

| 主题 | 要点 |
|------|------|
| 内存放大 | 文件大小乘 3 到 5 倍才是内存需求 |
| 逐行 | `for row in reader` 天然流式，勿累积 |
| 分块 | `chunksize` 按行，聚合或分批落盘 |
| 压缩 | gzip 等不支持随机访问，只能顺序读 |
| 并行 | CSV 不可按字节切，需先建边界或换格式 |
| 优化顺序 | 先 `usecols`/`dtype`/分块，再谈并行 |

下一篇进入最佳实践与总结，把 schema 约定、校验、缺失值、数据质量和安全串成一套可落地的工程规范。
