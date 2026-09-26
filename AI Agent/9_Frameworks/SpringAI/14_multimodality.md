# 1. 多模态能力概览

**结论先行**：Spring AI 用统一的模型抽象把文本、图片、音频、审核等能力收拢到一套 API 之下，业务代码通过 `Media`（媒体对象）描述非文本输入，从而不必为每种模态单独写一套调用逻辑。

## 1.1 什么是多模态

多模态（Multimodality）指模型能同时处理多种信息形态：文本（text）、图像（image）、音频（audio）、视频（video）。对大模型而言，非文本输入最终会被编码成 Token 序列，与文本 Token 拼接后一同送入模型。

## 1.2 Spring AI 的模型抽象

```
┌──────────────────────────────────────────────┐
│ ChatModel        文本对话（可含图片/音频输入） │
│ EmbeddingModel   文本 → 向量                  │
│ ImageModel       文本 → 图片                  │
│ AudioModel       语音转文字 / 文字转语音       │
│ ModerationModel  内容审核                     │
└──────────────────────────────────────────────┘
```

各抽象的职责：

| 抽象 | 方向 | 典型用途 |
|------|------|----------|
| ChatModel | 多模态输入 → 文本输出 | 看图问答、语音问答 |
| EmbeddingModel | 文本/图片 → 向量 | 检索、聚类 |
| ImageModel | 文本 → 图片 | 配图、素材生成 |
| AudioModel | 语音 ↔ 文本 | 转录、播报 |
| ModerationModel | 文本 → 审核结果 | 内容护栏 |

## 1.3 能力矩阵对比

不同模型对模态的支持差异很大，接入前必须核对：

| 模态 | 输入支持 | 输出支持 | 常见约束 |
|------|----------|----------|----------|
| 文本 | 全部 | 全部 | 上下文长度 |
| 图片 | 多数多模态模型 | 仅图像模型 | 分辨率、格式、数量上限 |
| 音频 | 部分模型 | 部分模型 | 时长、采样率 |
| 视频 | 少数模型 | 极少 | 帧率、时长、体积 |

> 能力以具体模型为准，抽象统一不代表底层能力统一。

## 1.4 多模态请求的内部处理

**结论先行**：Spring AI 的抽象负责"把媒体规范化为模型可接受的格式"，真正决定效果的是模型侧的编码方式。

```
Media(Resource)
   │
   ▼
按 MimeType 选择编码策略
   ├── 图片 → Base64 内联 / 上传取 URL
   ├── 音频 → 采样率转换 / 分片
   └── 视频 → 抽帧 / 分片
   │
   ▼
拼装为模型请求的多模态 content 数组
   │
   ▼
模型编码为视觉/音频 Token，与文本 Token 拼接
```

处理链路上的每一环都可能成为失败点：编码不支持、尺寸超限、格式不符。因此工程上应尽早做前置校验。

### 1.4.1 模态与 Token 的关系

| 模态 | Token 化方式 | 影响成本的关键因素 |
|------|--------------|--------------------|
| 文本 | 分词 | 字符数、语言 |
| 图片 | 切块编码 | 分辨率、细节档位 |
| 音频 | 时间片编码 | 时长 |
| 视频 | 抽帧 + 时间 | 帧数、时长 |

分辨率越高、时长越长，Token 消耗近似线性增长，这是多模态成本失控的主要来源。

# 2. 多模态输入

**结论先行**：多模态输入的核心是把非文本内容包装成 `Media`，再挂到 `UserMessage` 上；文本与媒体可同时存在，由模型综合理解。

## 2.1 图片输入

```java
var imageResource = new ClassPathResource("/images/chart.png");

UserMessage message = UserMessage.builder()
        .text("这张图表反映了什么趋势？")
        .media(new Media(MimeTypeUtils.IMAGE_PNG, imageResource))
        .build();

String answer = chatClient.prompt()
        .messages(message)
        .call()
        .content();
```

### 2.1.1 Media 的构成

