基础表单能提交，进阶表单讲的是"体验与无障碍"——`datalist` 给输入框加自动补全，`fieldset` 把字段逻辑分组，正确的 `autocomplete` 让浏览器记住你的信息，而无障碍表单让屏幕阅读器用户也能顺畅填写。本篇补齐这些"差一点就专业"的细节。

# 1. datalist 自动补全

## 1.1 给输入框加候选

```html
<!-- list 指向 datalist 的 id，输入时弹出候选 -->
<label for="city">城市</label>
<input id="city" list="cities">
<datalist id="cities">
  <option value="北京"></option>
  <option value="上海"></option>
  <option value="广州"></option>
</datalist>
```

★ **关键认知**：`datalist` 给文本框提供**候选建议**（输入时下拉提示，仍可自由输入），是"既能选又能填"的折中方案。⚠ 它**不是下拉框**——不强制选、不校验值必须在列表里；想要"只能选其一"应继续用 `select`。`input` 的 `list` 属性与 `datalist` 的 `id` 挂钩。

# 2. fieldset 分组

## 2.1 逻辑分组与禁用

```html
<!-- fieldset 把相关字段分组；legend 是组的标题 -->
<fieldset>
  <legend>收货地址</legend>
  <label for="province">省份</label>
  <input id="province" type="text">
  <label for="city">城市</label>
  <input id="city" type="text">
</fieldset>

<!-- disabled 整体禁用一组字段 -->
<fieldset disabled>
  <legend>已填信息（不可改）</legend>
  <input type="text">
</fieldset>
```

★ **关键认知**：`fieldset` 用 `<legend>` 给**一组字段加标题**，语义上属于一个整体（如"收货地址""支付方式"）；`disabled` 能**整组禁用**。⚠ 长表单用 fieldset 分组，比堆一堆字段清晰得多；屏幕阅读器会播报 legend，帮助用户理解字段归属。

## 2.2 单选钮的天然分组

```html
<fieldset>
  <legend>请选择支付方式</legend>
  <label><input type="radio" name="pay" value="wechat"> 微信</label>
  <label><input type="radio" name="pay" value="ali"> 支付宝</label>
</fieldset>
```

★ **关键认知**：单选钮（radio）配 `fieldset + legend` 是**无障碍标准姿势**——`legend` 说明"这组选什么"，个体 radio 分别带 label。⚠ 没有 legend 的裸单选组，读屏用户面对一串"选项 A、选项 B"却不知道在选什么。

# 3. autocomplete 自动填充

```html
<form autocomplete="on">
  <!-- 语义化 autocomplete 值：让浏览器/密码管理器正确回填 -->
  <label for="name">姓名</label>
  <input id="name" name="name" autocomplete="name">

  <label for="email">邮箱</label>
  <input id="email" name="email" autocomplete="email">

  <label for="pw">密码</label>
  <input id="pw" name="pw" type="password" autocomplete="new-password">
</form>
```

★ **关键认知**：`autocomplete` 用**语义化的值**（`name`/`email`/`new-password`/`street-address`…）告诉浏览器"这个字段是什么"，从而正确自动填充与触发密码管理器。⚠ 注册用 `new-password`、登录用 `current-password`——写错会导致密码管理器乱填或拒绝保存；敏感字段可用 `autocomplete="off"` 关闭。

# 4. 无障碍表单

## 4.1 输入框配 hint

```html
<label for="pw">密码</label>
<input id="pw" type="password" aria-describedby="pw-hint">
<p id="pw-hint">至少 8 位，含大小写字母</p>
```

★ **关键认知**：`aria-describedby` 把**补充说明**关联到字段——屏幕阅读器聚焦到输入框时会读这段说明，普通用户也能看到。⚠ 提示信息别只靠颜色/图标传达（"红框=错"色盲用户看不懂），文字 + aria 才是正路。

## 4.2 错误状态的可访问表达

```html
<input id="email" type="email" aria-invalid="true" aria-describedby="email-error">
<p id="email-error">请输入有效的邮箱地址</p>
```

★ **关键认知**：`aria-invalid="true"` 标记字段**处于错误状态**，读屏会播报"无效"；配合 `aria-describedby` 关联错误文案。⚠ 手写校验时，把错误**可访问地**暴露（而非只有一个红框），是无障碍表单与普通表单的分水岭。

# 5. 自定义控件与无障碍原则

## 5.1 别重新发明原生控件

★ **关键认知**：**优先用原生 `input`/`select`/`button`**——它们天生可访问（键盘可用、读屏可读、可聚焦）。⚠ 用 `<div>` + 一堆 JS "造"一个复选框是头号反模式：你要手动处理键盘、焦点、读屏、状态标识，漏任何一项就破坏了可访问性。除非视觉要求极特殊，否则"能原生就原生"。

## 5.2 用 button 而非 div 做可点击元素

```html
<!-- ✓ 语义正确：按钮就是按钮，键盘可聚焦、可回车触发 -->
<button type="button" onclick="doThing()">执行</button>

<!-- ✗ 反模式：div + onclick，无焦点、无键盘、无语义 -->
<!-- <div onclick="doThing()">执行</div> -->
```

★ **关键认知**：可点击的交互元素用 `<button>`；`<div onclick>` 无法被键盘触发、无按钮语义。⚠ 一个判断标准：**"这个东西用户能用 Tab 键到达、回车触发吗？"**——不能，就该换成真正的交互元素，或补全套 ARIA + 键盘处理（成本远超换标签）。

# 6. 本篇小结与下一步

★ **关键认知**：进阶表单的四个方向——**datalist 补全、fieldset 分组、autocomplete 回填、aria 无障碍**。核心原则：**能原生就不造轮子，交互元素用语义标签，状态变化用文字 + ARIA 暴露**。

下一步进入**阶段三 · 语义化与可访问性**，第 9 篇：语义化与文档大纲——`article`/`section`/`nav` 到底怎么分。