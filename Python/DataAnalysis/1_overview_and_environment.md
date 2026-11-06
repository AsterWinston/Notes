数据分析不是"会用几个函数"，而是一条从问题到结论、再到可复现结论的完整链路。本篇先把这条链路讲清楚，再把工具链和环境搭起来，后面九篇才有落脚点。

# 1. 数据分析在做什么

## 1.1 从问题到结论的工作流

一次正经的数据分析，通常要经过下面几步：

```text
┌────────────┐   ┌────────────┐   ┌────────────┐   ┌────────────┐
│ 明确问题    │ → │ 获取数据    │ → │ 清洗整理    │ → │ 探索分析    │
└────────────┘   └────────────┘   └────────────┘   └────────────┘
                                                          │
      ┌───────────────────────────────────────────────────┘
      ▼
┌────────────┐   ┌────────────┐   ┌────────────┐
│ 建模/统计   │ → │ 可视化表达  │ → │ 结论与复现  │
└────────────┘   └────────────┘   └────────────┘
```

每一步都可能回头：清洗时发现字段含义不对要改问题定义，探索时发现样本偏差要补数据。**它是带反馈环的循环，不是瀑布流程。**

★ **关键认知**：真正决定分析质量的往往不是模型，而是**问题定义和数据质量**。问题含糊，后面所有计算都是精致的废话；数据脏，模型越复杂错得越离谱。

## 1.2 一次分析的最小闭环

先看一个能跑通的最小例子，建立"输入 → 处理 → 输出"的直觉：

```python
import pandas as pd

# 构造一份极小的销售数据
df = pd.DataFrame({
    "city": ["北京", "上海", "北京", "广州", "上海", "北京"],
    "amount": [120, 80, 200, 50, 130, 90],
})

# 按城市汇总销售额，并从高到低排序
result = (
    df.groupby("city", as_index=False)["amount"]
      .sum()
      .sort_values("amount", ascending=False)
)

print(result)
#   city  amount
# 0   北京     410
# 1   上海     210
# 2   广州      50
```

这段代码已经覆盖了数据分析的三个核心动作：**读入结构化数据、按维度聚合、输出可解释结果**。后面的 Pandas 篇就是把它讲透。

★ **关键认知**：分析代码要能"从头跑到尾"。凡是依赖手工改中间结果、依赖某个单元格执行顺序的流程，都不算闭环，也无法复现。

# 2. 工具链分工

Python 数据栈看起来庞杂，真正的地基只有三个库。

## 2.1 NumPy：数值计算的地基

NumPy 提供 `ndarray`（多维同类型数组）和向量化运算。Pandas 的列、Matplotlib 的数据输入，底层几乎都是 NumPy 数组。

```python
import numpy as np

arr = np.array([1.0, 2.0, 3.0, 4.0])
print(arr.mean())        # 2.5
print(arr * 2)           # [2. 4. 6. 8.]，整数组一次运算
```

★ **关键认知**：NumPy 快，不是因为语法短，而是因为**同类型连续内存 + 底层 C 循环**。它的循环发生在 Python 之外，避开了逐元素解释执行的开销。

## 2.2 Pandas：表格数据的操作系统

Pandas 在 NumPy 之上加了"带标签的列"和"带标签的行索引"，提供类似电子表格但可编程的能力：筛选、连接、分组、透视、时间序列。

```python
import pandas as pd

s = pd.Series([10, 20, 30], index=["a", "b", "c"], name="score")
print(s["b"])            # 20，按标签取值
print(s.mean())          # 20.0
```

## 2.3 Matplotlib：把数字变成图

Matplotlib 负责画图。Seaborn 是它之上的统计绘图封装，语法更贴近 DataFrame。

```python
import matplotlib.pyplot as plt

plt.plot([1, 2, 3, 4], [1, 4, 9, 16], marker="o")
plt.xlabel("x")
plt.ylabel("y")
plt.title("square")
plt.show()
```

## 2.4 三者如何协作