| 字段 | 含义 | 说明 |
|------|------|------|
| MimeType | 媒体类型 | 决定编码与解析方式 |
| Resource | 数据来源 | 类路径、文件、字节、URL |
| Data | 原始字节 | 可延迟加载 |

`Media` 的第二个参数是 `Resource` 抽象，因此类路径资源、`FileSystemResource`、`ByteArrayResource` 都能直接传入。

## 2.2 多图输入

```java
UserMessage message = UserMessage.builder()
        .text("对比这两张图片的差异")
        .media(
                new Media(MimeTypeUtils.IMAGE_PNG, new ClassPathResource("/a.png")),
                new Media(MimeTypeUtils.IMAGE_JPEG, new ClassPathResource("/b.jpg"))
        )
        .build();
```

多图会显著增加 Token 消耗，多数模型对单次请求的图片数量有上限，超出会报错或被静默截断。

## 2.3 URL 图片

```java
new Media(MimeTypeUtils.IMAGE_PNG, URI.create("https://example.com/chart.png"));
```

> 注意：并非所有模型都支持 URL 图片，部分需要 Base64 内联。

### 2.3.1 URL 与内联的取舍

| 方式 | 优点 | 缺点 |
|------|------|------|
| URL 引用 | 请求体小、可复用 | 依赖网络可达、模型需支持 |
| Base64 内联 | 兼容性好、无外网依赖 | 请求体膨胀、占带宽与内存 |

## 2.4 从用户上传接收

```java
@PostMapping(value = "/vision", consumes = MediaType.MULTIPART_FORM_DATA_VALUE)
public String vision(@RequestPart("file") MultipartFile file,
                     @RequestParam String question) throws IOException {
    Media media = new Media(
            MimeTypeUtils.parseMimeType(file.getContentType()),
            new ByteArrayResource(file.getBytes()));

    return chatClient.prompt()
            .user(u -> u.text(question).media(media))
            .call()
            .content();
}
```

### 2.4.1 上传安全校验

上传入口是攻击面，必须校验：

- 文件大小上限（防止内存溢出）；
- MIME 类型白名单（不要信任客户端声明，应嗅探真实类型）；
- 文件内容合法性（图片可尝试解码验证）；
- 文件名不参与路径拼接。

## 2.5 音频输入

部分模型支持直接把音频作为输入，实现"语音直接问答"，跳过 ASR 中间步骤：

```java
UserMessage message = UserMessage.builder()
        .text("这段录音里提到了哪些行动项？")
        .media(new Media(MimeTypeUtils.parseMimeType("audio/mp3"),
                new ClassPathResource("/audio/meeting.mp3")))
        .build();
```

> 是否支持以模型为准；不支持时退回"ASR 转录 + 文本对话"的两段式方案。

## 2.6 多模态消息的组装顺序

**结论先行**：媒体与文本的相对顺序会影响模型的理解，推荐"先文本指令、后媒体内容"或"先媒体、后提问"，并保持同一会话内风格一致。

```java
// 推荐：先给指令，再给媒体
UserMessage msg = UserMessage.builder()
        .text("请判断这张图是否包含违规内容，并说明理由")
        .media(new Media(MimeTypeUtils.IMAGE_PNG, res))
        .build();
```

### 2.6.1 常见组装模式

| 模式 | 适用 | 示例 |
|------|------|------|
| 文本在前 | 指令明确的判定任务 | 先规则后图片 |
| 媒体在前 | 开放式描述任务 | 先图片后"这是什么" |
| 多轮引用 | 追问同一张图 | 首轮给图，后续只给文本 |

## 2.7 Token 估算与上限

多模态输入必须在入口处估算 Token，避免超限报错或成本飙升：

```java
public record MediaBudget(int maxImages, int maxImagePx, int maxAudioSec) {}

public void validate(List<Media> medias, MediaBudget b) {
    if (medias.size() > b.maxImages()) {
        throw new IllegalArgumentException("图片数量超限");
    }
    // 其余维度（分辨率、时长）按元数据校验
}
```

