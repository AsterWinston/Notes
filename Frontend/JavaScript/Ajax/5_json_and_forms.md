请求发不出去是一回事，发出去了后端收不到字段是另一回事。后者九成出在**请求体格式与 `Content-Type` 不匹配**：明明发的是 JSON，服务端却按表单解析；明明上传了文件，却手动写死了 multipart 边界。本篇把 JSON、URL 编码表单、`FormData` 和文件字段这几种常见格式讲透。

# 1. JSON 基础

## 1.1 语法与类型

JSON 是一种纯文本的数据交换格式，只支持六种值：

```json
{
  "id": 1001,
  "name": "张三",
  "active": true,
  "score": null,
  "tags": ["前端", "AJAX"],
  "address": { "city": "北京", "street": "长安街" }
}
```

| 类型 | 示例 | 注意 |
|------|------|------|
| 字符串 | `"hello"` | 必须双引号，不能单引号 |
| 数字 | `42`、`3.14` | 不支持 `NaN`、`Infinity` |
| 布尔 | `true`/`false` | 小写 |
| null | `null` | 没有 `undefined` |
| 数组 | `[1, 2, 3]` | 元素类型可混合 |
| 对象 | `{"a": 1}` | 键必须是双引号字符串 |

★ **关键认知**：JSON 不是 JavaScript 对象字面量。⚠ 键名不能省略引号、不能写注释、不能有尾逗号、不能用 `undefined` 和函数，这些在 JS 里合法，在 JSON 里全是语法错误。

## 1.2 JSON.stringify

```javascript
const user = {
  id: 1,
  name: "张三",
  createdAt: new Date("2026-01-01"),
  password: undefined,
  sayHi() {},
};

console.log(JSON.stringify(user));
// {"id":1,"name":"张三","createdAt":"2026-01-01T00:00:00.000Z"}
```

序列化时会发生这些"悄悄变化"：

| 值 | 结果 |
|----|------|
| `undefined`、函数、Symbol | 对象中直接省略；数组中变成 `null` |
| `Date` | 调用 `toISOString()` 变成字符串 |
| `NaN`、`Infinity` | 变成 `null` |
| `BigInt` | 抛 `TypeError` |
| 循环引用 | 抛 `TypeError: Converting circular structure to JSON` |

★ **关键认知**：`JSON.stringify` 会丢掉 `undefined` 和函数，所以"前端明明传了字段，后端说缺失"时，先检查该字段的值是否为 `undefined`。⚠ 需要保留时，显式转成 `null` 或空字符串，别指望序列化帮你兜底。

## 1.3 JSON.parse 与容错

```javascript
const data = JSON.parse('{"id":1,"name":"张三"}');

try {
  const broken = JSON.parse("<html>404</html>"); // 抛 SyntaxError
} catch (error) {
  console.error("响应不是合法 JSON");
}
```

★ **关键认知**：`fetch` 的 `response.json()` 内部就是 `JSON.parse`，**解析失败会 reject**。⚠ 如果服务器出错时返回的是 HTML 错误页，你看到的会是 `Unexpected token '<'`，而不是真实的状态码——先判断 `response.ok`，再决定是否解析。

# 2. 发送 JSON

## 2.1 请求端写法

```javascript
async function createUser(user) {
  const response = await fetch("/api/users", {
    method: "POST",
    headers: {
      "Content-Type": "application/json", // 关键：声明正文是 JSON
      "Accept": "application/json",       // 期望响应也是 JSON
    },
    body: JSON.stringify(user),          // 手动序列化
  });

  if (!response.ok) throw new Error(`HTTP ${response.status}`);
  return response.json();
}
```

★ **关键认知**：`fetch` **不会**因为你传了对象就自动 JSON 序列化，也不会自动加 `Content-Type`。⚠ 只写 `body: user` 会把对象转成 `"[object Object]"`，后端收到的是一串无意义的字符串。

## 2.2 后端如何接收

不同语言/框架读取 JSON 的方式不同，但共同点是：**JSON 在请求体里，不在表单参数里**。

```javascript
// Node/Express
app.use(express.json()); // 必须注册 JSON 解析中间件
app.post("/api/users", (req, res) => {
  console.log(req.body.name); // 已解析的对象
});
```

```java
// Java Servlet：不能 request.getParameter("name")
BufferedReader reader = request.getReader();
StringBuilder sb = new StringBuilder();
String line;
while ((line = reader.readLine()) != null) sb.append(line);
User user = new ObjectMapper().readValue(sb.toString(), User.class);
```

## 2.3 响应端的 Content-Type

服务器返回 JSON 时应设置 `Content-Type: application/json; charset=utf-8`，再输出序列化后的字符串。

★ **关键认知**：`Content-Type` 是"我发的是什么格式"的声明，请求和响应各自独立。⚠ 服务器返回 JSON 却把类型写成 `text/plain`，浏览器不会自动解析，Chrome 还可能触发 CORB 拦截，前端只能拿到空响应。

# 3. 表单编码

## 3.1 application/x-www-form-urlencoded

这是 HTML 表单的默认编码，格式就是 `key=value&key=value`：

```javascript
const body = "name=" + encodeURIComponent("张三") + "&age=20";

fetch("/api/login", {
  method: "POST",
  headers: { "Content-Type": "application/x-www-form-urlencoded" },
  body,
});
```

服务端(以 Servlet 为例)可以直接用 `request.getParameter("name")` 读取。这种格式简单、兼容性最好，但只适合扁平的键值对。

## 3.2 用 URLSearchParams 生成

```javascript
const params = new URLSearchParams({
  name: "张三",
  age: "20",
  tags: "a,b",
});

console.log(params.toString()); // name=%E5%BC%A0%E4%B8%89&age=20&tags=a%2Cb

fetch("/api/search", {
  method: "POST",
  body: params, // 浏览器自动设置 Content-Type
});
```

