# 1. 从 SFT 到对齐

结论先行：SFT（Supervised Fine-Tuning，监督微调）教会模型"如何回答"，本质是模仿；对齐（Alignment）教会模型"哪种回答更好"，本质是偏好排序。两者不是替代关系，而是流水线上的先后阶段。

```
SFT：学会"如何回答"（模仿）
对齐：学会"哪种回答更好"（偏好）
```

## 1.1 为什么 SFT 之后还要对齐

SFT 的损失函数只关心"复现标注答案"，它无法表达"两个都不错的答案里哪个更好"，也无法抑制模型在采样时出现的危险、啰嗦或空泛回答。对齐阶段引入偏好信号后，模型输出的分布会向人类偏好方向偏移。

| 维度 | SFT | 对齐 |
|------|-----|------|
| 目标 | 最大化标注答案似然 | 最大化偏好排序正确率 |
| 数据 | 指令-回答对 | 偏好对 / 打分 / 可验证奖励 |
| 损失 | 交叉熵 | 成对 / 排序 / 策略梯度 |
| 产出 | 会回答的模型 | 会取舍的模型 |
| 典型规模 | 数万到数百万条 | 数千到数万条 |

## 1.2 主流对齐方法全景

### 1.2.1 方法能力对比

| 方法 | 需要 RM | 需要 RL | 数据 |
|------|:-------:|:-------:|------|
| RLHF | 是 | 是 | 偏好对 |
| DPO | 否 | 否 | 偏好对 |
| KTO | 否 | 否 | 单条标签 |
| ORPO | 否 | 否 | 偏好对 |
| GRPO | 否 | 是 | 可验证奖励 |

说明：RM 指 Reward Model（奖励模型），RL 指 Reinforcement Learning（强化学习）。KTO（Kahneman-Tversky Optimization，卡尼曼-特沃斯基优化）只需单条"好/坏"标签；ORPO（Odds Ratio Preference Optimization，几率比偏好优化）把 SFT 与偏好对齐合并到一次训练。

### 1.2.2 决策路径

```
有偏好对？
 ├── 是 → 想省显存/不想训 RM？
 │        ├── 是 → DPO / SimPO / ORPO
 │        └── 否 → RLHF(PPO)
 └── 否 → 只有单条评分？
          ├── 是 → KTO
          └── 否 → 任务可验证（数学/代码）？
                   ├── 是 → GRPO
                   └── 否 → 先补数据
```

### 1.2.3 边界与坑

- 对齐不能弥补基座能力不足：SFT 没学会的能力，偏好训练也变不出来。
- 方法选择不是越新越好：数据形态（成对 vs 单条 vs 可验证）才是第一约束。
- RLHF 三模型（策略、参考、奖励）同驻显存，资源门槛最高。

## 1.3 方法演进

```
2017  RLHF(PPO)  —— 三模型，复杂
2023  DPO        —— 去 RM、去 RL
2024  KTO        —— 单条标签即可
2024  ORPO       —— 合并 SFT 与对齐
2024  SimPO      —— 去参考模型
2024  GRPO       —— 组内相对优势
```

| 阶段 | 代表 | 关键简化 |
|------|------|----------|
| 早期 | RLHF | 引入人类偏好 |
| 中期 | DPO | 消去奖励模型 |
| 近期 | SimPO/GRPO | 消去参考模型/价值模型 |

# 2. 偏好数据

结论先行：偏好数据的质量决定对齐上限，数量只影响收敛稳定性。几千条高质量偏好对通常优于几十万条噪声数据。

## 2.1 数据格式

```json
{
  "prompt": "解释什么是量子纠缠",
  "chosen": "量子纠缠是两个或多个粒子...（准确清晰）",
  "rejected": "量子纠缠就是很神奇的东西...（模糊）"
}
```

### 2.1.1 字段语义

