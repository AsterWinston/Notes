文件上传下载是 Axios 最容易"看起来能跑、上线就出问题"的领域：Content-Type 写错导致后端收不到文件，忘记 `responseType` 导致二进制被当 JSON 解析，错误响应本身也是 Blob 导致提示变成 `[object Blob]`。本篇把 FormData、进度、Blob 下载、文件名解析和大文件处理串起来。

# 1. FormData 上传

## 1.1 最小示例

```javascript
const file = document.querySelector('#file').files[0]
if (!file) return

const form = new FormData()
form.append('file', file) // 字段名要和后端约定一致
const { data } = await http.post('/upload', form, {
  timeout: 120000 // 上传大文件时适当放宽
})
```

## 1.2 多文件与额外字段

```javascript
const form = new FormData()
for (const file of input.files) form.append('files', file) // 多个同名文件

form.append('albumId', '42') // 普通字段一律转成字符串
form.append('meta', JSON.stringify({ source: 'web' }))
form.append('cover', coverFile, 'cover.png') // 第三个参数指定文件名
```

★ **关键认知**：FormData 的值只有三种：字符串、Blob/File。数字、布尔、对象都会被转成字符串或被忽略。⚠ 嵌套对象要先 `JSON.stringify`，并和后端约定按 JSON 解析，否则会得到 `[object Object]`。

## 1.3 不要手写 Content-Type

```javascript
// 错误：缺少 boundary，后端无法解析
http.post('/upload', form, {
  headers: { 'Content-Type': 'multipart/form-data' }
})

// 正确：交给浏览器生成
http.post('/upload', form)
```

`multipart/form-data` 必须带随机 boundary，用来分隔多个字段。浏览器发送 FormData 时会自动生成 `Content-Type: multipart/form-data; boundary=----WebKitFormBoundary...`。手动写死类型会把 boundary 弄丢。

★ **关键认知**：Axios 检测到请求体是 FormData 时，会删除你设置的 `Content-Type`，让运行时自己补。⚠ 但老版本或二次封装可能覆盖这个行为；最稳妥的做法就是**永远不要给 FormData 请求手动设置 Content-Type**。

## 1.4 Node 环境

Node 18+ 自带 `FormData` 和 `Blob`，可以直接 `append`；旧版本需要 `form-data` 包，并用 `form.getHeaders()` 显式拿请求头。

# 2. 上传进度

## 2.1 onUploadProgress

```javascript
await http.post('/upload', form, {
  onUploadProgress: (event) => {
    if (!event.total) return // 总大小未知时无法算百分比
    const percent = Math.round((event.loaded / event.total) * 100)
    console.log(`已上传 ${percent}%`)
  }
})
```

`event` 是 ProgressEvent，关键字段有 `loaded`（已传字节）、`total`（总字节，可能为 0）、`progress`（0 到 1 的比例）、`rate` / `estimated`（较新版本提供）。

## 2.2 计算与节流

```javascript
function createProgressHandler(onChange) {
  let lastPercent = -1
  return (event) => {
    if (!event.total) return
    const percent = Math.min(100, Math.round((event.loaded / event.total) * 100))
    if (percent !== lastPercent) {
      lastPercent = percent
      onChange(percent)
    }
  }
}

await http.post('/upload', form, {
  onUploadProgress: createProgressHandler(setPercent)
})
```

★ **关键认知**：进度回调触发非常频繁，直接在里面 `setState` 会导致大量渲染。⚠ 应按整数百分比去重，或用 `requestAnimationFrame` 节流；进度条只需要"看起来顺滑"，不需要每个字节都重绘。

## 2.3 取消上传

```javascript
const controller = new AbortController()
http.post('/upload', form, { signal: controller.signal })
  .catch((error) => {
    if (axios.isCancel(error)) console.log('用户取消上传')
  })

cancelButton.onclick = () => controller.abort()
```

取消会 abort XHR，已传出的分片可能已到达服务端。服务端需要自行清理临时文件，或依赖分片上传的"未完成即过期"机制。

