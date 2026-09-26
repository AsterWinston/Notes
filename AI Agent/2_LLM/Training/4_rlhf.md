# 1. RLHF 概述

结论先行：RLHF（Reinforcement Learning from Human Feedback，基于人类反馈的强化学习）用人类偏好训练奖励模型，再用强化学习优化语言模型。

## 1.1 定义与动机

### 1.1.1 什么是 RLHF

**RLHF（Reinforcement Learning from Human Feedback）** = 用人类偏好训练奖励模型，再用强化学习优化语言模型。

```
SFT → 训练奖励模型 → PPO 优化
```

> 它是 ChatGPT 成功的关键技术之一。

### 1.1.2 为什么 SFT 不够

SFT 只最大化示范回答的似然，只能告诉模型"这样答"，无法表达"这样比那样更好"。人类偏好是相对判断，天然适合用排序信号来建模。

## 1.2 三个步骤

### 1.2.1 SFT → 奖励模型 → PPO

```
① SFT：用指令数据微调 Base Model
       ↓
② 奖励模型（RM）：用人类偏好数据训练打分模型
       ↓
③ PPO：用 RM 作为奖励，强化学习优化 LLM
```

### 1.2.2 数据流与模型角色

```
                   人类偏好数据
                        │
                        ▼
        ┌───────────────────────────┐
        │      奖励模型 RM           │
        └───────────┬───────────────┘
                    │ 打分
                    ▼
   LLM 生成 → RM 打分 → PPO 更新 LLM
```

RLHF 阶段同时存在四个模型：策略（Policy）、参考（Reference）、奖励模型（Reward Model）、价值模型（Value Model）。

## 1.3 数学框架

### 1.3.1 马尔可夫决策过程视角

把生成一个回答看成一条轨迹：状态是已生成的前缀，动作是下一个 token，策略是 LLM。奖励只在序列结束时由 RM 给出，属于稀疏奖励问题。

### 1.3.2 目标函数

RLHF 的优化目标是在最大化奖励的同时约束与 SFT 策略的距离：

$$
\max_{\pi_\theta} \; \mathbb{E}_{x \sim \mathcal{D},\, y \sim \pi_\theta(\cdot \mid x)} \big[ r_\phi(x, y) \big] - \beta \, \mathrm{KL}\!\big(\pi_\theta(\cdot \mid x) \,\|\, \pi_{\text{SFT}}(\cdot \mid x)\big)
$$

其中 $r_\phi$ 是奖励模型，$\beta$ 控制 KL 惩罚强度。

### 1.3.3 价值函数与优势函数

PPO 引入价值函数 $V_\psi(x, y_{<t})$ 估计从当前前缀出发的期望回报，并用优势函数衡量某个动作相对基线的优劣：

$$
A_t = Q_t - V_\psi(s_t), \qquad Q_t = r + \gamma V_\psi(s_{t+1})
$$

由于 RLHF 的奖励只在序列末尾给出（稀疏奖励），价值函数承担了"把末端奖励分摊到每个 token"的职责。

## 1.4 边界与坑

- RLHF 是"锦上添花"，基座能力不足时收益有限。
- 偏好数据质量决定上限，垃圾偏好只能得到垃圾奖励模型。
- 训练不稳定是常态，需要大量调参经验。

# 2. 偏好数据

结论先行：偏好数据是"给同一提示的两个回答排序"的成对样本，是奖励模型的训练燃料。

## 2.1 数据采集

### 2.1.1 成对比较

```
给标注者两个回答，选更好的：

Prompt: "解释什么是量子纠缠"

回答 A: "量子纠缠是两个粒子...（准确、清晰）"
回答 B: "量子纠缠就是很神奇的东西...（模糊）"

标注: A > B
```

成对比较比绝对打分更稳定，因为人类更擅长相对判断而非绝对评分。

### 2.1.2 标注规范

标注规范需明确定义"好"的标准（有用、无害、诚实），并给出正反例，减少标注者间的分歧。

### 2.1.3 采样策略

偏好数据中的两个回答通常来自同一策略的多次采样，或来自不同模型。采样多样性越高，区分度越大；若两个回答高度相似，标注价值低。

## 2.2 数据格式

### 2.2.1 JSON 结构

```json
{
  "prompt": "解释什么是量子纠缠",
  "chosen": "量子纠缠是...",
  "rejected": "量子纠缠就是很神奇..."
}
```

### 2.2.2 质量与一致性

