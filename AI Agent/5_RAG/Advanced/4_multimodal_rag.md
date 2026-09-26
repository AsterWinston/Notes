# 1. 为什么需要多模态

本章结论先行：企业知识大量存在于**图片、图表、扫描件、视频**中，传统文本 RAG 对这些内容无能为力，多模态 RAG（Multimodal RAG，多模态检索增强生成）正是为弥补这一缺口而生。

## 1.1 企业知识的模态分布

结论：真实知识库是异构的，纯文本只是其中一部分。

### 1.1.1 模态分类

| 模态 | 典型载体 | 主要挑战 |
|------|----------|----------|
| 文本 | 网页、Word、Markdown | 切分与语义完整性 |
| 图片 | PNG/JPG、截图 | 无文本可索引 |
| 表格 | PDF 表格、Excel | 结构丢失 |
| 扫描件 | 影印 PDF | 需 OCR（光学字符识别） |
| 音频 | 会议录音 | 需 ASR（自动语音识别） |
| 视频 | 教程、直播 | 抽帧 + 转写 |

### 1.1.2 传统文本 RAG 的失效

传统文本 RAG 的失效方式是"静默"的：

- 解析器遇到图片直接跳过，图片信息彻底丢失；
- 表格被按字符流打散，"住宿费 500 元" 变成无意义的碎片；
- 扫描件解析出空文本，文档在索引中"消失"；
- 用户问"报销标准表里差旅城市怎么分档"，系统答"资料中未提及"。

失败不会报错，只会让召回率在无声中下降。

## 1.2 多模态 RAG 的定义与范围

### 1.2.1 定义

多模态 RAG 指在检索与生成环节中，能够处理至少一种非文本模态的 RAG 系统。其最小形态是"把图片转成文本再走文本 RAG"，完整形态是"图文映射到同一向量空间做跨模态检索"。

### 1.2.2 能力边界

| 能力 | 是否属于多模态 RAG |
|------|:------------------:|
| 图片 OCR 后索引 | 是（弱形态） |
| 图片向量检索 | 是（强形态） |
| 纯文本 RAG | 否 |
| 图像生成 | 否（属生成侧，非检索） |

## 1.3 术语

| 英文术语 | 中文含义 |
|----------|----------|
| OCR | 光学字符识别 |
| ASR | 自动语音识别 |
| CLIP | 对比语言-图像预训练模型 |
| Embedding | 向量嵌入 |
| Caption | 图注/图片描述 |
| Vision Model | 视觉语言模型 |

# 2. 两条技术路线

结论：所有多模态 RAG 方案最终都归结为两条路线——**转文本**与**多模态 Embedding**。前者简单可复用，后者语义完整但链路复杂。

## 2.1 路线一：转文本

### 2.1.1 机制拆解

```
图片 → OCR/多模态模型描述 → 文本 → 文本 RAG
✅ 实现简单、复用现有链路
❌ 转换有信息损失
```

核心是把非文本模态"投影"为文本，再交给既有的文本 RAG 流水线。投影方式有三种：OCR（提取文字）、Caption（生成描述）、结构化（表格转自然语言）。

### 2.1.2 代码示意

```java
public Document toTextDocument(byte[] imageBytes, String source) {
    String caption = visionModel.describe(imageBytes);
    return Document.from(caption, Metadata.from(Map.of(
            "contentType", "image",
            "source", source,
            "origin", "caption")));
}
```

### 2.1.3 边界与坑

| 坑 | 说明 | 对策 |
|----|------|------|
| 描述遗漏 | 模型只描述主物体 | 提示要求"逐项列出文字与数据" |
| OCR 错字 | 手写、低分辨率 | 保留原图链接 + 人工复核 |
| 结构丢失 | 表格行列关系丢失 | 转 Markdown 或自然语言 |
| 模态信息丢失 | 颜色、布局、箭头 | 关键图走路线二 |

## 2.2 路线二：多模态 Embedding

### 2.2.1 机制拆解

```
图片/文本 → 同一向量空间 → 跨模态检索
✅ 保留原始语义
❌ 需要多模态模型，链路复杂
```

图文分别编码，但落在同一空间，使得"用文本查图片"成为可能。代价是需要多模态编码模型、更大的存储与更复杂的链路。

### 2.2.2 代码示意

```java
Embedding textEmb = multimodalModel.embedText("报销标准");
Embedding imageEmb = multimodalModel.embedImage(imageBytes);
```

### 2.2.3 边界与坑