# 3. 下载 Blob

## 3.1 必须设置 responseType

```javascript
// 错误：默认按 JSON 解析，二进制内容会损坏
const res = await http.get('/files/report')

// 正确
const res = await http.get('/files/report', { responseType: 'blob' })
```

常用值：`json`（默认，对象）、`text`（字符串）、`blob`（浏览器下载）、`arraybuffer`（自行解析二进制）、`stream`（Node 大文件下载）。

## 3.2 触发浏览器下载

```javascript
async function downloadFile(url, filename) {
  const { data } = await http.get(url, { responseType: 'blob' })
  const objectUrl = URL.createObjectURL(data)
  const anchor = document.createElement('a')
  anchor.href = objectUrl
  anchor.download = filename
  document.body.appendChild(anchor)
  anchor.click()
  anchor.remove()
  setTimeout(() => URL.revokeObjectURL(objectUrl), 1000)
}
```

★ **关键认知**：`URL.createObjectURL` 创建的地址会一直占用内存，直到页面卸载或手动 `revokeObjectURL`。⚠ 批量下载时尤其要注意释放，否则几十个文件就能让内存明显上涨。

## 3.3 错误响应也是 Blob

这是下载功能最隐蔽的坑：`responseType: 'blob'` 时，如果服务端返回 400 且响应体是 JSON，Axios 依然会把它包成 Blob 放进 `error.response.data`。直接 `error.response.data.message` 拿到的是 `undefined`。

```javascript
async function readBlobError(error) {
  const data = error.response?.data
  if (data instanceof Blob) {
    const text = await data.text()
    try {
      return JSON.parse(text)
    } catch (parseError) {
      return { message: text || '下载失败' }
    }
  }
  return data
}

try {
  await downloadFile('/files/report', 'report.xlsx')
} catch (error) {
  if (axios.isAxiosError(error)) {
    const body = await readBlobError(error)
    console.error(body.message)
  }
}
```

# 4. Content-Disposition 文件名

## 4.1 响应头格式

```text
Content-Disposition: attachment; filename="report.xlsx"
Content-Disposition: attachment; filename*=UTF-8''%E6%8A%A5%E5%91%8A.xlsx
```

`filename` 是传统写法，遇到中文可能乱码；`filename*` 使用 RFC 5987 编码，支持 UTF-8。解析时优先取 `filename*`。

## 4.2 解析函数

```javascript
function parseFilename(disposition, fallback = 'download') {
  if (!disposition) return fallback

  const utf8Match = /filename\*\s*=\s*UTF-8''([^;]+)/i.exec(disposition)
  if (utf8Match?.[1]) {
    try {
      return decodeURIComponent(utf8Match[1].trim())
    } catch (error) {
      // 编码非法时退回普通 filename
    }
  }

  const plainMatch = /filename\s*=\s*"?([^";]+)"?/i.exec(disposition)
  return plainMatch?.[1]?.trim() || fallback
}

const response = await http.get('/files/report', { responseType: 'blob' })
const filename = parseFilename(response.headers['content-disposition'], 'report.xlsx')
```

## 4.3 跨域时要暴露响应头

跨域请求默认只能读取"安全响应头"，`Content-Disposition` 不在其中。服务端必须声明：

```text
Access-Control-Expose-Headers: Content-Disposition
```

否则前端拿到的 `response.headers['content-disposition']` 是 `undefined`，只能自己拼文件名。

# 5. 大文件与流式处理

## 5.1 分片上传

大文件不要一次性塞进 FormData，常见方案是分片：

```text
1. 计算文件 hash（可选，用于秒传/去重）
2. 调 init 接口，拿到 uploadId
3. 将文件 slice 成 N 个 chunk
4. 并发上传每个 chunk（通常 1 到 3 个并发）
5. 全部完成后调 complete 接口合并
6. 失败时按 chunk 重试，已完成的跳过
```