```text
原始文件 CSV/Excel/JSON
        │
        ▼
┌──────────────┐   读入并整理     ┌──────────────┐
│    Pandas     │ ───────────────→ │   DataFrame   │
└──────┬───────┘                  └──────┬───────┘
       │ 计算落到 ndarray                 │ 画图
       ▼                                 ▼
┌──────────────┐                  ┌──────────────┐
│    NumPy      │                  │  Matplotlib   │
└──────────────┘                  └──────────────┘
```

| 库 | 负责 | 典型对象 | 不该拿它做什么 |
|----|------|---------|---------------|
| NumPy | 数值数组运算 | `ndarray` | 存混合类型表格 |
| Pandas | 表格清洗与分析 | `Series`/`DataFrame` | 大规模矩阵数值计算 |
| Matplotlib | 绘图 | `Figure`/`Axes` | 数据清洗与聚合 |

★ **关键认知**：三者是**分层**关系，不是替代关系。Pandas 负责"带标签的语义"，NumPy 负责"裸数值的高速计算"，Matplotlib 只负责"呈现"。选错层会让代码又慢又难读。

# 3. 环境搭建

## 3.1 Python 3.11 与虚拟环境

数据分析项目必须隔离依赖。不同项目对 Pandas、NumPy 的版本要求可能冲突，装进全局环境迟早出事。

先确认 `python --version` 输出为 Python 3.11.x。

## 3.2 用 venv 建虚拟环境

`venv` 是标准库自带方案，轻量、无需额外安装：

```bash
# 在项目目录下创建名为 .venv 的环境
python -m venv .venv

# Windows PowerShell 激活
.\.venv\Scripts\Activate.ps1

# macOS / Linux 激活
source .venv/bin/activate

# 退出环境
deactivate
```

激活后提示符前会出现 `(.venv)`，此时 `pip install` 只装进这个环境。

```bash
pip install "numpy>=2.0" "pandas>=2.2" "matplotlib>=3.8" "seaborn>=0.13" "jupyterlab>=4.0"
pip freeze > requirements.txt   # 锁定当前版本，便于复现
```

## 3.3 用 conda 建环境

conda 的优势是能同时管理 Python 版本和非 Python 依赖（如 MKL、编译器），适合科学计算环境：

```bash
# 创建名为 da 的环境，指定 Python 3.11
conda create -n da python=3.11 -y
conda activate da

# 从 conda-forge 安装，包更新更及时
conda install -c conda-forge numpy pandas matplotlib seaborn jupyterlab -y

# 导出与还原环境
conda env export > environment.yml
conda env create -f environment.yml
```

| 方案 | 优点 | 适合场景 |
|------|------|---------|
| venv + pip | 标准库自带、轻量、通用 | 纯 Python 项目、部署到服务器 |
| conda | 管理 Python 版本与非 Python 依赖 | 科学计算、需要特定二进制包 |

★ **关键认知**：**一个项目一个环境**。不要把所有库都装进 base 或全局环境。环境隔离是"半年后还能跑起来"的第一道保险。

⚠ **不要混用 conda 与 pip 装同一个包**。先用 conda 装能装的，剩下的再用 pip，否则依赖关系会被两套包管理器互相覆盖。

## 3.4 版本固定与复现

```text
项目根目录/
├── .venv/            # 虚拟环境，不提交到 git
├── data/raw/         # 原始数据，只读不改
├── data/processed/   # 清洗后的数据
├── notebooks/        # 探索性分析
├── src/              # 可复用的处理函数
├── requirements.txt  # 依赖清单
└── README.md         # 数据来源与运行方式
```

⚠ **`requirements.txt` 里写死版本范围**（如 `pandas==2.2.3`），不要只写 `pandas`。否则半年后一次 `pip install` 就可能装到行为不兼容的新版本。

# 4. Jupyter：交互式分析的主场

## 4.1 为什么用 Jupyter

Jupyter Notebook 把代码、输出、图表、说明放在同一文档里，适合探索阶段：改一行、看一次结果、再改。分析思路还没定型时，这种即时反馈比"改脚本 → 重跑全流程"高效得多。

```bash
# 启动 JupyterLab（在已激活的环境中）
jupyter lab
```

## 4.2 单元格与执行顺序

