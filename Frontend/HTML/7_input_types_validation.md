`<input>` 是表单里的"瑞士军刀"，20 多种 `type` 各管一类数据——只用对类型，手机就会弹数字键盘、日期有原生日历、邮箱有格式校验。本篇讲清 input 类型全集、`required`/`minlength`/`pattern` 等校验属性，以及约束校验 API——很多"自己手写正则校验"的精力，其实浏览器早替你做了一半。

# 1. input 类型全集

## 1.1 文本类

```html
<input type="text">      <!-- 单行文本 -->
<input type="password">  <!-- 密码（打码显示） -->
<input type="email">     <!-- 邮箱（自带格式校验） -->
<input type="tel">       <!-- 电话（手机弹数字键盘） -->
<input type="url">       <!-- 网址 -->
<input type="search">    <!-- 搜索框 -->
```

★ **关键认知**：不同 `type` 不只是样式不同，更是**语义 + 行为**的双重切换——`email`/`url` 触发格式校验，`tel` 在手机上弹数字键盘，`search` 显示清除按钮。⚠ 用对类型是"移动端体验"的第一步：数字、电话、日期该用什么类型就用什么，别全用 `text` 了事。

## 1.2 数字与时间类

```html
<input type="number" min="0" max="100" step="5">   <!-- 数字（带步进器） -->
<input type="range" min="0" max="100">            <!-- 滑块 -->
<input type="date">      <!-- 日期（原生日历） -->
<input type="time">      <!-- 时间 -->
<input type="datetime-local">  <!-- 日期+时间 -->
<input type="month">     <!-- 月份 -->
<input type="week">      <!-- 周 -->
```

★ **关键认知**：日期时间类输入给用户**原生日历/选择器**，体验远好于"手输文本再解析"。⚠ 日期类 `value` 的格式是标准化的（`date` 是 `YYYY-MM-DD`），但**显示格式随浏览器/系统本地化**，不要假设显示出来的一定是某种格式。

## 1.3 选择与文件类

```html
<input type="checkbox">       <!-- 复选框（多选） -->
<input type="radio" name="g" value="a">   <!-- 单选（同名成组） -->
<input type="file" accept=".png,.jpg" multiple>  <!-- 文件选择 -->
<input type="color">          <!-- 颜色选择器 -->
<input type="hidden">         <!-- 隐藏字段（随表单提交但不显示） -->
```

★ **关键认知**：`checkbox` 独立多选，`radio` 靠**同名 `name`** 成组（同名则互斥单选）；`file` 用 `accept` 限格式、`multiple` 允许多文件。⚠ 单选钮必须**同名**才互斥；提交时 checkbox/radio **选中才提交 value**，没勾选则整个字段不提交。

# 2. 校验属性

## 2.1 required 与长度

```html
<!-- required：必填；minlength/maxlength：长度约束 -->
<input type="text" required>
<input type="password" minlength="8" maxlength="20">
```

★ **关键认知**：`required` 让字段**必填**（空着提交会被浏览器拦截并提示）；`minlength`/`maxlength` 限长度，`min`/`max` 限数值范围。⚠ 这是**浏览器原生校验**——手机端也会有对应提示，省去大量"手写 JS 校验 + 自绘报错框"的工作。

## 2.2 pattern 正则

```html
<!-- pattern：用正则约束输入格式（如 6 位数字验证码） -->
<input type="text" pattern="\d{6}" title="请输入 6 位数字">
```

★ **关键认知**：`pattern` 挂一个**正则**做格式校验，不匹配则提交被拦。⚠ 配套写 `title`——不匹配时浏览器显示的就是 title 里的说明，缺了它用户只得到一句生硬的"格式不匹配"。正则按**整个输入**匹配，隐式带 `^...$`。

# 3. 提交时的校验行为

```html
<form action="/submit">
  <!-- 校验只在"提交"那一刻触发；pattern/required 不通过则阻止提交 -->
  <input type="email" required>
  <button type="submit">提交</button>
</form>
```

★ **关键认知**：原生校验是**提交时拦截**——点提交那一下，不合法字段被标红、聚焦、弹出提示，请求根本不发出去。⚠ 它不是"实时校验"（边输边提示），实时体验要配合 JS；但拦截层面已经足够可靠，服务端仍需**二次校验**（前端校验可被绕过，只算体验优化）。

# 4. 约束校验 API

```html
<form id="myForm">
  <input id="email" type="email" required>
  <button type="submit">提交</button>
</form>
```

```javascript
const email = document.getElementById('email')

email.validity.typeMismatch   // 是否类型不匹配（邮箱格式）
email.validity.valueMissing   // 是否必填却为空
email.validity.valid          // 是否整体通过校验

email.setCustomValidity('自定义错误信息')  // 自定义校验消息

// 手动触发校验
document.getElementById('myForm').checkValidity()  // true/false
```

★ **关键认知**：`validity` 对象暴露**校验的细分结果**（`typeMismatch`、`valueMissing`、`tooLong`…），JS 据此做**自定义校验 UI**；`setCustomValidity` 注入自定义错误。⚠ 这套 API 是"开车手写表单校验"的官方通道——别从头造轮子去重写浏览器已有的规则。

# 5. 本篇小结与下一步

★ **关键认知**：用对 `type` 白拿手机键盘/原生日历/格式校验；用 `required`/`pattern`/`min/max` 让浏览器拦截非法提交；用 `validity` API 做自定义校验。记住"**前端校验只算体验，服务端校验才是底线**"。

下一步第 8 篇：表单进阶——`datalist` 自动补全、`fieldset` 分组、无障碍表单与自定义控件的正确姿势。