```javascript
async function uploadInChunks(file, chunkSize = 5 * 1024 * 1024) {
  const total = Math.ceil(file.size / chunkSize)

  for (let index = 0; index < total; index++) {
    const start = index * chunkSize
    const form = new FormData()
    form.append('chunk', file.slice(start, start + chunkSize))
    form.append('index', String(index))
    form.append('total', String(total))

    await http.post('/upload/chunk', form, { timeout: 0 })
    setPercent(Math.round(((index + 1) / total) * 100))
  }
}
```

★ **关键认知**：分片的**大小**和**并发数**要一起调。分片太小会增加请求数量与合并开销，太大则单次失败重传成本高。⚠ 5 MB 到 10 MB 是常见起点，具体要结合网络与后端限制压测。

## 5.2 Range 断点续传

服务端支持断点续传时，请求带 `Range: bytes=${downloadedBytes}-`，响应返回 `206 Partial Content` 和 `Content-Range`。前端把每次收到的数据追加到 IndexedDB，最终合并。⚠ 浏览器端实现完整断点续传成本不低，很多项目会优先交给下载管理器或桌面端。

## 5.3 Node 流式下载

Node 中可以用 `responseType: 'stream'`，避免把整个文件读进内存：

```javascript
import { createWriteStream } from 'node:fs'
import { pipeline } from 'node:stream/promises'

const response = await axios.get(url, { responseType: 'stream', timeout: 0 })
await pipeline(response.data, createWriteStream(target))
```

`pipeline` 会自动处理背压与错误销毁，比手动 `pipe` 更安全。

## 5.4 浏览器流式下载与体积上限

Axios 在浏览器走 XHR，拿不到 `ReadableStream`，`responseType: 'stream'` 在浏览器不可用。需要真正流式处理时改用 `fetch` 的 `response.body.getReader()`，把读到的 `Uint8Array` 写入 OPFS 或做增量处理。⚠ 浏览器端流式受存储与权限限制，普通下载仍推荐 Blob。另外用 `maxContentLength` / `maxBodyLength` 限制请求与响应体积，避免异常大包拖垮进程。

# 6. 常见坑

## 6.1 高频坑清单

| 坑 | 后果 | 改法 |
|----|------|------|
| 手写 multipart 的 Content-Type | 后端收到空文件或报缺少 boundary | 交给浏览器生成 |
| 忘记 `responseType: 'blob'` | 二进制被当 JSON 解析 | 下载接口显式设置 |
| 用 JSON 方式读 Blob 错误 | 拿不到后端消息 | 先判断 `instanceof Blob` 再 `text()` |
| 选同一文件不触发 change | 无法重新上传 | 处理完清空 `input.value` |
| 忘记释放 Object URL | 内存持续增长 | 用完 `revokeObjectURL` |

# 7. 练习

## 7.1 练习一：带进度的上传

实现图片上传：显示百分比进度、支持取消、成功后预览。进度按整数百分比去重，避免频繁渲染。

## 7.2 练习二：安全下载

实现 `download(url, fallbackName)`：使用 `responseType: 'blob'`，从 `Content-Disposition` 解析文件名，失败时能读出 Blob 里的 JSON 错误信息，并在结束后释放 Object URL。

## 7.3 练习三：分片上传

把 20 MB 文件按 5 MB 切片上传，串行执行并显示总进度；再改成最多 3 个并发，比较耗时与重试难度。

# 8. 小结

| 概念 | 要点 |
|------|------|
| FormData | 不要手写 Content-Type，让浏览器生成 boundary |
| 进度 | `onUploadProgress` / `onDownloadProgress`，注意 total 可能为 0 |
| 下载 | 必须设置 `responseType: 'blob'` |
| Blob 错误 | 错误响应也可能是 Blob，先 `text()` 再解析 |
| 文件名 | 优先 `filename*=UTF-8''`，跨域需 Expose-Headers |
| 大文件 | 分片、Range、Node stream，浏览器流式用 fetch |

下一篇进入 TypeScript：`AxiosInstance`、`AxiosRequestConfig`、泛型响应、拦截器类型，以及如何通过模块扩展给自定义配置字段加类型。
