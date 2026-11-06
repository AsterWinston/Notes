分析结论最终要给人看。一张图如果让人三秒内看不懂，问题通常不在数据，而在图表类型、坐标轴、颜色或标注。本篇讲清 Matplotlib 的结构、Seaborn 的定位、中文显示和保存规范，目标是画出"能直接放进报告"的图。

# 1. 先选对图

## 1.1 按问题选图表

| 想表达 | 首选图 | 不推荐 |
|--------|--------|--------|
| 随时间变化 | 折线图 | 饼图 |
| 类别比较 | 条形图 | 折线图 |
| 分布形态 | 直方图、箱线图、小提琴图 | 饼图 |
| 两变量关系 | 散点图 | 条形图 |
| 构成占比 | 堆叠条形图、100% 堆叠 | 3D 饼图 |
| 相关性矩阵 | 热力图 | 折线图 |
| 多组分布对比 | 箱线图、分组小提琴图 | 多张饼图 |

★ **关键认知**：图表类型由**变量类型和问题**决定，不由"哪个好看"决定。时间用折线、类别用条形、分布用直方/箱线、关系用散点——这条对应关系能解决八成的选择困难。

## 1.2 一张图的解剖

```text
┌──────────────────────────────┐
│  Title（结论，不是字段名）      │
│                              │
│  │            ●               │
│  │      ●  ●                 │  ← 数据区（Axes）
│  │  ●                        │
│  └───────────────────────    │
│    X label     Y label       │
│    Legend / Annotation       │
└──────────────────────────────┘
```

⚠ **标题写结论，不写字段名**。`"2026 年月度销售额"` 只是描述，`"Q3 销售额环比增长 18%"` 才是信息。图是论点，不是数据容器。

# 2. Matplotlib 基础

## 2.1 Figure 与 Axes

Matplotlib 有两层对象：

- `Figure`：整张画布，可包含多个子图。
- `Axes`：一个具体的绘图区，有坐标轴、刻度、图例。

```python
import matplotlib.pyplot as plt

fig, ax = plt.subplots(figsize=(8, 4))
ax.plot([1, 2, 3, 4], [1, 4, 9, 16], marker="o")
ax.set_xlabel("x")
ax.set_ylabel("y")
ax.set_title("平方关系")
plt.show()
```

★ **关键认知**：**优先用面向对象写法**（`fig, ax = plt.subplots()` 后调 `ax.xxx`），而不是 `plt.plot` 全局状态。多个子图、复杂布局、函数封装时，`ax` 写法不会互相污染。

## 2.2 多子图布局

```python
fig, axes = plt.subplots(1, 2, figsize=(10, 4))

axes[0].plot([1, 2, 3], [1, 4, 9])
axes[0].set_title("左图")

axes[1].bar(["A", "B", "C"], [3, 7, 5])
axes[1].set_title("右图")

fig.suptitle("两个子图", fontsize=14)
fig.tight_layout()      # 自动调整间距，防止标签重叠
plt.show()
```

## 2.3 常用图

```python
import numpy as np
import matplotlib.pyplot as plt

x = np.linspace(0, 10, 50)
y = np.sin(x)

fig, axes = plt.subplots(2, 3, figsize=(12, 6))

axes[0, 0].plot(x, y)                          # 折线
axes[0, 1].scatter(x, y, s=15)                 # 散点
axes[0, 2].bar(["A", "B", "C"], [3, 7, 5])     # 条形
axes[1, 0].hist(np.random.randn(500), bins=20) # 直方图
axes[1, 1].boxplot([np.random.randn(100), np.random.randn(100) + 1])
axes[1, 2].pie([30, 20, 50], labels=["A", "B", "C"], autopct="%1.0f%%")

fig.tight_layout()
plt.show()
```

## 2.4 样式与可读性

```python
fig, ax = plt.subplots(figsize=(8, 4))

ax.plot(x, y, label="sin(x)", color="#2E86AB", linewidth=2)
ax.plot(x, np.cos(x), label="cos(x)", color="#E4572E", linestyle="--")

ax.set_xlabel("x")
ax.set_ylabel("y")
ax.set_title("正弦与余弦")
ax.legend(loc="upper right", frameon=False)
ax.grid(axis="y", alpha=0.3)      # 只保留横向网格，减少干扰
ax.spines[["top", "right"]].set_visible(False)   # 去掉上右边框

fig.tight_layout()
plt.show()
```

⚠ **不要同时用网格、边框、背景色、阴影、3D 效果**。图表里的每一滴墨水都要么传递信息，要么删掉。

# 3. Seaborn

## 3.1 定位

Seaborn 建在 Matplotlib 之上，优势是**直接吃 DataFrame**、默认样式更现代、统计图（分布、回归、分类）开箱即用。

```python
import seaborn as sns
import matplotlib.pyplot as plt

sns.set_theme(style="whitegrid")     # 统一主题
```

## 3.2 常用统计图

```python
import pandas as pd
import seaborn as sns

df = sns.load_dataset("tips")        # 内置示例数据

# 分布
sns.histplot(data=df, x="total_bill", bins=20)

# 分类比较
sns.boxplot(data=df, x="day", y="total_bill", hue="smoker")

# 关系
sns.scatterplot(data=df, x="total_bill", y="tip", hue="time")

# 相关性热力图
corr = df.select_dtypes("number").corr()
sns.heatmap(corr, annot=True, fmt=".2f", cmap="RdBu_r", center=0)
```

| 需求 | Seaborn 函数 |
|------|-------------|
| 单变量分布 | `histplot` / `kdeplot` |
| 分组分布 | `boxplot` / `violinplot` |
| 两变量关系 | `scatterplot` / `lineplot` |
| 分类计数 | `countplot` |
| 相关性 | `heatmap` |
| 回归趋势 | `regplot` / `lmplot` |

