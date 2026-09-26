# 1. LLM 应用的 CI 特殊性

传统 CI（持续集成，Continuous Integration）做"单测 + 构建"即可保证代码正确性；LLM（大语言模型，Large Language Model）应用的行为由模型与 Prompt（提示词）共同决定，代码通过并不代表效果与成本达标。因此必须在流水线中新增**评测回归、成本检查、Prompt 变更验证**三个环节。

## 1.1 传统 CI 与 LLM CI 的差异

### 1.1.1 评测回归

传统测试断言确定输出（`assertEquals`），LLM 输出是非确定的自然语言，无法逐字比对。评测回归改为对指标设阈值：召回率、忠实度、相关性、有害性等，低于阈值即失败。

### 1.1.2 成本检查

一次调用成本从不到一分钱到数元不等，Prompt 变长或换更大模型都会推高成本。CI 需估算单次平均成本，超过预算即拦截。

### 1.1.3 Prompt 变更验证

Prompt 是"用自然语言写的代码"，改动往往比代码更频繁，却缺乏编译器保护。CI 需识别 Prompt 文件变更并触发专项评测。

## 1.2 流水线全景

```
提交代码 → 单测 → 构建 → 集成测试 → 评测回归 → 成本检查 → 部署
                                    ↑
                            这是 LLM 应用新增的关键环节
```

传统 CI：单测 + 构建。
LLM 应用还需：**评测回归、成本检查、Prompt 变更验证**。

## 1.3 边界与坑

- 评测本身有噪声，单次结果不可信，需多次采样取均值。
- 评测集若与线上分布偏离，门禁会"假绿"。
- 把评测放在 PR 必过项会让流水线很慢，需权衡采样规模。
- 成本检查依赖价格表，模型调价后需同步更新。

# 2. CI 流水线

CI 流水线把上述环节串成自动门禁。核心设计原则：快反馈前置（单测最快）、贵环节后置（评测最慢）、失败即停。

## 2.1 触发条件

### 2.1.1 PR 触发

拉取请求（Pull Request）触发单测、Lint、构建与评测回归，作为合并门禁。

### 2.1.2 主干触发

合并到 `main` 触发集成测试与镜像构建，产出可部署产物。

```yaml
on:
  pull_request:
  push:
    branches: [main]
```

## 2.2 任务编排

### 2.2.1 依赖关系

用 `needs` 声明前置任务，形成 DAG（有向无环图）。`test → integration → evaluation`，任一失败则后续不执行，节省资源。

### 2.2.2 缓存与并行

Maven/Gradle 依赖、Docker 层均可缓存。独立任务（Lint 与单测）可并行以缩短总时长。

## 2.3 完整工作流示例

```yaml
name: ci

on:
  pull_request:
  push:
    branches: [main]

jobs:
  test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Setup JDK
        uses: actions/setup-java@v4
        with:
          java-version: '21'
          distribution: 'temurin'

      - name: Unit tests
        run: ./mvnw -q test

      - name: Lint
        run: ./mvnw -q checkstyle:check

      - name: Build
        run: ./mvnw -q -DskipTests package

  integration:
    needs: test
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Integration tests (mocked model)
        run: ./mvnw -q verify -Pintegration

  evaluation:
    needs: integration
    if: github.event_name == 'pull_request'
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Run eval regression
        env:
          OPENAI_API_KEY: ${{ secrets.EVAL_API_KEY }}
        run: ./mvnw -q exec:java -Dexec.mainClass=EvalRunner

      - name: Check thresholds
        run: ./scripts/check-eval-thresholds.sh
```

## 2.4 边界与坑

- 评测任务使用真实模型密钥，务必用 `secrets` 且避免打印。
- `if` 条件不当会导致评测在主干上被跳过，失去门禁意义。
- 长流水线应设超时，防止挂起占用额度。
- 缓存密钥应加前缀隔离，避免跨分支污染。

# 3. 单元测试：Mock 模型

单元测试要快、可重复、零外部依赖。模型调用慢、贵、不稳定，因此必须 Mock（打桩替换）。

## 3.1 为什么要 Mock

| 维度 | 真实模型 | Mock 模型 |
|------|----------|-----------|
| 速度 | 秒级 | 微秒级 |
| 成本 | 有 | 无 |
| 稳定性 | 非确定 | 确定 |
| 适用 | 评测 | 单测 |

## 3.2 Mock 示例

### 3.2.1 异常回退

```java
@ExtendWith(MockitoExtension.class)
class ChatServiceTest {

    @Test
    void should_fallback_when_model_fails() {
        ChatModel mock = mock(ChatModel.class);
        when(mock.chat(any(String.class)))
                .thenThrow(new RuntimeException("模型不可用"));

        ChatService service = new ChatService(mock, fallbackService);
        assertEquals("服务暂时不可用，请稍后重试", service.ask("你好"));
    }
}
```

> **CI 中的单测绝不真调模型**（慢、贵、不稳定）。

### 3.2.2 正常路径与参数校验

