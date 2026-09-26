# 1. 为什么需要浏览器工具

结论先行：浏览器工具（Browser Automation，浏览器自动化）是 Agent 访问"必须经过真实浏览器渲染与交互才能获得的信息或操作"的手段。凡是页面依赖 JavaScript（脚本语言）动态生成、依赖登录态、需要点击提交的任务，静态 HTTP 抓取都无法胜任。

很多信息与操作只能通过网页完成：

```
- 网页内容抓取（JS 渲染的动态页面）
- 表单填写与提交
- 网页操作（点击、登录、下单）
- 截图与可视化验证
```

## 1.1 浏览器能补齐的能力

### 1.1.1 动态渲染内容

现代前端框架（React / Vue / SPA，单页应用）在服务端返回的 HTML 中往往只有一个空壳，真正的正文由浏览器执行 JavaScript 后注入 DOM（文档对象模型）。静态 HTTP 客户端拿到的只是骨架，因此必须由浏览器执行脚本、等待网络空闲后，才能读到完整内容。

```
静态抓取：HTTP GET -> 空壳 HTML -> 无正文
浏览器：  导航 -> 执行 JS -> 请求接口 -> 渲染 DOM -> 读取正文
```

### 1.1.2 交互型任务

除"读"之外，浏览器工具还能"写"：填写表单、点击按钮、上传文件、登录、翻页。这类任务本质是维护一个浏览器会话（Session）并驱动状态机，静态抓取完全无法完成。

### 1.1.3 可视化验证

截图（Screenshot）让 Agent 能对"页面是否渲染成功"做可视化确认，也便于人工审查与审计留痕。对于视觉类任务，截图是唯一的可核对证据。

## 1.2 何时不该用浏览器工具

### 1.2.1 静态页面的更优解

若目标页面是服务端渲染（SSR）或纯静态 HTML，直接用 HTTP 客户端 + Jsoup 解析更快、更省资源、更安全。启动一个 Chromium 实例的内存成本（500MB 以上）远高于一次 HTTP 请求。

### 1.2.2 纯 API 可替代场景

很多站点提供公开 API。优先使用官方 API，它更稳定、结构化更好、也更合规。只有当 API 不存在或权限不足时，才退化到浏览器自动化。

| 需求 | 推荐方案 |
|------|----------|
| 静态 HTML | HTTP + Jsoup |
| 有官方 API | 直接调用 API |
| JS 渲染页面 | 浏览器工具 |
| 需要登录/点击 | 浏览器工具 |
| 需要视觉核对 | 浏览器工具 + 截图 |

> 边界与坑：把浏览器当"万能抓取器"会迅速耗尽内存与并发额度。先判断是否真的需要 JS 执行。

# 2. 技术选型

结论先行：Java 生态首选 Playwright for Java，它在等待策略、自动重试、多浏览器一致性上优于老牌 Selenium。选型需同时考虑语言、维护成本与团队熟悉度。

| 方案 | 语言 | 特点 |
|------|------|------|
| Playwright | 多语言（含 Java） | 现代、稳定、推荐 |
| Selenium | 多语言 | 老牌、生态大 |
| Puppeteer | Node | 轻量 |
| Chrome DevTools Protocol | 底层 | 灵活 |

> Java 生态推荐 **Playwright for Java**。

## 2.1 主流方案对比

### 2.1.1 Playwright

Playwright 内置"自动等待"（Auto-waiting）：在执行点击、填充等操作前，会自动等待元素可交互，大幅减少手写 sleep。它还统一了 Chromium / Firefox / WebKit 三套内核的 API，跨浏览器测试成本最低。

### 2.1.2 Selenium 与 Puppeteer

Selenium 历史悠久、驱动生态丰富，但需要显式等待，代码更啰嗦；Puppeteer 只支持 Chromium 且以 Node 为主；CDP（Chrome DevTools Protocol，Chrome 开发者工具协议）最底层、最灵活，但需要自行处理会话与事件。

