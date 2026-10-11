表单是用户把数据**交给你**的唯一入口——注册、登录、搜索、下单，都靠它。本篇讲清 `<form>` 的提交机制、`input`/`select`/`textarea` 三大输入族、`label` 与控件的正确挂钩，以及 GET/POST 两种提交方式的区别。

# 1. form 表单容器

## 1.1 一个最小表单

```html
<!-- action：提交到哪；method：提交方式 -->
<form action="/submit" method="post">
  <input name="username" type="text">
  <button type="submit">提交</button>
</form>
```

★ **关键认知**：`<form>` 是表单的**容器与提交单元**——`action` 定接收地址、`method` 定提交方式。点击 `type="submit"` 的按钮，整个表单里**带 `name` 的控件值**会被打包成一个请求发出去。⚠ 没有 `name` 的控件**不会参与提交**，这是新手最容易漏的细节。

## 1.2 GET vs POST

| 维度 | GET | POST |
|------|-----|------|
| 数据位置 | URL 查询串（`?a=1&b=2`） | 请求体 |
| 可缓存/书签 | 可 | 不可 |
| 数据量 | 受 URL 长度限制 | 较大 |
| 适用 | 搜索、查询 | 登录、提交敏感/大量数据 |

★ **关键认知**：GET 把数据放 **URL**（可见、可分享、可缓存），适合**无副作用的查询**；POST 放**请求体**（更隐蔽、量大），适合**登录、创建、修改**这类有副作用的提交。⚠ **任何敏感数据（密码、token）绝不能用 GET**——它会留在 URL、历史记录和服务器日志里。

# 2. 输入三族

## 2.1 input：万能输入

```html
<input type="text" name="username" placeholder="请输入用户名" value="默认值">
```

★ **关键认知**：`<input>` 靠 `type` 切换身份——文本、密码、数字、日期、文件……（第 7 篇逐类展开）。它是**自闭合标签**，没有子内容。⚠ `value` 是初始值，`placeholder` 是占位提示（**不是 label 的替代品**，见 3.1）。

## 2.2 textarea：多行文本

```html
<!-- textarea 是双标签，内容放标签之间（不是 value 属性） -->
<textarea name="comment" rows="5" cols="40">初始多行文字</textarea>
```

★ **关键认知**：`textarea` 是多行文本，⚠ **初始内容写在标签之间**（不是 `value`），且标签间的**空格换行都会被当成内容**——所以开标签和内容之间别留空行。

## 2.3 select：下拉选择

```html
<label for="city">城市</label>
<select name="city" id="city">
  <option value="beijing">北京</option>
  <option value="shanghai">上海</option>
  <option value="guangzhou" selected>广州</option>  <!-- selected 默认选中 -->
</select>
```

★ **关键认知**：`select` 装多个 `option`，提交的是**被选中 option 的 `value`**（不是显示文字）。⚠ `selected` 标记默认选项；若不显式选中，默认是第一个 option。`value` 是机器读的值（如 `beijing`），标签间是给人看的名字，两者要分清。

# 3. label：控件与说明的挂钩

## 3.1 label 的两种关联

```html
<!-- 方式一：for 指向控件 id（推荐，最清晰） -->
<label for="name">姓名：</label>
<input id="name" type="text">

<!-- 方式二：把控件包进 label 里 -->
<label>姓名：<input type="text"></label>
```

★ **关键认知**：`label` 关联控件后，**点击文字等同于点击控件**（聚焦、勾选），大大提升可用性与可访问性。⚠ **绝不要用 `placeholder` 替代 `label`**——占位符一输入就消失，且屏幕阅读器不读它，这会导致用户"不知道自己在填什么"。`for` 的值必须与控件 `id` 精确匹配。

## 3.2 为什么 label 重要

★ **关键认知**：关联 label 有两大收益——**点击热区变大**（移动端好点、复选框更好勾）与**屏幕阅读器能播报字段名**（无障碍）。⚠ 无 label 的表单对键盘/读屏用户几乎不可用，是表单可访问性的头号问题。

# 4. 按钮与提交

```html
<form action="/login">
  <!-- 三种按钮：submit 提交、reset 重置、button 普通（配 JS 用） -->
  <button type="submit">提交</button>
  <button type="reset">重置</button>
  <button type="button" onclick="alert('hi')">普通按钮</button>
</form>
```

★ **关键认知**：`button` 的 `type` 决定行为——`submit` 触发提交（**表单里默认就是 submit**）、`reset` 重置表单、`button` 无默认行为（给 JS 用）。⚠ 常见坑：在表单里放一个 `button` 却漏写 `type="button"`，点击它就意外**提交了表单**——记事:"表单里的按钮，except 提交按钮都要显式写 `type`"。

# 5. 本篇小结与下一步

★ **关键认知**：表单四件套——`form`（提交容器）、`name`（决定是否参与提交）、三族输入（input/textarea/select）、`label`（可访问性关键）。GET 用于查询、POST 用于提交，敏感数据绝不进 URL。

下一步第 7 篇：input 全家族与校验——20+ 种 `type` 各管什么，`required`/`pattern` 如何让浏览器帮你校验。