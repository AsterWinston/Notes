# 1. 为什么需要采样

结论先行：语言模型每一步只输出一个词表上的概率分布，如何从这个分布里挑出下一个 token（词元），直接决定了输出的稳定性与多样性。贪心解码稳定但死板，采样则更自然多样。

## 1.1 贪心解码的局限

贪心解码（greedy decoding）每次都取概率最大的 token，是最简单的解码方式，但它有系统性缺陷。

### 1.1.1 重复与死循环

贪心倾向于反复选中同一批高概率 token，长文本里容易出现复读现象。当某段上下文把某个短语的概率推到极高时，模型会不断重复它，甚至进入"今天天气天气天气……"式的死循环。原因是贪心不回溯、不带随机性，一旦落入局部最优就很难跳出。

### 1.1.2 缺乏多样性

同一个 prompt（提示词）下，贪心永远给出同一条输出。对创意写作、头脑风暴、对话等任务，这等于浪费了模型分布里的大量可能性。

```
贪心（每次取最大概率）的问题：

  - 输出重复、死板
  - 缺乏多样性
  - 可能陷入循环

采样：按概率分布随机选择 → 更自然多样
```

## 1.2 采样的数学本质

### 1.2.1 从确定性到随机性

设模型在第 t 步输出 logits（未归一化得分）z，经过 softmax（归一化指数函数）得到概率 p：

```
p_i = exp(z_i) / sum_j exp(z_j)
```

贪心取 argmax_i p_i；采样则把 p 当作一个真正的分布，按 p_i 的概率抽取下标 i。前者是后者的退化情形。

### 1.2.2 语言模型即概率分布

自回归生成把整句概率分解为条件概率的连乘：

```
P(x_1, ..., x_T) = prod_t P(x_t | x_1, ..., x_{t-1})
```

每一步的 P(x_t | 前缀) 就是我们要采样的分布。采样策略的全部工作，就是把这个分布"改造"成更适合任务的形式，再抽样。

## 1.3 自回归生成流程

```
输入 prompt
     |
     v
[模型前向] --> logits (词表大小)
     |
     v
[采样策略]  temperature / top-k / top-p / 惩罚
     |
     v
   选中 token --> 追加到序列 --> 下一轮
```

- 边界与坑：采样只改变"选谁"，不改变模型本身；它不能凭空创造分布里没有的 token（除非人为加噪声）。
- 生产建议：把采样参数视为与 prompt 同等重要的接口参数，纳入版本管理与评测。

# 2. Temperature（温度）

结论先行：温度 T 是唯一作用在 softmax 之前的标量，通过缩放 logits 改变分布的"尖锐程度"，是最基础也最重要的采样旋钮。

## 2.1 定义与公式

### 2.1.1 softmax 回顾

原始分布由 logits 经 softmax 得到。温度在 softmax 之前对 logits 做除法：

```
probs = softmax(logits / T)
```

### 2.1.2 温度如何改变分布

- T → 0 时，最大 logit 的优势被无限放大，分布退化为 one-hot，等价贪心。
- T = 1 时，保持模型原始分布。
- T → ∞ 时，分布趋于均匀，等价随机从词表里均匀抽取。

从信息论看，温度单调地改变分布的熵（entropy，不确定性度量）：T 越大熵越高，T 越小熵越低。温度本质上是在"确定"和"随机"之间连续调节。

```
logits = [2.0, 1.0, 0.0]

T = 0.5: [0.867, 0.117, 0.016]  尖锐
T = 1.0: [0.665, 0.245, 0.090]  原始
T = 2.0: [0.506, 0.307, 0.187]  平坦
```

## 2.2 温度区间效果

```
T < 1：分布更尖锐 → 更确定
T = 1：原始分布
T > 1：分布更平坦 → 更随机
```

| 温度 | 效果 | 适用 |
|:----:|------|------|
| 0~0.3 | 确定、稳定 | 事实问答、代码 |
| 0.7 | 平衡 | 通用对话 |
| 1.0~2.0 | 随机、多样 | 创意写作 |

## 2.3 实现

```python
def apply_temperature(logits, temperature):
    if temperature == 0:
        return logits.argmax(-1)     # 贪心
    return logits / temperature
```

```java
static double[] applyTemperature(double[] logits, double temperature) {
    if (temperature == 0.0) {
        return new double[]{argmax(logits)};   // 贪心，返回索引
    }
    double[] scaled = new double[logits.length];
    for (int i = 0; i < logits.length; i++) {
        scaled[i] = logits[i] / temperature;
    }
    return scaled;
}

static int argmax(double[] a) {
    int best = 0;
    for (int i = 1; i < a.length; i++) {
        if (a[i] > a[best]) best = i;
    }
    return best;
}
```

## 2.4 边界与坑

