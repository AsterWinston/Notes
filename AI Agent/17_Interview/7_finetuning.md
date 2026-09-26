# 1. 什么时候该微调？

## 1.1 标准答案

**答**：微调是最后手段。优先级：

```
提示工程 → RAG → 微调
```

**该微调的信号**：提示已优化到极限仍不足；需要稳定的特定风格；领域术语理解差；需要小模型降本（蒸馏）。

## 1.2 深入原理

### 1.2.1 为什么把微调放最后

微调成本高、迭代慢、易引入灾难性遗忘；而提示与 RAG 可小时级迭代、可回滚。

### 1.2.2 微调擅长什么

微调改变的是**模型的分布与风格**，而非注入事实知识；知识更新应交给 RAG。

## 1.3 代码示例

```java
// 伪代码：决策树
Decision choose() {
    if (promptNotEnough()) return Decision.PROMPT;
    if (needFreshKnowledge()) return Decision.RAG;
    if (needStyleOrDistill()) return Decision.FINETUNE;
    return Decision.PROMPT;
}
```

## 1.4 常见追问

**Q：微调能让模型记住新事实吗？**
A：可以但不可靠且难更新，知识注入优先 RAG。

## 1.5 生产视角

微调应作为**有明确 ROI 的专项**：先量化收益目标（准确率、成本、延迟），再决定投入。

# 2. RAG 与微调的区别？

## 2.1 标准答案

| 需求 | RAG | 微调 |
|------|:---:|:----:|
| 补充新知识 | 是 | 是 |
| 知识实时更新 | 是 | 否 |
| 改变输出风格 | 否 | 是 |
| 提升领域能力 | 部分 | 是 |
| 压缩模型成本 | 否 | 是 |

**RAG 补知识，微调改能力/风格。**

## 2.2 深入原理

### 2.2.1 作用层不同

RAG 在**推理时**把外部知识拼入上下文；微调在**训练时**改变模型参数。

### 2.2.2 二者可叠加

先微调领域风格，再用 RAG 注入实时知识，是常见组合方案。

## 2.3 代码示例

```java
String ctx = retriever.search(q);
String answer = finetunedModel.generate(
    "参考资料：\n" + ctx + "\n问题：" + q);
```

## 2.4 常见追问

**Q：微调后还需要 RAG 吗？**
A：需要，微调不解决知识实时性。

## 2.5 生产视角

**默认 RAG，微调按需**；组合方案要分别度量"知识命中率"与"风格达标率"。

# 3. 全量微调与 LoRA 的区别？

## 3.1 标准答案

| 类型 | 参数量 | 显存 | 效果 |
|------|:------:|:----:|:----:|
| 全量 | 100% | 极高 | 最好 |
| LoRA | 0.1%~1% | 低 | 接近全量 |
| QLoRA | 同 LoRA | 更低 | 略低 |

**LoRA 是主流**，冻结基座，只训练低秩矩阵。

## 3.2 深入原理

### 3.2.1 全量微调的代价

需保存全部参数的梯度与优化器状态，显存约为参数量的 3~4 倍以上。

### 3.2.2 LoRA 的性价比

只训练极少量新增参数，显存与存储成本大幅下降，且可插拔、可多套并存。

## 3.3 代码示例

```java
LoraConfig cfg = LoraConfig.builder()
    .r(16)
    .loraAlpha(32)
    .targetModules(List.of("q_proj", "k_proj", "v_proj", "o_proj"))
    .build();
```

## 3.4 常见追问

**Q：LoRA 效果一定不如全量吗？**
A：多数任务接近全量，差距在大规模风格迁移时才明显。

## 3.5 生产视角

生产优先 LoRA/QLoRA：训练快、成本低、可回滚；仅当收益显著不足时才考虑全量。

# 4. LoRA 的原理？

## 4.1 标准答案

```
W' = W + B·A

W 冻结，A（d×r）、B（r×d）可训练，r << d
```

关键参数：`r`（8~64）、`alpha`（2×r）、`target_modules`（注意力 + FFN）。

**可合并到基座，无推理开销**。

## 4.2 深入原理

### 4.2.1 低秩假设

微调带来的权重变化具有低内在秩，用低秩乘积近似即可，参数量从 d² 降到 2dr。

### 4.2.2 初始化与缩放

A 随机初始化、B 初始化为零，使训练起点等价于原模型；缩放因子 alpha/r 控制更新幅度。

## 4.3 代码示例