### 2.7.1 边界与坑

- 图片数量、单图分辨率、单次请求总 Token 都有上限，任一项超限都会失败；
- 上限随模型不同而不同，不能硬编码单一常量；
- 客户端压缩可能降低识别质量，需在成本与准确率间权衡。

# 3. 图像生成（ImageModel）

**结论先行**：`ImageModel` 接收 `ImagePrompt`（含文本与选项），返回 `ImageResponse`，结果通常是 URL 或 Base64，需要再做持久化与合规处理。

## 3.1 基础用法

```java
ImageOptions options = ImageOptionsBuilder.builder()
        .model("dall-e-3")
        .width(1024)
        .height(1024)
        .build();

ImageResponse response = imageModel.call(
        new ImagePrompt("一只戴着眼镜写代码的橘猫，扁平插画风格", options));

String url = response.getResult().getOutput().getUrl();
```

## 3.2 参数与变体

| 参数 | 作用 | 常见约束 |
|------|------|----------|
| model | 选择生成模型 | 版本能力不同 |
| width/height | 输出尺寸 | 受模型支持列表限制 |
| quality | 画质档位 | 影响成本与延迟 |
| style | 风格倾向 | 部分模型专有 |
| n | 生成数量 | 多数限 1 |

### 3.2.1 提示词工程要点

- 明确主体、风格、构图、色调；
- 避免歧义与敏感词；
- 生成失败时先检查是否被内容策略拦截。

## 3.3 输出处理与合规

```java
Image img = response.getResult().getOutput();

if (img.getUrl() != null) {
    // 下载到自有存储，避免外链失效
    byte[] bytes = restClient.get().uri(img.getUrl())
            .retrieve().body(byte[].class);
    storage.save("generated/" + UUID.randomUUID() + ".png", bytes);
}
```

> 图像生成多用于营销素材、配图、原型设计；注意版权与内容合规。

### 3.3.1 边界与坑

- 生成结果可能包含不当内容，输出侧仍需审核；
- 外链 URL 通常有时效，必须及时转存；
- 生成内容可能涉及第三方版权风格，商用前需评估。

## 3.4 批量生成与异步

图像生成延迟通常为秒级甚至更长，不适合放在同步请求链路中：

```java
// 伪代码：异步任务 + 回调
@Async
public CompletableFuture<String> generateAsync(String prompt) {
    ImageResponse resp = imageModel.call(new ImagePrompt(prompt));
    String url = resp.getResult().getOutput().getUrl();
    return CompletableFuture.completedFuture(persist(url));
}
```

### 3.4.1 任务编排

| 阶段 | 动作 | 失败处理 |
|------|------|----------|
| 提交 | 生成任务 ID，立即返回 | 参数校验失败即拒绝 |
| 生成 | 调用 ImageModel | 重试 N 次后置失败 |
| 转存 | 下载并落库 | 保留原始 URL 兜底 |
| 通知 | 推送/轮询结果 | 超时提示 |

# 4. 语音能力

**结论先行**：语音能力分为转录（ASR，语音转文字）与合成（TTS，文字转语音）两个方向，二者与 LLM 组合即可构成语音 Agent。

## 4.1 语音转文字（Transcription）

```java
var audio = new ClassPathResource("/audio/meeting.mp3");

String text = transcriptionModel.call(
        new AudioTranscriptionPrompt(audio)).getResult().getOutput();
```

### 4.1.1 转录增强

长音频应分段处理，并附带领域词表提升识别准确率：

```java
AudioTranscriptionOptions opts = AudioTranscriptionOptions.builder()
        .language("zh")
        .prompt("以下是关于订单履约的会议")
        .build();

String text = transcriptionModel.call(
        new AudioTranscriptionPrompt(audio, opts)).getResult().getOutput();
```

## 4.2 文字转语音（Speech）

```java
byte[] audioBytes = speechModel.call(
        new SpeechPrompt("欢迎使用智能客服系统"))
        .getResult().getOutput();
```

