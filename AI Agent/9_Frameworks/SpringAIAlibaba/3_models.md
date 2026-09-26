# 1. 模型总览

## 1.1 支持的模型清单

结论先行：Spring AI Alibaba 以通义千问（Qwen，阿里自研大模型）为核心，向上覆盖对话、长文本、多模态与图像生成，并借助 OpenAI 兼容接口接入任意兼容厂商的模型。

| 模型 | 说明 |
|------|------|
| 通义千问（Qwen） | 阿里自研，中英双语 |
| 通义万相 | 图像生成 |
| 百炼平台 | 多模型聚合 |
| 其他 OpenAI 兼容 | 通过兼容接口 |

## 1.2 模型能力矩阵

机制：不同模型面向不同模态与任务，选型前应确认"输入模态 / 输出模态 / 上下文长度 / 成本"四个维度。

| 模型族 | 输入模态 | 输出模态 | 典型用途 |
|--------|----------|----------|----------|
| qwen-turbo | 文本 | 文本 | 高频轻量任务 |
| qwen-plus | 文本 | 文本 | 通用对话 |
| qwen-max | 文本 | 文本 | 复杂推理 |
| qwen-long | 长文本 | 文本 | 长文档问答 |
| qwen-vl | 文本 + 图像 | 文本 | 视觉理解 |
| 通义万相 | 文本 | 图像 | 文生图 |
| text-embedding-v3 | 文本 | 向量 | 检索、聚类 |

## 1.3 选型原则

- 按任务复杂度分层：简单任务走 turbo，复杂任务走 max；
- 按模态选型：需要"看图"选 qwen-vl，需要"出图"选通义万相；
- 按上下文选型：超长文档优先 qwen-long；
- 永远准备备用模型：单一模型不可用时必须能切换。

> 选型的本质是"效果 / 成本 / 延迟"的三角权衡，不存在通吃的最优解。

## 1.4 版本与兼容性

- 模型名（如 `qwen-plus`）会随时间迭代，同一名称背后可能是新版本；
- 若业务对输出稳定性要求高，应在配置中锁定带版本号的模型名；
- 兼容性变更（参数默认值、上下文长度）需在升级前做回归测试。

| 变更类型 | 影响 | 应对 |
|----------|------|------|
| 模型名指向新版本 | 输出风格变化 | 锁定版本 + 回归 |
| 上下文长度调整 | 长文可能超限 | 监控 Token |
| 参数默认值变化 | 结果漂移 | 显式设置参数 |

# 2. 通义千问接入

## 2.1 依赖与配置

### 2.1.1 Maven 依赖

结论先行：接入第一步是引入 DashScope 适配的 starter，并让 Spring Boot 自动装配完成 `ChatModel`。

```xml
<dependency>
    <groupId>com.alibaba.cloud.ai</groupId>
    <artifactId>spring-ai-alibaba-starter-dashscope</artifactId>
</dependency>
```

### 2.1.2 YAML 配置

机制：配置驱动是首选方式，密钥通过环境变量注入，绝不硬编码。

```yaml
spring:
  ai:
    dashscope:
      api-key: ${DASHSCOPE_API_KEY}
      chat:
        options:
          model: qwen-plus        # qwen-turbo / qwen-plus / qwen-max
          temperature: 0.7
```

> `api-key` 用 `${DASHSCOPE_API_KEY}` 占位，由运行环境注入，仓库中不出现真实密钥。

### 2.1.3 多环境配置

机制：用 Spring Profile 隔离环境差异，公共部分沉到 `application.yml`，差异部分放到 `application-{env}.yml`。

生产环境文件 `application-prod.yml`：

```yaml
spring:
  ai:
    dashscope:
      chat:
        options:
          model: qwen-max
```

开发环境文件 `application-dev.yml`：

```yaml
spring:
  ai:
    dashscope:
      chat:
        options:
          model: qwen-turbo
```

- 开发环境用便宜模型快速验证流程；
- 生产环境用高能力模型保证效果；
- 密钥统一由密钥管理服务注入，不随环境文件分发。

## 2.2 编程式接入

### 2.2.1 ChatModel 构建

当需要多模型并存（例如同时持有 turbo 与 max）时，用编程式构建更灵活。

```java
@Bean("plusModel")
ChatModel plusModel(@Value("${spring.ai.dashscope.api-key}") String apiKey) {
    return DashScopeChatModel.builder()
            .apiKey(apiKey)
            .modelName("qwen-plus")
            .build();
}
```

### 2.2.2 ChatClient 封装