## 2.2 选型决策

### 2.2.1 决策清单

```
需要 Java 且要稳定？            -> Playwright
需要兼容旧系统/大量现成驱动？    -> Selenium
只在 Node 环境且只用 Chromium？  -> Puppeteer
需要极细粒度控制网络/性能？      -> CDP
```

### 2.2.2 版本与依赖

Maven 依赖需与浏览器镜像版本对齐，否则会出现驱动不匹配。生产环境建议固定版本号，避免自动升级引入不兼容。

```xml
<dependency>
    <groupId>com.microsoft.playwright</groupId>
    <artifactId>playwright</artifactId>
    <version>1.40.0</version>
</dependency>
```

# 3. 基础用法

结论先行：把 Playwright 的生命周期（创建、关闭）与页面生命周期（打开、关闭）分开管理。Playwright 与 Browser 是重量级、长生命周期的；Page 是轻量级、短生命周期的，应"用完即关"。

## 3.1 生命周期管理

### 3.1.1 单例 Playwright 与 Browser

`Playwright.create()` 会启动 Node 驱动进程，`launch()` 会启动浏览器进程，二者都极其昂贵，绝不可每次请求创建。应做成单例或对象池。

### 3.1.2 Context 与 Page 隔离

BrowserContext 相当于一个独立的"隐身窗口"，拥有独立的 Cookie、LocalStorage。为避免状态残留，每个任务使用新 Context，用完即关。这样既能隔离登录态，也能避免跨任务数据串味。

```java
@Component
public class BrowserTools implements AutoCloseable {

    private final Playwright playwright;
    private final Browser browser;

    public BrowserTools() {
        this.playwright = Playwright.create();
        this.browser = playwright.chromium().launch(
                new BrowserType.LaunchOptions().setHeadless(true));
    }

    @Tool("访问网页并提取纯文本内容")
    public ToolResult fetchPage(@P("网页 URL") String url) {
        // 安全校验
        if (!urlValidator.isAllowed(url)) {
            return ToolResult.fail("URL 不在允许范围内");
        }

        try (Page page = browser.newPage()) {
            page.navigate(url, new Page.NavigateOptions()
                    .setTimeout(15_000));

            // 等待内容加载
            page.waitForLoadState(LoadState.NETWORKIDLE);

            String text = page.evaluate("() => document.body.innerText")
                    .toString();

            return ToolResult.ok(truncate(text, 8000));
        } catch (Exception e) {
            return ToolResult.fail("访问失败：" + e.getMessage());
        }
    }

    @Override
    public void close() {
        browser.close();
        playwright.close();
    }
}
```

## 3.2 页面抓取工具实现

### 3.2.1 等待策略

等待分三档：`NAVIGATE`（仅导航完成）、`LOAD`（资源加载完成）、`NETWORKIDLE`（网络空闲，约 500ms 无请求）。动态页面通常用 `NETWORKIDLE`，但会拖慢速度；更精确的做法是等待目标元素出现。

```java
// 三种等待策略的组合使用
page.navigate(url, new Page.NavigateOptions()
        .setWaitUntil(WaitUntilState.DOMCONTENTLOADED)
        .setTimeout(15_000));

// 先等网络空闲，再等目标元素，双保险
page.waitForLoadState(LoadState.NETWORKIDLE);
page.waitForSelector("#result-list",
        new Page.WaitForSelectorOptions()
                .setState(WaitForSelectorState.VISIBLE)
                .setTimeout(10_000));
```

### 3.2.2 文本提取与截断

用 `document.body.innerText` 获取"可见文本"，它自动跳过 `display:none` 的元素，比 `textContent` 更贴近人类阅读结果。提取后必须截断（如 8000 字符），避免超出模型上下文。

> 生产建议：截断时保留开头与结尾，中间省略，通常开头是标题、结尾是关键结论。