### 4.2.1 语音参数

| 参数 | 作用 | 说明 |
|------|------|------|
| voice | 音色 | 决定声线 |
| speed | 语速 | 影响自然度 |
| format | 输出格式 | mp3/wav 等 |

## 4.3 语音 Agent 组合

```
用户语音 ─▶ ASR ─▶ LLM（含工具调用）─▶ TTS ─▶ 用户
                     │
                     └─▶ 文本记录 / 审计
```

> 语音是语音助手、会议纪要、无障碍场景的基础。结合 ASR + LLM + TTS 即可搭建语音 Agent。

### 4.3.1 端到端延迟优化

- 流式 ASR：边识别边送入 LLM；
- 流式 TTS：LLM 首句生成后立即合成；
- 打断处理：用户插话时中断当前播报。

### 4.3.2 语音场景的质量指标

| 指标 | 含义 | 目标方向 |
|------|------|----------|
| WER | 词错误率 | 越低越好 |
| 首字延迟 | 从说话到首次响应 | 越低越自然 |
| 打断响应 | 用户插话后的反应时间 | 越短越好 |
| 合成自然度 | 主观听感评分 | 越高越好 |

# 5. 多模态 RAG

**结论先行**：多模态 RAG 让图片、扫描件也能进入知识库，工程上有两条路线：多模态 Embedding 直接检索，或 OCR/图注转文本后走文本 RAG。

图片、扫描件也能进入知识库：

```
文档/图片 → 多模态 Embedding（如 CLIP）→ 向量库
查询文本 → 文本 Embedding → 检索 → 多模态模型生成
```

## 5.1 两种技术路线

| 路线 | 原理 | 优点 | 缺点 |
|------|------|------|------|
| 多模态 Embedding | 图文映射到同一向量空间 | 保留视觉信息 | 模型与库支持有限 |
| OCR + 文本 RAG | 先转文本再检索 | 工程成熟、可解释 | 丢失版式与图形信息 |

要点：

- 使用**同一多模态空间**的 Embedding 模型（文本与图片映射到同一向量空间）；
- 或先对图片做 OCR/图注生成，再走文本 RAG（工程上更常见）。

## 5.2 多模态 Embedding 路线

```java
// 伪代码：图片与文本使用同一多模态 Embedding 模型
EmbeddingResponse imgVec = multimodalEmbedding.embed(
        new ImageEmbeddingRequest(imageResource));
EmbeddingResponse txtVec = multimodalEmbedding.embed(
        new TextEmbeddingRequest("季度营收趋势"));
// 二者可直接做相似度比较
```

### 5.2.1 边界与坑

- 图文向量空间必须由同一模型产出，混用不同模型会导致相似度失真；
- 多模态 Embedding 的召回质量通常低于专用文本模型，需用评测集验证。

## 5.3 OCR + 文本 RAG 路线

```
图片 ─▶ OCR/图注 ─▶ 文本分块 ─▶ 文本 Embedding ─▶ 向量库
查询 ─▶ 文本 Embedding ─▶ 检索 ─▶ LLM 生成
```

### 5.3.1 何时选择哪条路线

| 场景 | 推荐路线 |
|------|----------|
| 纯文本扫描件 | OCR + 文本 RAG |
| 图表、示意图为主 | 多模态 Embedding 或图注 |
| 图文混排文档 | OCR 保留阅读顺序 + 图注 |
| 强合规、需可解释 | OCR + 文本 RAG |

> 详见 `5_RAG/Advanced/4_multimodal_rag.md`。

## 5.4 多模态 RAG 的分块策略

**结论先行**：图片/图注的分块要保证"一个块能独立表达一个语义单元"，否则检索命中后模型仍无法作答。

| 内容类型 | 分块方式 | 备注 |
|----------|----------|------|
| 扫描页 | 按页/段落 | 保留阅读顺序 |
| 图表 | 一图一块 + 图注 | 图注由多模态模型生成 |
| 图文混排 | 段落级 + 关联图引用 | 维护图文对应关系 |

