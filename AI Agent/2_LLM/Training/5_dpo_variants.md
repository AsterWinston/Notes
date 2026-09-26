# 1. 为什么需要简化 RLHF

结论先行：RLHF 需要奖励模型与 PPO，复杂、不稳定且昂贵，DPO 等方法直接用偏好数据优化 LLM，跳过显式奖励模型与强化学习。

## 1.1 RLHF 的痛点

### 1.1.1 复杂度与成本

```
RLHF 的问题：
- 需要训练奖励模型
- 需要 PPO（四个模型）
- 训练不稳定、调参困难
- 成本高
```

**DPO 等方法直接用偏好数据优化 LLM，跳过奖励模型与 RL。**

四个模型同时驻留显存，训练与推理开销巨大；奖励模型还需单独的标注与训练流程。

### 1.1.2 不稳定性

PPO 对学习率、KL 系数、优势归一化等超参极其敏感，一次跑崩往往需要重来，工程成本高。

### 1.1.3 离线与在线

RLHF 属于在线（Online）方法，策略必须实时生成样本；DPO 属于离线（Offline）方法，直接消费固定的偏好数据集，训练更像监督学习。

| 维度 | 在线 RLHF | 离线 DPO |
|------|-----------|----------|
| 数据 | 实时生成 | 固定数据集 |
| 可探索 | 是 | 否 |
| 工程复杂度 | 高 | 低 |
| 稳定性 | 差 | 好 |

## 1.2 直接偏好优化的思路

### 1.2.1 核心洞察

RLHF 的最优策略与奖励函数存在闭式关系：给定奖励 $r$，最优策略为

$$
\pi^*(y \mid x) = \frac{1}{Z(x)} \pi_{\text{ref}}(y \mid x) \exp\!\left( \frac{r(x, y)}{\beta} \right)
$$

反解可得奖励可由策略与参考模型的概率比表示，于是无需显式奖励模型。

### 1.2.2 方法谱系

```
RLHF（PPO）
   │  去掉 RL
   ▼
DPO ──► IPO（正则化）
   │
   ├──► KTO（单标签）
   ├──► ORPO（合并 SFT）
   ├──► SimPO（无参考模型）
   └──► GRPO（组内相对优势）
```

### 1.2.3 隐式奖励的通用性

DPO 的隐式奖励 $r(x,y)=\beta\log\frac{\pi_\theta(y|x)}{\pi_{\text{ref}}(y|x)}$ 可复用于离线评估，无需额外训练打分模型。

## 1.3 边界与坑

- DPO 类方法假设偏好数据离线可得，无法像在线 RL 那样探索新回答。
- 跳过奖励模型意味着失去可复用的通用奖励信号。

## 1.4 生产建议

- 资源有限时优先 DPO 类方法，可显著降低工程复杂度。
- 仍建议保留一份奖励模型用于离线评测。

# 2. DPO

结论先行：DPO（Direct Preference Optimization，直接偏好优化）用策略与参考模型的概率比构造隐式奖励，直接在偏好对上做监督式优化。

## 2.1 原理推导

### 2.1.1 从 RLHF 目标到闭式解

RLHF 目标：

$$
\max_{\pi} \; \mathbb{E}\big[ r(x, y) \big] - \beta \, \mathrm{KL}(\pi \| \pi_{\text{ref}})
$$

其最优解为 $\pi^*(y \mid x) \propto \pi_{\text{ref}}(y \mid x) \exp(r(x,y)/\beta)$，反解得

$$
r(x, y) = \beta \log \frac{\pi^*(y \mid x)}{\pi_{\text{ref}}(y \mid x)} + \beta \log Z(x)
$$

### 2.1.2 隐式奖励

把上式代入 Bradley-Terry 偏好模型，配分函数 $Z(x)$ 在成对比较中被约掉，得到可直接优化的 DPO 损失。

### 2.1.3 梯度直觉

对 DPO 损失求梯度可得：

$$
\nabla_\theta \mathcal{L} = -\beta \, \sigma(\hat{r}_l - \hat{r}_w) \big( \nabla_\theta \log \pi_\theta(y_w) - \nabla_\theta \log \pi_\theta(y_l) \big)
$$

权重 $\sigma(\hat{r}_l - \hat{r}_w)$ 表示"当前模型判错的置信度"：错得越离谱，梯度越大。这使 DPO 自动聚焦于难样本。

## 2.2 损失函数

### 2.2.1 公式