```java
Tensor forward(Tensor x) {
    Tensor base = x.matmul(W);            // 冻结路径
    Tensor delta = x.matmul(A).matmul(B); // 低秩路径
    return base.add(delta.mul(alpha / r));
}
```

## 4.4 常见追问

**Q：r 越大越好吗？**
A：不是，过大会增加参数与过拟合风险，通常 8~64。

## 4.5 生产视角

合并 LoRA 后推理无额外开销；未合并则便于按租户切换适配器，需按场景权衡。

# 5. QLoRA 是什么？

## 5.1 标准答案

**答**：4bit 量化基座 + LoRA。

```
70B 模型可在单卡 48GB 上微调
效果接近全量微调
```

## 5.2 深入原理

### 5.2.1 三重省显存

4bit 量化基座（NF4）、双重量化、分页优化器，共同把显存需求压到可单卡承载。

### 5.2.2 精度损失

基座被量化会带来轻微效果损失，但 LoRA 部分以高精度训练，整体仍接近全量。

## 5.3 代码示例

```java
BitsAndBytesConfig q = BitsAndBytesConfig.builder()
    .loadIn4Bit(true)
    .quantType("nf4")
    .doubleQuant(true)
    .computeDtype("bfloat16")
    .build();
```

## 5.4 常见追问

**Q：QLoRA 与 LoRA 训练速度？**
A：QLoRA 更省显存，速度略慢，但可训练更大模型。

## 5.5 生产视角

QLoRA 让中小团队也能微调大模型，但需在**目标硬件**上实测吞吐与效果，避免纸面可行。

# 6. SFT 的关键点？

## 6.1 标准答案

1. **只在 assistant 部分计算损失**；
2. 数据质量 > 数量（1000 条优质 > 10 万条低质）；
3. 学习率 1e-5~2e-5，轮数 1~3；
4. 防灾难性遗忘（低学习率 + 数据混合）。

## 6.2 深入原理

### 6.2.1 掩码的重要性

若对用户输入也计算损失，模型会学习"复述问题"，应把 prompt 部分标签置为 -100 屏蔽。

### 6.2.2 质量优先

低质数据会污染分布且训练时无法自动识别，因此清洗是收益最高的环节。

## 6.3 代码示例

```java
long[] labels = inputIds.clone();
for (int i = 0; i < promptLen; i++) {
    labels[i] = -100;   // 屏蔽 prompt，只学 assistant
}
```

## 6.4 常见追问

**Q：轮数为什么这么少？**
A：SFT 数据量小，多轮极易过拟合与遗忘。

## 6.5 生产视角

SFT 数据要**版本化与可追溯**，每次训练记录数据快照，保证效果可复现。

# 7. 灾难性遗忘如何缓解？

## 7.1 标准答案

- 低学习率；
- 少轮数（1~3）；
- 数据混合（混入通用数据）；
- LoRA（只训练少量参数）；
- 正则化。

## 7.2 深入原理

### 7.2.1 遗忘的本质

参数在领域数据上大幅更新，覆盖了原有的通用能力，表现为通用任务退化。

### 7.2.2 数据混合比例

在领域数据中混入 10%~30% 通用数据，可显著缓解通用能力下降。

## 7.3 代码示例

```java
List<Sample> mix = new ArrayList<>();
mix.addAll(domainData);                  // 领域
mix.addAll(sample(generalData, 0.2));    // 20% 通用
shuffle(mix);
```

## 7.4 常见追问

**Q：LoRA 为何更抗遗忘？**
A：只更新少量参数，对基座扰动小。

## 7.5 生产视角

上线前必须做**通用能力回归**，把"遗忘率"作为硬性准入门槛。

# 8. DPO 与 RLHF 的区别？

## 8.1 标准答案

| 维度 | RLHF | DPO |
|------|:----:|:---:|
| 奖励模型 | 需要 | 不需要 |
| 强化学习 | PPO | 无 |
| 模型数 | 4 | 2 |
| 稳定性 | 差 | 好 |

**DPO 是当前主流**。

## 8.2 深入原理

### 8.2.1 RLHF 链路

先训奖励模型（RM），再用 PPO 优化策略，链路长、超参多、易不稳定。

### 8.2.2 DPO 的简化

直接用偏好对（chosen/rejected）构造损失，把奖励隐式编码进策略，省去 RM 与 RL。

## 8.3 代码示例

```java
double dpoLoss(LogProb chosen, LogProb rejected,
               LogProb refChosen, LogProb refRejected, double beta) {
    double a = beta * (chosen.value() - refChosen.value());
    double b = beta * (rejected.value() - refRejected.value());
    return -logSigmoid(a - b);
}
```