### 5.4.1 图注生成示例

```java
String caption = chatClient.prompt()
        .user(u -> u.text("用一句中文描述这张图的关键信息")
                .media(new Media(MimeTypeUtils.IMAGE_PNG, chartRes)))
        .call()
        .content();
// caption 与图片一起入库，检索时以文本召回、以图片补全上下文
```

# 6. 内容审核（ModerationModel）

**结论先行**：审核应作为输入与输出两侧的标配护栏，输入侧拦截违规请求，输出侧拦截违规生成。

## 6.1 输入审核

```java
ModerationResponse result = moderationModel.call(
        new ModerationPrompt(userInput));

if (result.getResult().getOutput().isFlagged()) {
    throw new IllegalArgumentException("输入包含违规内容");
}
```

## 6.2 输出审核

```java
String answer = chatClient.prompt().user(q).call().content();

ModerationResponse check = moderationModel.call(new ModerationPrompt(answer));
if (check.getResult().getOutput().isFlagged()) {
    return "抱歉，回答涉及不适宜内容，已拦截。";
}
return answer;
```

### 6.2.1 审核的局限

- 审核模型本身有误判率，需配合人工申诉通道；
- 对多模态内容（图片/音频）的审核能力依平台而异；
- 审核会引入额外延迟，建议异步或抽样。

> 审核是护栏的一环，详见 `15_Security/5_guardrails.md`。

## 6.3 审核策略配置

**结论先行**：审核应做成可配置的策略，而不是散落在业务代码里的硬编码判断。

| 策略项 | 取值 | 说明 |
|--------|------|------|
| 审核时机 | 输入/输出/两者 | 建议两者 |
| 命中动作 | 拦截/降级/告警 | 按风险分级 |
| 抽样比例 | 0~100% | 高并发下降低开销 |
| 申诉通道 | 有/无 | 误判兜底 |

# 7. 生产注意事项

**结论先行**：多模态的成本、延迟、隐私风险都高于纯文本，必须从资源、成本、兼容三个维度做工程约束。

## 7.1 资源与性能

| 风险 | 建议 |
|------|------|
| 大文件内存溢出 | 限制大小、流式处理、临时文件 |
| 延迟高 | 多模态推理更慢，需异步 + 进度反馈 |

### 7.1.1 上传处理流水线

```
上传 ─▶ 大小/MIME 校验 ─▶ 落临时文件 ─▶ 异步任务 ─▶ 结果回调
```

## 7.2 成本与隐私

| 风险 | 建议 |
|------|------|
| 成本高 | 图片/音频 Token 消耗大，按需调用 |
| 隐私泄露 | 上传的图片/音频含敏感信息，需脱敏与合规审查 |
| 生成内容违规 | 输出侧也要审核 |

### 7.2.1 成本控制手段

- 图片先缩放/压缩到模型可用分辨率；
- 音频先 VAD 切分，静音段不上传；
- 对重复请求做缓存（按内容哈希）。

## 7.3 兼容与降级

| 风险 | 建议 |
|------|------|
| 格式不兼容 | 校验 MIME 类型与模型支持范围 |
| 模型不支持某模态 | 设计降级路径（如音频转文本） |

### 7.3.1 降级矩阵

| 主路径 | 降级路径 | 触发条件 |
|--------|----------|----------|
| 语音直接问答 | ASR + 文本对话 | 模型不支持音频输入 |
| 多模态 Embedding 检索 | OCR + 文本检索 | 无多模态模型 |
| URL 图片 | Base64 内联 | 模型不支持 URL |

# 8. 小结

- Spring AI 用统一的 `Media` 抽象支持图片、音频等多模态输入。
- 图像生成、语音转录/合成各有独立模型抽象，注意输出侧的合规处理。
- 多模态 RAG 与语音 Agent 是当前两大热点方向。
- 多模态的成本、延迟、隐私风险都高于纯文本，需谨慎设计。
- 审核（Moderation）应作为输入输出两侧的标配护栏。
