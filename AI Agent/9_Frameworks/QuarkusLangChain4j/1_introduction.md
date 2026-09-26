# 1. 定位

## 1.1 一句话定位

### 1.1.1 结论先行

**Quarkus LangChain4j 是 LangChain4j 的 Quarkus 官方扩展（位于 Quarkiverse 孵化组织），把 LangChain4j 的 LLM 能力层与 Quarkus 的编译期运行时层结合，面向云原生与 GraalVM 原生镜像场景。**

它不是一个独立的 AI 框架，而是"能力 + 运行时"的叠加：能力层几乎完全复用 LangChain4j，运行时层则替换为 Quarkus 的编译期容器。

### 1.1.2 分层机制

```
LangChain4j（能力层）
  ├── ChatLanguageModel / StreamingChatLanguageModel
  ├── EmbeddingModel / EmbeddingStore
  ├── Tools / Memory / RAG
  └── AiServices（运行时动态代理）
        │  编译期扩展（quarkus-langchain4j-core）
        ▼
Quarkus（运行时层）
  ├── 编译期注入（build-time injection）
  ├── 快速启动 / 低内存
  ├── GraalVM 原生镜像
  └── CDI / REST / 配置 / 可观测
```

关键点在于中间那条"编译期扩展"通道：Quarkus 扩展在构建期扫描 `@RegisterAiService`，为每个接口生成实现类并注册为 CDI Bean，因此运行期不需要反射创建代理。

### 1.1.3 与纯 LangChain4j 的差异

| 维度 | 纯 LangChain4j | Quarkus LangChain4j |
|------|----------------|---------------------|
| Bean 创建 | 手写 `AiServices.builder()` | 注解 + 编译期生成 |
| 模型装配 | 手动 new / 工厂 | 配置驱动 + CDI |
| 启动开销 | 运行时初始化 | 构建期预计算 |
| 原生镜像 | 需自行配置 | 扩展内置支持 |
| 配置来源 | 代码 / 自建 | `application.properties` |
| 作用域 | 自行管理 | CDI Scope |

## 1.2 能力矩阵

### 1.2.1 LangChain4j 提供什么

- 统一的 `ChatLanguageModel` 抽象，屏蔽 OpenAI、Anthropic、Ollama、Gemini 等厂商差异。
- `EmbeddingModel` / `EmbeddingStore`，用于向量化与向量检索。
- `@Tool` 工具调用、`ChatMemory` 对话记忆、`RetrievalAugmentor` 检索增强。
- `AiServices` 声明式接口，把接口方法映射为 LLM 调用。

### 1.2.2 Quarkus 提供什么

- 编译期 DI（ArC，Quarkus 的 CDI 实现），替代 Spring 的运行期反射容器。
- 统一配置系统 `quarkus.*`，支持 profile、环境变量、Secret。
- REST（RESTEasy Reactive）、健康检查、Micrometer 指标开箱即用。
- GraalVM 原生镜像构建链，扩展作者负责注册反射与资源。

### 1.2.3 能力边界

| 能力 | 是否由扩展提供 | 备注 |
|------|----------------|------|
| 模型调用 | 是 | 多厂商 |
| 流式输出 | 是 | `Multi<String>` |
| 工具调用 | 是 | `@Tool` |
| 对话记忆 | 是 | 内存 / Redis |
| RAG | 是 | 复用 LangChain4j |
| Agent 编排 | 部分 | 依赖 LangChain4j 版本 |
| 提示词运营平台 | 否 | 需自建 |

## 1.3 版本与坐标

### 1.3.1 Maven 坐标

```xml
<dependency>
    <groupId>io.quarkiverse.langchain4j</groupId>
    <artifactId>quarkus-langchain4j-openai</artifactId>
    <version>0.24.0</version>
</dependency>
```

### 1.3.2 版本兼容

| 组件 | 约束 |
|------|------|
| JDK | 17 或 21（推荐 21 LTS） |
| Quarkus | 与扩展版本强绑定 |
| LangChain4j | 由扩展传递依赖锁定 |
| GraalVM | 与 Quarkus 版本对应 |

