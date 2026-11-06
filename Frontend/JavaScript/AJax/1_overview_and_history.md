AJAX 这个词今天听上去有点老派，但它描述的事情一点不过时：**网页在不整页刷新的前提下，用 JavaScript 向服务器要数据，再把结果填回页面**。本篇从概念、同步与异步的区别讲起，梳理它从 XMLHttpRequest 到 fetch 的历史，并说明它与单页应用(SPA)的关系。

# 1. AJAX 是什么

## 1.1 一个最小的请求

先看结果，再拆概念。下面这段代码在页面加载后向服务器要一段文本，并把它塞进 `<div>`，整个过程页面不会跳转：

```html
<button id="load">加载问候语</button>
<div id="output"></div>

<script>
  document.getElementById("load").addEventListener("click", async () => {
    const response = await fetch("/api/greeting"); // 发一个 HTTP 请求
    const text = await response.text();            // 读出响应正文
    document.getElementById("output").textContent = text; // 局部更新 DOM
  });
</script>
```

★ **关键认知**：AJAX 不是一门语言，也不是某个具体 API，而是**一类交互模式**——用脚本发请求、拿数据、局部更新。实现它的工具可以是 `XMLHttpRequest`、`fetch`，也可以是 Axios 等封装库；只要满足"页面不整体刷新"这个特征，就属于 AJAX 的范畴。⚠ 不要把 AJAX 和某个库画等号，面试里说"AJAX 就是 jQuery 的 `$.ajax`"是典型误解。

## 1.2 三个关键词

| 关键词 | 含义 | 容易混淆的点 |
|--------|------|--------------|
| Asynchronous | 请求发出后不阻塞主线程，结果通过回调/Promise 交付 | 异步不等于"一定更快"，它只是不阻塞 |
| JavaScript | 发起请求和更新页面的执行者 | 请求本身由浏览器网络层完成 |
| XML | 早期常用的数据格式 | 今天绝大多数接口返回 JSON，名字只是历史遗留 |

★ **关键认知**：AJAX 的名字里有 XML，但**数据格式与 AJAX 无关**。服务器返回 JSON、纯文本、HTML 片段、二进制流都可以，浏览器只负责把字节交给你，怎么解析由 `responseType` 或 `Response` 的方法决定。

# 2. 同步与异步交互

## 2.1 同步请求：等结果，页面卡住

"同步"在这里指**发起请求后，JavaScript 执行流停下来等响应**。在浏览器里，同步 XHR 会冻结主线程，用户点击、滚动、动画全部停摆：

```javascript
const xhr = new XMLHttpRequest();
xhr.open("GET", "/api/data", false); // 第三个参数 false 表示同步
xhr.send();                          // 这一行会阻塞，直到响应到达
console.log(xhr.status);
```

⚠ **关键认知**：现代浏览器已经不允许在页面主线程对跨域请求使用同步 XHR，控制台会直接报错。即便同源，同步请求也会造成"页面假死"，**生产代码里不应再出现 `open(..., false)`**。

## 2.2 异步请求：先返回，结果稍后到

异步请求发出后，JavaScript 立刻继续执行后面的代码，等网络层拿到响应再通过事件或 Promise 通知你：

```javascript
const xhr = new XMLHttpRequest();
xhr.open("GET", "/api/data", true); // true 表示异步
xhr.onload = () => {
  console.log("稍后才执行", xhr.status); // 响应到达时才运行
};
xhr.send();
console.log("先执行这一行"); // 不会等待网络
```

## 2.3 两种模式的对比

| 维度 | 同步 | 异步 |
|------|------|------|
| 主线程 | 被阻塞，页面假死 | 保持响应 |
| 代码顺序 | 请求后立刻能拿到结果 | 结果在回调/Promise 中出现 |
| 错误处理 | 用 `try/catch` 包住 `send` | 监听 `error` 事件或 `catch` |
| 现代可用性 | 基本被禁用 | 标准做法 |