| 字段 | 类型 | 说明 |
|------|------|------|
| `prompt` | string | 用户输入，必须与线上分布一致 |
| `chosen` | string | 偏好答案，DPO 中作为正样本 |
| `rejected` | string | 次选答案，DPO 中作为负样本 |
| `id` | string | 可选，用于追溯与去重 |

### 2.1.2 变体格式

KTO 使用单条标签格式：

```json
{"prompt": "解释量子纠缠", "completion": "量子纠缠是...", "label": true}
```

### 2.1.3 构造流程

```
线上/离线采样
   ↓ 对同一 prompt 生成 2~4 个回答
人工或强模型标注偏好
   ↓ 过滤（长度、重复、脏数据）
偏好对数据集
```

## 2.2 数据质量

### 2.2.1 采样策略

- 同一 prompt 的多个回答应来自相同或相近策略，避免风格差异直接暴露答案。
- 覆盖真实业务分布：长度、语言、领域、难度都要分层。
- 保留难样本（chosen 与 rejected 差距小），它们信息量最大。

### 2.2.2 常见缺陷

| 缺陷 | 后果 | 对策 |
|------|------|------|
| chosen 明显更长 | 模型学到"越长越好" | 长度归一化或截断 |
| 标注者偏好不一致 | 噪声梯度 | 多人交叉标注 + 一致性检验 |
| prompt 分布偏移 | 线上效果差 | 用线上真实请求 |
| 重复样本过多 | 过拟合少数模式 | 去重 |

### 2.2.3 生产建议

- 数据版本化，记录每次训练的偏好集哈希。
- 保留 10% 数据作为留出集，用于评估对齐是否真的提升。

## 2.3 数据规模

| 目标 | 偏好对数量 | 说明 |
|------|:----------:|------|
| 风格微调 | 500~2000 | 数据质量最关键 |
| 能力对齐 | 2000~10000 | 覆盖多任务 |
| 安全对齐 | 5000+ | 需覆盖攻击面 |

> 超过万条后边际收益递减，此时应优先提升数据多样性而非数量。

# 3. DPO 实战

结论先行：DPO（Direct Preference Optimization，直接偏好优化）把 RLHF 的"训奖励模型 + PPO"两步压缩为一个监督式损失，无需奖励模型、无需采样，是当前最主流的对齐方式。

## 3.1 训练配置

```python
from trl import DPOTrainer, DPOConfig

training_args = DPOConfig(
    output_dir="./dpo-output",
    per_device_train_batch_size=2,
    gradient_accumulation_steps=8,
    learning_rate=5e-7,              # 比 SFT 小得多
    num_train_epochs=1,               # 1~2 轮
    beta=0.1,                         # KL 约束强度
    bf16=True,
    gradient_checkpointing=True)

trainer = DPOTrainer(
    model=model,
    ref_model=ref_model,              # SFT 后的模型
    args=training_args,
    train_dataset=preference_dataset,
    tokenizer=tokenizer)

trainer.train()
```

### 3.1.1 最小可运行示例

```python
import torch
from datasets import load_dataset
from transformers import AutoModelForCausalLM, AutoTokenizer

model = AutoModelForCausalLM.from_pretrained(
    "Qwen/Qwen2.5-0.5B-Instruct",
    torch_dtype=torch.bfloat16, device_map="auto")
ref_model = AutoModelForCausalLM.from_pretrained(
    "Qwen/Qwen2.5-0.5B-Instruct",
    torch_dtype=torch.bfloat16, device_map="auto")
tokenizer = AutoTokenizer.from_pretrained("Qwen/Qwen2.5-0.5B-Instruct")

dataset = load_dataset("json", data_files="prefs.jsonl", split="train")

trainer = DPOTrainer(
    model=model, ref_model=ref_model, args=training_args,
    train_dataset=dataset, tokenizer=tokenizer)
trainer.train()
trainer.save_model("./dpo-final")
```

## 3.2 关键参数

| 参数 | 建议 |
|------|------|
| `beta` | 0.1~0.5 |
| 学习率 | 5e-7 ~ 1e-6 |
| 轮数 | 1~2 |
| ref_model | SFT 后的模型 |