# 4. 网页操作工具

结论先行：点击、输入、截图三个动作构成最小操作集。所有操作都必须设超时，否则页面卡死会拖垮整个 Agent。

## 4.1 点击与输入

### 4.1.1 定位器 Locator

Playwright 推荐 Locator API（`page.getByRole`、`page.locator`），它自带重试与严格模式，比原始的 `page.click(selector)` 更稳。CSS 选择器（层叠样式表选择器）应尽量语义化，避免依赖易变的 class 名。

### 4.1.2 表单填写

填写后通常需要触发 `blur` 或回车才能让前端框架感知变化。对于受控组件，`fill` 可能不触发 `input` 事件，此时需改用逐字符 `type`。

```java
@Tool("点击网页上的元素")
ToolResult click(@P("URL") String url,
                 @P("CSS 选择器") String selector) {
    try (Page page = browser.newPage()) {
        page.navigate(url);
        page.click(selector, new Page.ClickOptions().setTimeout(5000));
        return ToolResult.ok("点击成功");
    } catch (Exception e) {
        return ToolResult.fail("点击失败：" + e.getMessage());
    }
}

@Tool("在网页表单中输入内容")
ToolResult fillForm(@P("URL") String url,
                    @P("选择器") String selector,
                    @P("内容") String value) {
    try (Page page = browser.newPage()) {
        page.navigate(url);
        page.fill(selector, value);
        return ToolResult.ok("输入成功");
    } catch (Exception e) {
        return ToolResult.fail("输入失败：" + e.getMessage());
    }
}

@Tool("对网页截图")
ToolResult screenshot(@P("URL") String url,
                      @P("截图保存路径") String path) {
    try (Page page = browser.newPage()) {
        page.navigate(url);
        page.screenshot(new Page.ScreenshotOptions()
                .setPath(Path.of(path))
                .setFullPage(true));
        return ToolResult.ok("已截图：" + path);
    } catch (Exception e) {
        return ToolResult.fail("截图失败：" + e.getMessage());
    }
}
```

## 4.2 截图与文件

### 4.2.1 全页截图

`setFullPage(true)` 会滚动拼接整页，适合长页面；若只关心首屏，去掉该选项更快。截图路径必须限制在受控目录内，避免路径穿越。

### 4.2.2 下载与上传

上传用 `setInputFiles`；下载需先 `waitForDownload` 再落盘。二者都可能成为数据外泄通道，应配合出网白名单与目录白名单。

```java
// 上传：把本地文件绑定到 input 元素
page.setInputFiles("#upload", Path.of("/safe/input.csv"));

// 下载：等待下载事件，再保存到受控目录
Download download = page.waitForDownload(() ->
        page.click("#export-button"));
download.saveAs(Path.of("/safe/output.csv"));
```

> 边界与坑：`page.click` 若选择器命中多个元素会抛异常（严格模式）。优先用 `.first()` 或更精确的定位器。

# 5. 提取结构化内容

结论先行：结构化提取比"整页纯文本"更省上下文、更利于下游程序消费。优先用语义标签（`article`、`main`）而非易变的 class。

## 5.1 语义标签优先

### 5.1.1 正文提取

`article`、`main`、`.content` 这类语义容器通常包含正文主体。用 `locator(...).first()` 取第一个匹配，避免多元素报错。

```java
@Tool("从网页提取结构化信息")
ToolResult extractStructured(@P("URL") String url) {
    try (Page page = browser.newPage()) {
        page.navigate(url);

        // 提取标题与正文
        String title = page.title();
        String content = page.locator("article, main, .content")
                .first().innerText();

        return ToolResult.ok(Map.of(
                "url", url,
                "title", title,
                "content", truncate(content, 8000)));
    } catch (Exception e) {
        return ToolResult.fail("提取失败：" + e.getMessage());
    }
}
```

## 5.2 结构化字段抽取