★ **关键认知**：异步的本质是**把"等待"从主线程挪走**。浏览器网络层在后台收数据，收完再把任务排进事件循环。所以你看到的现象是"代码先往下走，回调后执行"，而不是"网络比代码慢"这么简单。

# 3. AJAX 的历史

## 3.1 前 AJAX 时代：整页刷新

早期 Web 的交互模型非常直接：

```text
用户点击链接/提交表单
        │
        ▼
浏览器发起整页请求
        │
        ▼
服务器渲染完整 HTML
        │
        ▼
浏览器丢弃旧页面，重新渲染
```

这种模式下，哪怕只想校验一个用户名，也要重新加载导航、样式和整张页面，体验割裂、流量浪费。

## 3.2 XMLHttpRequest 的出现

1999 年前后，微软在 IE 中引入 `XMLHTTP` ActiveX 对象，后来标准化为 `XMLHttpRequest`。它让脚本可以在后台发请求，页面不必刷新。2005 年，Jesse James Garrett 在一篇文章里把这种技术组合命名为 **AJAX**，随后 Gmail、Google Maps 等产品让它广为人知。

## 3.3 从回调到 Promise

XHR 的事件模型是回调式的，多个请求串联时容易写出层层嵌套的"回调地狱"。ES6 带来 Promise，`fetch` 则把请求抽象成 Promise，让链式写法成为主流：

```javascript
// 回调式：请求成功后再发第二个请求
xhr.onload = () => {
  const id = JSON.parse(xhr.responseText).id;
  const next = new XMLHttpRequest();
  next.onload = () => console.log(next.responseText);
  next.open("GET", `/api/detail/${id}`);
  next.send();
};
```

```javascript
// Promise 式：扁平、可读
fetch("/api/list")
  .then((res) => res.json())
  .then((list) => fetch(`/api/detail/${list[0].id}`))
  .then((res) => res.json())
  .then((detail) => console.log(detail));
```

★ **关键认知**：技术演进的主线不是"API 换名字"，而是**控制流表达能力的提升**——从事件回调，到 Promise 链，再到 `async/await` 的同步写法。底层仍然是同一次 HTTP 请求，变的是你组织异步代码的方式。

## 3.4 今天的标准栈

| 阶段 | 代表工具 | 特点 |
|------|----------|------|
| 早期 | `XMLHttpRequest` | 事件驱动，兼容性极好 |
| Promise 时代 | `fetch` | 浏览器原生、基于 Promise |
| 工程化时代 | Axios、ky 等 | 拦截器、自动 JSON、取消、重试 |
| 框架时代 | React Query、SWR | 缓存、去重、后台刷新 |

# 4. AJAX 与 SPA 的关系

## 4.1 SPA 的核心思路

**单页应用(Single Page Application)** 指整个站点只有一个 HTML 文档，页面切换由前端路由完成，数据通过 AJAX 拉取，视图由框架重新渲染：

```text
┌─────────────────────────────────────────┐
│  index.html（只加载一次）                │
│  ┌───────────┐   ┌───────────────────┐  │
│  │ 前端路由  │──▶│ 渲染对应视图组件  │  │
│  └───────────┘   └─────────┬─────────┘  │
│                            │ fetch      │
│                            ▼            │
│                    后端 JSON 接口        │
└─────────────────────────────────────────┘
```

## 4.2 路由接管了"页面跳转"

传统多页应用里，每次点击链接都是一次文档请求；SPA 里，点击链接被 `history.pushState` 拦截，只更新 URL 和视图，数据由 AJAX 补上：

```javascript
document.querySelector("a.internal").addEventListener("click", (event) => {
  event.preventDefault();                 // 阻止浏览器整页跳转
  history.pushState({}, "", "/users");    // 只改地址栏
  loadUsers();                            // 用 AJAX 换内容
});
```

★ **关键认知**：SPA 与 AJAX 是**互相成就但不等价**的关系。SPA 一定大量使用 AJAX；但用了 AJAX 的页面不一定是 SPA——传统页面里做一个"用户名异步校验"也属于 AJAX。⚠ 判断标准是"页面切换是否由前端路由接管"，而不是"有没有发异步请求"。

