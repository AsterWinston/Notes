AJAX 是 jQuery "三件套"里的最后一块：选择器管找元素，事件管交互，AJAX 管和后端说话。今天 `fetch` 已经普及，但存量项目里 `$.ajax` 仍然遍地都是，而且它返回的 `jqXHR` 自带 Promise 风格接口，理解它和原生 Promise 的差异，是读老代码、排查请求问题的必备技能。

# 1. AJAX 基础

## 1.1 最小请求

```javascript
// 最简写法：只给 URL，默认 GET
$.ajax('/api/user')

// 常用写法：配置对象
$.ajax({
  url: '/api/user',
  method: 'GET',            // jQuery 3.x 推荐 method，type 是旧别名
  dataType: 'json',         // 期望服务器返回 JSON
  success: function (data) {
    console.log(data)
  }
})
```

★ **关键认知**：`$.ajax()` 返回的不是普通数据，而是一个 **jqXHR 对象**。它既是 XMLHttpRequest 的包装（可以 `.abort()`），又实现了 Promise 风格接口（可以 `.done()`、`.fail()`、`.then()`）。这意味着"发请求"和"处理结果"可以拆开写。

## 1.2 全局默认配置

```javascript
$.ajaxSetup({
  timeout: 10000,                     // 10 秒超时
  headers: { 'X-Requested-With': 'XMLHttpRequest' }
})
```

⚠ `$.ajaxSetup` 是**全局**的，会影响页面上所有请求，包括第三方插件发出的请求。多人协作或引入插件时慎用，更稳妥的是封装自己的 `request()` 函数。

# 2. 快捷方法

## 2.1 $.get 与 $.post

```javascript
// $.get(url, data, success, dataType)
$.get('/api/list', { page: 1 }, function (data) {
  console.log(data)
}, 'json')
```

快捷方法的参数顺序固定，容易记错。新代码建议统一用 `$.ajax({...})`，具名配置更清晰。

## 2.2 $.getJSON、load 与表单序列化

```javascript
$.getJSON('/api/config', function (config) {   // 等价 dataType: 'json'
  console.log(config.theme)
})

$('#sidebar').load('/partials/sidebar.html')     // 取 HTML 并插入

// 表单序列化：name=张三&age=20
const query = $('#form').serialize()
$.post('/api/save', query)
```

★ **关键认知**：`serialize()` 只收集**有 name 且未 disabled** 的控件。没有 `name` 的输入框、被禁用的字段不会出现在结果里，这是"表单提交后少字段"的最常见原因。

# 3. $.ajax 详解

## 3.1 常用参数

| 参数 | 说明 | 默认值 |
|------|------|--------|
| `url` | 请求地址 | 当前页 |
| `method` | HTTP 方法 | `"GET"` |
| `data` | 发送的数据，对象或字符串 | 无 |
| `dataType` | 期望的响应类型 | 智能推断 |
| `timeout` | 超时毫秒数 | 无 |
| `headers` | 自定义请求头 | 无 |
| `beforeSend` | 发送前回调，返回 `false` 可取消 | 无 |

## 3.2 dataType 与 JSON

```javascript
$.ajax({
  url: '/api/user',
  dataType: 'json',        // 告诉 jQuery 按 JSON 解析
  success: function (data) {
    console.log(data.name) // data 已经是对象，不用 JSON.parse
  }
})

// 发送 JSON 请求体
$.ajax({
  url: '/api/user',
  method: 'POST',
  contentType: 'application/json; charset=UTF-8',
  data: JSON.stringify({ name: '张三' }),
  processData: false       // 别把已序列化的字符串再转一次
})
```

⚠ 只写 `contentType: 'application/json'` 但 `data` 仍传对象，jQuery 会调用 `toString()` 得到 `[object Object]`。必须自己 `JSON.stringify`，并关掉 `processData`。

## 3.3 三种回调与状态