$$
\mathcal{L}_{\text{DPO}} = - \mathbb{E}_{(x, y_w, y_l)} \left[ \log \sigma \!\left( \beta \log \frac{\pi_\theta(y_w \mid x)}{\pi_{\text{ref}}(y_w \mid x)} - \beta \log \frac{\pi_\theta(y_l \mid x)}{\pi_{\text{ref}}(y_l \mid x)} \right) \right]
$$

直觉：提高 chosen 相对参考模型的概率，同时降低 rejected 相对参考模型的概率。

### 2.2.2 代码与直觉

```
核心洞察：
  奖励模型的最优解与策略有闭式关系
  → 可以跳过 RM，直接用偏好数据优化 LLM

目标：
  提高 chosen 的概率，降低 rejected 的概率
```

```python
def dpo_loss(policy, ref_policy, chosen, rejected, beta=0.1):
    # 计算策略与参考模型的 log 概率比
    pi_chosen = logprob(policy, chosen) - logprob(ref_policy, chosen)
    pi_rejected = logprob(policy, rejected) - logprob(ref_policy, rejected)

    # 最大化 chosen 优于 rejected
    return -F.logsigmoid(beta * (pi_chosen - pi_rejected)).mean()
```

### 2.2.3 Java 示例

```java
public static double dpoLoss(double logpPolicyChosen,
                             double logpRefChosen,
                             double logpPolicyRejected,
                             double logpRefRejected,
                             double beta) {
    double chosen = logpPolicyChosen - logpRefChosen;
    double rejected = logpPolicyRejected - logpRefRejected;
    double margin = beta * (chosen - rejected);
    return -logSigmoid(margin);
}

private static double logSigmoid(double x) {
    if (x >= 0) {
        return -Math.log1p(Math.exp(-x));
    }
    return x - Math.log1p(Math.exp(x));
}
```

### 2.2.4 序列级与 token 级概率

DPO 使用整段回答的对数概率（token 概率之和）。长度差异会让长回答的绝对概率更小，因此部分实现改用长度归一化（见 SimPO）。

## 2.3 与 RLHF 对比

| 对比 | RLHF | DPO |
|------|:----:|:---:|
| 奖励模型 | 需要 | 不需要 |
| 强化学习 | PPO | 无 |
| 模型数 | 4 | 2（Policy + Ref） |
| 稳定性 | 差 | 好 |
| 成本 | 高 | 低 |
| 效果 | 好 | 相当 |

> DPO 是当前**最主流的偏好对齐方法**。

## 2.4 边界与坑

- DPO 对参考模型敏感，参考模型应与策略同源（通常用 SFT 模型）。
- beta 过小会过度偏离，过大则学不动。
- 偏好数据中的噪声会被直接放大，需严格筛选。

## 2.5 生产建议

- 参考模型固定且冻结，用 SFT 后的 checkpoint。
- 监控 chosen/rejected 的隐式奖励差，判断训练是否有效。
- 1~2 轮即可，轮数过多易过拟合。

# 3. DPO 的变体总览

结论先行：DPO 的变体从数据需求、参考模型、训练阶段等维度改进基础方法。

## 3.1 谱系

| 方法 | 改进点 |
|------|--------|
| DPO | 基础 |
| IPO | 解决过拟合 |
| KTO | 只需"好/坏"标签，不需成对 |
| ORPO | 合并 SFT 与偏好优化 |
| SimPO | 无需参考模型 |
| GRPO | 组内相对优势（DeepSeek 用） |

## 3.2 选择维度

选择方法时主要看三点：是否有成对偏好数据、是否有额外显存放参考模型、是否需要在线探索。

## 3.3 边界与坑

- 变体并非全都更优，很多只在特定数据条件下有优势。
- 混用不同损失需重新调参，不能照搬 DPO 的超参。

## 3.4 生产建议

- 先用 DPO 建立基线，再逐个尝试变体并做 A/B 对比。
- 记录每种方法的隐式奖励差与人工评分，避免凭感觉选型。

# 4. IPO

结论先行：IPO（Identity Preference Optimization）通过加入正则化解决 DPO 在偏好确定性高时的过拟合。

## 4.1 动机

```
问题：DPO 在偏好数据确定性高时容易过拟合
IPO：加入正则化，限制优化幅度
```

当偏好数据几乎无噪声时，DPO 的 log-sigmoid 会持续推大分差，导致过拟合。

## 4.2 方法

IPO 用平方损失替换 log-sigmoid，把分差回归到一个目标值：

$$
\mathcal{L}_{\text{IPO}} = \mathbb{E}\left[ \left( \log \frac{\pi_\theta(y_w \mid x)}{\pi_{\text{ref}}(y_w \mid x)} - \log \frac{\pi_\theta(y_l \mid x)}{\pi_{\text{ref}}(y_l \mid x)} - \frac{1}{2\beta} \right)^2 \right]
$$