### 5.2.1 CSS 选择器映射

把"字段名 -> 选择器"写成配置，即可把页面映射为 JSON。这样新增站点无需改代码，只需加配置。

```java
// 伪代码：字段映射
Map<String, String> schema = Map.of(
        "title", "h1.product-title",
        "price", ".price .amount",
        "stock", "#stock-status");

Map<String, String> data = schema.entrySet().stream()
        .collect(Collectors.toMap(
                Map.Entry::getKey,
                e -> page.locator(e.getValue()).first().innerText()));
```

### 5.2.2 表格与列表

表格用 `page.locator("table tr")` 遍历行，再取 `td`；列表用 `locator("li").allInnerTexts()`。注意分页，需循环点击"下一页"直到按钮消失。

| 结构 | 选择器 | 返回 |
|------|--------|------|
| 表格行 | `table tr` | 行集合 |
| 单元格 | `td` | 文本 |
| 列表项 | `li` | 文本数组 |
| 链接 | `a[href]` | 属性 |

> 生产建议：结构化提取应附带"来源 URL + 抓取时间"，便于溯源与去重。

## 5.3 抽取的边界与反例

### 5.3.1 反例：整页 innerText 当结构化数据

直接把整页 `innerText` 当结果返回，会混入导航、推荐、页脚等噪声，且下游无法直接消费。结构化抽取的价值在于"只取需要的字段"。

### 5.3.2 反例：硬编码单一站点选择器

为每个站点写死选择器会导致维护成本随站点数线性增长。更可持续的是"通用提取 + 站点覆盖"的分层策略。

| 做法 | 可维护性 | 准确率 |
|------|----------|--------|
| 整页文本 | 高 | 低 |
| 硬编码选择器 | 低 | 高 |
| 通用 + 覆盖 | 中高 | 高 |
| 模型抽取 | 高 | 中 |

> 边界与坑：模型抽取（把 HTML 交给模型直接输出 JSON）灵活但成本高、可能幻觉，适合字段不固定的场景。

# 6. 安全防护（重点）

结论先行：浏览器工具是 SSRF（Server-Side Request Forgery，服务端请求伪造）的高危入口，因为它由攻击者可控的 URL 驱动，且运行在内网环境中。白名单 + 内网屏蔽是底线。

## 6.1 SSRF 防护

### 6.1.1 白名单校验

先校验协议（只允许 http/https），再校验域名白名单。域名匹配必须做"后缀 + 点"匹配，避免 `evil-example.com` 绕过 `example.com`。

```java
// 正确：要求 host 等于白名单域名，或以 ".白名单域名" 结尾
private boolean hostAllowed(String host) {
    return ALLOWED_DOMAINS.stream().anyMatch(d ->
            host.equals(d) || host.endsWith("." + d));
}

// 错误：host.endsWith("example.com") 会放行 evil-example.com
```

### 6.1.2 DNS 重绑定与重定向

攻击者可让域名先解析到公网、请求时再解析到内网（DNS Rebinding，DNS 重绑定）。因此解析后必须校验实际 IP，且禁止跟随到内网的 3xx 重定向。

```java
@Component
public class UrlValidator {

    private static final Set<String> ALLOWED_DOMAINS = Set.of(
            "docs.example.com", "support.example.com");
    private static final Set<String> BLOCKED_HOSTS = Set.of(
            "169.254.169.254",            // 云元数据
            "localhost", "127.0.0.1");

    public boolean isAllowed(String url) {
        try {
            URI uri = URI.create(url);

            // ① 协议
            if (!Set.of("http", "https").contains(uri.getScheme())) {
                return false;
            }

            // ② 域名白名单
            String host = uri.getHost();
            if (ALLOWED_DOMAINS.stream().noneMatch(host::endsWith)) {
                return false;
            }

            // ③ 内网地址防护（SSRF）
            if (BLOCKED_HOSTS.contains(host)) {
                return false;
            }
            InetAddress addr = InetAddress.getByName(host);
            if (addr.isSiteLocalAddress() || addr.isLoopbackAddress()) {
                return false;
            }
            return true;
        } catch (Exception e) {
            return false;                 // 解析失败也拒绝
        }
    }
}
```