```javascript
$.ajax({
  url: '/api/list',
  success: function (data, textStatus, jqXHR) {
    // HTTP 2xx 时调用
  },
  error: function (jqXHR, textStatus, errorThrown) {
    // 网络错误、超时、非 2xx、解析失败都会进来
  }
})
```

| textStatus | 含义 |
|-----------|------|
| `"success"` | 成功 |
| `"timeout"` | 超时 |
| `"abort"` | 被 `abort()` 取消 |
| `"parsererror"` | 响应解析失败，如 JSON 格式错误 |
| `"error"` | HTTP 错误，如 404、500 |

# 4. Deferred 与 Promise

## 4.1 jqXHR 的双重身份

```javascript
const xhr = $.ajax('/api/user')

xhr.done(function (data) { console.log('成功', data) })
xhr.fail(function (jqXHR, textStatus) { console.log('失败', textStatus) })
xhr.always(function () { console.log('结束，无论成败') })
xhr.abort()   // 取消请求
```

## 4.2 then 的链式转换

```javascript
$.ajax('/api/user/1')
  .then(function (user) {
    return $.ajax('/api/orders/' + user.id)  // 返回值传给下一个 then
  })
  .then(function (orders) { render(orders) })
  .catch(function (err) { console.error('任一环节失败', err) })
```

```text
请求链：取用户 → 取该用户订单 → 渲染
任一环节 reject，直接跳到 catch
```

★ **关键认知**：`.then()` 返回的是**新的 promise**，因此可以转换值、串联请求、统一捕获异常；`.done()` / `.fail()` 返回的仍是原对象，只做"监听"，不具备转换能力。需要"请求 A 成功后再发请求 B"，用 `then`；只是"成功后做个副作用"，用 `done`。

## 4.3 与原生 Promise 的差异

| 维度 | jqXHR / Deferred | 原生 Promise |
|------|------------------|--------------|
| 取消 | `.abort()` | 需 `AbortController` |
| 进度 | `notify()` / `.progress()` | 无 |
| 完成回调 | `.done()` / `.fail()` / `.always()` | `.then()` / `.catch()` / `.finally()` |
| 返回值 | 部分方法返回原对象 | 总是返回新 Promise |

⚠ 不要把 jqXHR 直接丢给只接受原生 Promise 的代码。`await $.ajax(...)` 在 jQuery 3.x 中通常可用，因为它是 thenable；但 `.abort()`、`.done()` 这些独有能力，原生 Promise 链上没有。

# 5. 错误处理

## 5.1 区分错误类型

```javascript
$.ajax({
  url: '/api/data',
  dataType: 'json',
  timeout: 5000
}).fail(function (jqXHR, textStatus, errorThrown) {
  if (textStatus === 'timeout') {
    console.warn('请求超时')
  } else if (textStatus === 'abort') {
    console.warn('请求被取消')
  } else {
    console.warn('HTTP 错误', jqXHR.status, jqXHR.statusText)
  }
})
```

## 5.2 statusCode 与全局钩子

```javascript
$.ajax({
  url: '/api/user/1',
  statusCode: {
    404: function () { console.warn('用户不存在') }
  }
})

$(document).on('ajaxError', function (event, jqXHR) {
  if (jqXHR.status === 401) location.href = '/login'
})
```

全局事件还有 `ajaxStart`、`ajaxSend`、`ajaxSuccess`、`ajaxComplete`、`ajaxStop`，适合统一做 loading 遮罩。⚠ 全局钩子无法区分业务来源，容易把"用户主动取消"也当成错误弹窗，建议只处理明确的鉴权、网络类错误。

# 6. 并发

## 6.1 $.when 聚合

```javascript
$.when(
  $.getJSON('/api/user'),
  $.getJSON('/api/orders'),
  $.getJSON('/api/messages')
).done(function (userRes, ordersRes, messageRes) {
  // 每个参数都是 [data, textStatus, jqXHR] 数组
  render(userRes[0], ordersRes[0], messageRes[0])
}).fail(function () {
  console.warn('至少有一个请求失败')
})
```