## 4.3 边界与坑

- 目标分差 $1/(2\beta)$ 的选取影响效果，需调参。
- 在噪声较大的数据上，IPO 可能不如 DPO 激进有效。

## 4.4 生产建议

- 偏好数据非常干净且希望稳定时可选 IPO。
- 与 DPO 做 A/B 对比后再决定。

# 5. KTO

结论先行：KTO（Kahneman-Tversky Optimization）只需单条数据的"好/坏"标签，不需要成对偏好。

## 5.1 动机

```
问题：DPO 需要成对数据（chosen + rejected）
KTO：只需单条数据的"好/坏"标签

适用：只有点赞/点踩数据，没有成对偏好
```

## 5.2 方法

```json
[
  {"prompt": "...", "completion": "...", "label": "good"},
  {"prompt": "...", "completion": "...", "label": "bad"}
]
```

KTO 借鉴前景理论（Prospect Theory）的损失厌恶思想，对好样本与坏样本分别使用不同权重的效用函数。

## 5.3 边界与坑

- 好坏样本比例失衡时需调权重。
- 单标签信息量低于成对比较，理论上限可能略低。

## 5.4 生产建议

- 已有大量点赞/点踩日志时，KTO 是低成本选择。
- 保留一部分成对数据用于评测。

# 6. ORPO

结论先行：ORPO（Odds Ratio Preference Optimization）把 SFT 与偏好优化合并为一步训练。

## 6.1 动机

```
把 SFT 与偏好优化合并为一步：
  loss = SFT_loss + λ × OR_loss

优点：省一次训练
```

## 6.2 方法

OR 项用胜率比（Odds Ratio）对比 chosen 与 rejected，与 SFT 的负对数似然加权相加：

$$
\mathcal{L}_{\text{ORPO}} = \mathcal{L}_{\text{SFT}} + \lambda \, \mathcal{L}_{\text{OR}}
$$

## 6.3 边界与坑

- 省了 SFT 阶段，但合并损失使超参（尤其 $\lambda$）更难调。
- 基座较弱时，合并训练可能两头不讨好。

## 6.4 生产建议

- 从干净基座直接做偏好对齐、想省一次训练时可选 ORPO。
- 分别对比"先 SFT 再 DPO"与 ORPO 的效果。

# 7. SimPO

结论先行：SimPO（Simple Preference Optimization）用长度归一化的对数概率作为隐式奖励，完全不需要参考模型。

## 7.1 动机

DPO 需要同时加载参考模型，显存翻倍。SimPO 去掉参考模型，降低资源需求。

## 7.2 方法

SimPO 用平均对数概率作为奖励，并引入目标奖励间隔 $\gamma$：

$$
\mathcal{L}_{\text{SimPO}} = - \log \sigma\!\left( \frac{\beta}{|y_w|} \log \pi_\theta(y_w \mid x) - \frac{\beta}{|y_l|} \log \pi_\theta(y_l \mid x) - \gamma \right)
$$

长度归一化抑制模型偏好长回答。

## 7.3 边界与坑

- 无参考模型意味着缺少锚点，训练初期可能不稳定。
- $\gamma$ 需调参，过大导致学不动。

## 7.4 生产建议

- 显存紧张时优先 SimPO。
- 监控回答长度，确认长度归一化生效。

# 8. GRPO

结论先行：GRPO（Group Relative Policy Optimization，组相对策略优化）用同一问题的组内相对排名计算优势，无需价值模型。

## 8.1 动机

**DeepSeek 采用的强化学习方法**：

```
GRPO（Group Relative Policy Optimization）：
  对同一问题生成一组回答
  用组内相对排名计算优势
  → 不需要 Value Model（比 PPO 省一半显存）
```

## 8.2 方法

### 8.2.1 组内相对优势

```
问题 → 生成 N 个回答 → 打分 → 组内相对优势 → 更新
```

对同一提示采样 $N$ 个回答，用组内奖励的均值与标准差归一化得到优势：

$$
A_i = \frac{r_i - \mathrm{mean}(\{r_1, \dots, r_N\})}{\mathrm{std}(\{r_1, \dots, r_N\})}
$$

这省去了 PPO 的价值网络，同时保留 KL 惩罚。

### 8.2.2 与 PPO 对比

| 对比 | PPO | GRPO |
|------|:---:|:----:|
| Value Model | 需要 | 不需要 |
| 显存 | 高 | 低 |
| 稳定性 | 中 | 好 |