`ChatModel` 是底层模型，`ChatClient` 是面向业务的封装，携带默认系统提示。

```java
@Bean
ChatClient chatClient(ChatModel chatModel) {
    return ChatClient.builder(chatModel)
            .defaultSystem("你是一个专业的中文助手。")
            .build();
}
```

## 2.3 运行验证与边界

- 启动后调用 `/actuator/health`，确认自动装配无冲突；
- 首次调用打印实际使用的 `model`，避免配置未生效；
- 边界：多个 `ChatModel` Bean 会导致注入歧义，需用 `@Qualifier` 或 `@Primary`；
- 生产建议：把模型名、温度等做成可热更新的配置项。

## 2.4 常见错误与排查

| 现象 | 可能原因 | 排查 |
|------|----------|------|
| 启动报缺少 ChatModel | 未引入 starter | 检查依赖 |
| 401 未授权 | 密钥错误/未注入 | 打印密钥长度 |
| 模型名无效 | 拼写或区域不支持 | 核对控制台 |
| 响应被截断 | max-tokens 过小 | 调大上限 |
| 注入歧义 | 多个 ChatModel | `@Qualifier` |

- 边界：把密钥打印到日志是严重安全问题，排查时只打印"是否存在/长度"；
- 生产建议：启动时做一次"冒烟调用"，尽早暴露配置错误。

# 3. 模型分层与路由

## 3.1 分层模型

结论先行：把模型按"能力 / 成本"分层，让不同任务走不同档位，是控制成本最有效的手段。

| 模型 | 定位 | 成本 |
|------|------|:----:|
| qwen-turbo | 快速、便宜 | 低 |
| qwen-plus | 平衡 | 中 |
| qwen-max | 最强 | 高 |
| qwen-long | 长上下文 | 中 |

机制：分层后，路由层根据任务类型选择档位，避免"所有请求都打 max"造成的浪费。

## 3.2 路由实现

### 3.2.1 静态路由

静态路由按任务类型硬编码映射，实现简单、可预测。

```java
public ChatClient route(String taskType) {
    return switch (taskType) {
        case "classify", "extract" -> turboClient;
        case "complex" -> maxClient;
        default -> plusClient;
    };
}
```

### 3.2.2 动态路由

动态路由先由小模型做难度判定，再决定用哪一档。

```java
public ChatClient dynamicRoute(String question) {
    String level = turboClient.prompt()
            .user("判断难度，只回答 EASY 或 HARD：\n" + question)
            .call().content();
    return level.contains("HARD") ? maxClient : plusClient;
}
```

## 3.3 边界与生产建议

- 边界：动态路由本身多一次模型调用，短任务可能得不偿失；
- 边界：路由判定出错会把复杂任务降级，导致效果下降；
- 生产建议：路由规则做成配置表，按流量灰度调整；
- 生产建议：为每档模型设置独立的限流与配额。

# 4. 参数与采样

## 4.1 常用参数

结论先行：温度、top-p、最大长度是影响输出最直接的三个参数。

| 参数 | 作用 | 典型取值 |
|------|------|----------|
| temperature | 随机性 | 0.0 ~ 1.0 |
| top_p | 核采样 | 0.7 ~ 1.0 |
| max_tokens | 输出上限 | 按业务设 |
| seed | 复现随机 | 固定值 |

## 4.2 参数对输出的影响

- 分类 / 抽取等确定性任务：temperature 取 0；
- 创作 / 头脑风暴：temperature 取 0.7 ~ 0.9；
- 温度与 top-p 一般不同时调，避免随机性叠加失控。

```yaml
spring:
  ai:
    dashscope:
      chat:
        options:
          model: qwen-plus
          temperature: 0.2
          max-tokens: 2048
```

## 4.3 边界与坑

- 边界：temperature=0 也不保证完全确定，后端实现与并行计算仍可能带来差异；
- 边界：`max-tokens` 设得过小会截断答案，且被截断时往往不报错；
- 生产建议：把参数纳入 Prompt 版本管理，与提示词一起回归测试。

# 5. Embedding 接入

## 5.1 配置方式

结论先行：Embedding 用于把文本转成向量，是 RAG 与语义检索的基础。

```yaml
spring:
  ai:
    dashscope:
      embedding:
        options:
          model: text-embedding-v3
```

## 5.2 编程式构建

```java
EmbeddingModel embeddingModel = DashScopeEmbeddingModel.builder()
        .apiKey(apiKey)
        .modelName("text-embedding-v3")
        .build();
```