- T=0 在实现里通常特判为贪心，直接除法会除零。
- 温度过高（>1.5）会让低概率垃圾 token 大量进入，事实性骤降。
- 温度会与 top-p 相互作用：先除温度再截断，截断集合会随温度变化。
- 不同框架对 temperature=0 的处理不同（有的报错，有的走贪心），需确认。

## 2.5 生产建议

- 事实问答、代码、工具调用：T ∈ [0, 0.3]。
- 通用对话：T ≈ 0.7。
- 创意写作：T ∈ [0.9, 1.2]，很少需要超过 1.2。
- 把温度写进配置而非硬编码，便于按任务切换。

# 3. Top-k 采样

结论先行：Top-k 固定保留概率最高的 k 个 token 后重新归一化采样，用固定预算剪掉长尾噪声，简单但不够自适应。

## 3.1 原理

只在概率最高的 k 个候选里采样，其余置为负无穷，再在截断集合上重新归一化。

```
原始分布: [0.3, 0.2, 0.15, 0.1, 0.08, ...]
Top-k=3:  [0.3, 0.2, 0.15] → 重新归一化后采样
```

重归一化公式：

```
p'_i = p_i / sum_{j in TopK} p_j
```

截断采样会"放大头部"：原本只有 0.15 的 token，归一化后概率被抬升，比原始分布更容易被抽中。

## 3.2 实现

```python
def top_k_filter(logits, k):
    top_k_values, _ = torch.topk(logits, k)
    threshold = top_k_values[:, -1:]
    logits[logits < threshold] = float('-inf')
    return logits
```

```java
static void topKFilter(double[] logits, int k) {
    int n = logits.length;
    Integer[] idx = new Integer[n];
    for (int i = 0; i < n; i++) idx[i] = i;
    java.util.Arrays.sort(idx, (a, b) -> Double.compare(logits[b], logits[a]));
    double threshold = logits[idx[k - 1]];
    for (int i = 0; i < n; i++) {
        if (logits[i] < threshold) logits[i] = Double.NEGATIVE_INFINITY;
    }
}
```

## 3.3 对比

| 优点 | 缺点 |
|------|------|
| 排除低概率噪声 | k 固定，不适应分布 |
| 实现简单、开销小 | 尖锐分布时可能引入过多候选 |

## 3.4 边界与坑

- k 太小 → 接近贪心；k 太大 → 噪声回归。
- k 固定是它的根本弱点：分布尖锐时 k=50 相当于把大量垃圾放进候选；分布平坦时 k=50 又可能截断过多有效候选。
- 与 top-p 同时启用时，谁先谁后由框架决定，结果可能不同。

## 3.5 生产建议

- 作为兜底保护：k 取 20~100，防止极端低概率 token。
- 现代实践中常直接用 top-p 取代 top-k，二者择一。
- 记录实际生效的 k 值，便于复现。

# 4. Top-p（核采样）

结论先行：Top-p 又称核采样（nucleus sampling），按累积概率动态选取"最小充分集合"，集合大小随分布自适应，是目前最推荐的截断方式。

## 4.1 原理

### 4.1.1 累积概率

把概率从大到小排序，累加直到超过阈值 p，取这一段作为候选集（核）：

```
排序后概率: [0.4, 0.3, 0.15, 0.1, 0.05]
Top-p=0.9: 累积到 0.4+0.3+0.15+0.1=0.95 > 0.9
           → 取前 4 个
```

### 4.1.2 与 Top-k 对比

- Top-k 控制"个数"，Top-p 控制"概率质量"。
- 分布尖锐时 Top-p 自动只留少数 token；分布平坦时自动放多。这正是它优于 Top-k 的地方。

| 优点 | 缺点 |
|------|------|
| 自适应分布 | 计算稍复杂 |
| 比 top-k 更灵活 | 极端情况下候选数仍可能很多 |

## 4.2 实现

```python
def top_p_filter(logits, p):
    sorted_logits, sorted_idx = torch.sort(logits, descending=True)
    probs = softmax(sorted_logits)
    cumsum = torch.cumsum(probs, dim=-1)

    # 移除累积概率超过 p 的 token
    mask = cumsum - probs > p
    sorted_logits[mask] = float('-inf')

    return sorted_logits.scatter(1, sorted_idx, sorted_logits)
```

```java
static void topPFilter(double[] logits, double p) {
    int n = logits.length;
    Integer[] idx = new Integer[n];
    for (int i = 0; i < n; i++) idx[i] = i;
    java.util.Arrays.sort(idx, (a, b) -> Double.compare(logits[b], logits[a]));

    double sum = 0.0;
    for (double v : logits) sum += Math.exp(v);   // 简化：未减最大值
    double cum = 0.0;
    for (int r = 0; r < n; r++) {
        int i = idx[r];
        double pr = Math.exp(logits[i]) / sum;
        if (cum > p) logits[i] = Double.NEGATIVE_INFINITY;
        cum += pr;
    }
}
```