### 3.2.1 beta 的作用

`beta` 控制模型偏离参考模型的程度：越大越保守（接近 SFT 模型），越小越激进（容易过拟合与退化）。它是 DPO 里最需要调的超参数。

### 3.2.2 显存优化

```python
# 用 LoRA 做 DPO，显存大幅下降
from peft import LoraConfig

peft_config = LoraConfig(
    r=16, lora_alpha=32,
    target_modules=["q_proj", "v_proj"],
    lora_dropout=0.05)
```

> 使用 LoRA 时若未显式提供 `ref_model`，部分实现会自动用禁用适配器的模型作为参考。

### 3.2.3 边界与坑

- 不提供 `ref_model` 时，参考模型必须与训练模型同源，否则梯度有偏。
- 学习率用 SFT 的量级（1e-5）会迅速崩坏。
- 批次太小（<1）会导致梯度噪声过大，配合梯度累积使用。

# 4. DPO 的直觉

结论先行：DPO 的目标只有一句话——让 chosen 的概率相对 rejected 提高，同时用参考模型约束偏离幅度。

## 4.1 损失函数

```
目标：让 chosen 的概率相对 rejected 提高

  loss = -log σ(β × [log π(chosen)/π_ref(chosen)
                    - log π(rejected)/π_ref(rejected)])

β：约束强度（越大越不偏离参考模型）
```

### 4.1.1 逐项拆解

| 符号 | 含义 |
|------|------|
| `π` | 当前策略模型（正在训练） |
| `π_ref` | 参考模型（冻结的 SFT 模型） |
| `β` | KL 约束强度 |
| `σ` | sigmoid 函数 |
| `log π/π_ref` | 相对对数概率（隐式奖励） |

### 4.1.2 为什么不需要奖励模型

DPO 证明：在 KL 约束下最优策略与奖励函数存在闭式映射，因此可以把"奖励"直接改写成策略与参考策略的比值，从而消去显式奖励模型。

### 4.1.3 与 RLHF 目标的关系

RLHF 的目标是在最大化奖励的同时约束与参考模型的 KL 散度：

```
max E[r(x,y)] - β · KL(π || π_ref)
```

DPO 对该目标求闭式最优解并反解出奖励，得到仅含策略比值的损失，因此两者理论目标一致，只是实现路径不同。

## 4.2 梯度直觉

```
当 chosen 相对概率高于 rejected → 损失小 → 梯度小
当两者接近或反向           → 损失大 → 梯度大
β 越大，隐式奖励被压缩，更新越温和
```

### 4.2.1 边界与反例

- 若 chosen 与 rejected 由完全不同模型生成，比值会编码风格差异而非质量差异。
- 若参考模型与初始策略不同源，`π/π_ref` 的基线失真。

# 5. GRPO（DeepSeek 用的方法）

结论先行：GRPO（Group Relative Policy Optimization，组相对策略优化）用"同一问题的一组回答的组内相对得分"替代 PPO 的价值模型，省去 Value Model，显存显著下降，适合数学、代码等可验证任务。

## 5.1 算法流程

```
对同一问题生成一组回答：
  问题 → [回答1, 回答2, ..., 回答N]
           ↓ 打分（可验证奖励）
        [0.8, 0.2, 0.9, 0.5, ...]
           ↓ 组内相对优势
  优势 = (得分 - 组内均值) / 组内标准差
           ↓
      更新策略
```

### 5.1.1 伪代码

```python
for problem in batch:
    responses = policy.generate(problem, n=group_size)
    rewards = [reward(problem, r) for r in responses]
    mean, std = mean(rewards), std(rewards) + 1e-4
    for r, rew in zip(responses, rewards):
        adv = (rew - mean) / std
        loss += -adv * logprob(r) + beta * kl(policy, ref)
    loss.backward()
```

## 5.2 奖励函数