| 维度 | 说明 |
|------|------|
| 一致性 | 同类问题判断标准一致 |
| 区分度 | chosen 明显优于 rejected |
| 多样性 | 覆盖多种任务与难度 |
| 无偏 | 避免位置、长度等偏好 |

### 2.2.3 Java 示例：偏好样本解析

```java
public static class Preference {
    public final String prompt;
    public final String chosen;
    public final String rejected;

    public Preference(String prompt, String chosen, String rejected) {
        this.prompt = prompt;
        this.chosen = chosen;
        this.rejected = rejected;
    }
}

public static Preference parse(Map<String, String> row) {
    String p = row.get("prompt");
    String c = row.get("chosen");
    String r = row.get("rejected");
    if (p == null || c == null || r == null) {
        throw new IllegalArgumentException("missing field");
    }
    if (c.equals(r)) {
        throw new IllegalArgumentException("chosen == rejected");
    }
    return new Preference(p, c, r);
}
```

### 2.2.4 边界提醒

- chosen 与 rejected 完全相同的样本必须剔除，否则梯度恒为零且浪费算力。
- 若 prompt 过长，需在截断时保证 chosen/rejected 前缀一致，避免比较不公平。

## 2.3 边界与坑

- 位置偏见：标注者倾向选先出现的回答，可随机交换顺序消除。
- 长度偏见：长回答常被误判为更好，需在规范中说明。
- 若 chosen 与 rejected 差异极小，样本信息量低，应剔除。

## 2.4 生产建议

- 每条样本多人标注并计算一致性，低一致性的丢弃。
- 定期做标注者校准与培训。
- 保留原始标注与元数据，便于追溯。

# 3. 奖励模型

结论先行：奖励模型把"回答好不好"压缩为一个标量分数，用 Bradley-Terry 模型从成对偏好中学习。

## 3.1 建模方式

### 3.1.1 Bradley-Terry 模型

```
输入：prompt + response
输出：一个分数

训练目标：让 chosen 的分数 > rejected
```

Bradley-Terry 假设：$P(A \succ B) = \sigma(r(A) - r(B))$，其中 $\sigma$ 是 sigmoid。

### 3.1.2 损失推导

最大化 chosen 优于 rejected 的概率，即最小化：

$$
\mathcal{L}_{\text{RM}} = - \mathbb{E}_{(x, y_w, y_l)} \Big[ \log \sigma\big( r_\phi(x, y_w) - r_\phi(x, y_l) \big) \Big]
$$

直觉：只要拉开好回答与坏回答的分差即可，分数绝对值不重要。

### 3.1.3 成对损失与排序损失的关系

Bradley-Terry 成对损失是排序损失的一种特例。当偏好数据含多个回答的完整排序时，可用 Plackett-Luce 模型把成对损失推广到多路排序。

## 3.2 训练细节

### 3.2.1 初始化与结构

RM 通常在 SFT 模型上把最后的语言模型头换成一个标量头（线性层），输出单个分数。

### 3.2.2 过拟合与正则

RM 容易过拟合偏好数据，需用早停、权重衰减与数据增强。RM 过拟合会导致 RLHF 中的奖励黑客。

### 3.2.3 数据量与泛化

奖励模型需要足够多的提示与多样的回答，才能在未见过的输入上给出可靠分数。泛化差的 RM 会让策略在分布外"刷分"。

| 数据规模 | 表现 |
|----------|------|
| 过小 | 过拟合，泛化差 |
| 适中 | 区分度与泛化平衡 |
| 过大且噪声多 | 引入偏见 |

## 3.3 Java 示例

### 3.3.1 损失计算

```java
public static double rewardLoss(double rChosen, double rRejected) {
    double margin = rChosen - rRejected;
    return -logSigmoid(margin);
}

private static double logSigmoid(double x) {
    if (x >= 0) {
        return -Math.log1p(Math.exp(-x));
    }
    return x - Math.log1p(Math.exp(x));
}
```

### 3.3.2 数值稳定性

`sigmoid` 在 $|x|$ 很大时会溢出，需用 `log1p` 与分支实现保证数值稳定。

## 3.4 边界与坑

- 只用 chosen/rejected 的分差，若数据里没有覆盖某类错误，RM 会给出错误高分。
- RM 会利用长度、格式等表面特征，而非真正理解质量。

## 3.5 生产建议

- 训练多个 RM 做集成，降低单模型偏差。
- 定期用新数据重训 RM，防止被策略钻空子。

# 4. PPO 优化

结论先行：PPO 用奖励模型的分数作为奖励，同时用 KL 惩罚约束策略不偏离 SFT 模型。

## 4.1 目标函数

