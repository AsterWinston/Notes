静态文件与上传功能几乎每个 Web 服务都会遇到，但也是最容易写出路径穿越、超大文件、错误类型漏洞的地方。本篇从静态目录、单页应用回退讲到单多文件上传、下载和安全限制。

# 1. 静态文件服务

## 1.1 Static 与 StaticFS

Gin 提供几种静态文件方法：

```go
r.Static("/assets", "./public/assets") // URL 前缀 -> 本地目录
r.StaticFile("/favicon.ico", "./public/favicon.ico")
r.StaticFS("/files", http.Dir("./public/files"))
```

| 方法 | 用途 |
|------|------|
| `Static` | 映射整个目录 |
| `StaticFile` | 映射单个文件 |
| `StaticFS` | 使用自定义 `http.FileSystem`，便于嵌入文件或加权限控制 |

`Static` 内部会做目录遍历防护，但不要把整个项目根目录映射出去：

```go
// 错误：可能暴露源码、配置和密钥
r.Static("/", ".")

// 正确：只暴露专门存放静态资源的目录
r.Static("/assets", "./public/assets")
```

★ **关键认知**：静态目录的根路径就是安全边界。只要目录里放的是可公开文件，映射出去问题不大；一旦把项目根目录或用户上传目录直接映射，就可能泄露配置、日志甚至源码。

## 1.2 单页应用回退

前端使用 History 路由时，刷新 `/users/42` 会向服务器请求这个路径。服务器如果没有对应路由，就会返回 404：

```text
浏览器请求 /users/42
        │
        ▼
Gin 找不到 /users/42
        │
        ▼
返回 404，但前端需要 index.html 来接管路由
```

常见做法是把未匹配的 GET 请求回退到 `index.html`：

```go
r.NoRoute(func(c *gin.Context) {
    if c.Request.Method == http.MethodGet {
        c.File("./public/index.html")
        return
    }

    c.JSON(http.StatusNotFound, gin.H{
        "code":    "NOT_FOUND",
        "message": "接口不存在",
    })
})
```

⚠ 不要把所有 404 都回退到 HTML，否则 API 请求会收到 HTML 而不是 JSON。至少要根据路径前缀或 `Accept` 头区分前端路由和 API 请求。

# 2. 文件上传

## 2.1 单文件上传

HTML 表单：

```html
<form action="/upload" method="post" enctype="multipart/form-data">
    <input type="file" name="file">
    <button type="submit">上传</button>
</form>
```

Gin Handler：

```go
func Upload(c *gin.Context) {
    file, err := c.FormFile("file")
    if err != nil {
        c.JSON(http.StatusBadRequest, gin.H{
            "code":    "INVALID_FILE",
            "message": "请选择要上传的文件",
        })
        return
    }

    // 不要直接使用 file.Filename 拼接路径
    dst := filepath.Join("./uploads", filepath.Base(file.Filename))
    if err := c.SaveUploadedFile(file, dst); err != nil {
        c.JSON(http.StatusInternalServerError, gin.H{
            "code":    "SAVE_FAILED",
            "message": "保存文件失败",
        })
        return
    }

    c.JSON(http.StatusOK, gin.H{
        "name": file.Filename,
        "size": file.Size,
    })
}
```

`file.Filename`、`file.Size`、`file.Header` 分别表示原始文件名、字节数和文件头。`file.Size` 是客户端声明的值，不能完全信任，服务端仍要限制实际读取大小。

## 2.2 多文件上传

```go
form, err := c.MultipartForm()
if err != nil {
    c.JSON(http.StatusBadRequest, gin.H{"code": "INVALID_FORM"})
    return
}

files := form.File["files"]
if len(files) == 0 {
    c.JSON(http.StatusBadRequest, gin.H{"code": "NO_FILE"})
    return
}
if len(files) > 5 {
    c.JSON(http.StatusBadRequest, gin.H{"code": "TOO_MANY_FILES"})
    return
}

for _, file := range files {
    dst := filepath.Join("./uploads", filepath.Base(file.Filename))
    if err := c.SaveUploadedFile(file, dst); err != nil {
        c.JSON(http.StatusInternalServerError, gin.H{"code": "SAVE_FAILED"})
        return
    }
}
```

## 2.3 保存与命名

直接使用用户文件名有三个问题：

1. **重名覆盖**：两个用户都上传 `avatar.png`，后一个覆盖前一个。
2. **路径穿越**：文件名可能是 `../../app.conf`。
3. **非法字符**：不同文件系统对特殊字符支持不同。