> 版本以官方最新为准；扩展版本号与 Quarkus 平台版本存在对应关系，升级需成对进行。

### 1.3.3 边界与坑

- 混用不同小版本的 `quarkus-langchain4j-*` 扩展可能引发 CDI 构建期报错。
- 手动覆盖 LangChain4j 传递依赖版本容易与扩展内部的 API 假设冲突。
- 扩展处于 0.x 阶段，小版本间存在破坏性变更，必须锁定版本。

## 1.4 生产建议

- 用 `quarkus-langchain4j-bom` 或平台 BOM 统一管理版本，避免手工指定。
- 升级前先在测试环境跑一次原生镜像构建，确认反射配置未失效。
- 在 `pom.xml` 中显式记录扩展版本与对应 Quarkus 版本，形成升级台账。

# 2. 核心优势

## 2.1 启动快与内存低

### 2.1.1 结论先行

**Quarkus 把大量工作在构建期完成，运行期只做最小初始化，因此启动可达毫秒级、常驻内存显著低于传统框架。**

### 2.1.2 机制拆解

传统框架在启动时扫描类路径、解析注解、创建 Bean；Quarkus 在构建期生成字节码记录这些元数据，运行期直接读取。

```
构建期（build-time）          运行期（runtime）
  扫描注解                       读取已生成的元数据
  解析配置                       按需实例化
  生成 Bean 字节码               启动 HTTP 层
  预计算反射信息
```

### 2.1.3 数据参考

| 指标 | JVM 模式 | 原生镜像 |
|------|----------|----------|
| 启动时间 | 数百毫秒 | 数十毫秒 |
| 常驻内存 | 数百 MB | 数十 MB |
| 镜像体积 | 较大 | 较小 |

> 具体数值随应用规模变化，应以自身压测为准。

## 2.2 编译期注入

### 2.2.1 CDI 与 build-time

Quarkus 使用 ArC 作为 CDI 实现。与 Spring 不同，Bean 的依赖关系在构建期确定，运行期不做类路径扫描。

### 2.2.2 对 AI Service 的意义

`@RegisterAiService` 标注的接口会在构建期被生成实现，并注册为 CDI Bean，因此可以直接 `@Inject`，且原生镜像下无需额外反射注册。

### 2.2.3 与 Spring 容器的差异

| 维度 | Quarkus ArC | Spring |
|------|-------------|--------|
| 扫描时机 | 构建期 | 启动期 |
| 代理方式 | 生成的字节码 | CGLIB / JDK 代理 |
| 未满足依赖 | 构建期失败 | 启动期失败 |
| 原生镜像 | 天然友好 | 需额外配置 |

## 2.3 声明式编程模型

### 2.3.1 代码

```java
@RegisterAiService
public interface Assistant {

    @SystemMessage("你是一个专业的中文助手。")
    String chat(@UserMessage String message);
}
```

### 2.3.2 注解到调用的映射

接口即契约：方法签名表达调用意图，注解表达提示词与配置，无需手写样板代码。

| 元素 | 语义 |
|------|------|
| 接口 | 一个 AI Service |
| 方法 | 一次 LLM 调用 |
| 参数 | 消息或模板变量 |
| 返回值 | 文本 / 对象 / 流 |

## 2.4 云原生集成

### 2.4.1 能力清单

| 能力 | 扩展 |
|------|------|
| REST | quarkus-resteasy-reactive |
| 健康检查 | quarkus-smallrye-health |
| 指标 | quarkus-micrometer-registry-prometheus |
| 容错 | quarkus-smallrye-fault-tolerance |
| 配置 | quarkus-config |

### 2.4.2 与容器平台的契合

```
Serverless 平台
  ├── 冷启动敏感  → 原生镜像毫秒级启动
  ├── 内存计费    → 常驻内存低
  ├── 水平伸缩    → 无状态设计
  └── 快速销毁    → 无需优雅停机优化
```

### 2.4.3 边界与坑