注意 mask 的判定用的是"累加前的累积和"（cumsum - probs），保证累积刚好越过 p 的那个 token 仍被保留。

## 4.3 边界与坑

- p 太小（<0.5）会过度截断，接近贪心；p=1.0 等于不截断。
- 数值稳定性：softmax 前应先减去最大值，否则 exp 溢出。
- 与 temperature 的顺序：先温度后 top-p 是主流约定，需与框架对齐。
- 平局概率（重复 logits）排序不稳定，可能影响候选边界。

## 4.4 生产建议

- 通用默认：top_p=0.9；需要更保守用 0.8，更发散用 0.95。
- 不要同时精调 top-k 与 top-p，二者择一为主，另一作硬上限。
- 把 top_p 与 temperature 组合成"配置档位"，避免逐请求乱调。

# 5. 常用组合与参数

结论先行：单独一个旋钮往往不够，工业界最常用的是 temperature + top_p 组合；不建议同时叠加 top-k 与 top-p。

## 5.1 推荐组合

```
推荐组合：temperature + top_p
  事实任务：T=0.2, top_p=0.9
  通用对话：T=0.7, top_p=0.9
  创意写作：T=1.0, top_p=0.95

不要同时用 top_k 和 top_p（通常一个就够）
```

## 5.2 场景参数表

| 场景 | temperature | top_p |
|------|:-----------:|:-----:|
| 事实问答 | 0.1~0.3 | 0.9 |
| 代码生成 | 0~0.2 | 0.95 |
| 通用对话 | 0.7 | 0.9 |
| 创意写作 | 0.9~1.2 | 0.95 |

## 5.3 参数相互影响

- 温度先作用于 logits，改变分布形状；top-p 再在变形后的分布上截断。
- 高温度 + 低 top-p 可以做到"候选多但每个都不极端"，是常见的稳健搭配。
- 低温度 + 高 top-p 则基本等价贪心。

## 5.4 生产建议

- 建立少量"预设档"（精确 / 平衡 / 创造），而不是暴露裸参数给业务方。
- 参数改动需回归评测，避免悄悄劣化事实性。
- 记录每步的随机种子（seed），便于问题复现。

# 6. 重复惩罚

结论先行：长文本生成中模型会重复已出现内容，通过对已生成 token 的 logits 施加惩罚可有效缓解。

## 6.1 问题来源

- 自回归模型在已出现的上下文上条件概率偏高，形成正反馈。
- 贪心或低温度进一步放大重复。
- 训练数据中的模板化文本也会诱发复读。

## 6.2 三类惩罚

| 参数 | 说明 |
|------|------|
| `repetition_penalty` | > 1 惩罚重复（乘除式） |
| `frequency_penalty` | 按出现次数惩罚 |
| `presence_penalty` | 出现即惩罚（与次数无关） |

## 6.3 实现

```python
def apply_repetition_penalty(logits, generated, penalty=1.2):
    for token_id in set(generated):
        if logits[token_id] > 0:
            logits[token_id] /= penalty
        else:
            logits[token_id] *= penalty
    return logits
```

```java
static void applyRepetitionPenalty(double[] logits,
                                   int[] generated, double penalty) {
    boolean[] seen = new boolean[logits.length];
    for (int t : generated) seen[t] = true;
    for (int i = 0; i < logits.length; i++) {
        if (!seen[i]) continue;
        if (logits[i] > 0) logits[i] /= penalty;
        else               logits[i] *= penalty;
    }
}
```

frequency / presence 惩罚通常直接减去与出现次数成正比的量：

```
z'_i = z_i - alpha * count(i)          // frequency
z'_i = z_i - beta  * 1[count(i) > 0]   // presence
```

## 6.4 边界与坑

- penalty 过大（>1.5）会让正常复述（如引用、代码变量名）也被惩罚，输出怪异。
- 正负 logit 用乘除、否则用加减，混用会导致行为不一致。
- 惩罚作用于整个已生成序列，长上下文下累积效应可能过强。

## 6.5 生产建议

- 默认 repetition_penalty ∈ [1.0, 1.2]，超过 1.3 需谨慎。
- 代码、引用类任务慎用强惩罚。
- 与温度配合：高温度时重复问题减轻，惩罚可调小。

# 7. Beam Search（束搜索）

结论先行：Beam Search 同时维护 k 条候选路径，追求整体高概率序列，适合翻译、摘要等确定性任务，不适合开放对话。

## 7.1 原理

### 7.1.1 搜索过程

每一步对当前 k 条路径分别扩展，保留累计对数概率最高的 k 条，最后取全局最优序列。

```
Beam=2:
  step1: [今天] [天气]
  step2: [今天天气] [天气真] ...
  最终选累计概率最高的序列
```

