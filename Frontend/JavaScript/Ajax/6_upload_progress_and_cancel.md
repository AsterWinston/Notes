文件传到一半用户关了页面、弱网下请求一直转圈、十个请求同时发出把浏览器连接池堵死——这些都指向同一组能力：**进度、取消、超时、并发控制**。本篇分别用 XHR 和 fetch 实现，并给出可直接复用的并发池写法。

# 1. 上传进度

## 1.1 XHR 的 upload 事件

`xhr.upload` 是一个独立的事件目标，专门上报请求体发送进度：

```javascript
const xhr = new XMLHttpRequest();
xhr.open("POST", "/api/upload", true);

xhr.upload.onprogress = (event) => {
  if (event.lengthComputable) {
    const percent = (event.loaded / event.total) * 100;
    console.log(`上传 ${percent.toFixed(1)}%`);
  }
};

xhr.upload.onload = () => console.log("请求体已全部发送");

xhr.onload = () => {
  if (xhr.status >= 200 && xhr.status < 300) {
    console.log("服务器处理完成", xhr.responseText);
  }
};

xhr.send(formData);
```

| 事件 | 触发时机 |
|------|----------|
| `loadstart` | 开始发送请求体 |
| `progress` | 发送过程中周期性触发 |
| `load` | 请求体发送完毕 |
| `error` | 上传失败 |
| `abort` | 被 `abort()` 取消 |
| `timeout` | 超过 `xhr.timeout` |
| `loadend` | 发送结束(成功或失败) |

★ **关键认知**：`xhr.upload.onload` 只代表"字节发出去了"，**不代表服务器处理成功**。⚠ 真正的业务结果要看 `xhr.onload` 里的 `status`；把上传进度 100% 当成"上传成功"是常见误判。

## 1.2 lengthComputable

```javascript
xhr.upload.onprogress = (event) => {
  if (!event.lengthComputable) return console.log(`已发送 ${event.loaded} 字节`);
  bar.style.width = `${(event.loaded / event.total) * 100}%`;
};
```

★ **关键认知**：`lengthComputable` 表示浏览器是否知道总大小。⚠ 使用分块传输或服务器未给出长度时它为 `false`，此时 `event.total` 是 0，直接相除会得到 `Infinity` 或 `NaN`，进度条就乱了。

## 1.3 为什么 fetch 做不了上传进度

`fetch` 的 `Request.body` 虽然可以是 `ReadableStream`，但需要 `duplex: "half"`，浏览器与服务端支持都很有限，实际项目里并不可靠。

⚠ **关键认知**：截至现代浏览器，**带上传进度的可靠方案仍是 XHR**。需要"进度条 + 取消 + 重试"时，用 XHR 封装或选支持进度回调的请求库，不必强行用 fetch。

# 2. 下载进度

## 2.1 XHR 的 progress

```javascript
const xhr = new XMLHttpRequest();
xhr.open("GET", "/api/large-file", true);
xhr.responseType = "blob";

xhr.onprogress = (event) => {
  if (event.lengthComputable) {
    const percent = (event.loaded / event.total) * 100;
    console.log(`下载 ${percent.toFixed(1)}%`);
  }
};

xhr.onload = () => {
  const url = URL.createObjectURL(xhr.response); // 生成临时下载地址
  Object.assign(document.createElement("a"), { href: url, download: "file.zip" }).click();
  URL.revokeObjectURL(url);                      // 用完释放
};

xhr.send();
```

## 2.2 fetch 流式下载进度

```javascript
async function download(url, onProgress, signal) {
  const response = await fetch(url, { signal });
  if (!response.ok) throw new Error(`HTTP ${response.status}`);
  const total = Number(response.headers.get("content-length")) || 0;
  const reader = response.body.getReader();
  const chunks = [];
  let loaded = 0;
  while (true) {
    const { done, value } = await reader.read();
    if (done) break;
    chunks.push(value);
    loaded += value.length;
    onProgress(loaded, total);
  }
  return new Blob(chunks);
}
```