## 8.4 常见追问

**Q：DPO 需要参考模型吗？**
A：需要，用 SFT 后模型作为参考（ref_model）约束偏移。

## 8.5 生产视角

偏好数据（chosen/rejected）的构造成本常高于算法本身，应建立**标注规范与质检**流程。

# 9. DPO 的关键参数？

## 9.1 标准答案

| 参数 | 建议 |
|------|------|
| beta | 0.1~0.5 |
| 学习率 | 5e-7 ~ 1e-6 |
| 轮数 | 1~2 |
| ref_model | SFT 后的模型 |

## 9.2 深入原理

### 9.2.1 beta 的作用

beta 控制与参考模型的偏离程度：越大越保守，越小越激进，过小易退化。

### 9.2.2 学习率与轮数

DPO 学习率比 SFT 低一个量级，轮数 1~2，防止过拟合偏好对。

## 9.3 代码示例

```java
DpoConfig cfg = DpoConfig.builder()
    .beta(0.1)
    .learningRate(5e-7)
    .numEpochs(1)
    .refModel(sftModel)
    .build();
```

## 9.4 常见追问

**Q：beta 如何调？**
A：从 0.1 起步，观察对参考模型的 KL 偏离与评测表现。

## 9.5 生产视角

DPO 训练要监控**KL 偏离与长度漂移**，异常增长往往预示即将退化。

# 10. GRPO 是什么？

## 10.1 标准答案

**答**：DeepSeek 用的强化学习方法。

```
同一问题生成一组回答 → 打分 → 组内相对优势 → 更新
不需要 Value Model（比 PPO 省一半显存）
适合可验证奖励（数学、代码）
```

## 10.2 深入原理

### 10.2.1 组内相对优势

对同一问题采样多条回答，用组内均值作为基线，优势 = 得分 − 组均值，省去价值网络。

### 10.2.2 适用前提

奖励必须可自动验证（答案正确性、单元测试通过），否则难以稳定打分。

## 10.3 代码示例

```java
List<Double> rewards = samples.stream().map(this::score).toList();
double mean = avg(rewards);
for (int i = 0; i < samples.size(); i++) {
    double advantage = rewards.get(i) - mean;   // 相对优势
    update(samples.get(i), advantage);
}
```

## 10.4 常见追问

**Q：GRPO 与 PPO 区别？**
A：GRPO 用组内基线替代价值模型，显存更省、实现更简。

## 10.5 生产视角

GRPO 依赖**可靠的奖励函数**：奖励被刷（reward hacking）是最大风险，需设计防作弊校验。

# 11. 微调数据怎么准备？

## 11.1 标准答案

| 来源 | 质量 |
|------|:----:|
| 人工标注 | 最高 |
| 真实业务数据 | 高 |
| 强模型生成 | 中 |
| 失败案例回流 | 高 |

**清洗**：长度、质量、去重、有害、格式。**配比**：通用 + 领域 + 推理 + 代码 + 安全。

## 11.2 深入原理

### 11.2.1 数据决定上限

微调效果主要由数据质量与分布决定，算法与超参只是放大器。

### 11.2.2 分布匹配

训练数据分布应贴近线上真实输入，否则离线好、线上差。

## 11.3 代码示例

```java
List<Sample> clean = raw.stream()
    .filter(this::lengthOk)
    .filter(this::formatOk)
    .distinct()
    .filter(this::notHarmful)
    .toList();
```

## 11.4 常见追问

**Q：合成数据能用吗？**
A：能，但需强模型生成 + 人工抽检，防止错误传播。

## 11.5 生产视角

建立**数据飞轮**：线上失败案例回流、清洗、标注、再训练，持续迭代。

# 12. 如何部署微调后的模型？

## 12.1 标准答案

| 方式 | 说明 |
|------|------|
| vLLM | 高吞吐，OpenAI 兼容 |
| Ollama | 本地，极简 |
| TGI | HuggingFace |
| 合并 LoRA | 推理更快 |

**vLLM 支持动态加载 LoRA**。

## 12.2 深入原理

### 12.2.1 推理引擎的关键

连续批处理（continuous batching）与分页注意力（PagedAttention）决定吞吐。

### 12.2.2 合并与动态加载

合并 LoRA 简化部署但难以多租户；动态加载可在同一基座上切换多套适配器。

## 12.3 代码示例

```java
OpenAiChatModel model = OpenAiChatModel.builder()
    .baseUrl("http://vllm.internal:8000/v1")
    .modelName("my-finetuned")
    .build();
```