★ **关键认知**：传 `URLSearchParams` 时，`fetch` 会自动把 `Content-Type` 设为 `application/x-www-form-urlencoded;charset=UTF-8`，不用手写。⚠ 空格在该格式里会被编码成 `+`，与 `encodeURIComponent` 的 `%20` 不同，两者都能被服务端正确解码，不要混着手动拼接。

## 3.3 手动拼接的坑

⚠ `+`、`&`、`=`、`#`、`%` 在表单里都有特殊含义，手动拼接时不对键和值调用 `encodeURIComponent` 就会截断参数，用 `URLSearchParams` 最省心。

# 4. FormData

## 4.1 从表单直接构造

```html
<form id="profile">
  <input name="nickname" value="小明">
  <input name="age" type="number" value="18">
  <input name="avatar" type="file" accept="image/*">
</form>
```

```javascript
const form = document.getElementById("profile");
const data = new FormData(form); // 自动收集带 name 的控件

fetch("/api/profile", {
  method: "POST",
  body: data, // 不要手动设置 Content-Type
});
```

★ **关键认知**：用 `new FormData(form)` 时，控件必须有 `name` 属性才会被收集；文件输入会自动变成文件字段。⚠ 一定要让浏览器自己设置 `Content-Type`，因为它需要写入 `boundary` 分隔符。

## 4.2 手动构造

```javascript
const data = new FormData();
data.append("nickname", "小明");
data.append("age", "18");
data.append("tags", "前端");
data.append("tags", "AJAX"); // 同名键可追加多个

data.set("age", "20");         // 覆盖同名键
console.log(data.get("age"));  // "20"
data.delete("tags");           // 删除所有同名键
```

| 方法 | 行为 |
|------|------|
| `append(key, value)` | 追加，同名保留多个 |
| `set(key, value)` | 覆盖，同名只留一个 |
| `get(key)` | 取第一个值 |
| `getAll(key)` | 取全部值 |
| `delete(key)` | 删除所有同名键 |
| `has(key)` | 是否存在 |

## 4.3 文件字段

```javascript
const input = document.querySelector('input[type="file"]');
const file = input.files[0];

const data = new FormData();
data.append("avatar", file);              // 使用原文件名
data.append("avatar", file, "自定义名.png"); // 指定文件名

fetch("/api/upload", { method: "POST", body: data });
```

★ **关键认知**：`FormData` 里的文件是 `File`/`Blob` 对象，**不是路径字符串**。⚠ 浏览器出于安全不会把本地绝对路径交给 JS，所以后端收到的只有文件名和字节内容，别指望拿到 `C:\Users\...`。

# 5. Content-Type 对照表

| 发送内容 | 推荐 Content-Type | 服务端读取方式 |
|----------|-------------------|----------------|
| JSON 对象 | `application/json` | 读请求体后反序列化 |
| 普通表单 | `application/x-www-form-urlencoded` | `getParameter`/表单解析器 |
| 含文件的表单 | `multipart/form-data; boundary=...` | multipart 解析器 |
| 纯文本 | `text/plain` | 读请求体 |
| 二进制 | `application/octet-stream` | 读输入流 |

## 5.1 不要手写 multipart 的 boundary

```javascript
// 错误：手写 Content-Type 会丢掉 boundary
fetch("/api/upload", {
  method: "POST",
  headers: { "Content-Type": "multipart/form-data" },
  body: formData,
});
```

★ **关键认知**：`multipart/form-data` 的正文由多段组成，每段之间靠 `boundary` 分隔，而这个随机字符串是浏览器在序列化 `FormData` 时生成的。⚠ 手动写死 `Content-Type` 会让 boundary 丢失，服务端无法切分字段，表现为"文件收到了但文件名和字段全乱"或直接解析失败。

## 5.2 JSON 与 FormData 不能混用

⚠ 不要在一个请求里既发 `FormData` 又在字段中塞 JSON 字符串，服务端只会按 multipart 规则解析；要么整体用 JSON，要么整体用表单。

# 6. 一个完整的混合示例

```javascript
async function submitProfile(formElement) {
  const formData = new FormData(formElement);

  // 追加非表单字段
  formData.append("source", "web");

  const response = await fetch("/api/profile", {
    method: "POST",
    body: formData, // Content-Type 交给浏览器
    credentials: "include",
  });

  if (!response.ok) {
    const message = await response.text();
    throw new Error(`提交失败(${response.status})：${message}`);
  }

  return response.json();
}
```

# 7. 练习

1. 写一个函数，把对象转成 `application/x-www-form-urlencoded` 字符串，要求对键和值都正确编码。
2. 做一个包含文本字段和头像文件的上传表单，用 `FormData` 提交，并在发送前打印出所有键值确认内容。
3. 故意把 JSON 请求的 `Content-Type` 改成 `text/plain`，观察后端行为，并解释为什么字段读不到。

# 8. 小结

| 概念 | 要点 |
|------|------|
| JSON | 只支持六种类型，键必须双引号 |
| stringify | 会丢弃 `undefined`/函数，循环引用报错 |
| JSON 请求 | 手动 `JSON.stringify` + `Content-Type: application/json` |
| URL 编码 | `key=value&`，优先用 `URLSearchParams` |
| FormData | 同名 `append`、覆盖 `set`、文件是 `File` 对象 |
| multipart | 绝不手写 `Content-Type`，让浏览器生成 boundary |

下一篇进入上传进度与取消，讲 `xhr.upload` 的进度事件、fetch 的流式下载，以及用 `AbortController` 做取消、超时和并发控制。