| 攻击 | 防护 |
|------|------|
| SSRF | 域名白名单 + 内网屏蔽 |
| 云元数据窃取 | 屏蔽 169.254.169.254 |
| 恶意页面 | 容器隔离 + 无持久状态 |
| 数据外泄 | 出网白名单 |

## 6.2 其他攻击面

### 6.2.1 云元数据

云厂商的元数据服务（如 `169.254.169.254`）可返回临时凭证。除屏蔽该 IP 外，还应屏蔽整个链路本地网段（`169.254.0.0/16`）与内网网段（`10.0.0.0/8`、`172.16.0.0/12`、`192.168.0.0/16`）。

### 6.2.2 下载与外泄

页面可通过表单或 JS 把数据 POST 到外部。应对容器做"默认拒绝出网 + 仅放行白名单"的网络策略，并监控异常出网流量。

> 边界与坑：只校验首次 URL 不够，浏览器内部的 XHR/fetch 请求同样可能访问内网，需在网络层（而非仅应用层）做隔离。

# 7. 容器化浏览器

结论先行：浏览器必须运行在隔离沙箱中，限制内存、CPU、网络与文件系统。容器化是成本最低、落地最快的隔离手段。

## 7.1 为什么容器化

浏览器进程权限高、攻击面大（解析任意 HTML/JS）。容器能限制其影响范围：即使被恶意页面攻破，也只能影响容器内部。

```java
// 在沙箱容器中运行浏览器
ProcessBuilder pb = new ProcessBuilder(
        "docker", "run", "--rm",
        "--network", "sandbox-net",
        "--memory", "1g",
        "mcr.microsoft.com/playwright/java:v1.40.0-jammy",
        "java", "-jar", "/app/browser-agent.jar");
```

> 浏览器占资源大（内存 500MB+），建议**独立服务 + 池化**。

## 7.2 镜像与网络

### 7.2.1 资源限制

除 `--memory` 外，还应设置 `--cpus`、`--pids-limit`（限制进程数，防 fork 炸弹）与只读文件系统 `--read-only`。

### 7.2.2 无持久状态

每次任务使用全新容器或全新 Context，任务结束即销毁，确保 Cookie、缓存、下载文件不残留。

| 限制项 | 参数 | 目的 |
|--------|------|------|
| 内存 | `--memory 1g` | 防内存耗尽 |
| CPU | `--cpus 1` | 防抢占 |
| 进程 | `--pids-limit 256` | 防 fork 炸弹 |
| 网络 | `--network sandbox-net` | 出网受控 |
| 文件 | `--read-only` | 防写盘 |

# 8. 与 Agent 集成

结论先行：浏览器工具应作为普通 `@Tool` 注册给 Agent，由 Agent 自主决定何时调用。关键是把 URL 校验也作为前置约束注入，而不是寄希望于模型自觉。

## 8.1 工具注册

```java
Assistant browserAgent = AiServices.builder(Assistant.class)
        .chatModel(model)
        .tools(new BrowserTools(), new UrlValidator())
        .build();

String result = browserAgent.chat(
        "帮我查一下 https://docs.example.com/api 的接口列表");
```

> 浏览器工具常与**联网搜索**配合：搜索找 URL → 浏览器读取内容。

## 8.2 搜索 + 浏览链路

### 8.2.1 两段式

先搜索获得候选 URL，再由模型筛选 1~3 个最相关 URL，最后逐一用浏览器读取。这样既保证时效性，又避免盲目遍历。

### 8.2.2 结果回灌

抓取结果应附带 URL 与标题回灌给模型，让回答可溯源。对超长内容做截断，并在提示中声明"网页内容为不可信数据"。

