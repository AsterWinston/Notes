
# 1. 概述

Feign 原生不直接支持 Multipart 文件上传，需要引入 `feign-form` 和 `feign-form-spring` 扩展模块。文件下载则通过返回 `byte[]` 或 `ResponseEntity<byte[]>` 实现。

# 2. 依赖引入

```xml
<dependency>
    <groupId>io.github.openfeign</groupId>
    <artifactId>feign-core</artifactId>
</dependency>
<dependency>
    <groupId>io.github.openfeign.form</groupId>
    <artifactId>feign-form</artifactId>
    <version>3.8.0</version>
</dependency>
<dependency>
    <groupId>io.github.openfeign.form</groupId>
    <artifactId>feign-form-spring</artifactId>
    <version>3.8.0</version>
</dependency>
```

Spring Cloud OpenFeign 较新版本已自动包含 `feign-form-spring`。

# 3. 单文件上传

## 3.1 Feign 客户端

```java
@FeignClient(name = "file-service", configuration = MultipartSupportConfig.class)
public interface FileClient {

    @PostMapping(value = "/api/files/upload",
                 consumes = MediaType.MULTIPART_FORM_DATA_VALUE)
    String uploadFile(@RequestPart("file") MultipartFile file);
}
```

## 3.2 配置类

```java
public class MultipartSupportConfig {
    @Bean
    public Encoder feignFormEncoder(ObjectFactory<HttpMessageConverters> converters) {
        return new SpringFormEncoder(new SpringEncoder(converters));
    }
}
```

**关键点**：`SpringFormEncoder` 是 `feign-form-spring` 提供的，它会将 `@RequestPart` 标注的 `MultipartFile` 参数编码为 multipart/form-data 格式。

## 3.3 调用方式

```java
@RestController
public class UploadController {

    private final FileClient fileClient;

    @PostMapping("/upload")
    public String upload(@RequestParam("file") MultipartFile file) {
        return fileClient.uploadFile(file);
    }
}
```

# 4. 多文件上传

```java
@FeignClient(name = "file-service", configuration = MultipartSupportConfig.class)
public interface FileClient {

    @PostMapping(value = "/api/files/batch-upload",
                 consumes = MediaType.MULTIPART_FORM_DATA_VALUE)
    List<String> uploadMultipleFiles(@RequestPart("files") MultipartFile[] files);

    // 或使用 List
    @PostMapping(value = "/api/files/upload-list",
                 consumes = MediaType.MULTIPART_FORM_DATA_VALUE)
    List<String> uploadFileList(@RequestPart("files") List<MultipartFile> files);
}
```

# 5. 文件 + 附加参数上传

```java
@FeignClient(name = "file-service", configuration = MultipartSupportConfig.class)
public interface FileClient {

    @PostMapping(value = "/api/files/upload-with-meta",
                 consumes = MediaType.MULTIPART_FORM_DATA_VALUE)
    String uploadWithMetadata(
        @RequestPart("file") MultipartFile file,
        @RequestPart("meta") FileMeta meta
    );
}
```

```java
// DTO
public class FileMeta {
    private String title;
    private String description;
    private List<String> tags;
    // getter/setter...
}
```

服务端接收：

```java
@PostMapping(value = "/api/files/upload-with-meta",
             consumes = MediaType.MULTIPART_FORM_DATA_VALUE)
public String upload(@RequestPart("file") MultipartFile file,
                     @RequestPart("meta") FileMeta meta) {
    // ...
}
```

# 6. 文件下载

## 6.1 返回 byte[]

```java
@FeignClient(name = "file-service")
public interface FileClient {

    @GetMapping(value = "/api/files/{id}/download")
    byte[] downloadFile(@PathVariable String id);
}
```

```java
@Service
public class FileDownloadService {

    private final FileClient fileClient;

    public void download(String fileId, HttpServletResponse response) throws IOException {
        byte[] data = fileClient.downloadFile(fileId);
        response.setContentType("application/octet-stream");
        response.setHeader("Content-Disposition", "attachment; filename=file.pdf");
        response.getOutputStream().write(data);
    }
}
```

