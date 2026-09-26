# 1. SFT 全流程

## 1.1 流程总览

**结论先行**：SFT（Supervised Fine-Tuning，监督微调）的核心是把"指令-回答"数据灌入模型，使其学会遵循指令，全流程为数据、基座、配置、训练、评估、部署六步。

```
① 准备数据（指令-回答）
② 选择基座模型
③ 配置训练（LoRA + 超参）
④ 训练
⑤ 评估
⑥ 部署
```

## 1.2 各阶段目标

**结论先行**：每个阶段都有明确的输入与验收标准，跳过任何一步都会把问题带到线上。

| 阶段 | 输入 | 输出 | 验收 |
|------|------|------|------|
| 数据 | 原始语料 | 指令-回答集 | 质量抽检通过 |
| 基座 | 模型名 | 权重 | 能正常推理 |
| 配置 | 超参 | 训练参数 | 显存可跑 |
| 训练 | 数据 + 配置 | 适配器 | 损失正常下降 |
| 评估 | 适配器 + 留出集 | 指标 | 优于基座 |
| 部署 | 适配器 | 服务 | 延迟与质量达标 |

## 1.3 边界与常见误区

**结论先行**：SFT 只教"如何回答"，不教"哪种回答更好"，偏好对齐属于后续阶段。

**反例**：把偏好数据直接当 SFT 数据训练，模型只会模仿被选中的回答，无法学到相对优劣，属于典型误用。

# 2. 数据准备

## 2.1 数据格式

**结论先行**：推荐使用带角色的多轮消息格式，兼容主流训练框架。

### 2.1.1 多轮消息格式

```json
{"messages": [
  {"role": "system", "content": "你是一个专业的 Java 技术助手。"},
  {"role": "user", "content": "解释 Spring 的 IoC 容器"},
  {"role": "assistant", "content": "IoC（控制反转）..."}
]}
```

> 只在 assistant 部分计算损失。

### 2.1.2 单轮指令格式

**结论先行**：单轮数据可用 `instruction/input/output` 三字段表达，便于从传统指令集迁移。

```json
{"instruction": "解释 IoC", "input": "", "output": "IoC 是控制反转..."}
```

## 2.2 损失掩码

**结论先行**：训练损失只应计算在 assistant 回答上，user 与 system 部分的标签置为 -100 以忽略。

**机制**：分词后把非回答位置的标签设为 -100，交叉熵在计算时跳过这些位置。

```python
labels = input_ids.clone()
labels[user_token_positions] = -100
loss = cross_entropy(logits, labels)
```

### 2.2.1 掩码的实现方式

**结论先行**：两种主流做法：直接置 -100，或使用 completion-only 数据整理器按模板自动掩码。

| 方式 | 适用 | 说明 |
|------|------|------|
| 手动置 -100 | 自定义数据 | 完全可控 |
| CompletionOnlyLM | 标准模板 | 按响应模板自动切分 |
| 仅拼接 assistant | 简单数据 | 整段都是回答 |

**坑**：若模板字符串与分词器渲染不一致，自动掩码会切错位置，导致损失计算到 user 文本上。

## 2.3 数据质量与数量

**结论先行**：SFT 阶段数据质量远比数量重要，数千条高质量样本通常优于数十万条噪声数据。

| 维度 | 要求 | 说明 |
|------|------|------|
| 正确性 | 高 | 错误答案会被学会 |
| 一致性 | 高 | 风格与格式统一 |
| 多样性 | 中 | 覆盖任务类型 |
| 去重 | 必做 | 重复样本放大偏差 |
| 隔离 | 必做 | 训练/测试不得泄漏 |

# 3. 完整训练脚本

## 3.1 端到端脚本

**结论先行**：一个完整的 SFT 脚本包含模型加载、LoRA 配置、数据加载、训练参数与训练执行五个部分。