- 原生镜像的构建产物与 JVM 产物不同，部署脚本需区分。
- 健康检查探针若依赖外部模型可用性，会导致启动期反复重启。

## 2.5 边界与坑

- 编译期注入意味着无法在运行期动态新增 Bean 类型（需借助编程式 API）。
- 热重载在开发模式可用，但修改注解后需等待重新编译。
- 大量 Bean 会增加构建时间，扩展数量应保持克制。

## 2.6 生产建议

- 对启动敏感的场景优先评估原生镜像收益。
- 不要在静态初始化块中执行网络调用，否则会拖慢原生镜像启动。
- 用 `@ApplicationScoped` 而非 `@Singleton` 作为默认作用域，保持 CDI 语义。

# 3. 与 Spring Boot 版对比

## 3.1 维度对比

### 3.1.1 结论先行

**两者能力层都基于 LangChain4j，差异主要在运行时模型：Quarkus 偏编译期与资源效率，Spring Boot 偏生态与熟悉度。**

### 3.1.2 对比表

| 维度 | Quarkus | Spring Boot |
|------|---------|-------------|
| 启动 | 毫秒级 | 秒级 |
| 内存 | 低 | 高 |
| 原生镜像 | 一等公民 | 支持 |
| 依赖注入 | 编译期（ArC） | 运行期（反射） |
| 生态 | 增长中 | 最大 |
| 上手 | 需学 CDI | 熟悉度高 |
| 配置前缀 | `quarkus.langchain4j.*` | `langchain4j.*` |
| 热重载 | quarkus:dev | devtools |

### 3.1.3 配置映射

| 语义 | Quarkus | Spring Boot |
|------|---------|-------------|
| API Key | `quarkus.langchain4j.openai.api-key` | `langchain4j.openai.api-key` |
| 模型名 | `...chat-model.model-name` | `...chat-model.model-name` |
| 温度 | `...chat-model.temperature` | `...chat-model.temperature` |
| 超时 | `...chat-model.timeout` | `...chat-model.timeout` |

## 3.2 选型建议

### 3.2.1 选 Quarkus 的信号

- 部署在 Serverless / Knative / Cloud Run，冷启动敏感。
- 容器密度高，资源成本是核心指标。
- 团队已接受或愿意学习 CDI 与 Quarkus 生态。

### 3.2.2 选 Spring Boot 的信号

- 需要大量成熟第三方库与企业集成。
- 团队对 Spring 生态高度熟悉，迁移成本不划算。
- 应用为常驻型，启动与内存不是瓶颈。

### 3.2.3 边界与坑

- 不要因为"原生镜像"而盲目迁移：若依赖库不支持原生，收益可能被抵消。
- 两者 API 命名相近但配置前缀不同，迁移时容易漏改配置。
- 注解名可能相同（如 `@Tool`），但装配方式不同，不能直接复制代码。

## 3.3 生产建议

- 用同一套提示词与评测集在两种实现间做 A/B，比较的是资源指标而非功能。
- 若已用 Spring Boot 且稳定，除非有明确资源诉求，否则不建议重写。
- 若确有迁移需求，优先迁移无状态、启动敏感的服务。

# 4. 依赖引入

## 4.1 核心依赖

```xml
<dependency>
    <groupId>io.quarkiverse.langchain4j</groupId>
    <artifactId>quarkus-langchain4j-openai</artifactId>
    <version>0.24.0</version>
</dependency>
```

> 版本以官方最新为准。

## 4.2 可选扩展

| 扩展 | 用途 |
|------|------|
| quarkus-langchain4j-openai | OpenAI 兼容模型 |
| quarkus-langchain4j-ollama | 本地模型 |
| quarkus-langchain4j-pgvector | pgvector 向量库 |
| quarkus-langchain4j-redis | Redis 向量库 |
| quarkus-langchain4j-easy-rag | 开箱即用 RAG |

## 4.3 BOM 管理