★ **关键认知**：fetch 的下载进度不是事件，而是**读流的副作用**——每 `read()` 一次就累加一次字节数。⚠ `chunks` 会一直留在内存里，下载超大文件时应改用 `showSaveFilePicker` 或流式写盘，而不是全部攒成 `Blob`。

# 3. AbortController

## 3.1 基本用法

```javascript
const controller = new AbortController();

fetch("/api/slow", { signal: controller.signal })
  .then((response) => response.json())
  .then((data) => console.log(data))
  .catch((error) => {
    if (error.name === "AbortError") {
      console.log("请求已取消");
    } else {
      console.error(error);
    }
  });

// 需要取消时
controller.abort();
```

`abort()` 会让 fetch 的 Promise 以 `DOMException` 拒绝，`name` 为 `"AbortError"`；也可以传入原因，之后通过 `controller.signal.reason` 读取。

★ **关键认知**：`AbortController` 是**一次性的**——调用 `abort()` 后再复用同一个 signal 发请求，新请求会立刻被取消。⚠ 每个新请求都应创建新的 controller，除非你确实想"一次取消一组请求"。

## 3.2 取消多个请求

```javascript
const controller = new AbortController();
const { signal } = controller;
const requests = ["a", "b", "c"].map((id) => fetch(`/api/${id}`, { signal }));

Promise.allSettled(requests); // 三个请求共享同一个 signal
controller.abort();           // 一次全部取消
```

这适合"离开页面时取消本页所有请求"的场景。⚠ 共享 signal 后无法单独取消其中一个，需要精细控制就为每个请求建独立 controller。

## 3.3 超时

### 3.3.1 AbortSignal.timeout

```javascript
try {
  const response = await fetch("/api/slow", {
    signal: AbortSignal.timeout(5000), // 5 秒后自动 abort
  });
} catch (error) {
  if (error.name === "TimeoutError") {
    console.error("请求超时");
  }
}
```

### 3.3.2 手动控制与超时组合

```javascript
const controller = new AbortController();
// 两个信号任意一个触发，都会取消请求
const signal = AbortSignal.any([controller.signal, AbortSignal.timeout(5000)]);

fetch("/api/slow", { signal }).catch((error) => {
  if (error.name === "TimeoutError") return console.error("超时");
  if (error.name === "AbortError") return console.log("用户取消");
  console.error("其他错误", error);
});
```

★ **关键认知**：`AbortSignal.timeout()` 触发的是 `TimeoutError`，而 `controller.abort()` 触发的是 `AbortError`，两者要分开处理——超时通常提示"网络较慢，请重试"，主动取消则不该弹错误。⚠ 旧代码里常用 `setTimeout(() => controller.abort(), ms)`，那样只会得到 `AbortError`，无法区分超时和手动取消。

## 3.4 XHR 的取消与超时

```javascript
const xhr = new XMLHttpRequest();
xhr.open("GET", "/api/slow", true);
xhr.timeout = 5000;
xhr.onabort = () => console.log("已取消");
xhr.ontimeout = () => console.log("超时");
xhr.onload = () => console.log(xhr.status);
xhr.send();
xhr.abort(); // 取消
```

⚠ XHR 没有 `AbortSignal` 接口，`abort()` 之后 `readyState` 会回到 0 并触发 `abort` 事件；再次使用必须先 `open()`。

# 4. 并发控制

## 4.1 为什么需要限流

浏览器对同一主机的并发连接数有限制(HTTP/1.1 通常 6 个左右)，一次性发几十个请求只会让它们排队，还可能拖慢关键请求。常见策略：

| 策略 | 适用场景 |
|------|----------|
| 并发池 | 批量上传、批量拉详情 |
| 请求去重 | 同一资源被多个组件同时请求 |
| 防抖/节流 | 搜索联想、滚动加载 |
| 优先级队列 | 关键接口优先 |

## 4.2 一个并发池

```javascript
async function runPool(tasks, limit = 4) {
  const results = new Array(tasks.length);
  let cursor = 0;

  async function worker() {
    while (cursor < tasks.length) {
      const index = cursor++;
      results[index] = await tasks[index](); // 失败会中断本次 worker
    }
  }

  const workers = Array.from(
    { length: Math.min(limit, tasks.length) },
    () => worker()
  );

  await Promise.all(workers);
  return results;
}

// 用法：把每个请求包成函数，一次最多跑 3 个
const tasks = urls.map((url) => async () => (await fetch(url)).json());
const data = await runPool(tasks, 3);
```