Notebook 由单元格（cell）组成，分代码、Markdown 两类。代码单元格按你**手动执行的顺序**运行，而不是按页面从上到下的顺序。

```python
# 单元格 1
x = 10

# 单元格 2
x = x + 5

# 单元格 3
print(x)      # 若先跑 1、3，结果是 10；跑全 1、2、3 才是 15
```

★ **关键认知**：Notebook 的内核是一个**有状态的 Python 进程**，变量不会因为页面刷新而重置。所谓的"执行顺序"是隐藏状态，这正是 Notebook 最常见 bug 的来源。

## 4.3 使用纪律

- 交付前执行 **Restart Kernel and Run All**，确认从头到尾无报错、结果一致。
- 编号靠前的单元格不要依赖后面才定义的变量。
- 探索性代码留在 notebook，稳定的清洗逻辑抽到 `src/` 里的函数，供 notebook 调用。
- 输出大的 DataFrame 用 `.head()`，别让几万行把文件撑爆。

⚠ **删除单元格或改变执行顺序后，旧变量仍然留在内存里**。看到的"正确结果"可能来自已经不存在的代码。凡是结果可疑，先重启内核再跑。

# 5. 一个完整的最小示例

把读入、清洗、聚合、绘图串起来：

```python
import pandas as pd
import matplotlib.pyplot as plt

# 1. 读入
df = pd.read_csv("data/raw/sales.csv")

# 2. 初探：看形状、列名、类型、缺失
print(df.shape)
print(df.dtypes)
print(df.isna().sum())

# 3. 清洗：日期转类型、金额转数值、丢弃金额缺失行
df["date"] = pd.to_datetime(df["date"], errors="coerce")
df["amount"] = pd.to_numeric(df["amount"], errors="coerce")
df = df.dropna(subset=["date", "amount"])

# 4. 聚合：按月汇总
monthly = (
    df.set_index("date")
      .resample("ME")["amount"]
      .sum()
)

# 5. 输出
print(monthly)
monthly.plot(kind="line", marker="o", title="Monthly Sales")
plt.tight_layout()
plt.savefig("monthly_sales.png", dpi=150)
plt.show()
```

这段代码的每一行在后续都有专篇展开：`pd.to_datetime`、`to_numeric` 在清洗篇，`resample` 在时间序列篇，`plot` 与 `savefig` 在可视化篇。

★ **关键认知**：分析脚本的正确顺序是**先体检、再清洗、后分析**。上来就 `groupby` 往往是在给脏数据做无意义的汇总。

# 6. 常见错误

| 错误 | 现象 | 正确做法 |
|------|------|---------|
| 全局装一堆包 | 项目之间版本互相踩踏 | 一项目一虚拟环境 |
| Notebook 不按顺序跑 | 结果无法复现 | 交付前 Restart & Run All |
| 只写 `pandas` 不锁版本 | 半年后行为不一致 | 固定版本号 |
| 直接改 `data/raw` | 原始数据被破坏、无法回溯 | 原始数据只读，产物写 `processed` |
| 用 Pandas 做逐行循环 | 慢到不可用 | 向量化或 `groupby` |

# 7. 练习

1. 用 `venv` 建一个环境，安装 NumPy、Pandas、Matplotlib，导出 `requirements.txt`，再删掉环境重建一次，确认能还原。
2. 在 Jupyter 里创建三个单元格，故意打乱执行顺序，观察变量取值变化；然后 Restart & Run All，对比结果。

# 8. 小结

| 主题 | 要点 |
|------|------|
| 分析流程 | 问题 → 获取 → 清洗 → 探索 → 建模 → 表达 → 复现，带反馈环 |
| NumPy | 同类型连续内存，向量化运算的地基 |
| Pandas | 带标签的表格，负责清洗、聚合、连接、时间序列 |
| Matplotlib | 只负责呈现，Seaborn 是其统计封装 |
| 环境 | 一项目一环境，锁版本，原始数据只读 |

下一篇进入 NumPy，从 `ndarray` 的内存布局讲起，把 shape、dtype、索引切片、广播和向量化一次讲透，为后面所有 Pandas 操作打地基。
