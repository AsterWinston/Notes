页面的"快慢"很大程度在 HTML 这一层就被决定了——脚本放哪、样式怎么加载、图片何时才下载。本篇讲清浏览器渲染 HTML 的"关键渲染路径"、`defer`/`async` 两种脚本加载、`preload`/`prefetch` 两种预加载,以及图片懒加载,让首屏真正快起来。

# 1. 关键渲染路径

## 1.1 从 HTML 到首屏

```
下载 HTML → 解析 HTML → 构建 DOM
                │
          遇到 <script> ──▶ 阻塞解析,下载并执行脚本(可能改 DOM)
                │
          遇到 <link rel=stylesheet> ──▶ 下载 CSS(阻塞渲染,不阻塞 DOM 构建)
                │
          DOM + CSSOM → 渲染树 → 布局 → 绘制  → 首屏
```

★ **关键认知**：浏览器边下载边解析,**`<script>` 会阻塞 HTML 解析**(因为它可能 `document.write` 改内容),**CSS 会阻塞渲染**(拿到样式才能画)。⚠ 因此脚本放哪里、加载方式,直接决定"什么时候能画出首屏"——这就是性能优化的主战场。

## 1.2 脚本放 body 底部的由来

```html
<body>
  <p>内容优先呈现</p>
  <!-- 脚本放 body 末尾:内容先解析渲染完,再加载执行脚本 -->
  <script src="app.js"></script>
</body>
```

★ **关键认知**：把 `<script>` 放 **`</body>` 之前**(而非 head),让 DOM 先构建、内容先渲染,再下载执行脚本——这是"首屏快"的经典做法。⚠ 现代更优雅的替代是 `defer`(见下),但"脚本别堵在 head"的原则不变。

# 2. defer 与 async

## 2.1 三种脚本加载对比

```html
<!-- 普通:下载阻塞、执行阻塞(最慢) -->
<script src="a.js"></script>

<!-- defer:并行下载,DOM 解析完按顺序执行 -->
<script src="a.js" defer></script>

<!-- async:并行下载,一下载完就立即执行(乱序) -->
<script src="a.js" async></script>
```

★ **关键认知**：`defer` 与 `async` 都让**下载不阻塞解析**——区别在**执行时机**:`defer` 等 DOM 解析完**按文档顺序执行**(适合依赖 DOM、需有序的脚本);`async` 一下载完**立即执行、无序**(适合独立的第三方脚本,如统计、广告)。⚠ 依赖顺序的脚本用 `defer`,别用 `async`,否则可能"后下载的先执行"导致依赖错乱。

## 2.2 决策表

| 场景 | 用 |
|------|-----|
| 依赖 DOM、需按序执行的应用脚本 | `defer` |
| 独立的第三方脚本(统计/广告) | `async` |
| 内联脚本(无下载) | 直接放 body 底 |
| 需要最简、模块化 | `<script type="module">`(自带 defer 语义) |

★ **关键认知**：`type="module"` 的脚本**自带 defer 行为**(延迟执行 + 有序),写现代前端(打包后的 ESM)时基本不用额外写 defer。⚠ 但 module 脚本默认也 **strict mode** 与**跨域策略**(CORS)更严,加载第三方非 module 脚本时注意。

# 3. preload 与 prefetch

## 3.1 两种预加载

```html
<!-- preload:提前加载"本页马上要用"的资源(字体、首屏图) -->
<link rel="preload" href="font.woff2" as="font" type="font/woff2" crossorigin>
<link rel="preload" href="hero.jpg" as="image">

<!-- prefetch:空闲时加载"下一页可能要用"的资源 -->
<link rel="prefetch" href="/next-page.js" as="script">
```

★ **关键认知**：`preload` 是"**本页要用、现在就提前拉**"(字体、首屏大图),`prefetch` 是"**未来才用、空闲时偷跑**"(下一页的资源)。⚠ 别滥用 preload——预加载没用的资源反而抢带宽、拖慢首屏;`as` 属性必须有,它告诉浏览器资源类型以正确优先级加载。

## 3.2 典型应用

```html
<head>
  <!-- 字体通常是"发现得晚、首屏又急着用"的典型,preload 很值 -->
  <link rel="preload" href="/fonts/main.woff2" as="font" crossorigin>
  <!-- 首屏 LCP 大图提前加载 -->
  <link rel="preload" href="/hero.webp" as="image">
</head>
```

★ **关键认知**：preload 最划算的两个对象——**字体**(CSS 里才被发现,常晚于首屏需要)与 **LCP 首屏大图**(最大内容绘制慢)。⚠ 字体 preload **必加 `crossorigin`**(即使同源),否则浏览器会忽略,导致字体加载两次。

# 4. 图片懒加载

## 4.1 loading="lazy"

```html
<!-- loading="lazy":图片滚进视口才加载(首屏图不要懒加载) -->
<img src="photo.jpg" loading="lazy" alt="照片">

<!-- iframe 同样支持 -->
<iframe src="..." loading="lazy"></iframe>
```

★ **关键认知**：`loading="lazy"` 是**原生懒加载**——图片/iframe 滚到视口附近才真正下载,长页面能省下大量首屏流量。⚠ **首屏图(LCP)不要懒加载**(它本来就该立刻加载,懒加载反而延后);懒加载的是首屏之外的长尾图片。

## 4.2 为什么首屏图不能懒加载

★ **关键认知**：LCP(Largest Contentful Paint,最大内容绘制)是核心性能指标,首屏大图正是 LCP 的常见主体。给首屏图加 lazy = 主动推迟 LCP = 性能反而变差。⚠ 一句话:**视口内的图 eager(立即),视口外的图 lazy**——`loading="lazy"` 只该给"屏幕外"的图。

# 5. 性能清单

| 手法 | 针对 | 收益 |
|------|------|------|
| 脚本放 body 底 / defer | 阻塞解析 | 加速 DOM 构建 |
| CSS 尽早放 head | 样式就绪 | 减少闪烁 |
| preload 字体/首屏图 | 关键资源 | 加快首屏 |
| prefetch 下一页 | 未来资源 | 加速跳转 |
| loading="lazy" | 长尾图片 | 省首屏流量 |
| picture/srcset | 响应式图片 | 移动端省流量 |

★ **关键认知**：性能的 HTML 层心法——**阻塞的东西挪走或变流(defer),关键的东西提前(preload),不着急的东西延后(lazy)**。把握"关键渲染路径 + 资源加载时机",首屏优化就成功了大半。

# 6. 本篇小结与下一步

★ **关键认知**：脚本阻塞是性能头号敌人(defer/async 化解),preload 抢跑关键资源,prefetch 偷跑未来资源,lazy 延后长尾资源。核心:**只让"首屏真正需要的"第一时间加载**。

下一步第 13 篇:模板与框架协同——HTML 模板语法、模板引擎,以及 Vue/React 到底"接管"了 HTML 的哪部分。