★ **关键认知**：并发池的本质是**固定数量的 worker 轮流取任务**，不是"每批 3 个、等最慢的完成再下一批"。⚠ 上面用 `Promise.all` 时，任一任务失败会让整体提前 reject，但其他 worker 仍在跑；需要"全部结果都要"就改用 `Promise.allSettled` 或在任务内部捕获错误。

## 4.3 请求去重

用一个 `Map` 缓存"进行中的 Promise"：相同 URL 的请求直接复用，结束后删除键，允许下次重新请求。这样能避免"三个组件同时请求同一份用户信息"造成的重复流量。⚠ 只对**读请求**做去重，写请求重复执行可能产生副作用。

## 4.4 上传取消的完整示例

```javascript
function uploadFile(file, { onProgress, signal }) {
  return new Promise((resolve, reject) => {
    const xhr = new XMLHttpRequest();
    const form = new FormData();
    form.append("file", file);

    xhr.open("POST", "/api/upload", true);
    xhr.timeout = 60000;
    xhr.upload.onprogress = (event) => {
      if (event.lengthComputable) onProgress(event.loaded / event.total);
    };

    xhr.onload = () => {
      if (xhr.status >= 200 && xhr.status < 300) resolve(JSON.parse(xhr.responseText));
      else reject(new Error(`HTTP ${xhr.status}`));
    };
    xhr.onerror = () => reject(new Error("网络错误"));
    xhr.ontimeout = () => reject(new Error("上传超时"));
    xhr.onabort = () => reject(new DOMException("上传取消", "AbortError"));

    signal?.addEventListener("abort", () => xhr.abort(), { once: true });
    xhr.send(form);
  });
}
```

# 5. 常见坑

| 坑 | 说明 | 处理 |
|----|------|------|
| 把上传 100% 当成功 | 只代表字节发出 | 等 `onload` 看 `status` |
| `total` 为 0 时相除 | 得到 `NaN`/`Infinity` | 先判断 `lengthComputable` |
| 复用已 abort 的 signal | 新请求立刻失败 | 每个请求新建 controller |
| 超时与取消混淆 | 两者错误名不同 | 分别判断 `TimeoutError`/`AbortError` |
| 取消后不清理 UI | 进度条卡住 | 在 `finally` 里复位状态 |
| 大文件全量进内存 | 页面崩溃 | 流式写盘或分片上传 |

★ **关键认知**：取消不是"删掉请求"，而是**让浏览器停止等待并释放连接**。⚠ 对于已经到达服务器的上传，取消只能阻止前端继续发送，服务器可能已经写入部分数据，后端需要支持断点或清理临时文件。

# 6. 练习

1. 用 XHR 做一个文件上传：显示百分比、支持取消按钮、上传完成后显示服务器返回的文件地址。
2. 用 fetch + `AbortSignal.timeout` 实现 3 秒超时的请求，并区分"超时"和"用户取消"两种提示。
3. 写一个并发池，把 10 个请求限制在最多 4 个并发，统计总耗时并与不限流版本对比。

# 7. 小结

| 能力 | 方案 | 关键点 |
|------|------|--------|
| 上传进度 | `xhr.upload.onprogress` | 判断 `lengthComputable` |
| 下载进度 | XHR `onprogress` / fetch 读流 | fetch 需手动累加字节 |
| 取消 | `AbortController` / `xhr.abort()` | signal 一次性，abort 后不可复用 |
| 超时 | `AbortSignal.timeout()` | 错误名是 `TimeoutError` |
| 并发 | 固定 worker 池 | 不要一次性发全部请求 |
| 去重 | 缓存 in-flight Promise | 只对读请求使用 |

下一篇进入错误处理与最佳实践，把网络错误、HTTP 错误、解析错误和业务错误分层处理，并讨论重试、幂等、请求封装与前端安全。