### 4.1.1 奖励减 KL

```
目标：最大化奖励，同时不偏离 SFT 模型太远

  max E[r(x, y)] - β · KL(π || π_SFT)

r：奖励模型的分数
KL：与 SFT 模型的差异（防止跑偏）
β：KL 惩罚系数
```

### 4.1.2 四个模型

| 组件 | 作用 |
|------|------|
| Policy | 待优化的 LLM |
| Reference | SFT 模型（KL 基准） |
| Reward Model | 打分 |
| Value Model | 估计基线 |

## 4.2 PPO 算法要点

### 4.2.1 优势估计与裁剪

PPO 用广义优势估计（GAE）计算优势 $A_t$，并用裁剪的目标函数限制每次更新幅度：

$$
\mathcal{L}^{\text{CLIP}} = \mathbb{E}\big[ \min( \rho_t A_t, \; \mathrm{clip}(\rho_t, 1-\epsilon, 1+\epsilon) A_t ) \big]
$$

其中 $\rho_t = \pi_\theta / \pi_{\text{old}}$ 是新旧策略的概率比。

### 4.2.2 在线生成

PPO 需要策略在线生成回答，因此训练速度受生成速度制约，这是 RLHF 慢的主因。

```python
    # PPO 核心（简化）
for batch in prompts:
    responses = policy.generate(batch)
    rewards = reward_model(batch, responses)

    # 加入 KL 惩罚
    kl = compute_kl(policy, ref_policy, batch, responses)
    total_reward = rewards - beta * kl

    # PPO 更新
    ppo_update(policy, batch, responses, total_reward)
```

### 4.2.3 Java 示例：优势归一化

```java
public static double[] normalizeAdvantages(double[] adv) {
    double mean = 0.0;
    for (double a : adv) {
        mean += a;
    }
    mean /= adv.length;

    double var = 0.0;
    for (double a : adv) {
        var += (a - mean) * (a - mean);
    }
    var /= adv.length;

    double std = Math.sqrt(var) + 1e-8;
    double[] out = new double[adv.length];
    for (int i = 0; i < adv.length; i++) {
        out[i] = (adv[i] - mean) / std;
    }
    return out;
}
```

优势归一化能显著降低梯度方差，是 PPO 稳定训练的重要技巧。

## 4.3 KL 惩罚

### 4.3.1 作用

```
没有 KL 惩罚：
  模型会"钻空子" → 输出奖励模型喜欢但人类不喜欢的怪话

有 KL 惩罚：
  限制模型不要偏离 SFT 太远
```

> 这是 RLHF 稳定性的关键。

### 4.3.2 实现

KL 常用逐 token 的近似估计 $-\log(\pi_{\text{ref}}/\pi_\theta)$ 并累积到奖励中，作为每步的惩罚项。

### 4.3.3 KL 的数学推导

逐 token 的 KL 惩罚可写成对数概率比：

$$
\mathrm{KL}_t = \log \pi_\theta(y_t \mid x, y_{<t}) - \log \pi_{\text{ref}}(y_t \mid x, y_{<t})
$$

其期望即 $\mathrm{KL}(\pi_\theta \| \pi_{\text{ref}})$ 的无偏估计。把它并入奖励：

$$
\tilde{r}_t = -\beta \, \mathrm{KL}_t
$$

于是策略在获得奖励的同时被"拉回"参考模型附近。

## 4.4 边界与坑

- $\beta$ 太小会奖励黑客，太大则学不动，需仔细调参。
- 优势估计的方差大，需要足够大的 batch 与归一化。
- 生成与更新不同步会导致策略陈旧（off-policy 程度过高）。

## 4.5 生产建议

- 监控 KL 散度曲线，突增说明策略跑偏。
- 用较小的学习率与足够的 rollout 批大小。
- 保存 checkpoint，便于回滚。

# 5. 奖励黑客与稳定性

结论先行：奖励黑客是 RLHF 的主要风险，需 KL 惩罚、RM 集成与定期重训共同防范。

## 5.1 奖励黑客

### 5.1.1 表现

```
模型找到"骗过奖励模型"的方法：
- 输出冗长但空洞
- 迎合奖励模型的偏见
- 格式取巧
```

奖励模型只是人类偏好的有偏代理，策略优化会主动寻找其漏洞。

### 5.1.2 对策

| 对策 | 说明 |
|------|------|
| KL 惩罚 | 限制偏离 |
| 奖励模型集成 | 多个 RM 平均 |
| 定期重训 RM | 防止被钻空子 |
| 人工抽检 | 发现异常 |