```xml
<dependencyManagement>
    <dependencies>
        <dependency>
            <groupId>io.quarkiverse.langchain4j</groupId>
            <artifactId>quarkus-langchain4j-bom</artifactId>
            <version>0.24.0</version>
            <type>pom</type>
            <scope>import</scope>
        </dependency>
    </dependencies>
</dependencyManagement>
```

## 4.4 边界与坑

- 引入多个模型扩展（如同时 openai 与 ollama）时，需明确默认模型，否则装配歧义。
- 部分扩展在原生镜像下需要额外资源注册，需查看扩展文档。
- BOM 只管理版本，不引入依赖；仍需显式声明需要的扩展。

## 4.5 生产建议

- 只用 BOM 管理版本，不在各 dependency 中散落版本号。
- 依赖引入后立即执行一次 `./mvnw package`，尽早暴露构建期冲突。
- 定期检查扩展的变更日志，评估升级风险。

# 5. 配置

## 5.1 基础配置

```properties
quarkus.langchain4j.openai.api-key=${OPENAI_API_KEY}
quarkus.langchain4j.openai.chat-model.model-name=gpt-4o-mini
quarkus.langchain4j.openai.chat-model.temperature=0.3
```

## 5.2 常用参数

| 配置项 | 含义 | 建议 |
|--------|------|------|
| api-key | 访问密钥 | 走环境变量 |
| chat-model.model-name | 模型名 | 显式指定 |
| chat-model.temperature | 采样温度 | 抽取类 0~0.2 |
| chat-model.timeout | 超时 | 显式设置 |
| chat-model.log-requests | 记录请求 | 生产关闭 |
| chat-model.log-responses | 记录响应 | 生产关闭 |

## 5.3 Profile 与环境覆盖

```properties
%dev.quarkus.langchain4j.openai.chat-model.model-name=gpt-4o-mini
%prod.quarkus.langchain4j.openai.chat-model.model-name=gpt-4o
%prod.quarkus.langchain4j.openai.chat-model.log-requests=false
```

### 5.3.1 边界与坑

- `${...}` 引用的环境变量缺失时，启动期即失败（快速失败，属预期行为）。
- 将密钥写入 `application.properties` 并提交，是最常见的安全事故。
- Profile 配置优先级高于默认配置，但低于系统属性。

### 5.3.2 生产建议

- 密钥统一来自 Secret / 环境变量，仓库中只保留占位符。
- 用 profile 区分 dev/prod 的模型与日志级别。

## 5.4 多模型装配

### 5.4.1 命名模型

```properties
quarkus.langchain4j.openai.chat-model.model-name=gpt-4o-mini
quarkus.langchain4j.openai.chat-model.responses-api-key=${OPENAI_API_KEY}
```

### 5.4.2 边界与坑

- 多个同名模型 Bean 会导致注入歧义，需用 `@ModelName` 或限定符区分。
- 不同厂商的配置前缀不同，注意不要串用。

# 6. 声明式 AI Service

## 6.1 最小示例

```java
@RegisterAiService
public interface Assistant {

    @SystemMessage("你是一个专业的中文助手。")
    String chat(@UserMessage String message);
}
```

## 6.2 注解详解

### 6.2.1 @RegisterAiService

标记接口为 AI Service，构建期生成实现并注册为 Bean。可配置模型、工具、记忆、检索增强器等。

### 6.2.2 @SystemMessage / @UserMessage

- `@SystemMessage`：系统提示，定义角色与约束，可放在接口或方法上。
- `@UserMessage`：用户消息模板，支持 `{{it}}` 引用参数或命名占位符。

```java
@RegisterAiService
public interface Extractor {

    @UserMessage("从文本抽取字段，输出 JSON：{{it}}")
    Person extract(String text);
}
```

### 6.2.3 返回类型

| 返回类型 | 语义 |
|----------|------|
| `String` | 纯文本 |
| 自定义 POJO | 结构化输出 |
| `Multi<String>` | 流式 |
| `TokenStream` | 更细粒度流式 |

## 6.3 注入与调用