```python
from transformers import AutoModelForCausalLM, AutoTokenizer, TrainingArguments
from peft import LoraConfig, get_peft_model
from trl import SFTTrainer, DataCollatorForCompletionOnlyLM
from datasets import load_dataset

# ① 加载模型与分词器
model_name = "Qwen/Qwen2.5-7B"
tokenizer = AutoTokenizer.from_pretrained(model_name)
model = AutoModelForCausalLM.from_pretrained(
    model_name, torch_dtype=torch.bfloat16, device_map="auto")

# ② LoRA 配置
lora_config = LoraConfig(
    r=16, lora_alpha=32, lora_dropout=0.05,
    target_modules=["q_proj", "k_proj", "v_proj", "o_proj",
                    "gate_proj", "up_proj", "down_proj"],
    task_type="CAUSAL_LM")

model = get_peft_model(model, lora_config)
model.print_trainable_parameters()      # 打印可训练参数比例

# ③ 数据
dataset = load_dataset("json", data_files="train.jsonl", split="train")

# ④ 训练参数
args = TrainingArguments(
    output_dir="./output",
    per_device_train_batch_size=4,
    gradient_accumulation_steps=4,
    learning_rate=2e-4,
    num_train_epochs=3,
    warmup_ratio=0.03,
    lr_scheduler_type="cosine",
    bf16=True,
    logging_steps=10,
    save_strategy="epoch",
    gradient_checkpointing=True)        # 省显存

# ⑤ 训练
trainer = SFTTrainer(
    model=model,
    args=args,
    train_dataset=dataset,
    tokenizer=tokenizer,
    max_seq_length=2048)

trainer.train()
trainer.save_model("./lora-adapter")
```

## 3.2 逐段解析

**结论先行**：理解每一步的作用，是排查训练异常的前提。

### 3.2.1 模型与分词器

**机制**：分词器负责把消息模板渲染为 token 序列；`torch_dtype=torch.bfloat16` 降显存，`device_map="auto"` 自动分配设备。

### 3.2.2 LoRA 配置

**机制**：`get_peft_model` 按 `target_modules` 注入适配器并冻结其余参数，`print_trainable_parameters` 用于确认比例是否合理。

### 3.2.3 训练参数

**机制**：`gradient_accumulation_steps` 与批大小共同决定有效批大小；`gradient_checkpointing` 用时间换显存。

## 3.3 保存与复现

**结论先行**：保存适配器之外，还要保存训练配置与数据版本，才能复现。

```
./lora-adapter/
  adapter_config.json
  adapter_model.safetensors
./output/
  training_args.bin
  trainer_state.json     # 损失与学习率曲线
```

**坑**：只保存权重而丢失超参，会导致后续无法解释效果差异。

### 3.3.1 复现所需的最小信息

**结论先行**：复现一次 SFT 至少需要四类信息，缺一不可。

| 类别 | 内容 |
|------|------|
| 代码 | 训练脚本与库版本 |
| 配置 | 超参快照 |
| 数据 | 版本与划分方式 |
| 随机性 | 随机种子 |

# 4. 关键超参数

## 4.1 参数总览

**结论先行**：SFT 的超参数不多，但学习率、轮数与精度对结果影响最大。

| 参数 | 建议 | 说明 |
|------|------|------|
| 学习率 | 1e-4 ~ 2e-4（LoRA） | 比全量高 |
| 轮数 | 2~3 | 过多过拟合 |
| 批大小 | 视显存 | — |
| 梯度累积 | 4~16 | 模拟大批 |
| Warmup | 3% | 稳定初期 |
| 调度 | cosine | 常用 |
| 精度 | bf16 | 稳定 |
| 梯度检查点 | 开 | 省显存 |

## 4.2 学习率与调度

**结论先行**：LoRA 的学习率通常比全量微调高一个数量级，调度以 cosine 加 warmup 最常用。

**机制**：warmup 在初期逐步抬升学习率，避免随机初始化的适配器造成大幅扰动；cosine 在后期平滑衰减，利于收敛。

### 4.2.1 学习率与训练方式的关系