```
用户问题 -> 搜索 -> 候选 URL -> 浏览器读取 -> 综合回答
```

# 9. 性能与稳定性

结论先行：浏览器工具的性能瓶颈在"启动成本"与"等待策略"。用池化解决前者，用元素等待解决后者。

## 9.1 等待与超时

### 9.1.1 拒绝固定 sleep

固定 `Thread.sleep` 既慢又不可靠。应等待具体元素或网络空闲。

### 9.1.2 分层超时

导航超时、操作超时、整体任务超时三者都要设，且逐层收紧，防止单点卡死拖垮链路。

| 问题 | 对策 |
|------|------|
| 页面加载慢 | 设置超时 + 等待策略 |
| 动态内容 | 等待特定元素/网络空闲 |
| 反爬 | 遵守 robots.txt、限速 |
| 资源占用 | 池化 + 独立服务 |
| 元素找不到 | 重试 + 更稳健的选择器 |

```java
// 等待特定元素出现，而非固定睡眠
page.waitForSelector(".result-list",
        new Page.WaitForSelectorOptions().setTimeout(10_000));
```

## 9.2 池化与并发

### 9.2.1 浏览器池

维护固定数量的 Browser 实例，任务从池中借用、归还。池大小按内存预算设定，超出则排队而非无限扩容。

```java
// 伪代码：简易池化
Browser browser = browserPool.borrow(Duration.ofSeconds(5));
try {
    // 使用 browser.newContext() 隔离状态
} finally {
    browserPool.release(browser);
}
```

池化要解决三个问题：启动成本高（避免每次 launch）、状态污染（用新 Context 隔离）、资源失控（设上限）。归还前应清理 Context，并对"疑似损坏"的实例做重建。

| 策略 | 说明 |
|------|------|
| 固定池 | 池大小固定，排队等待 |
| 弹性池 | 按负载扩容，但有硬上限 |
| 健康检查 | 借出前探测实例可用性 |
| 超时重建 | 借用超时则新建实例 |

### 9.2.2 反爬与限速

遵守 `robots.txt`，对同一域名限速，设置合理的 User-Agent。过度抓取会触发封禁，得不偿失。

> 边界与坑：并发过高时，多个浏览器实例争抢 CPU 会导致所有页面都变慢，出现"雪崩式超时"。应给总并发设上限。

# 10. 生产注意事项

结论先行：把浏览器工具当作"不可信、高成本、高权限"的外部依赖来治理：隔离、限流、监控、审计缺一不可。

## 10.1 风险清单

| 风险 | 建议 |
|------|------|
| SSRF | 域名白名单 + 内网屏蔽 |
| 无沙箱 | 容器隔离 |
| 资源占用大 | 独立服务 + 池化 |
| 无限等待 | 设置超时 |
| 反爬违规 | 遵守 robots + 限速 |
| 状态残留 | 每次新 context |
| 无监控 | 监控成功率与耗时 |

## 10.2 监控与告警

### 10.2.1 指标

采集成功率、P95 耗时、超时率、内存占用、池等待时长。任一指标劣化都可能是页面改版或反爬升级的早期信号。

### 10.2.2 审计

记录每次访问的 URL、时间、耗时、结果大小与截图。审计日志既是排障依据，也是合规证据。

> 生产建议：对失败请求保留截图，能极大加速"选择器失效"类问题的定位。

# 11. 小结

- 浏览器工具处理动态网页与网页操作。
- Java 推荐 Playwright，需容器隔离。
- SSRF 防护是重中之重（白名单 + 内网屏蔽）。
- 资源占用大，建议独立服务 + 池化。
- 与搜索工具配合，形成"找 URL → 读内容"的链路。
- 能用静态抓取或 API 的场景，不要动用浏览器。
- 等待用元素条件而非固定睡眠，超时分层设置。
