可访问性(a11y)让残障用户——视障、听障、运动障碍——也能正常使用你的页面。它的两条腿是**用对语义**与**ARIA 补语义**。本篇讲清"ARIA 第一守则"、role/state/property 三要素、键盘导航与焦点管理,以及最容易踩的几个 ARIA 陷阱。

# 1. 什么是可访问性与 ARIA

## 1.1 ARIA 补语义的缺口

★ **关键认知**：ARIA(Accessible Rich Internet Applications)是一组**补充语义的属性**,当原生 HTML 无法表达必要的可访问语义时才使用。⚠ **ARIA 第一守则**:**能用原生 HTML 就用原生**——`<button>` 天然可访问,别用 `<div role="button">` 去重新发明它。ARIA 只是原生语义不够时的补丁,不是替代品。

## 1.2 ARIA 的三大类

| 类别 | 含义 | 示例 |
|------|------|------|
| role(角色) | 元素"是什么" | `role="navigation"`、`role="dialog"` |
| state(状态) | 元素"现在怎样" | `aria-expanded="true"`、`aria-checked="true"` |
| property(属性) | 元素的静态特征 | `aria-label`、`aria-describedby` |

★ **关键认知**：ARIA 的三类——**role 定义该是什么**(如"这是一个对话框")、**state 描述当前状态**(如"它是展开的")、**property 提供静态信息**(如"它的名字叫'关闭'")。⚠ role 一旦写了就**不能随意改**(它建立了与无障碍树的契约),state 则可随交互动态变化。

# 2. 常见 role 与命名

## 2.1 地标角色

```html
<header role="banner">页面头部</header>       <!-- 或直接语义标签 -->
<nav role="navigation">导航</nav>             <!-- 天然已是导航 -->
<main role="main">主内容</main>               <!-- 天然已是主内容 -->
<footer role="contentinfo">页脚</footer>
```

★ **关键认知**：地标(Landmark)角色让用户**在页面区块间快速跳转**。⚠ 用了 `<nav>`/`<main>` 等语义标签就**已有对应地标角色,无需重复声明 role**——重复甚至可能引起冲突。这就是"原生优先"的直接体现。

## 2.2 命名:aria-label 与 aria-labelledby

```html
<!-- aria-label:直接给元素起一个可访问名字 -->
<button aria-label="关闭对话框">×</button>

<!-- aria-labelledby:引用别处文字作名字 -->
<h2 id="dlg-title">设置</h2>
<button aria-labelledby="dlg-title">×</button>

<!-- aria-describedby:引用补充说明 -->
<input aria-describedby="pwd-hint">
<p id="pwd-hint">至少 8 位</p>
```

★ **关键认知**：可访问名(Accessible Name)是读屏播报的名字——`aria-label` 直接给名,`aria-labelledby` 复用别处文字,`aria-describedby` 追加说明。⚠ "关闭"按钮只有一个 `×` 字符时,读屏会读出"乘号",必须用 `aria-label="关闭"` 补上正确的名字——这是 ARIA 最经典的用途。

# 3. 键盘导航与焦点

## 3.1 焦点顺序

★ **关键认知**：键盘用户靠 **Tab 键**在可聚焦元素间移动,焦点顺序默认是**文档顺序**。⚠ 只要 HTML结构语义清晰、顺序自然,焦点顺序通常就正确;**别用 tabindex 乱改**——那会制造"键盘陷阱"或反直觉的跳转。可聚焦的天然元素:链接、按钮、表单控件。

## 3.2 tabindex 的正确姿势

```html
<!-- tabindex="-1":可编程聚焦,但不进 Tab 顺序(常用于弹窗、跳转目标) -->
<div id="modal" tabindex="-1">…</div>

<!-- tabindex="0":自定义元素进入 Tab 顺序 -->
<div tabindex="0" role="button">可键盘聚焦的自定义控件</div>

<!-- ⚠ 正数 tabindex 是反模式:强制指定顺序,难维护 -->
<!-- <div tabindex="3">…</div> -->
```

★ **关键认知**：`tabindex="-1"` 表示**可聚焦但不参与 Tab 顺序**(弹窗打开后手动 `focus()` 用);`tabindex="0"` 让自定义元素进 Tab 顺序。⚠ **避免正数 `tabindex`**(1、2、3…):它硬编码顺序,极难维护,几乎总是应该用文档顺序 + DOM 位置来解决。

# 4. 状态与动态内容

## 4.1 常见 state 示例

```html
<!-- 折叠面板:aria-expanded 表达展开/收起 -->
<button aria-expanded="false" aria-controls="panel">展开</button>
<div id="panel" hidden>面板内容</div>

<!-- 手风琴、选项卡等复合组件的 state 表达 -->
<div role="tablist">
  <button role="tab" aria-selected="true" aria-controls="tab1">标签1</button>
</div>
<div role="tabpanel" id="tab1">内容1</div>
```

★ **关键认知**：state 让动态 UI 对读屏"透明"——`aria-expanded` 表达展开态、`aria-selected` 表达选中态、`aria-controls` 关联被控元素。⚠ 这些 state **必须随交互更新**(点开面板要把 `aria-expanded` 改成 true),否则读屏用户看到的是"永远未展开"的过期状态——**ARIA 配上却不同步,比没有更糟**。

## 4.2 动态内容的通知

```html
<!-- aria-live:告诉读屏"这块内容会动态变化,变了请播报" -->
<div aria-live="polite">
  <p>购物车已更新</p>
</div>

<!-- role="alert":重要的即时警告 -->
<div role="alert">提交失败,请重试</div>
```

★ **关键认知**：`aria-live` 让**异步变化的内容被读屏通知**——`polite`(温和,等用户空闲再播报)与 `assertive`(急切,立即)两档;`role="alert"` 是即时的错误/警告。⚠ 表单提交失败、消息到达这类"看不见的变化",没有 aria-live 读屏用户就完全无感知。

# 5. ARIA 的常见陷阱

| 陷阱 | 正解 |
|------|------|
| `<div role="button">` 重造按钮 | 直接用 `<button>` |
| 用了 aria-expanded 却不同步 | 交互时同步更新 state |
| 正数 tabindex 硬编码顺序 | 用文档顺序 + tabindex="-1"/"0" |
| 只靠颜色表达状态 | 状态同时用文字/ARIA 表达 |
| aria-label 覆盖了可见文字 | label 应补充而非冲突 |

★ **关键认知**：ARIA 最大的坑是**"配了不管"**——声明了 role/state,却不在交互里维护,结果给读屏用户一个"说谎"的界面。⚠ 金句:**"没有 ARIA 的坏 ARIA,好过乱用的 ARIA"**——拿不准就用原生语义,别硬上 ARIA。

# 6. 本篇小结与下一步

★ **关键认知**：可访问性 = **原生语义打底 + ARIA 补缺口 + 键盘可用 + 状态同步**。第一守则"原生优先";命名用 aria-label/labelledby;动态内容用 aria-live;state 必须跟着交互走。

下一步第 11 篇:嵌入与组合——iframe/canvas/svg/template,以及它们与 Web Components 的边界。