## 5.3 维度与批量

| 维度 | 说明 |
|------|------|
| 向量维度 | v3 支持多档输出维度 |
| 批量上限 | 单次请求文本条数有上限 |
| 计费 | 按输入 Token 计费 |

- 维度越高，表达力越强，但存储与检索成本越高；
- 批量接口可显著降低往返延迟，但需处理部分失败。

### 5.3.1 维度选择权衡

| 维度档 | 表达力 | 存储 | 适用 |
|:------:|:------:|:----:|------|
| 低 | 弱 | 省 | 粗粒度召回 |
| 中 | 中 | 中 | 通用 RAG |
| 高 | 强 | 贵 | 高精度检索 |

### 5.3.2 批量与并发

- 单次批量有上限，超限需分片提交；
- 并发提交要限流，避免触发平台 QPS 限制；
- 批量返回需按输入顺序对齐，防止错位。

## 5.4 边界与生产建议

- 边界：更换 Embedding 模型必须**全量重建**向量库，否则新旧向量不可比；
- 边界：超长文本需先切分再向量化，避免截断丢信息；
- 生产建议：向量维度与模型名写入元数据，便于迁移与回溯；
- 生产建议：对 Embedding 做本地缓存，重复文本不重复计费。

# 6. 多模态（VL）

## 6.1 图文理解

结论先行：qwen-vl 支持图文混合输入，用于看图问答、票据识别、图表解读。

```java
// 通义千问 VL（视觉理解）
UserMessage message = UserMessage.builder()
        .text("这张图说明了什么？")
        .media(new Media(MimeTypeUtils.IMAGE_PNG, imageResource))
        .build();

String answer = chatClient.prompt().messages(message).call().content();
```

机制：图像以 `Media`（媒体对象）形式与文本一起封装进 `UserMessage`，由模型统一编码。

### 6.1.1 多图输入

```java
UserMessage message = UserMessage.builder()
        .text("对比这两张图，指出差异。")
        .media(new Media(MimeTypeUtils.IMAGE_PNG, image1))
        .media(new Media(MimeTypeUtils.IMAGE_PNG, image2))
        .build();
```

- 多图会显著增加 Token 与延迟，需评估必要性；
- 图片顺序会影响语义，提示词中应说明顺序含义。

## 6.2 图像生成（通义万相）

| 能力 | 说明 |
|------|------|
| 文生图 | 文本描述生成图像 |
| 图像编辑 | 基于原图改图 |
| 风格控制 | 指定画风 |

### 6.2.1 图像编辑

```java
// 伪代码：基于原图做局部重绘
ImageResponse resp = wanxClient.edit(
        EditRequest.builder()
                .image(originUrl)
                .mask(maskUrl)
                .prompt("把背景换成海边")
                .build());
```

- 编辑依赖原图与掩码，需先上传对象存储；
- 生成结果需做尺寸与格式校验后再落库。

```java
// 伪代码：调用通义万相生成图像
ImageResponse resp = wanxClient.generate(
        ImageRequest.builder()
                .prompt("一只在雪地里的橘猫，写实风格")
                .size("1024x1024")
                .build());
```

## 6.3 边界与生产建议

- 边界：图像体积大，直传会增加带宽与 Token 成本，应压缩或先传对象存储再给 URL；
- 边界：VL 对细小文字与复杂图表仍可能识别错误，关键场景需人工复核；
- 生产建议：图片走对象存储 + 预签名 URL，避免把二进制塞进请求体；
- 生产建议：对生成内容做安全审核，拦截违规图像。

# 7. 百炼平台

## 7.1 平台能力

结论先行：百炼是阿里云的大模型服务平台，提供多模型聚合与托管能力。

```
百炼 = 阿里云的大模型服务平台
  ├── 多模型聚合（通义 + 第三方）
  ├── 应用编排
  ├── 知识库
  └── 统一 API
```

## 7.2 接入方式

```yaml
spring:
  ai:
    dashscope:
      api-key: ${DASHSCOPE_API_KEY}
      base-url: https://dashscope.aliyuncs.com
```

机制：通过 `base-url` 指向百炼网关，同一套 API Key 即可访问平台上的多个模型。

## 7.3 边界与生产建议

- 边界：托管服务意味着数据在云端，需评估数据出境与合规；
- 边界：平台侧限流与配额由云厂商控制，突增流量可能被限；
- 生产建议：应用侧仍要做本地限流与熔断，避免被平台限流后整体雪崩；
- 生产建议：记录每次调用的模型与版本，便于问题追溯。