## 4.3 SPA 带来的新问题

| 问题 | 说明 |
|------|------|
| 首屏变慢 | 需要先下载 JS，再发请求拿数据 |
| SEO 难度 | 内容不在初始 HTML 里，需要 SSR/预渲染 |
| 状态管理 | 多页共享数据、缓存、失效策略变复杂 |
| 错误边界 | 一个接口失败可能影响整块视图 |

# 5. 最小示例：同一件事的两种写法

## 5.1 XMLHttpRequest 版

```javascript
function loadWithXhr(url) {
  const xhr = new XMLHttpRequest();
  xhr.open("GET", url, true);          // 异步 GET
  xhr.responseType = "json";           // 让浏览器直接解析 JSON
  xhr.onload = () => {
    if (xhr.status >= 200 && xhr.status < 300) {
      console.log("成功", xhr.response); // 已解析好的对象
    } else {
      console.error("HTTP 错误", xhr.status);
    }
  };
  xhr.onerror = () => console.error("网络错误"); // 断网、DNS 失败等
  xhr.send();
}
```

## 5.2 fetch 版

```javascript
async function loadWithFetch(url) {
  const response = await fetch(url);       // 只在网络层失败时 reject
  if (!response.ok) {                      // 4xx/5xx 不会自动抛错
    throw new Error(`HTTP ${response.status}`);
  }
  return response.json();                  // 返回 Promise，需再 await
}
```

## 5.3 逐段说明

1. `open` 只负责"准备请求"，还没有真正发出；`send` 才触发网络。
2. `responseType = "json"` 省去手动 `JSON.parse`，但只在 XHR 里有这个属性。
3. `fetch` 的 Promise 在**收到响应头**时就 resolve，不等正文下载完；正文要用 `json()`、`text()` 再读一次。
4. `response.ok` 是 `status` 在 200-299 的简写，**必须自己检查**，这是 fetch 最容易踩的坑。

★ **关键认知**：XHR 把"请求状态"和"HTTP 状态"分成 `readyState` 与 `status` 两套；fetch 则把"网络成功"和"业务成功"拆成两层 Promise。⚠ 新手常把 `fetch` 的 resolve 当成"请求成功"，结果 404 也进了 `then`，这是必须建立的条件反射。

# 6. 常见误区

| 误区 | 正确理解 |
|------|----------|
| AJAX 必须用 XML | 数据格式任意，JSON 是主流 |
| AJAX 一定比整页刷新快 | 省的是传输和渲染，接口本身可能更慢 |
| fetch 失败会抛错 | 只有网络错误才 reject，4xx/5xx 需查 `ok` |
| 异步就是多线程 | 浏览器用事件循环调度，JS 仍是单线程 |
| SPA 等于 AJAX | SPA 是应用架构，AJAX 是数据交互手段 |

# 7. 练习

1. 用 `fetch` 写一个按钮：点击后请求 `/api/time`，把返回的纯文本显示在页面上，并处理 404 与断网两种情况。
2. 把第 5.1 节的 XHR 示例改写成 `async/await` 版本，要求用 `try/catch` 捕获网络错误。
3. 画一张时序图，标出传统整页请求与 AJAX 请求在"用户点击 → 页面更新"之间的差异。

# 8. 小结

| 概念 | 要点 |
|------|------|
| AJAX | 脚本发请求、局部更新页面，不是具体 API |
| 同步/异步 | 同步阻塞主线程，异步通过事件/Promise 交付结果 |
| 历史 | XHR → Promise/fetch → 封装库与数据请求框架 |
| SPA | 前端路由接管页面切换，AJAX 负责取数 |
| 核心陷阱 | fetch 不因 HTTP 错误而 reject，必须检查 `ok` |

下一篇进入 `XMLHttpRequest`，把它的生命周期、`readyState`、事件模型和常见坑讲透，作为理解所有请求封装的地基。