### 8.2.3 裁剪与 KL

GRPO 沿用 PPO 的裁剪目标与 KL 惩罚，只是把优势换成组内归一化版本：

$$
\mathcal{L} = \mathbb{E}\left[ \min(\rho_i A_i, \mathrm{clip}(\rho_i, 1-\epsilon, 1+\epsilon) A_i) \right] - \beta \, \mathrm{KL}
$$

## 8.3 边界与坑

- 组大小 $N$ 太小则优势估计噪声大，太大则计算开销高。
- 需要可自动打分的任务（数学、代码）才能发挥优势。

## 8.4 生产建议

- 在可验证奖励任务上优先 GRPO。
- 监控组内奖励方差，方差过小说明区分度不足。

# 9. 方法选择

结论先行：按数据形态与资源条件选择方法，没有万能最优。

## 9.1 决策表

| 场景 | 推荐 |
|------|------|
| 有成对偏好数据 | DPO |
| 只有单条评分 | KTO |
| 想省一次训练 | ORPO |
| 无参考模型资源 | SimPO |
| 大规模 RL | GRPO |
| 追求稳定 | DPO |

## 9.2 组合策略

实践中常先用 SFT 打底，再用 DPO/SimPO 对齐，必要时对可验证任务补 GRPO。

## 9.3 边界与坑

- 不要一次上多种损失，先建立单方法基线。
- 数据形态决定方法，强行套用会浪费资源。

# 10. 训练要点

结论先行：偏好对齐的学习率比 SFT 更小、轮数更少，数据质量决定效果。

## 10.1 关键参数

```python
    # DPO 关键参数
beta = 0.1              # KL 约束强度
learning_rate = 5e-7    # 比 SFT 更小
epochs = 1~2            # 容易过拟合

    # 数据质量比数量重要
    # 参考模型用 SFT 后的模型
```

| 参数 | 建议 |
|------|------|
| beta | 0.1~0.5 |
| 学习率 | 5e-7 ~ 1e-6 |
| 轮数 | 1~2 |

## 10.2 工程实践

### 10.2.1 参考模型管理

参考模型冻结且与策略同源，用 SFT 后的 checkpoint，并确保分词与模板一致。

### 10.2.2 日志与监控

记录隐式奖励差、KL、回答长度与训练损失，用于判断健康度。

### 10.2.3 数据去重与配比

偏好数据需去重并平衡好/坏比例；混入少量 SFT 数据可缓解灾难性遗忘。

## 10.3 边界与坑

- 学习率沿用 SFT 值会导致训练崩溃。
- 参考模型用了错误版本（如基座而非 SFT）会显著掉点。
- 偏好数据未去重会过拟合。

## 10.4 生产建议

- 用 LoRA 做偏好对齐可进一步省显存。
- 训练与评测数据严格隔离。

# 11. 生产注意事项

结论先行：偏好对齐的落地风险集中在过拟合、数据质量与参考模型配置。

## 11.1 工程风险

| 风险 | 建议 |
|------|------|
| 过拟合 | 少轮数 + beta 调节 |
| 数据质量差 | 严格筛选偏好数据 |
| 参考模型错误 | 用 SFT 后的模型 |
| 灾难性遗忘 | 混入 SFT 数据 |
| 无评测 | 对齐后评测 |

## 11.2 评测闭环

### 11.2.1 自动评测

用 LLM-as-Judge 与基准集做快速迭代，关注有用性、安全性与格式遵循。

### 11.2.2 人工评测

对关键维度做盲评，作为上线裁决依据。

### 11.2.3 回归测试

除对齐指标外，还需回归测试通用能力，确认未因偏好对齐而显著退化。

## 11.3 生产建议

- 建立奖励差、KL、长度、人工评分四位一体的监控。
- 灰度发布，逐步放量。
- 保留基座与各阶段 checkpoint，支持回滚。

# 12. 小结

## 12.1 核心结论

- DPO 跳过奖励模型与 RL，直接用偏好数据优化。
- 主流变体：IPO、KTO、ORPO、SimPO、GRPO。
- KTO 只需单条标签，SimPO 无需参考模型，GRPO 不需价值模型。
- 训练轮数少（1~2），学习率小（5e-7 ~ 1e-6）。
- 偏好数据质量决定效果，参考模型须用 SFT 后的模型。

## 12.2 落地清单

- 按数据形态与资源选方法，先用 DPO 建立基线。
- 建立自动与人工评测闭环，并做通用能力回归测试。
- 混入少量 SFT 数据防遗忘，保存 checkpoint 支持回滚。