推荐使用服务端生成的文件名：

```go
ext := filepath.Ext(file.Filename)
name := fmt.Sprintf("%d_%s%s", time.Now().UnixNano(), uuid.NewString(), ext)
dst := filepath.Join("./uploads", name)
```

更严谨的做法是根据文件内容计算哈希，便于去重和校验：

```text
文件名 = sha256(文件内容) + 扩展名
```

★ **关键认知**：原始文件名只用于展示和下载时的建议名称，不能作为磁盘路径的一部分。磁盘路径必须由服务端控制。

# 3. 文件下载

```go
r.GET("/download/:name", func(c *gin.Context) {
    name := c.Param("name")
    path := filepath.Join("./uploads", filepath.Base(name))
    c.FileAttachment(path, name) // 设置 Content-Disposition: attachment
})
```

| 方法 | 行为 |
|------|------|
| `c.File` | 直接返回文件，浏览器可能内联展示 |
| `c.FileAttachment` | 设置下载文件名，浏览器通常下载 |

⚠ 下载接口同样要防止路径穿越。`filepath.Base` 只能去掉目录部分，不能替代权限校验；用户只能下载自己有权访问的文件。

# 4. 安全与限制

## 4.1 大小限制

Gin 的 `MaxMultipartMemory` 控制解析 multipart 表单时内存中保留的最大字节数，超出部分会写入临时文件，但它**不是上传大小上限**：

```go
r := gin.Default()
r.MaxMultipartMemory = 8 << 20 // 8 MiB 仅控制内存缓冲
```

真正的请求体限制应放在入口：

```go
r.Use(func(c *gin.Context) {
    c.Request.Body = http.MaxBytesReader(c.Writer, c.Request.Body, 10<<20)
    c.Next()
})
```

或者在 Nginx、网关层限制 `client_max_body_size`。多层限制更可靠，因为应用层限制发生得较晚。

## 4.2 路径穿越与类型校验

路径穿越示例：

```text
用户文件名：../../etc/passwd
直接拼接：./uploads/../../etc/passwd
实际路径：./etc/passwd
```

防御措施：

- 永远不要直接拼接用户提供的路径。
- 使用 `filepath.Base` 取文件名，拒绝包含路径分隔符的输入。
- 保存后检查最终路径是否仍在目标目录内。
- 使用服务端生成的文件名。

文件类型校验不能只看扩展名：

```go
f, err := file.Open()
if err != nil {
    // 处理错误
    return
}
defer f.Close()

head := make([]byte, 512)
n, _ := f.Read(head)
contentType := http.DetectContentType(head[:n]) // 根据内容推断类型
```

扩展名可以伪造，内容类型也可能被构造。高安全场景还应做病毒扫描、图片解码验证和存储隔离。

# 5. 常见错误

| 现象 | 常见原因 | 解决方式 |
|------|----------|----------|
| 上传失败但无错误 | 上传目录不存在或没有写权限 | 启动时创建目录并检查权限 |
| 文件互相覆盖 | 使用了原始文件名 | 服务端生成唯一文件名 |
| 下载文件名乱码 | `Content-Disposition` 编码问题 | 使用 `FileAttachment` 或正确编码 |
| 内存暴涨 | 大文件全部读入内存 | 使用流式处理并限制请求体 |
| 路径穿越 | 直接拼接用户文件名 | 使用 `filepath.Base` + 目录边界检查 |

# 6. 动手练习

实现头像上传接口：

1. 只允许 `image/jpeg` 和 `image/png`。
2. 单文件最大 2 MiB。
3. 使用 UUID 生成磁盘文件名，保留原扩展名。
4. 保存到 `./uploads/avatars`，返回可访问 URL。
5. 增加下载接口，并验证 `../` 文件名无法读取目录外文件。

验收标准：上传成功、类型错误、超限、路径穿越四类请求都有明确响应。

# 7. 小结

| 主题 | 要点 |
|------|------|
| 静态服务 | 只映射公开目录，不要把项目根目录暴露出去 |
| SPA 回退 | 区分前端路由和 API 请求，避免 API 收到 HTML |
| 上传 | 不信任原始文件名，服务端生成唯一名称 |
| 下载 | `FileAttachment` 设置下载名，仍要做权限校验 |
| 安全 | 请求体大小、内容类型、路径穿越三层防御 |

下一篇进入 JWT 认证，讲清认证与授权的区别、Token 结构、签发校验流程以及生产环境的安全实践。