```java
@Test
void should_return_model_answer() {
    ChatModel mock = mock(ChatModel.class);
    when(mock.chat("你好")).thenReturn("你好，有什么可以帮你？");

    ChatService service = new ChatService(mock, fallbackService);
    assertEquals("你好，有什么可以帮你？", service.ask("你好"));
    verify(mock, times(1)).chat(anyString());
}
```

## 3.3 边界与坑

- Mock 过度会让测试与实现强耦合，重构时大面积报错。
- Mock 掉的正是关键集成点时，单测通过但线上失败。
- 断言非确定输出时应用规则匹配（正则、包含）而非全等。
- 忘记 `verify` 调用次数，可能漏测重试/缓存逻辑。

# 4. 集成测试：Testcontainers

集成测试验证"真实组件协作"，但模型仍用 Mock，避免不确定性与成本。Testcontainers 在 CI 中按需启动真实数据库/向量库容器。

## 4.1 为什么用 Testcontainers

- 环境与生产一致（真实 PostgreSQL/pgvector）。
- 无需维护共享测试数据库，隔离性好。
- 生命周期随测试自动启停。

## 4.2 示例

### 4.2.1 真实向量库 + Mock Embedding

```java
@Testcontainers
class RagIntegrationTest {

    @Container
    static PostgreSQLContainer<?> postgres = new PostgreSQLContainer<>("pgvector/pgvector:pg16");

    @Test
    void should_retrieve_relevant_document() {
        // 使用真实向量库 + Mock Embedding
        VectorStore store = PgVectorStore.builder(
                jdbcTemplate(postgres), mockEmbeddingModel()).build();
        // ...
    }
}
```

### 4.2.2 数据库迁移验证

在容器启动后执行 Flyway/Liquibase 迁移，验证 schema 与索引创建成功，防止上线时迁移失败。

```java
@Container
static PostgreSQLContainer<?> postgres =
        new PostgreSQLContainer<>("pgvector/pgvector:pg16");

@BeforeAll
static void migrate() {
    Flyway.configure().dataSource(
            postgres.getJdbcUrl(), postgres.getUsername(), postgres.getPassword())
          .load().migrate();
}
```

## 4.3 边界与坑

- CI Runner 需支持 Docker，否则 Testcontainers 无法启动。
- 镜像拉取慢会拖长流水线，建议预热或缓存。
- 集成测试仍可能因端口/资源竞争而偶发失败，需重试策略。
- 不要在集成测试中调用真实模型，否则退回"慢、贵、不稳"。

# 5. 评测回归（关键环节）

评测回归是 LLM 应用 CI 的核心增量：用固定评测集衡量效果，并设阈值门禁，防止"改一处、坏一处"。

## 5.1 评测指标

### 5.1.1 检索指标

| 指标 | 含义 | 参考阈值 |
|------|------|----------|
| Recall@k | 前 k 条命中相关文档比例 | ≥ 0.85 |
| MRR | 首个相关结果排名倒数均值 | ≥ 0.70 |
| Hit Rate | 至少命中一条的比例 | ≥ 0.90 |

### 5.1.2 生成指标

| 指标 | 含义 | 参考阈值 |
|------|------|----------|
| Faithfulness | 回答是否忠于检索内容 | ≥ 0.90 |
| Answer Relevance | 回答与问题相关性 | ≥ 0.80 |
| Latency | 平均/P99 延迟 | 不劣化 |

## 5.2 评测运行器

```java
public class EvalRunner {

    public static void main(String[] args) {
        List<EvalCase> cases = loadCases("eval/cases.json");
        EvalReport report = evaluator.evaluate(cases);

        System.out.printf("Recall@5: %.3f%n", report.recallAt5());
        System.out.printf("Faithfulness: %.3f%n", report.faithfulness());
        System.out.printf("AvgLatency: %d ms%n", report.avgLatencyMs());
        System.out.printf("AvgCost: $%.4f%n", report.avgCost());

        // 阈值校验
        if (report.recallAt5() < 0.85) {
            System.err.println("召回率未达标");
            System.exit(1);
        }
        if (report.faithfulness() < 0.90) {
            System.err.println("忠实度未达标");
            System.exit(1);
        }
    }
}
```

## 5.3 阈值门禁

退出码非 0 即流水线失败，阻止合并。阈值应基于历史基线动态调整，而非拍脑袋。

## 5.4 边界与坑

- 评测集需版本化，随业务更新，避免过时。
- 生成式指标本身常用 LLM 评判（LLM-as-Judge），需固定评判模型版本。
- 单次评测波动大，建议多次采样取均值并设置信区间。
- 评测耗时与成本高，PR 可用子集，发版前跑全量。

# 6. Prompt 变更的 CI

Prompt 是高频变更对象，需要"改 Prompt 即跑对应评测"的自动联动。

## 6.1 变更检测

```yaml
# 检测 Prompt 文件变更，自动跑相关评测
- name: Detect prompt changes
  id: prompt_changes
  run: |
    if git diff --name-only origin/main | grep -q '^resources/prompts/'; then
      echo "changed=true" >> $GITHUB_OUTPUT
    fi
```