```python
# GRPO 核心：用可验证奖励
def reward(problem, response):
    if problem.type == "math":
        return 1.0 if extract_answer(response) == problem.answer else 0.0
    if problem.type == "code":
        return 1.0 if run_tests(response, problem.tests) else 0.0
    return 0.0
```

### 5.2.1 组内归一化

| 统计量 | 作用 |
|--------|------|
| 组内均值 | 基线，替代价值模型 |
| 组内标准差 | 归一化，稳定梯度 |
| 组大小 N | 常见 8~64，越大基线越稳、开销越高 |

### 5.2.2 与 PPO 对比

| 对比 | PPO | GRPO |
|------|:---:|:----:|
| Value Model | 需要 | 不需要 |
| 显存 | 高 | 低 |
| 奖励 | RM | 可验证奖励 |

### 5.2.3 边界与反例

- 组内得分全相同（如全对或全错）时标准差为 0，优势消失，样本无梯度。
- 奖励不可验证（开放式写作）时需另配奖励模型，优势退化为 RM 信号。
- 组大小过小会放大噪声，过大则吞吐下降。

# 6. 方法选择

结论先行：先看数据形态，再看资源约束，最后看任务是否可验证；不要为了追新而选方法。

## 6.1 选择表

| 场景 | 推荐 |
|------|------|
| 有成对偏好数据 | DPO |
| 只有单条评分 | KTO |
| 想省一次训练 | ORPO |
| 数学/代码（可验证） | GRPO |
| 无参考模型资源 | SimPO |
| 追求稳定 | DPO |

## 6.2 补充说明

### 6.2.1 SimPO

SimPO（Simple Preference Optimization，简单偏好优化）去掉参考模型，用长度归一化的平均对数概率作为隐式奖励，省显存但更易受长度影响。

### 6.2.2 ORPO

ORPO 在 SFT 损失上叠加几率比惩罚，一次训练同时完成模仿与对齐，适合从基座直接对齐、跳过独立 SFT 的场景。

### 6.2.3 组合策略

| 组合 | 效果 |
|------|------|
| SFT → DPO | 最稳，推荐 |
| SFT → DPO → GRPO | 可验证任务再提升 |
| 基座 → ORPO | 省一次训练 |
| SFT → SimPO | 无参考模型资源时 |

# 7. 训练要点

结论先行：对齐训练的核心是"小步慢走"——小学习率、少轮数、强约束、混数据。

```
① 数据质量远比数量重要（几千条优质数据即可）
② 学习率要小（5e-7 量级）
③ 轮数要少（1~2 轮）
④ 参考模型用 SFT 后的模型
⑤ 混入部分 SFT 数据防遗忘
```

## 7.1 学习率与轮数

| 参数 | 安全区 | 危险区 |
|------|--------|--------|
| 学习率 | 5e-7 ~ 1e-6 | > 1e-5 |
| 轮数 | 1~2 | > 3 |
| beta | 0.1~0.5 | < 0.01 |

## 7.2 防遗忘

将 5%~20% 的 SFT 数据混入偏好训练批次，可显著缓解灾难性遗忘（Catastrophic Forgetting，指模型在新任务上训练后丢失旧能力）。

## 7.3 超参搜索建议

```python
# 小网格搜索，先粗后细
grid = {
    "learning_rate": [5e-7, 1e-6],
    "beta": [0.1, 0.3, 0.5],
    "num_train_epochs": [1, 2]}
```

- 先用少量步数快速筛 beta 与学习率，再跑完整训练。
- 每个配置都用固定留出集算胜率，避免只看训练损失。

# 8. 常见问题

结论先行：对齐问题九成出在数据与超参上，先查数据质量，再查轮数与 beta。

| 问题 | 原因 | 对策 |
|------|------|------|
| 过拟合 | 轮数多/beta 小 | 减轮数、增 beta |
| 效果不明显 | 数据质量差 | 提升数据质量 |
| 灾难性遗忘 | 无 SFT 数据 | 混入 SFT 数据 |
| 输出退化 | 训练过度 | 早停 |
| 长度失控 | 偏好长度 | 长度正则 |