## 6.2 返回 feign.Response（推荐）

```java
@FeignClient(name = "file-service")
public interface FileClient {

    @GetMapping(value = "/api/files/{id}/download")
    feign.Response download(@PathVariable String id);
}
```

```java
public void download(String fileId, HttpServletResponse response) throws IOException {
    feign.Response feignResponse = fileClient.download(fileId);

    // 转发响应头
    for (Map.Entry<String, Collection<String>> header : feignResponse.headers().entrySet()) {
        response.setHeader(header.getKey(), String.join(",", header.getValue()));
    }

    // 转发响应体
    InputStream inputStream = feignResponse.body().asInputStream();
    OutputStream outputStream = response.getOutputStream();
    IOUtils.copy(inputStream, outputStream);
}
```

## 6.3 流式下载（大文件）

```java
// Feign 的 Response.body() 本身就是 InputStream，支持流式读取
// 不一次性加载到内存

public void streamDownload(String fileId, OutputStream targetStream) throws IOException {
    feign.Response response = fileClient.download(fileId);
    try (InputStream is = response.body().asInputStream()) {
        byte[] buffer = new byte[8192];
        int bytesRead;
        while ((bytesRead = is.read(buffer)) != -1) {
            targetStream.write(buffer, 0, bytesRead);
        }
    }
}
```

# 7. SpringFormEncoder 源码解析

```java
public class SpringFormEncoder extends FormEncoder {

    public SpringFormEncoder(Encoder encoder) {
        super(encoder);
    }

    @Override
    public void encode(Object object, Type bodyType, RequestTemplate template)
            throws EncodeException {
        // 检查 Content-Type 是否为 multipart/form-data
        Map<String, Collection<String>> headers = template.headers();
        if (isFormData(headers)) {
            // 委托给 FormEncoder 的多部分编码逻辑
            super.encode(object, bodyType, template);
        } else {
            // 非 multipart，委托给 SpringEncoder
            delegate.encode(object, bodyType, template);
        }
    }
}
```

```java
// FormEncoder 的核心逻辑
public class FormEncoder implements Encoder {

    @Override
    public void encode(Object object, Type bodyType, RequestTemplate template) {
        // 遍历方法参数
        // 对 MultipartFile 参数，读入字节并写入多部分 body
        // 对普通参数，按表单字段编码
    }
}
```

# 8. MultipartFile 传递原理

```
调用方：
  @RequestPart("file") MultipartFile file
    │
    ▼
  SpringFormEncoder 读取 file.getBytes()
    │
    ▼
  构建 multipart/form-data body
  ------WebKitFormBoundary
  Content-Disposition: form-data; name="file"; filename="test.pdf"
  Content-Type: application/pdf

  <file bytes>
  ------WebKitFormBoundary--
    │
    ▼
  HTTP POST (Feign 发送)

服务方：
  StandardMultipartHttpServletRequest 解析
    │
    ▼
  MultipartFile 对象注入到 Controller
```

# 9. 文件上传大小限制

```yaml
spring:
  servlet:
    multipart:
      max-file-size: 10MB        # 单个文件最大
      max-request-size: 50MB     # 总请求最大
```

Feign 作为客户端，需要确保：
- 客户端不超时（`read-timeout` 设置足够大）
- 超大文件考虑分片上传、断点续传

# 10. 最佳实践

1. **大文件使用 `feign.Response` 接收** — 避免一次性加载 byte[] 到内存
2. **上传统一使用 `SpringFormEncoder`** — 不要手动拼接 multipart body
3. **设置合理的超时** — 文件传输耗时长，`read-timeout` 应适当放大
4. **服务端返回 progress 接口** — 大文件上传配合进度查询接口
5. **考虑使用 OSS** — 超大数据直接上传到 OSS（对象存储），Feign 只传 URL