用对数概率避免下溢：

```
score(x_1..x_t) = sum_{s=1}^{t} log P(x_s | x_1..x_{s-1})
```

### 7.1.2 长度归一化

连乘概率天然偏向短句，需除以长度做归一化：

```
normalized = score / t^alpha     (alpha 常用 0.6~1.0)
```

## 7.2 与采样的对比

| 优点 | 缺点 |
|------|------|
| 更可能找到高概率序列 | 缺乏多样性 |
| 适合翻译/摘要 | 不适合开放对话 |

> Beam Search 在**确定性任务**（翻译）上好，在**开放生成**上不如采样。

## 7.3 边界与坑

- Beam 越大计算与显存开销越高（近似线性）。
- 高概率序列不等于高质量，容易出现"安全但平庸"的输出。
- 无随机性，同一输入永远同一输出。
- 需配合长度惩罚，否则倾向过短或过长。

## 7.4 生产建议

- 机器翻译、语音识别、结构化抽取可用 beam=3~5。
- 对话、创意任务用采样而非 beam。
- 若必须用 beam，叠加多样化束（diverse beam）可缓解同质化。

# 8. 其他采样方法

结论先行：除主流方法外，还有典型采样、对比解码、约束解码等，用于特定质量或格式需求。

## 8.1 典型采样（Typical Sampling）

按"信息量接近期望熵"来选 token，保留分布中"典型"的部分，兼顾质量与多样性。适合对流畅度与多样性同时敏感的场景。其判定条件是每个候选 token 的信息量 -log p 接近分布的期望熵，偏离过远的（既不过于确定也不过于随机）被剔除。

## 8.2 对比解码（Contrastive Decoding）

用大模型与小模型的 logits 之差放大"大模型独有"的信号，提升连贯性，常用于缓解小模型重复问题：

```
score = log p_large - alpha * log p_small
```

## 8.3 约束解码（Constrained Decoding）

在每步把不满足格式（JSON、正则、语法）的 token 概率置零，保证输出合法。对工具调用、结构化输出非常关键。

```java
// 约束解码示意：只允许符合当前文法状态的 token
static void constrain(double[] logits, boolean[] allowedMask) {
    for (int i = 0; i < logits.length; i++) {
        if (!allowedMask[i]) {
            logits[i] = Double.NEGATIVE_INFINITY;   // 屏蔽非法 token
        }
    }
}
```

约束解码通常借助有限状态机或下推自动机跟踪"当前合法 token 集合"，每步更新状态。

| 方法 | 目标 | 典型场景 |
|------|------|----------|
| 典型采样 | 质量+多样性 | 开放生成 |
| 对比解码 | 连贯性 | 小模型增强 |
| 约束解码 | 格式合法 | JSON/工具调用 |

## 8.4 生产建议

- 结构化输出优先用约束解码而非事后解析修复。
- 对比解码需额外小模型，权衡成本。
- 典型采样参数敏感，需评测后再上线。

# 9. 各策略对比

结论先行：没有万能策略，选择取决于任务对"确定性 vs 多样性"的偏好。

## 9.1 总览表

| 策略 | 多样性 | 质量 | 适用 |
|------|:------:|:----:|------|
| 贪心 | 最低 | 稳定 | 确定性任务 |
| Temperature | 可调 | 可调 | 通用 |
| Top-k | 中 | 好 | 通用 |
| Top-p | 中 | 好 | 通用（推荐） |
| Beam Search | 最低 | 高（特定任务） | 翻译/摘要 |

## 9.2 选择决策

```
需要唯一确定输出? ── 是 ──> 贪心 / 约束解码
        │
        否
        v
结构化/工具调用? ── 是 ──> 低温度 + 约束解码
        │
        否
        v
开放生成 ──> temperature + top-p（推荐组合）
```

## 9.3 生产注意事项

| 风险 | 建议 |
|------|------|
| temperature 过高 | 输出跑偏 |
| 同时用多个策略 | 通常一个够 |
| 固定参数 | 按任务调整 |
| 无重复惩罚 | 长文本易重复 |
| 用 Beam 做对话 | 输出死板 |

## 9.4 生产建议汇总

- 用配置档位管理参数，纳入评测与版本控制。
- 记录 seed 与生效参数，保证可复现。
- 对关键业务做 A/B，避免主观调参。

# 10. 小结

## 10.1 核心结论

- 采样让输出更自然多样，贪心则死板。
- Temperature 控制随机性，Top-p 自适应截断。
- 推荐组合：temperature + top_p。
- 重复惩罚解决长文本重复问题。

## 10.2 工程实践

- Beam Search 适合确定性任务，不适合开放对话。
- 约束解码保证结构化输出合法，是工程落地的关键补充。
- 参数应配置化、可评测、可复现，而非随手硬编码。