| 坑 | 说明 | 对策 |
|----|------|------|
| 空间未对齐 | 图文向量不可比 | 只用同一模型编码 |
| 中文效果差 | CLIP 中文偏弱 | 选 BGE-VL 等中文模型 |
| 存储翻倍 | 图片向量维度大 | 量化 + 分层存储 |
| 排序困难 | 图文分数尺度不同 | 分路排序后融合 |

## 2.3 路线对比与选型

### 2.3.1 对比表

| 维度 | 转文本 | 多模态 Embedding |
|------|:------:|:----------------:|
| 实现难度 | 低 | 高 |
| 信息保留 | 有损失 | 完整 |
| 成本 | OCR/描述成本 | 模型 + 存储成本 |
| 跨模态检索 | 不支持 | 支持 |
| 可解释性 | 高（有文本） | 低 |
| 推荐 | 起步方案 | 进阶方案 |

### 2.3.2 选型决策

```
是否只需要图中的文字？
  ├── 是 → 转文本（OCR）
  └── 否 → 是否需要"用文本搜图"？
            ├── 是 → 多模态 Embedding
            └── 否 → 转文本（Caption）
```

# 3. 路线一：转文本（推荐起步）

结论：转文本能覆盖大多数企业场景，成本最低、复用性最好，应作为默认方案。

## 3.1 图片描述生成

### 3.1.1 机制拆解

用视觉语言模型（Vision-Language Model）为图片生成详尽描述，描述质量直接决定检索效果。提示词应显式要求覆盖：可见文字、数值、图表趋势、空间关系。

### 3.1.2 代码示例（LangChain4j 风格）

```java
interface ImageCaptioner {

    @UserMessage("详细描述这张图片的内容，包括所有文字、数据、图表信息。")
    String caption(UserMessage message);
}

public Document processImage(byte[] imageBytes) {
    UserMessage msg = UserMessage.from(
            TextContent.from("详细描述这张图片的内容，包括所有文字、数据、图表信息。"),
            ImageContent.from(imageBytes, "image/png"));

    String caption = model.chat(msg).aiMessage().text();

    return Document.from(caption, Metadata.from(Map.of(
            "type", "image",
            "source", source,
            "hasImage", true)));
}
```

### 3.1.3 边界与坑

| 坑 | 说明 | 对策 |
|----|------|------|
| 描述太笼统 | 检索命不中 | 要求列举具体数值与名词 |
| 幻觉数值 | 模型编造数字 | 提示"看不清则说明不确定" |
| 长图截断 | 一张图信息过多 | 分区域裁剪后分别描述 |
| 成本高 | 每图一次调用 | 批量 + 缓存相同图片哈希 |

## 3.2 表格转文本

### 3.2.1 机制拆解

表格有两种转法：**逐行自然语言**（适合检索）与 **Markdown**（适合保结构）。检索场景推荐前者，因为自然语言更贴近用户提问的措辞。

```java
// 表格 → 逐行自然语言
"住宿费标准为 500 元，上限 800 元。"
"餐饮费标准为 100 元，上限 200 元。"
```

### 3.2.2 代码示例

```java
public List<Document> tableToDocuments(Table table, String source) {
    List<Document> docs = new ArrayList<>();
    String header = String.join("、", table.headers());

    for (Row row : table.rows()) {
        String sentence = header + "：" + row.valuesAsSentence();
        docs.add(Document.from(sentence, Metadata.from(Map.of(
                "contentType", "table",
                "source", source,
                "row", row.index()))));
    }
    return docs;
}
```

每行一个 Document 便于精确定位，同时保留表头拼接以恢复语义。

## 3.3 扫描件 OCR

### 3.3.1 机制拆解

扫描件没有文本层，必须先 OCR。可选方案：

| 方案 | 优点 | 缺点 |
|------|------|------|
| Tesseract | 免费、本地 | 中文精度一般 |
| PaddleOCR | 中文强 | 需自建服务 |
| 云 OCR | 精度高 | 成本、隐私 |

### 3.3.2 代码示例

```java
String text = ocrEngine.recognize(image);   // Tesseract / PaddleOCR / 云 OCR
```

OCR 后仍需按版面做阅读顺序重排，否则多栏文档的文本会错乱。

# 4. 路线二：多模态 Embedding

结论：当需要"用文本检索图片"或保留图片的视觉语义时，必须使用多模态 Embedding。

## 4.1 模型选型

### 4.1.1 模型表

| 模型 | 说明 |
|------|------|
| CLIP | 图文对齐的经典模型 |
| SigLIP | CLIP 改进版 |
| BGE-VL | 中文多模态 |
| Jina CLIP | 多语言 |
| Cohere Embed v3 | 支持图像 |