```text
$.when(a, b, c)：全部 resolve 才成功，任意一个 reject 则失败；其余请求不会自动取消。
```

⚠ `$.when` 只负责"等待与聚合"，**不会**因为一个失败就取消其他请求。需要"一个失败全部取消"时，要自己保存每个 jqXHR 并调用 `.abort()`。

## 6.2 动态数组

```javascript
const urls = ['/api/a', '/api/b', '/api/c']
const requests = urls.map(function (url) {
  return $.getJSON(url)
})

$.when.apply($, requests).done(function () {
  const results = Array.prototype.map.call(arguments, function (res) {
    return res[0]
  })
  console.log(results)
})
```

控制并发数时，核心思路是"维护一个运行中计数，完成一个再补一个"，而不是一次性把几百个请求全部发出去。

# 7. 取消与竞态

## 7.1 abort 取消请求

```javascript
let xhr = null

$('#search').on('input', function () {
  if (xhr) xhr.abort()          // 取消上一次未完成的请求
  xhr = $.getJSON('/api/search', { q: this.value })
    .done(function (data) { render(data) })
    .fail(function (jqXHR, textStatus) {
      if (textStatus !== 'abort') console.warn('搜索失败')
    })
})
```

★ **关键认知**：搜索框、筛选器这类"输入即请求"的场景，**竞态**比性能更致命：先发的请求可能后回来，把新结果覆盖成旧结果。标准解法是"取消上一次"或用递增序号丢弃过期响应。⚠ `abort()` 会触发 `fail`，`textStatus` 为 `"abort"`，记得与真正的错误区分。

# 8. 常见坑

| 坑 | 表现 | 解决 |
|----|------|------|
| `contentType: json` 但没 stringify | 服务端收到 `[object Object]` | `JSON.stringify` + `processData: false` |
| 表单字段丢失 | 提交后少参数 | 检查 `name` 与 `disabled` |
| JSON 解析失败 | `parsererror` | 检查响应头与返回内容 |
| 全局 `ajaxSetup` 污染 | 插件请求被改 | 封装局部请求函数 |
| 忘记处理 abort | 取消也弹错误 | 判断 `textStatus === 'abort'` |
| 并发失败不取消其他 | 后台请求继续 | 保存 jqXHR 手动 abort |
| 同步请求 | 页面卡死 | 不要用 `async: false` |

# 9. 练习

## 9.1 练习一：搜索防抖 + 取消

实现一个搜索框：输入停止 300ms 后发请求；若上一次请求未完成，先 `abort()`；结果按关键词渲染。要求处理 `abort` 不弹错误。

## 9.2 练习二：并发聚合

同时请求用户信息、订单列表、消息数三个接口，全部成功后一次性渲染；任意一个失败时显示统一错误提示，并说明其余请求是否需要取消。

## 9.3 练习三：封装 request

用 `$.ajax` 封装一个 `request(options)`，要求：① 返回原生 Promise；② 统一处理 401 跳转；③ 支持超时；④ 保留取消能力。

# 10. 小结

| 主题 | 要点 |
|------|------|
| $.ajax | 配置对象清晰，返回 jqXHR |
| 快捷方法 | get/post/getJSON/load 参数顺序易错 |
| 数据格式 | JSON 请求体要 stringify 且关 processData |
| Deferred | done/fail 监听，then 可转换串联 |
| 并发 | `$.when` 聚合，失败不自动取消其他 |
| 取消 | `abort()` 触发 fail，需与真错误区分 |
| 竞态 | 取消上一次或用序号丢弃过期响应 |

下一篇进入插件开发，讲 `$.fn` 扩展、链式返回、默认参数合并、状态存放与命名冲突，以及如何判断一个插件值不值得引入。