```java
@Inject
Assistant assistant;

public String ask(String question) {
    return assistant.chat(question);
}
```

### 6.3.1 边界与坑

- 接口方法返回值需是可序列化/可映射类型，复杂对象依赖模型输出结构。
- 提示词写在注解里，改动需重新编译，不适合频繁运营调整。
- AI Service 接口不应继承业务接口，避免代理冲突。

### 6.3.2 生产建议

- 提示词模板集中管理，避免散落在各方法注解中。
- 对结构化输出使用明确的 JSON 契约，并做结果校验。

## 6.4 编程式 API 兜底

### 6.4.1 何时用

当需要动态切换模型、运行期拼装提示词，或注解模型无法表达时，改用编程式 `AiServices`。

### 6.4.2 示例

```java
Assistant assistant = AiServices.builder(Assistant.class)
        .chatLanguageModel(model)
        .chatMemory(memory)
        .build();
```

> 编程式 API 失去部分编译期优化与原生镜像友好性，仅在必要时使用。

# 7. 适用场景

## 7.1 典型场景

| 场景 | 说明 |
|------|------|
| Serverless | 启动快、内存低 |
| 容器密集部署 | 资源效率高 |
| 原生镜像 | GraalVM |
| 事件驱动 | 与 Quarkus 生态 |
| 边缘 / IoT | 资源受限 |

## 7.2 反例

### 7.2.1 反例清单

| 不适用 | 原因 |
|--------|------|
| 依赖庞大 Spring 生态 | 迁移成本高 |
| 需要大量成熟三方库 | 生态覆盖不足 |
| 团队不熟 CDI | 学习曲线 |
| 动态插件化架构 | 编译期模型不友好 |

### 7.2.2 边界与坑

- 反例并非绝对禁止，而是提示成本收益比不佳，需显式权衡。
- 若核心依赖不支持原生，可退回 JVM 模式，仍保留 Quarkus 的开发体验。

## 7.3 生产建议

- 用最小可行应用先验证依赖在原生镜像下的兼容性，再决定规模化。
- 把"是否原生"作为独立决策项，而非默认开启。

# 8. 生产注意事项

## 8.1 风险清单

| 风险 | 建议 |
|------|------|
| 生态较小 | 评估成熟度 |
| 原生镜像限制 | 测试反射配置 |
| 团队不熟 Quarkus | 学习成本 |
| 版本迭代 | 锁定版本 |
| 密钥硬编码 | 环境变量 / Secret |
| 无监控 | 集成 Micrometer |

## 8.2 落地清单

- [ ] 版本用 BOM 锁定
- [ ] 密钥来自环境变量
- [ ] 配置超时与重试
- [ ] 原生镜像构建纳入 CI
- [ ] 指标与健康检查接入
- [ ] 提示词与评测集版本化

## 8.3 边界与坑

- 原生镜像构建耗时数分钟，不宜放在每次提交的主流程中。
- 反射配置缺失往往在运行期才暴露，需在 CI 中跑一次原生冒烟测试。
- 扩展 0.x 阶段的破坏性变更需通过版本锁定与回归测试兜底。

## 8.4 生产建议

- 建立"JVM 模式日常开发 + 原生模式发布前验证"的双轨流程。
- 对模型调用设置超时、重试、熔断三层防护。
- 为每个 AI Service 定义明确的所有权与变更流程。

# 9. 小结

## 9.1 要点回顾

- Quarkus LangChain4j 是 LangChain4j 的 Quarkus 扩展，能力层复用、运行时层替换。
- 优势：启动快、内存低、编译期注入、原生镜像一等公民。
- 声明式 AI Service（CDI + 注解）是核心编程模型。
- 与 Spring Boot 版的差异主要在运行时模型与生态成熟度。
- 适合云原生、Serverless、资源受限场景。
- 生态较小、原生镜像兼容性需评估，版本需锁定，密钥需外置。

## 9.2 落地行动

- 用 BOM 锁定版本，密钥外置，接入指标与健康检查。
- 为每个 AI Service 建立契约测试与所有权。