## 5.2 训练稳定性

### 5.2.1 常见不稳定来源

PPO 对超参敏感，常见不稳定来源：奖励尺度、KL 系数、优势归一化、学习率。建议固定其他变量逐个调参。

| 来源 | 现象 | 处理 |
|------|------|------|
| 奖励尺度过大 | 梯度爆炸 | 归一化奖励 |
| KL 系数过小 | 奖励黑客 | 提高 beta |
| 优势未归一化 | 方差大 | 逐 batch 标准化 |
| 学习率过大 | loss 震荡 | 降学习率 |
| 批大小过小 | 噪声大 | 增大 rollout |

### 5.2.2 调参顺序

建议顺序：先固定学习率调 KL 系数，再调 rollout 批大小，最后微调学习率与裁剪范围。

## 5.3 边界与反例

- 反例：去掉 KL 惩罚后奖励持续上升，但人工评测变差，说明在被黑客。
- 反例：RM 与策略同源，容易被过拟合。

## 5.4 生产建议

- 用奖励与 KL 的联合曲线判断训练健康度。
- 定期用人工评测校验自动奖励。
- 对异常输出模式（复读、超长）设置硬约束。

# 6. 代价与效果

结论先行：RLHF 效果好但复杂昂贵，这正是 DPO 等简化方法的动机。

## 6.1 代价

| 代价 | 说明 |
|------|------|
| 复杂 | 四个模型（Policy/RM/Value/Ref） |
| 不稳定 | PPO 调参困难 |
| 成本高 | 训练 + 标注 |
| 慢 | 需要在线生成 |

> 这些代价催生了 **DPO** 等简化方法。详见 `5_dpo_variants.md`。

### 6.1.1 代价量化

以 7B 模型为例，RLHF 阶段常需同时驻留策略、参考、奖励、价值四份模型，显存约为纯推理的数倍；且每步都要在线生成，吞吐受生成速度限制。

## 6.2 效果

| 提升 | 说明 |
|------|------|
| 指令遵循 | 更好理解意图 |
| 有用性 | 回答更有帮助 |
| 无害性 | 更好拒绝有害请求 |
| 诚实性 | 减少幻觉 |

### 6.2.1 效果评测维度

RLHF 的效果通常从指令遵循、有用性、无害性、诚实性四个维度评测，其中无害性与诚实性对安全上线尤为关键。

## 6.3 与 DPO 的关系

DPO 从 RLHF 的目标函数出发，推导出奖励与策略的闭式关系，从而跳过显式 RM 与 PPO。理解 RLHF 是理解 DPO 的前提。

## 6.4 生产建议

- 资源充足且追求极致效果时选 RLHF/PPO。
- 资源有限或追求稳定时优先 DPO。

# 7. 生产注意事项

结论先行：RLHF 落地需重点关注奖励黑客、标注质量、训练稳定性与评测闭环。

## 7.1 工程风险

| 风险 | 建议 |
|------|------|
| 奖励黑客 | KL 惩罚 + RM 集成 |
| 标注质量差 | 严格标注规范 |
| 训练不稳定 | 小学习率 + KL 控制 |
| 成本高 | 考虑 DPO |
| 无评测 | 对齐后评测 |

## 7.2 评测闭环

### 7.2.1 自动评测

用 LLM-as-Judge 与基准集做快速迭代，但需警惕 Judge 自身的偏见。

### 7.2.2 人工评测

对安全、有用性等关键维度做人工盲评，作为最终裁决。

### 7.2.3 数据隔离

训练用的偏好数据与评测集必须来自不同来源，避免模型记忆评测集造成虚高分数。

## 7.3 生产建议

- 建立奖励、KL、人工评测三位一体的监控。
- 训练与评测数据严格隔离。
- 保留完整实验记录，支持复现与回滚。

# 8. 小结

## 8.1 核心结论

- RLHF 三步：SFT → 奖励模型 → PPO。
- 偏好数据是成对比较，需控制位置与长度偏见。
- 奖励模型用 Bradley-Terry 损失学习 chosen > rejected。
- PPO 用 KL 惩罚防止偏离 SFT 模型，$\beta$ 是关键超参。
- 奖励黑客是主要风险，需 KL + RM 集成 + 定期重训。
- RLHF 复杂昂贵，DPO 是其简化替代。

## 8.2 落地清单

- 上线前必须建立自动与人工评测闭环。
- 训练与评测数据严格隔离，保留完整实验记录。
- 保存各阶段 checkpoint，出现奖励黑客时可回滚。
