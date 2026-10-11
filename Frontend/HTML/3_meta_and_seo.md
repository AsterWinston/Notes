`<head>` 里的内容用户看不到，却决定了三件大事：**会不会乱码**、**移动端显示是否正常**、**搜索引擎怎么认识你**。本篇讲清 `<meta>` 的核心用法、`viewport` 这个移动端命门、Open Graph 与结构化数据这两套"让分享和搜索更懂你"的协议。

# 1. meta 的两种形态

## 1.1 charset：字符编码

```html
<!-- 必须放 head 最前（前 1024 字节内），否则浏览器可能乱码后重解析 -->
<meta charset="UTF-8">
```

★ **关键认知**：`charset="UTF-8"` 声明文档的**字符编码**，几乎涵盖全球所有文字（含中文、emoji），是现代唯一正确选择。⚠ 它必须出现在 `<head>` 的**前 1024 字节内**——浏览器碰到中文再回头解析编码会乱码；历史遗留的 GBK/GB2312 只在老旧系统里才需要考虑。

## 1.2 name + content：命名元信息

```html
<!-- name 是"什么信息"，content 是"信息内容" -->
<meta name="viewport" content="width=device-width, initial-scale=1.0">
<meta name="description" content="本页内容的简介，展示在搜索结果下方">
<meta name="author" content="张三">
<meta name="keywords" content="HTML, 前端, 教程">  <!-- 已基本被搜索引擎忽略 -->
```

★ **关键认知**：`<meta>` 的主流用法是 **name/content 键值对**——声明描述、作者、视口等。⚠ `keywords`（关键词）**早已被主流搜索引擎弃用**（早年堆关键词作弊太严重），别再花心思；`description` 仍有价值，会显示在搜索结果摘要里，写不写体验差别明显。

# 2. viewport：移动端的命门

## 2.1 没有 viewport 会怎样

```html
<!-- 不加 viewport：移动端会以"桌面版 980px 宽"渲染再缩小，文字小到看不清 -->
<!-- 加上它：视口宽度 = 设备宽度，页面按真实尺寸布局 -->
<meta name="viewport" content="width=device-width, initial-scale=1.0">
```

★ **关键认知**：`width=device-width` 让**布局视口等于设备屏幕宽度**，这是移动端页面千里之堤的第一块砖；少了它，手机浏览器默认按约 980px 的桌面宽度渲染，整页缩成一团。⚠ `initial-scale=1.0` 设置初始缩放为 1，防止某些浏览器默认缩放；两者通常成对出现，是响应式（CSS 第 10 篇）的前提。

## 2.2 viewport 的进一步配置

```html
<!-- 常用配置：禁止用户缩放、最大缩放（一般不建议禁止缩放，影响无障碍） -->
<meta name="viewport" content="width=device-width, initial-scale=1.0, maximum-scale=1.0, user-scalable=no">
```

★ **关键认知**：viewport 可配 `initial-scale`（初始缩放）、`maximum-scale`（最大缩放）、`user-scalable`（是否允许缩放）。⚠ **慎用 `user-scalable=no`**：禁止缩放会伤害低视力用户的可访问性，除游戏/图表等特殊场景外不建议。

# 3. 标题与图标

## 3.1 title 与 link 系列

```html
<head>
  <title>冠军产品官网 - 让生活更简单</title>

  <!-- favicon 站点图标：显示在标签页/收藏夹 -->
  <link rel="icon" href="/favicon.ico">

  <!-- 引入外部样式表 -->
  <link rel="stylesheet" href="/styles.css">
</head>
```

★ **关键认知**：`<title>` 是**唯一必填**的 head 元素，决定标签页标题、搜索结果标题、收藏夹名称，是 SEO 权重最高的一处。`<link>` 承担"关联外部资源"的职责——图标、样式表、预加载（第 12 篇）。⚠ title 要"具体而非空泛"（"产品名 - 功能描述"优于"首页"），每个页面都应有独特 title。

# 4. Open Graph：社交分享的卡片

## 4.1 让分享链接有富信息

```html
<!-- Open Graph 协议：被微信、Facebook、Twitter 等读取，生成分享卡片 -->
<meta property="og:title" content="文章标题">
<meta property="og:description" content="分享时的摘要">
<meta property="og:image" content="https://example.com/cover.png">
<meta property="og:url" content="https://example.com/article/1">
<meta property="og:type" content="article">
```

★ **关键认知**：Open Graph（OG）是一套 **`property` 属性的 meta 规范**，社交平台抓它来生成分享卡片（标题、封面、摘要）。没有它，分享出去要么空白要么乱抓。⚠ 注意 OG 用 `property` 而非 `name`，前缀 `og:`；`og:image` 的图要为分享服务（尺寸合适、可公开访问），这是"分享体验"的关键。

## 4.2 结构化数据 JSON-LD

```html
<!-- 结构化数据：用机器可读的方式标注"这是什么"（文章/商品/评分） -->
<script type="application/ld+json">
{
  "@context": "https://schema.org",
  "@type": "Article",
  "headline": "文章标题",
  "author": "张三"
}
</script>
```

★ **关键认知**：**结构化数据（Schema.org）** 是对"页面内容是什么"的机器可读标注，搜索引擎据此生成**富摘要**（评分星、面包屑、价格）。最常用格式是 **JSON-LD**（写在 `application/ld+json` 的 script 里，与 HTML 结构解耦）。⚠ 它与 OG 分工不同：OG 面向社交分享卡片，JSON-LD 面向搜索富摘要。

# 5. SEO 的 head 清单

★ **关键认知**：head 里的 SEO 要点一句话——**title 精准 + description 诱人 + charset/viewport 正确 + OG/结构化数据齐全**。一张清单：

| 项 | 标签 | 作用 |
|----|------|------|
| 标题 | `<title>` | 搜索结果 + 标签页标题 |
| 摘要 | `<meta name="description">` | 搜索摘要 |
| 编码 | `<meta charset="UTF-8">` | 防乱码 |
| 视口 | `<meta name="viewport">` | 移动端适配 |
| 分享卡片 | `og:*` | 社交富信息 |
| 富摘要 | JSON-LD | 搜索结构化展示 |

# 6. 本篇小结与下一步

★ **关键认知**：head 里看不见，却管着**编码、移动端、搜索与分享**四件大事。核心是 `charset`+`viewport` 保基础体验，`title`+`description` 保搜索，`OG`+`JSON-LD` 保分分享与富摘要。

下一步第 4 篇：文本与内容标签——标题、段落、列表、引用、代码，如何用语义标签把一篇文章组织得明明白白。