## 7.4 与应用编排集成

机制：百炼的应用编排把提示词、知识库、工具串成应用，对外暴露统一 API。

```
应用编排 = 提示词 + 知识库 + 工具 + 模型
   └── 对外暴露统一 API（应用 ID）
```

- 优点：非研发也能调整编排，迭代快；
- 缺点：编排逻辑在云端，版本与调试可控性弱于代码；
- 生产建议：核心链路仍用代码编排，边缘场景用托管编排。

# 8. OpenAI 兼容接口

## 8.1 兼容模式

结论先行：通义提供 OpenAI 兼容接口，可用 `OpenAiChatModel` 直接调用。

```java
// 通义也提供 OpenAI 兼容接口
ChatModel model = OpenAiChatModel.builder()
        .baseUrl("https://dashscope.aliyuncs.com/compatible-mode")
        .apiKey(System.getenv("DASHSCOPE_API_KEY"))
        .modelName("qwen-plus")
        .build();
```

## 8.2 迁移与切换

| 维度 | DashScope 原生 | OpenAI 兼容 |
|------|----------------|-------------|
| API 风格 | 阿里自有 | OpenAI 风格 |
| 切换成本 | 高 | 低 |
| 特性覆盖 | 完整 | 部分 |

> 兼容接口便于跨框架/跨厂商切换。

机制：兼容层把请求翻译成 OpenAI 协议，使已有 OpenAI 代码几乎零改动即可切到通义。

## 8.3 边界与坑

- 边界：兼容层通常不覆盖厂商独有参数（如部分多模态字段），复杂能力仍需原生 SDK；
- 边界：兼容层的错误码与限流语义可能与原生不同；
- 生产建议：若长期只用通义，优先原生 SDK；若要跨厂商，用兼容接口抽象。

# 9. 高可用与容错

## 9.1 重试与超时

结论先行：模型调用是远程依赖，必须显式设置超时与重试，否则线程会被拖死。

```java
RetryTemplate retry = RetryTemplate.builder()
        .maxAttempts(3)
        .fixedBackoff(500)
        .retryOn(DashScopeException.class)
        .build();

String answer = retry.execute(ctx ->
        chatClient.prompt().user(q).call().content());
```

## 9.2 降级与备用模型

| 场景 | 降级策略 |
|------|----------|
| max 不可用 | 切 plus |
| 全部不可用 | 返回缓存答案 |
| 超时 | 返回兜底话术 |

- 降级链要有顺序，且每次降级都应打点记录；
- 兜底话术不能伪装成真实答案，避免误导用户。

## 9.3 限流与配额

- 在应用侧做令牌桶限流，保护下游模型；
- 按租户 / 用户维度设置配额，防止单点刷爆；
- 监控配额使用率，提前扩容。

# 10. 生产注意事项

## 10.1 风险清单

| 风险 | 建议 |
|------|------|
| 密钥硬编码 | 环境变量/密钥管理 |
| 单一模型 | 分层 + 备用 |
| 配额限制 | 限流 + 监控 |
| 数据合规 | 评估出境 |
| 版本变化 | 锁定版本 |

## 10.2 配置分层

- 公共配置（base-url）放默认 profile；
- 环境差异（密钥、模型档位）放环境 profile；
- 敏感项走密钥管理服务，不落配置文件。

## 10.3 成本与合规

- 按模型档位与租户统计 Token 与费用；
- 对长文本与多模态做成本预估，避免超预算；
- 涉及个人数据与出境的场景，先做合规评估再上线。

## 10.4 上线检查清单

| 检查项 | 通过标准 |
|--------|----------|
| 密钥 | 全部来自环境变量 |
| 模型锁定 | 配置含明确模型名 |
| 超时重试 | 已显式配置 |
| 降级链 | 已定义并测试 |
| 限流 | 应用侧已生效 |
| 监控 | Token/成本/延迟可见 |
| 合规 | 已完成评估 |

- 边界：清单是底线，不是上限；复杂场景需补充专项检查；
- 生产建议：把清单固化为发布门禁，未通过不允许上线。

# 11. 小结

## 11.1 核心要点

- Spring AI Alibaba 深度适配通义千问与百炼。
- 模型分层：turbo（快）、plus（平衡）、max（强）。
- 支持 Embedding 与多模态（VL）。
- 也提供 OpenAI 兼容接口。

## 11.2 落地清单

- 密钥外置，配置分层与限流。
- 锁定模型版本，准备降级链与配额监控。