```
文本 "报销标准"  →  ┐
                    ├─ 同一向量空间 → 跨模态检索
图片（报销标准表）→  ┘
```

### 4.1.2 选型考虑

| 考虑项 | 说明 |
|--------|------|
| 语言 | 中文优先选 BGE-VL |
| 维度 | 维度越高存储越大 |
| 部署 | 能否本地部署（隐私） |
| 许可 | 商用许可 |
| 生态 | 是否被向量库支持 |

## 4.2 实现思路

### 4.2.1 机制拆解

图文编码后写入同一个集合，检索时用文本向量近邻搜索即可命中图片。若同时保留图注文本向量，可做双路召回再融合。

### 4.2.2 代码示例

```java
// 文本与图片映射到同一空间
Embedding textEmb = multimodalModel.embedText("报销标准");
Embedding imageEmb = multimodalModel.embedImage(imageBytes);

// 用文本查询检索图片
List<Result> results = store.search(textEmb);
```

Spring AI 风格的向量库写入：

```java
vectorStore.add(List.of(
        new Document(imageId, caption, Map.of("contentType", "image")),
        new Document(textId, "报销标准", Map.of("contentType", "text"))));
```

### 4.2.3 边界与坑

| 坑 | 说明 | 对策 |
|----|------|------|
| 模型混用 | 图文空间不一致 | 严格同一模型 |
| 图注与图并存 | 重复召回 | 去重或加权 |
| 更新困难 | 图片变更需重编码 | 记录内容哈希 |

## 4.3 跨模态检索的排序

结论：图文分数尺度不同，直接混排会失真，应分路排序后融合。

```
文本路 topN ─┐
             ├─ 归一化 → 加权融合 → Rerank → 最终
图片路 topN ─┘
```

融合权重需用评测集调优，不能凭直觉设定。

# 5. 图文混排文档处理

结论：PDF 等混排文档需要按"块"解析——文本块、图片块、表格块分别处理，再用统一元数据串联。

## 5.1 解析与切分

### 5.1.1 机制拆解

```
PDF 页面
  ├── 文本块 → 文本 Embedding
  ├── 图片块 → 图片 Embedding + 图注
  └── 表格块 → Markdown 或结构化
        │
        ▼
  统一存储（带类型元数据）
        │
        ▼
  检索时按类型过滤/展示
```

解析器需要保留版面信息（页码、坐标、阅读顺序），否则块之间的关系会丢失。

### 5.1.2 代码示意

```java
record ContentBlock(String type, String text, byte[] image, int page) {}
```

## 5.2 元数据设计

### 5.2.1 字段表

| 字段 | 说明 |
|------|------|
| contentType | text / image / table |
| page | 页码 |
| source | 文件名 |
| caption | 图注 |
| imageUrl | 原图地址 |
| section | 所属章节标题 |

### 5.2.2 代码示例

```java
Metadata.from(Map.of(
        "contentType", "image",       // text / image / table
        "page", 12,
        "source", "manual.pdf",
        "caption", caption));
```

元数据既用于检索过滤（如"只在表格中检索"），也用于生成阶段回填引用。

## 5.3 边界与坑

| 坑 | 说明 | 对策 |
|----|------|------|
| 图片与正文脱节 | 图注和段落分离 | 按版面邻近关系绑定 |
| 跨页表格 | 表头在上一页 | 检测续页并合并 |
| 图片重复 | 页眉 logo 被反复入库 | 按哈希去重 |
| 顺序错乱 | 多栏排版 | 坐标聚类重排 |

# 6. 多模态生成

结论：检索到图片后，应把图片本身传给视觉语言模型，而非仅依赖图注，以避免转换损失。

## 6.1 检索到图片后的生成

### 6.1.1 机制拆解

生成阶段有两种输入：文本上下文与图片内容。当命中的是图片时，直接把图片作为多模态消息的一部分传入，让模型看图作答。

### 6.1.2 代码示例

```java
public String answerWithImage(String question, List<Document> docs) {
    List<Content> contents = new ArrayList<>();
    contents.add(TextContent.from("依据以下资料回答问题：" + question));

    for (Document doc : docs) {
        if ("image".equals(doc.getMetadata().get("contentType"))) {
            contents.add(ImageContent.from(doc.getMetadata().get("imageUrl").toString()));
        } else {
            contents.add(TextContent.from(doc.getText()));
        }
    }

    return visionModel.chat(UserMessage.from(contents.toArray(new Content[0])))
            .aiMessage().text();
}
```

## 6.2 引用与展示

### 6.2.1 机制拆解