## 8.1 诊断顺序

```
输出变差？
 ├── 先看训练/验证损失是否背离 → 过拟合
 ├── 再看 chosen/rejected 奖励差 → 数据是否可分
 ├── 再看通用能力基准 → 是否遗忘
 └── 最后看输出长度分布 → 是否长度偏差
```

## 8.2 反例

- 只盯训练损失下降，结果验证集偏好正确率反而下降，是典型过拟合。
- 数据里 chosen 全部更长，模型会学会"堆砌长度"而非提升质量。

## 8.3 一个完整排错案例

```
现象：DPO 后模型回答变短且空泛
排查：
  1) 训练损失正常下降           → 非欠拟合
  2) 验证胜率无提升             → 数据或目标问题
  3) 统计 chosen 长度 < rejected → 长度偏置
  4) 检查数据：rejected 多为长而空洞的回答
结论：模型学到"短=好"
对策：长度归一化 + 补充高质量长回答
```

# 9. 评估

结论先行：对齐没有绝对指标，必须用"对齐前后对比"来证明增益，常见做法是 LLM-as-Judge 胜率。

## 9.1 胜率评估

```python
def evaluate_alignment(model, test_prompts):
    # 用 judge 评估
    wins = 0
    for prompt in test_prompts:
        response = model.generate(prompt)
        if judge.is_better(response, baseline_response(prompt)):
            wins += 1
    return wins / len(test_prompts)
```

## 9.2 评估维度

| 维度 | 方法 |
|------|------|
| 偏好胜率 | LLM-as-Judge 对比 |
| 通用能力 | MMLU、GSM8K 等基准 |
| 安全性 | 红队测试集 |
| 长度分布 | 统计均值/方差 |

### 9.2.1 评估的坑

- Judge 自身有位置偏差与长度偏好，需交换顺序评估两次取平均。
- 只用训练同分布的测试集会高估效果。

### 9.2.2 基准回退检测

| 检查项 | 工具 | 阈值 |
|--------|------|------|
| 通用知识 | MMLU | 下降 < 2% |
| 数学推理 | GSM8K | 下降 < 2% |
| 指令遵循 | IFEval | 不下降 |
| 安全拒答 | 红队集 | 不下降 |

> 任何一项明显回退，都应降低学习率或减少轮数后重训。

# 10. 生产注意事项

结论先行：对齐训练上线前必须建立"数据质量—超参约束—回归评测"三道闸门。

| 风险 | 建议 |
|------|------|
| 过拟合 | 少轮数 + beta |
| 数据质量差 | 严格筛选 |
| 灾难性遗忘 | 混 SFT 数据 |
| 无评测 | 对比对齐前后 |
| 奖励黑客 | KL 约束 |

## 10.1 奖励黑客

奖励黑客（Reward Hacking）指模型找到奖励函数漏洞、刷高分却不真正满足意图。对策：KL 约束、人工抽检、多奖励加权。

## 10.2 上线清单

```
□ 偏好数据已去重、已分层、已留出
□ beta / 学习率 / 轮数在安全区
□ 对齐前后胜率与通用基准均无回退
□ 混入 SFT 数据防遗忘
□ 输出长度分布已检查
□ 模型与数据版本已记录
```

## 10.3 回滚策略

- 每次训练保存 checkpoint，保留至少上一个可用版本。
- 上线灰度：先 5% 流量，观察胜率与安全指标再放量。
- 记录数据哈希、超参、评测结果，保证可复现。

# 11. 小结

结论先行：对齐的本质是把"哪种回答更好"的偏好信号注入模型，方法选择由数据形态决定。

## 11.1 关键结论

- 偏好对齐让模型学会"哪种回答更好"。
- DPO 跳过 RM 与 RL，是当前主流。
- GRPO 用可验证奖励，适合数学/代码。

## 11.2 实践要点

- 学习率小（5e-7）、轮数少（1~2）。
- 必须对比对齐前后效果。