## 6.2 专项评测

```yaml
- name: Run prompt eval
  if: steps.prompt_changes.outputs.changed == 'true'
  run: ./mvnw -q exec:java -Dexec.mainClass=PromptEvalRunner
```

按 Prompt 归属（如某业务线）只跑相关评测子集，缩短反馈。

## 6.3 边界与坑

- `git diff` 基准分支写错会导致检测失效，需明确 `origin/main`。
- Prompt 内联在代码里会绕过文件检测，应外置到 `resources/prompts/`。
- Prompt 变更常伴随模型参数变化，评测需一并覆盖。
- 未记录 Prompt 版本，出问题无法归因。

# 7. 成本检查

成本是 LLM 应用的"新性能指标"，必须在 CI 中量化并设阈值。

## 7.1 成本估算

按输入/输出 token 单价与平均 token 数估算单次成本。批量估算需使用评测集或采样日志。

## 7.2 阈值门禁

```yaml
- name: Cost regression check
  run: |
    AVG_COST=$(./mvnw -q exec:java -Dexec.mainClass=CostEstimator | tail -1)
    if (( $(echo "$AVG_COST > 0.05" | bc -l) )); then
      echo "单次调用成本超阈值: $AVG_COST"
      exit 1
    fi
```

## 7.3 边界与坑

- `bc` 在部分 Runner 未预装，可用 `awk` 替代。
- 成本估算依赖价格表，需随模型调价维护。
- 只算平均不算 P99 会漏掉长尾高成本请求。
- 缓存命中率变化会显著影响成本，需纳入统计。

# 8. CD 流水线

CD（持续交付/部署，Continuous Delivery/Deployment）把通过 CI 的产物安全送上生产。LLM 应用推荐"金丝雀 + 自动验证 + 逐步放量"。

## 8.1 构建与推送

```yaml
name: cd

on:
  push:
    tags: ['v*']

jobs:
  deploy:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Build and push image
        run: |
          docker build -t registry/ai-app:${{ github.ref_name }} .
          docker push registry/ai-app:${{ github.ref_name }}
```

## 8.2 金丝雀部署

### 8.2.1 部署金丝雀

```yaml
      - name: Deploy canary (10%)
        run: |
          kubectl set image deployment/ai-app-canary \
            ai-app=registry/ai-app:${{ github.ref_name }}
          kubectl scale deployment/ai-app-canary --replicas=1
```

### 8.2.2 验证与提升

```yaml
      - name: Wait and verify
        run: ./scripts/verify-canary.sh

      - name: Promote to production
        run: |
          kubectl set image deployment/ai-app \
            ai-app=registry/ai-app:${{ github.ref_name }}
```

验证脚本检查错误率、P99 延迟、成本与评测分数，全部达标才放量。

## 8.3 边界与坑

- 金丝雀未接真实流量则验证无效，需在入口做流量切分。
- 自动提升缺少人工确认，高风险系统应设手动审批。
- 镜像 tag 用 `github.ref_name` 需保证唯一且不可覆盖。
- 回滚步骤必须在 CD 中显式定义并演练。

# 9. 分环境策略

## 9.1 环境矩阵

| 环境 | 触发 | 评测 | 模型 |
|------|------|------|------|
| dev | 每次提交 | 单测 | 小模型/Mock |
| test | 合并到 main | 集成测试 | 小模型 |
| staging | 发版前 | 完整评测 | 与生产同 |
| prod | 打标签 | 冒烟 + 灰度 | 正式模型 |

## 9.2 晋级规则

- 低级环境失败不晋级。
- staging 与生产用同一模型与配置，确保可比。
- prod 发布走金丝雀，逐步放量。

## 9.3 边界与坑

- 各环境配置漂移会让 staging 验证失去意义。
- dev/test 用小模型可能掩盖大模型特有问题（如上下文长度）。
- 生产密钥与测试密钥必须隔离，防止误用。

# 10. 生产注意事项

## 10.1 风险清单

| 风险 | 建议 |
|------|------|
| CI 真调模型 | 用 Mock |
| 无评测回归 | 必加环节 |
| 评测不稳定 | 多次采样取均值 |
| 无成本检查 | 加成本阈值 |
| 密钥进 CI 日志 | 用 secrets 并脱敏 |
| 无灰度 | 金丝雀发布 |

## 10.2 检查清单

- [ ] 单测零模型调用。
- [ ] 集成测试用 Testcontainers。
- [ ] 评测集版本化且含阈值门禁。
- [ ] Prompt 变更自动触发专项评测。
- [ ] 成本检查覆盖均值与长尾。
- [ ] CD 含金丝雀验证与回滚步骤。
- [ ] 所有密钥走 secrets 且日志脱敏。

# 11. 小结

- LLM 应用 CI 新增**评测回归**与**成本检查**。
- 单测 Mock 模型，集成测试用 Testcontainers。
- Prompt 变更触发专项评测。
- CD 用金丝雀发布 + 自动验证 + 逐步放量。
- 密钥用 CI secrets，绝不进日志。