多模态引用需要同时给出文本来源与图片链接，前端才能渲染缩略图并支持点击查看原图。

### 6.2.2 代码示例

```java
record Citation(int index, String source, int page, String imageUrl) {}
```

## 6.3 边界与坑

| 坑 | 说明 | 对策 |
|----|------|------|
| 图片过大 | 超出模型限制 | 缩放/裁剪后再传 |
| 多图超上下文 | Token 爆炸 | 限制图片数量 |
| 图片不可访问 | 链接失效 | 用带签名的临时 URL |
| 模型不看图 | 只读图注 | 提示明确要求结合图片 |

# 7. 语音与视频

结论：音视频本质上是"时间序列模态"，处理关键是先转写再切片，并保留时间戳以便定位。

## 7.1 音频处理

### 7.1.1 机制拆解

| 类型 | 处理 |
|------|------|
| 音频 | ASR 转文字 → 文本 RAG |
| 视频 | 抽帧 + ASR + 字幕 → 多模态索引 |
| 会议记录 | 说话人分离 + 摘要 |

转写结果需按时间戳切片，每片作为独立 Document。

### 7.1.2 代码示例

```java
String transcript = transcriptionModel.call(
        new AudioTranscriptionPrompt(audioResource)).getResult().getOutput();
```

## 7.2 视频处理

### 7.2.1 机制拆解

```
视频 → 音频轨 → ASR → 带时间戳文本
     → 视频轨 → 关键帧抽取 → 图注/图片向量
     → 合并（按时间戳对齐）
```

### 7.2.2 代码示例

```java
record VideoSegment(long startMs, long endMs, String transcript,
                    String keyframeUrl) {}
```

# 8. 路线对比

结论：没有万能路线，实践中常见的是"转文本为主、多模态为辅"的混合方案。

## 8.1 综合对比表

| 维度 | 转文本 | 多模态 Embedding |
|------|:------:|:----------------:|
| 实现难度 | 低 | 高 |
| 信息保留 | 有损失 | 完整 |
| 成本 | OCR/描述成本 | 模型 + 存储成本 |
| 跨模态检索 | 不支持 | 支持 |
| 可解释性 | 高（有文本） | 低 |
| 推荐 | 起步方案 | 进阶方案 |

## 8.2 混合方案

```
文字密集的扫描件 → OCR 转文本
含图表的关键图 → 多模态 Embedding
普通照片 → Caption 转文本
```

> 实践建议：**先用"转文本"覆盖 80% 场景**，确有跨模态需求再上多模态 Embedding。

# 9. 生产注意事项

结论：多模态 RAG 的生产风险集中在"转换质量、存储成本、隐私合规"三方面。

## 9.1 风险与对策

### 9.1.1 风险表

| 风险 | 建议 |
|------|------|
| 图注质量差 | 用强多模态模型 + 提示要求详尽 |
| 图片存储成本 | 对象存储 + CDN |
| 转换信息损失 | 保留原图链接供人工查看 |
| 检索排序 | 图文结果分别排序后融合 |
| 隐私 | 图片可能含敏感信息，需审核 |
| 成本 | 多模态调用更贵，按需 |

### 9.1.2 成本控制

| 手段 | 说明 |
|------|------|
| 哈希去重 | 相同图片只编码一次 |
| 分级处理 | 关键图用强模型，其余用弱模型 |
| 缓存 | 图注与向量缓存 |
| 按需转码 | 大图先缩放到合理分辨率 |

## 9.2 存储与隐私

### 9.2.1 存储

图片本体与向量分开存储：向量入向量库，图片入对象存储，元数据用 `imageUrl` 关联。

### 9.2.2 隐私

| 风险 | 对策 |
|------|------|
| 人脸/证件 | 入库前检测并脱敏 |
| 水印/机密标识 | OCR 检测敏感标记 |
| 越权访问 | 图片 URL 需鉴权与过期 |
| 数据出境 | 本地化模型部署 |

# 10. 小结

## 10.1 核心结论

- 多模态 RAG 处理图片、表格、扫描件、音视频。
- 两条路线：转文本（简单）与多模态 Embedding（完整）。
- 表格建议转自然语言，扫描件需 OCR，音视频先转写再切片。
- 生成阶段应把图片本身传给视觉模型，而非仅依赖图注。

## 10.2 落地清单

- 建议先用"转文本"覆盖多数场景，确有跨模态需求再升级。
- 元数据标注 `contentType`，便于检索过滤与前端展示。
- 保留原图链接与来源信息，兼顾可解释性与人工复核。
- 用哈希去重、分级处理与缓存控制多模态成本；图片含敏感信息时入库前必须审核。