**结论先行**：训练方式不同，学习率量级差异明显，不能混用。

| 训练方式 | 学习率量级 |
|----------|:----------:|
| 全量微调 | 1e-5 ~ 2e-5 |
| LoRA | 1e-4 ~ 2e-4 |
| QLoRA | 1e-4 ~ 2e-4 |
| DPO | 5e-7 ~ 1e-6 |

**边界**：LoRA 学习率之所以更高，是因为可训练参数少、梯度信号弱；把它套到全量微调会直接发散。

## 4.3 轮数与批大小

**结论先行**：轮数过多是过拟合的首要原因，2~3 轮通常足够。

| 数据量 | 建议轮数 |
|--------|:--------:|
| < 1k | 3~5 |
| 1k~10k | 2~3 |
| > 10k | 1~2 |

**边界**：有效批大小 = 单卡批大小 × 梯度累积 × 卡数，调整时需同步调整学习率。

## 4.4 精度与检查点

**结论先行**：优先使用 bf16，其动态范围大、不易溢出；梯度检查点用约 20% 的时间换约 50% 的激活显存。

# 5. 显存优化

## 5.1 优化手段总览

**结论先行**：显存优化手段可叠加使用，优先级为 QLoRA > 梯度检查点 > bf16 > 梯度累积。

| 技术 | 节省 |
|------|------|
| LoRA/QLoRA | 大量 |
| 梯度检查点 | ~50% |
| 梯度累积 | 允许小批 |
| bf16 | ~50% |
| FlashAttention | 注意力显存 |
| DeepSpeed ZeRO | 分布式 |

## 5.2 配置示例

**结论先行**：三个开关即可显著降低显存占用。

```python
args = TrainingArguments(
    gradient_checkpointing=True,
    bf16=True,
    optim="paged_adamw_8bit")        # 8bit 优化器
```

### 5.2.1 优化项选择顺序

**结论先行**：先上 LoRA/QLoRA，再开梯度检查点与 bf16，最后才考虑分布式。

```
1. LoRA / QLoRA        （最大收益）
2. 梯度检查点          （约省一半激活）
3. bf16                （约省一半精度显存）
4. 梯度累积            （用小批模拟大批）
5. FlashAttention      （注意力显存）
6. DeepSpeed ZeRO      （多卡切分）
```

## 5.3 边界与权衡

**结论先行**：省显存的手段大多以时间或效果为代价，需按硬件条件取舍。

| 手段 | 代价 |
|------|------|
| 梯度检查点 | 训练变慢 |
| 8bit 优化器 | 精度略降 |
| 梯度累积 | 单步更慢 |
| 4bit 量化 | 效果略降 |

**坑**：开启梯度检查点后必须关闭 `use_cache`，否则可能报错或显存反而升高。

# 6. 训练监控

## 6.1 关键指标

**结论先行**：训练损失、验证损失、学习率与梯度范数是判断训练健康度的四项核心指标。

```
关键指标：
  训练损失（应平稳下降）
  验证损失（应同步下降，回升即过拟合）
  学习率曲线
  梯度范数（异常大 = 不稳定）
```

### 6.1.1 指标的正常区间

**结论先行**：指标本身没有绝对阈值，关键是看趋势与相对变化。

| 指标 | 健康表现 | 异常信号 |
|------|----------|----------|
| 训练损失 | 平稳下降后趋缓 | 不降或剧烈震荡 |
| 验证损失 | 与训练同步下降 | 提前回升 |
| 梯度范数 | 稳定在个位数 | 突然飙升 |
| 学习率 | 按调度平滑变化 | 跳变 |

## 6.2 早停与回调

**结论先行**：早停是防止过拟合最直接的手段，依据验证损失触发。

```python
# 早停
EarlyStoppingCallback(early_stopping_patience=2)
```

## 6.3 异常识别

**结论先行**：损失曲线形态能快速定位问题类型。