## 12.4 常见追问

**Q：vLLM 与 Ollama 如何选？**
A：生产高并发选 vLLM；本地开发与验证选 Ollama。

## 12.5 生产视角

部署需统一**限流、熔断、健康检查与灰度**，并把自部署模型纳入既有网关治理。

# 13. 量化有哪些格式？

## 13.1 标准答案

| 格式 | 场景 |
|------|------|
| AWQ/GPTQ | GPU 4bit |
| GGUF | CPU/本地 |
| INT8 | 几乎无损失 |
| INT4 | 性价比拐点 |

**量化后必须评测**。

## 13.2 深入原理

### 13.2.1 量化的取舍

以更低位宽换显存与速度，代价是精度损失，损失大小取决于校准数据与算法。

### 13.2.2 格式与硬件

GPU 推理多用 AWQ/GPTQ，CPU 或本地多用 GGUF，需与推理引擎匹配。

## 13.3 代码示例

```java
record QuantEval(String format, double accuracy, long vramMB, double qps) {}

List<QuantEval> report = List.of(
    eval("fp16", 1.00, 28000, 40),
    eval("int8", 0.995, 15000, 70),
    eval("int4", 0.980, 8000, 110));
```

## 13.4 常见追问

**Q：量化能救回微调损失吗？**
A：不能，量化是额外损失，需与微调效果叠加评估。

## 13.5 生产视角

量化必须用**业务评测集**验证，不能只看通用榜单，且需覆盖边界样本。

# 14. 微调后如何评估？

## 14.1 标准答案

1. **对比基座**（必须）；
2. 目标能力是否提升；
3. 通用能力是否退化（灾难性遗忘）；
4. 安全对齐是否破坏；
5. 格式遵循是否稳定。

## 14.2 深入原理

### 14.2.1 三向对比

基座、SFT 后、偏好对齐后三方对比，才能分离各阶段贡献。

### 14.2.2 维度覆盖

需同时看能力、安全、格式，任一维度退化都可能导致上线事故。

## 14.3 代码示例

```java
record FinetuneEval(
    double targetGain,        // 目标能力提升
    double generalDrop,       // 通用能力下降
    double safetyScore,       // 安全对齐
    double formatPassRate) {} // 格式遵循
```

## 14.4 常见追问

**Q：为什么必须对比基座？**
A：否则无法判断提升来自微调还是评测波动。

## 14.5 生产视角

把评估接入**发布门禁**：目标能力未提升或通用能力下降超阈值，禁止上线。

# 15. 微调的常见坑？

## 15.1 标准答案

| 坑 | 对策 |
|----|------|
| 数据质量差 | 严格筛选 |
| 轮数过多 | 1~3 轮 + 早停 |
| 灾难性遗忘 | 低学习率 + 混数据 |
| 数据泄漏 | 训练/测试隔离 |
| 不对比基座 | 必须对比 |
| 全量微调成本高 | 用 LoRA |

## 15.2 深入原理

### 15.2.1 数据泄漏

若评测样本混入训练集，指标虚高且线上崩盘，必须严格隔离。

### 15.2.2 过拟合

小数据多轮训练会记住样本而非学习规律，表现为离线好、线上差。

## 15.3 代码示例

```java
void assertNoLeak(Set<String> train, Set<String> test) {
    Set<String> overlap = new HashSet<>(train);
    overlap.retainAll(test);
    if (!overlap.isEmpty()) throw new IllegalStateException("数据泄漏");
}
```

## 15.4 常见追问

**Q：如何及早发现过拟合？**
A：留出验证集，观察验证损失回升即停。

## 15.5 生产视角

把常见坑固化为**训练前检查清单**，并在 CI 中自动校验数据隔离与配比。

# 16. 小结

## 16.1 核心结论

- 微调是最后手段，优先提示与 RAG；
- LoRA/QLoRA 是主流，成本低效果好；
- SFT 只在 assistant 部分算损失；
- DPO 跳过 RM 与 RL，GRPO 用可验证奖励；
- 必须对比基座评估效果。

## 16.2 速记清单

| 主题 | 一句话 |
|------|--------|
| 顺序 | 提示 → RAG → 微调 |
| 方法 | LoRA/QLoRA 优先 |
| 数据 | 质量优先、防泄漏 |
| 对齐 | DPO 主流，GRPO 用可验证奖励 |
| 部署 | vLLM 高吞吐、LoRA 可动态加载 |
| 评估 | 三向对比 + 通用能力回归 |