★ **关键认知**：Seaborn 负责"快速画出统计结构"，Matplotlib 负责"精细控制"。**探索阶段用 Seaborn，交付阶段用 Matplotlib 调细节**，两者不冲突，Seaborn 画完仍可拿 `ax` 继续改。

⚠ **`sns.load_dataset` 需要联网下载示例数据**，离线环境会失败。生产代码不要依赖它，换成自己的 DataFrame。

# 4. 中文显示

Matplotlib 默认字体不含中文，直接写中文会显示成方框。需要指定支持中文的字体：

```python
import matplotlib.pyplot as plt

# Windows 常用字体
plt.rcParams["font.sans-serif"] = ["Microsoft YaHei", "SimHei"]
# macOS 可换 "PingFang SC"，Linux 可换 "Noto Sans CJK SC"
plt.rcParams["axes.unicode_minus"] = False   # 解决负号显示成方块
```

设置后重新绘图即可。若仍乱码，先清缓存：

```python
import matplotlib
print(matplotlib.get_cachedir())   # 删除该目录下的字体缓存后重启内核
```

★ **关键认知**：`font.sans-serif` 要写**系统里真实存在的字体名**。写一个不存在的字体不会报错，只会静默回退到默认字体，于是继续乱码。用 `matplotlib.font_manager` 查可用字体：

```python
from matplotlib import font_manager

names = {f.name for f in font_manager.fontManager.ttflist}
print([n for n in names if "Hei" in n or "YaHei" in n])
```

⚠ **`axes.unicode_minus=False` 必须配**。中文字体里的负号和 ASCII 负号不是同一个字形，不加这行，负数刻度会变成方框。

# 5. 保存与可访问性

## 5.1 保存

```python
fig, ax = plt.subplots(figsize=(8, 4))
ax.plot([1, 2, 3], [2, 4, 3])
ax.set_title("示例")

fig.savefig(
    "chart.png",
    dpi=200,                 # 报告用 200，印刷用 300
    bbox_inches="tight",     # 裁掉多余白边
    transparent=False,
    facecolor="white",
)
```

- 位图用 `png`，矢量图用 `svg` 或 `pdf`。
- 保存要在 `plt.show()` **之前**，`show()` 之后画布可能被清空。
- 报告里优先 `svg`，放大不糊。

## 5.2 可访问性

| 做法 | 原因 |
|------|------|
| 不只用颜色区分 | 约 8% 男性有红绿色弱 |
| 颜色 + 形状/线型双编码 | 黑白打印也能分辨 |
| 对比度足够 | 浅灰字配白底看不清 |
| 字号不小于 9pt | 投影和打印可读 |
| 给出数据来源与单位 | 图脱离正文也能读懂 |

```python
# 色盲友好调色板
sns.color_palette("colorblind")

# 同时用线型区分
ax.plot(x, y1, linestyle="-", label="A")
ax.plot(x, y2, linestyle="--", label="B")
```

★ **关键认知**：**颜色是辅助编码，不是唯一编码**。用颜色区分组别时，再叠加线型、标记点或直接标注，图在灰度打印和色盲读者眼里才仍然可读。

⚠ **不要用彩虹色（jet）表示连续数值**。它的亮度变化不均匀，会制造出并不存在的"分界"。连续量用 `viridis`、`cividis` 这类感知均匀的色图。

# 6. 常见坑

| 坑 | 现象 | 正确做法 |
|----|------|---------|
| 中文变方框 | 缺中文字体 | 设 `font.sans-serif` + `unicode_minus` |
| 保存是空白 | `show()` 后保存 | 先 `savefig` 再 `show` |
| 标签被裁掉 | 边距不足 | `bbox_inches="tight"` |
| 多图互相覆盖 | 用全局 `plt` 状态 | `fig, ax` 面向对象 |
| 饼图类别太多 | 无法比较 | 改条形图 |
| 纵轴不从 0 开始 | 夸大差异 | 条形图必须从 0 起 |
| 用彩虹色图 | 制造假分界 | 用 `viridis` |
| 只靠颜色区分 | 色盲/黑白不可读 | 加线型或标记 |

⚠ **条形图纵轴截断是误导**。从 90 开始画，10 和 12 的差异看起来像十倍。折线图可以按需截断，条形图不行——长度本身是编码。

# 7. 练习

1. 用同一份月度销售数据，分别画折线图和条形图，说明哪种更适合表达趋势、哪种更适合比较单月高低。
2. 配置中文字体，画一张带负数的折线图，确认标题、图例、负号都正常显示，并保存为 `svg`。
3. 用 Seaborn 画分组箱线图，同时用 `hue` 和不同标记区分组别，检查转成灰度后是否仍能分辨。

# 8. 小结

| 主题 | 要点 |
|------|------|
| 选图 | 时间折线、类别条形、分布直方/箱线、关系散点 |
| Figure/Axes | 画布与绘图区分离，优先面向对象写法 |
| 布局 | `subplots` + `tight_layout` |
| Seaborn | 直接吃 DataFrame，统计图开箱即用 |
| 中文 | 指定真实存在的字体，关闭 `unicode_minus` |
| 保存 | 先 `savefig`，`dpi` 与 `bbox_inches` 按用途选 |
| 可访问性 | 颜色 + 形状双编码，避免彩虹色和截断条形轴 |

下一篇进入性能与内存，讲清向量化为什么快、dtype 如何影响内存、分块读取怎么做、Pandas 的 copy 与视图机制，以及如何定位真正的性能瓶颈。
