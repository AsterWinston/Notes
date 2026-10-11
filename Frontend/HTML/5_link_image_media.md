文字之外，网页的生命力还来自链接、图片与音视频。`<a>` 撑起整个互联网的跳转，`<img>` 与 `picture` 决定图片怎么加载与适配，`audio`/`video` 带来媒体播放。本篇讲清它们的属性、语义，以及"图片该用哪个标签""链接怎么安全打开"这些高频实操点。

# 1. 链接 a

## 1.1 最基本的跳转

```html
<!-- href 是目标地址，链接文字是可见部分 -->
<a href="https://example.com">访问示例站</a>

<!-- 页面内锚点：跳转到 id 为 section-2 的元素 -->
<a href="#section-2">跳转到第二节</a>
<h2 id="section-2">第二节</h2>

<!-- 邮箱/电话 -->
<a href="mailto:hello@example.com">发邮件</a>
<a href="tel:+8612345678900">拨打电话</a>
```

★ **关键认知**：`<a>` 的 `href` 决定跳到哪——**外部 URL、页面内锚点（`#id`）、mailto/tel 协议**都可。⚠ 链接文字要有意义（"点我"不如"查看文档"）：它既影响可访问性（屏幕阅读器朗读的是链接文字）也影响 SEO。

## 1.2 target 与 rel

```html
<!-- target="_blank" 新标签页打开；务必配 rel="noopener" 防安全漏洞 -->
<a href="https://example.com" target="_blank" rel="noopener noreferrer">新标签打开</a>

<!-- rel="nofollow"：告诉搜索引擎不要传播权重（用于不可信的外链） -->
<a href="https://example.com" rel="nofollow">不可信外链</a>
```

★ **关键认知**：`target="_blank"` 在新标签页打开链接；⚠ 一旦用了它，就**必须配 `rel="noopener"`**——否则新页面能通过 `window.opener` 反向控制你的页面（著名的 tabnabbing 攻击）。`rel="noreferrer"` 附带隐藏来源页。`nofollow` 是给搜索机器人的"别跟进"指令。

# 2. 图片 img

## 2.1 基础用法

```html
<!-- src 图片地址，alt 替代文本（图片加载失败或无障碍必备） -->
<img src="logo.png" alt="公司 Logo" width="200" height="100">
```

★ **关键认知**：`alt`（替代文本）是 `<img>` 最重要的属性——图片打不开时显示、屏幕阅读器朗读它。⚠ **永远写 `alt`**：能描述图片内容就写描述，纯装饰图写 `alt=""`（空值告诉屏幕阅读器忽略它）。漏写 alt 是无障碍的第一大高频问题。

## 2.2 图片的文件选择

| 格式 | 特点 | 适用 |
|------|------|------|
| PNG | 无损、支持透明 | Logo、图标、截图 |
| JPEG | 有损、体积小 | 照片 |
| GIF | 动画、颜色少 | 简单动图 |
| WebP | 更小、支持透明/动图 | 现代浏览器的首选 |
| SVG | 矢量、无限放大 | 图标、插画 |

★ **关键认知**：选图格式是性能的一环——照片用 JPEG/WebP、图标用 SVG/PNG。⚠ WebP 体积常比 JPEG 再省 25%~35%，已是现代默认推荐，配合 `<picture>`（见下）可优雅兼容老浏览器。

## 2.3 响应式图片 picture

```html
<!-- picture：按屏幕/格式选择加载哪张图 -->
<picture>
  <!-- 支持 WebP 的浏览器用这张 -->
  <source srcset="photo.webp" type="image/webp">
  <!-- 回退到 JPEG -->
  <img src="photo.jpg" alt="风景照片">
</picture>

<!-- srcset：按屏幕像素密度选择 -->
<img src="small.jpg"
     srcset="small.jpg 1x, large.jpg 2x"
     alt="响应式图片">
```

★ **关键认知**：`<picture>` 与 `srcset` 实现**按需加载**——`picture` 按**格式/媒体条件**换图，`srcset` 按**像素密度/宽度**换图，移动端不用下载巨大的桌面图。⚠ 浏览器按条件**只下载命中的一张**，这是响应式图片性能的本质收益。

# 3. 多媒体 audio / video

## 3.1 audio 音频

```html
<!-- controls 显示播放控件；多 source 尝试逐个加载 -->
<audio controls>
  <source src="audio.mp3" type="audio/mpeg">
  <source src="audio.ogg" type="audio/ogg">
  你的浏览器不支持音频。
</audio>
```

## 3.2 video 视频

```html
<!-- controls 播放控件；poster 播放前封面；preload 预加载策略 -->
<video controls poster="cover.jpg" width="640">
  <source src="video.mp4" type="video/mp4">
  <source src="video.webm" type="video/webm">
  你的浏览器不支持视频。
</video>
```

★ **关键认知**：`audio`/`video` 用 `<source>` 列**多个格式**让浏览器挑支持的（如 mp4 + webm 覆盖主流）；`controls` 显示原生播放条，`poster` 给视频封面。⚠ `<video>` 里的文字（"不支持视频"）是**回退内容**——只有不支持 source 时才显示。`autoplay` 属性在有声音时会被浏览器自动禁止，别指望它强制出声。

# 4. 图文搭配的语义

## 4.1 figure / figcaption

```html
<!-- figure：自包含的图文单元（图片 + 配文），可独立于正文移动 -->
<figure>
  <img src="chart.png" alt="2025 年销售柱状图">
  <figcaption>图 1：2025 年各季度销售数据</figcaption>
</figure>
```

★ **关键认知**：`figure` + `figcaption` 把**"一张图 + 它的说明文字"**绑成语义单元，用于插图、图表、代码示例等"可独立存在"的媒体。⚠ 用它的判断标准："这段图文拿出来放到别处，意思还成立吗？"成立就用 figure，纯装饰图不需要。

# 5. 本篇小结与下一步

★ **关键认知**：`<a>` 要写有意义的文字 + 新开标签配 `noopener`；`<img>` 必写 `alt`；响应式用 `picture`/`srcset`；图文用 `figure`；媒体用 `source` 列多格式。这些组合起来，链接安全和图片性能都过关。

下一步进入**阶段二 · 表单与交互**，第 6 篇：表单基础——`form`/`input`/`select`/`label` 如何组织一次"能提交的数据"。