| 现象 | 可能原因 |
|------|----------|
| 损失不降 | 学习率过低、掩码错误 |
| 损失震荡 | 学习率过高 |
| 损失 NaN | 精度溢出、数据异常 |
| 验证损失回升 | 过拟合 |

# 7. 常见问题

## 7.1 损失异常

| 问题 | 原因 | 对策 |
|------|------|------|
| Loss 不降 | 学习率太低 | 提高学习率 |
| Loss NaN | 学习率太高/精度问题 | 降学习率、用 bf16 |
| Loss 为 0 | 标签未掩码 | 检查损失掩码 |

## 7.2 过拟合与遗忘

| 问题 | 原因 | 对策 |
|------|------|------|
| 过拟合 | 轮数多/数据少 | 早停、增数据 |
| 灾难性遗忘 | 学习率太高 | 降学习率、混数据 |
| 效果差 | 数据质量差 | 检查数据 |

## 7.3 资源问题

| 问题 | 原因 | 对策 |
|------|------|------|
| 显存不足 | 批太大 | 梯度累积 + 检查点 |
| 训练过慢 | 序列过长 | 截断、打包 |
| 卡间不均衡 | 分片不当 | 调整 device_map |

# 8. 评估

## 8.1 评估方法

**结论先行**：评估应包含自动指标与人工/模型评审两部分，单一指标容易误导。

**机制**：对生成任务常用任务准确率、格式合规率与评审胜率三类指标。

```python
# 用留出集评估
def evaluate(model, test_cases):
    correct = 0
    for case in test_cases:
        output = model.generate(case["prompt"])
        if judge(case["expected"], output):
            correct += 1
    return correct / len(test_cases)
```

## 8.2 对比基座

**结论先行**：必须与基座对比，否则无法判断微调是否带来净收益。

```python
# 对比基座
base_score = evaluate(base_model, test_cases)
ft_score = evaluate(finetuned_model, test_cases)
print(f"基座: {base_score}, 微调后: {ft_score}")
```

> **必须对比基座**，否则不知微调是否有效。

## 8.3 评估边界

**结论先行**：留出集必须与训练集严格隔离，并覆盖线上真实分布。

**坑**：用训练数据当测试集会产生虚高的指标，是最常见的评估错误。

# 9. 生产注意事项

## 9.1 数据与训练风险

| 风险 | 建议 |
|------|------|
| 数据质量差 | 严格筛选 |
| 轮数过多 | 早停 |
| 灾难性遗忘 | 低学习率 + 混数据 |
| 数据泄漏 | 训练/测试隔离 |

## 9.2 部署与监控

**结论先行**：上线后需持续监控输出质量与分布漂移，模型效果会随数据分布变化而衰减。

| 监控项 | 说明 |
|--------|------|
| 输出合规率 | 格式与安全 |
| 延迟 | 端到端响应 |
| 人工抽检 | 质量兜底 |
| 版本 | 基座与适配器绑定 |

### 9.2.1 灰度与回滚

**结论先行**：上线采用灰度发布，先放小流量，指标达标再全量，异常时一键回滚到上一适配器。

**生产建议**：保留至少一个历史适配器版本，并记录其对应的基座版本，确保回滚可执行。

## 9.3 合规与安全

**结论先行**：训练数据需确保来源合规，输出需过滤敏感内容。

**生产建议**：建立数据来源台账，部署侧加输出过滤与审计日志。

# 10. 小结

## 10.1 要点回顾

- SFT 流程：数据 → 基座 → 配置 → 训练 → 评估 → 部署。
- 只在 assistant 部分计算损失。
- 关键超参：学习率 1e-4~2e-4、轮数 2~3、cosine 调度。
- 显存优化：QLoRA + 梯度检查点 + bf16。
- 必须对比基座评估效果。

## 10.2 实践清单

- 数据先做去重与训练/测试隔离。
- 固定随机种子，保存配置快照。
- 用验证损失触发早停，防止过拟合。
- 上线前跑通"基座 vs 微调"对比。
