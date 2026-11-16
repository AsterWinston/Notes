HTML 的容错既是福气也是坑——它从不报错,于是错误长年累月地藏在页面里。这篇把"最佳实践"落成可执行的清单:用校验器抓错、避开最常见的误用、守住结构与可访问性的底线,再补上性能、SEO 与安全(rel=noopener、SRI、CSP)三块短板,让 HTML 真正写得"对"而不只是"能跑"。

# 1. 用校验器抓错

## 1.1 在线校验

★ **关键认知**：因为浏览器容错,**"能打开" ≠ "写对了"**。标准做法是过一遍 **W3C 校验器**(validator.w3.org)——输入 URL、上传文件或粘贴代码,它会报告所有**结构性错误**与警告。这是 HTML 少有的"客观对错"来源。工程上可把校验搬进 CI:`html-validate`(npm)能以退出码阻断合并,比"上线前手动贴一次"可靠。

## 1.2 常见校验错误解读

```text
Error: End tag div seen, but there were open elements.
  → 有 div 没闭合(或嵌套错乱)

Warning: Section lacks heading.
  → section 应该有个标题(h1~h6)

Error: Duplicate ID.
  → 同一个 id 出现了两次
```

★ **关键认知**：校验器的报错主要三类——**未闭合/错嵌套、语义缺失(section 无标题)、id 重复**。⚠ 别被满屏报错吓到:先修"未闭合"这类**结构性**错误,它们是牵一发动全身的根源;语义类警告可以按优先级逐步消化。

# 2. 最常见的误用清单

## 2.1 "看起来对,其实错"

| 误用 | 为什么错 | 正解 |
|------|---------|------|
| 用 `<table>` 做布局 | 表格语义是"数据",不是"排版" | CSS(Flex/Grid)布局 |
| 用 `<br>`、空 `<p>` 撑间距 | 把视觉塞进结构 | CSS margin/padding |
| 用 `<b>`/`<i>` 表示重要 | 无语义 | `<strong>`/`<em>` |
| `<div onclick>` 当按钮 | 不可键盘、无可访问 | `<button>` |
| 图片不写 `alt` | 无障碍缺失 | 写 alt 或 `alt=""` |
| 多个 `h1` | 文档层级混乱 | 每页一个 h1 |
| 拖框架做静态页 | 过度工程 | 静态页就用纯 HTML |

★ **关键认知**：误用的根源多是**"用结构标签做样式的事"**或**"用无语义标签做语义的事"**。⚠ 一个万用自检:**"我是不是在用 HTML 标签去达到一个 CSS/交互本应负责的效果?"**——是,大概率写错了。目视生僻,校验器的语义警告能替你抓出来。

## 2.2 嵌套合法性

```html
<!-- ✗ 非法:块级元素里塞进不合法位置 -->
<!-- <p><div>块级不能随便塞进 p</div></p> -->

<!-- ✓ 合法:块级元素各自独立,不互相嵌套 -->
<p>一段文字</p>
<div>另一个块</div>
```

★ **关键认知**：HTML 有**内容模型(content model)**——哪些元素能包含哪些,是规范定死的(如 `<ul>` 只能直接含 `<li>`,`<p>` 不能含块级)。⚠ 浏览器会容错"修复"非法嵌套,但修复结果是**不可预测**的,最终 DOM 可能和直觉相去甚远——这也是"能跑≠对"的又一体现。

# 3. 结构性最佳实践

## 3.1 骨架规范

```html
<!DOCTYPE html>
<html lang="zh-CN">
  <head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>具体页面标题</title>
    <meta name="description" content="本页的一句话简介">
    <link rel="canonical" href="https://example.com/page">
  </head>
  <body>
    <header>…</header>
    <main><h1>…</h1></main>
    <footer>…</footer>
  </body>
</html>
```

★ **关键认知**：一个健康的 HTML 文件应有——**正确的 DOCTYPE、声明 lang、charset + viewport 齐全、唯一 h1、语义化 header/main/footer**。⚠ 这套骨架接近"模板"了,新项目开始时直接拷一份,比每次从空文件想起更省心;`lang` 不只给翻译用,读屏靠它选发音规则、字体渲染也参考它。

## 3.2 命名与格式约定

```html
<!-- 类名:小写 + 连字符(BEM 等约定见 CSS 第 19 篇) -->
<div class="user-card">
  <div class="user-card__title">…</div>
</div>

<!-- 缩进统一、属性用小写、引号统一(双引号) -->
<a href="#top" class="link">回到顶部</a>
```

★ **关键认知**：HTML 命名虽无强制,但团队约定能大幅提升可读性——**类名小写 + 连字符**、**缩进统一**、**属性小写**、**引号风格统一**。⚠ 这些靠工具(Lint / Prettier)自动保证最省心,别靠人肉记忆。

# 4. 可访问性底线

★ **关键认知**：HTML 可访问性的**最低底线**——**每张图有 alt、每个输入有 label、每个交互元素是真正的按钮/链接、每页有唯一 h1、结构用语义标签、iframe 有 title**。⚠ 满足这六条,你的页面就超过了大多数;更深的部分(ARIA、焦点管理)见第 10 篇。

# 5. 性能与 SEO 要点

## 5.1 head 里顺手能做的

| 目的 | 写法 |
|------|------|
| 首屏关键图提前加载 | `<link rel="preload" as="image" href="hero.jpg">` |
| 提前连第三方域名 | `<link rel="preconnect" href="https://cdn.example.com">` |
| 规范搜索权重入口 | `<link rel="canonical" href="...">` |
| 社交分享卡片 | `<meta property="og:title/og:image" content="...">` |
| 移动端适配 | `<meta name="viewport" content="width=device-width, initial-scale=1">` |

★ **关键认知**：性能类 `<link>` 的价值在于**把"浏览器将来自己发现资源"变成"现在就告诉它"**——`preload` 是给首屏关键资源的插队指令,`preconnect` 把 DNS+TLS 握手提前做掉。⚠ `preload` 滥用反噬(预载了却没用到会被控制台警告并浪费带宽),它只该给"首屏必需且无法被爬发现"的资源(如 hero 图、关键字体);SEO 大头仍在 title/description/结构化数据,详见第 3 篇。

## 5.2 资源三件套与脚本加载位

```html
<img src="photo.jpg" alt="场景描述" width="1200" height="800" loading="lazy">
<script src="app.js" defer></script>   <!-- 下载不阻塞解析,DOM 就绪后执行 -->
```

★ **关键认知**：图片写死 `width/height`(或 CSS aspect-ratio)让浏览器预留盒子,避免资源到达时**布局位移(CLS)**;`loading="lazy"` 只给首屏以下的图——⚠ 首屏 LCP 大图加 lazy 反而变慢。脚本放 `<body>` 末尾或用 `defer`(保留顺序、DOM 完成后执行);`async` 无序,只给统计脚本这类独立小包;内联大段 JS 既阻塞又缓存不友好。

# 6. 安全要点(HTML 层的三件事)

## 6.1 rel="noopener" 与外站链接

```html
<a href="https://other.example" target="_blank" rel="noopener noreferrer">
  外部站点   <!-- 切断新窗口通过 window.opener 操控原页面的能力 -->
</a>
```

★ **关键认知**：`target="_blank"` 的新页面能通过 `window.opener.location` **改写你的原页面**(反向钓鱼,即"tabnabbing")。⚠ 现代浏览器已对跨源 `target=_blank` **默认隐含 noopener**,但历史浏览器/同源场景仍有洞,而且"依赖默认值"本身就是不安全写法——外站链接**显式写 `rel="noopener"`** 是零成本保险。

## 6.2 SRI:给外链脚本上防伪校验

```html
<link rel="stylesheet" href="https://cdn.example.com/reset.css"
      integrity="sha384-BKc+C2fE7C7jK8G7..." crossorigin="anonymous">
<script src="https://cdn.example.com/lib.js"
        integrity="sha512-9NfJ..." crossorigin="anonymous"></script>
```

★ **关键认知**：**SRI(Subresource Integrity)** 是把资源的**哈希摘要写进 `integrity` 属性**,浏览器下载后比对,不一致就拒绝执行——防的是 CDN 被投毒/中间人篡改第三方脚本。⚠ 从 CDN 生成工具复制摘要时注意算法至少 SHA-384;同源的本地资源不需要 SRI(攻击者若能改你的服务器文件,哈希同样能改);CDN 静默升级版本会让哈希不匹配**直接白屏**,要锁版本再上 SRI。

## 6.3 CSP 基础(声明式安全边界)

```html
<!-- meta 或 HTTP 响应头都可;响应头更常见且更强 -->
<meta http-equiv="Content-Security-Policy"
      content="default-src 'self';
               script-src 'self' https://cdn.example.com;
               img-src 'self' data:;
               object-src 'none';
               base-uri 'self';
               frame-ancestors 'none'">
```

★ **关键认知**：**CSP(内容安全策略)** 用一张"白名单"告诉浏览器**只从哪些来源加载哪类资源**,是对 XSS 的纵深防御——内联 `<script>` 默认被禁(应改用 nonce/hash 放行,而非 `unsafe-inline`);`frame-ancestors 'none'` 防点击劫持,比 X-Frame-Options 更细。⚠ 落地节奏:先用 `Content-Security-Policy-Report-Only` + `report-uri` 收集违规报告,确认不误伤再切强 enforcement;`script-src` 放行某个失控的第三方域,等于把安全策略交给它。

# 7. 最佳实践总清单

| 维度 | 底线动作 |
|------|---------|
| 结构 | 校验器无 error;DOCTYPE/lang/charset/viewport 齐全;嵌套合法 |
| 语义 | 先选语义标签再退 div/span;唯一 h1;section 带标题 |
| 可访问性 | 图 alt、输入 label、交互用 button/a、iframe title |
| 性能 | 图片带宽高与合适 lazy;script defer/置底;关键资源 preload |
| SEO | title 精准、description 有内容、canonical/OG 就位 |
| 安全 | 外链 rel=noopener、CDN 资源上 SRI、部署 CSP(先 Report-Only) |

★ **关键认知**：这张表就是"HTML 写得对"的验收单——每一行都能在源码里逐项打钩。⚠ 真正让清单生效的方式是**工具化**:校验器进 CI、Prettier 管格式、人只做审查不做检查。

# 8. 本篇小结与下一步

★ **关键认知**：最佳实践四条腿——**校验器抓结构错、避开常见误用、守住可访问底线、补上性能/SEO/安全三件套**(preload/宽高/defer、canonical/OG、noopener/SRI/CSP)。核心心法:别用结构标签做样式的事,别用无语义标签做语义的事,"能跑≠对",且尽量让工具替你守规矩。

下一步第 15 篇:面试要点与总结——HTML 高频面试题 + 全系列知识地图,收官。
