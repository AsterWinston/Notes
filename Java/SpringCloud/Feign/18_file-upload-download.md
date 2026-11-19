# 1. 概述

Feign 原生不直接支持 Multipart 文件上传，需要引入 `feign-form` 和 `feign-form-spring` 扩展模块。文件下载则通过返回 `byte[]` 或 `ResponseEntity<byte[]>` 实现。

## 1.1 先看清这条路的整体代价

上传/下载是 Feign 最不擅长的场景，机制根源在 `RequestTemplate` 的 body 是**完整的 `byte[]`**（而非流）：`Contract` 解析 + Encoder 编码产出的是可重复读取的字节数组，这让重试成为可能（`7_interceptor.md` §8.1 每次尝试重放同一模板），代价是**文件多大、堆就吃多少，且至少两份**——`MultipartFile.getBytes()` 一份、模板 body 一份（SpringFormEncoder 内部还可能再组包）。所以本篇所有"最佳实践"都指向同一个结论：**超过内存预算的文件不该走 Feign**，对象存储直传 + Feign 只传 URL（§10.5）才是正解；Feign 文件接口适合的是"几百 KB 的附件/图片转发"量级。

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

包含了不等于生效了：自动配置的 Encoder 仍是 `SpringEncoder`，multipart 接口会报"无法序列化 MultipartFile"类错误（Jackson 把 MultipartFile 当普通 Bean 序列化成 `{name, originalFilename...}` 的 JSON——比报错更糟，**它不报错但服务端收到的是元数据 JSON 而非文件字节**）。必须显式注册 §3.2 的 `SpringFormEncoder` Bean 才接通，这正是 `6_encoder-decoder.md` §8 说的现成轮子。

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

包装 delegate 的写法是硬性要求而非风格偏好（`6_encoder-decoder.md` §8 同）：`new SpringFormEncoder()` 无参版内部用 `Encoder.Default`，同一接口里**非文件的 JSON 参数会退化成 toString**。配置类不加 `@Component`、只经 `configuration` 挂载的作用域纪律也已内置在示例里（`4_configuration.md` §5.1）——全局扫它会把所有客户端的 Encoder 换成表单版，普通 JSON 接口一起变行为。

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

这种"入口收文件、原样转发给下游"的中转模式，内存账要按**峰值并发 × 文件大小 × 3~4 份**估算（MultipartFile 磁盘溢出前的内存副本 + getBytes 拷贝 + 模板 body + 引擎写出缓冲）。Tomcat 默认 `file-size-threshold: 0` 让 MultipartFile 落盘，但 `getBytes()` 又把整份读回内存——大文件中转链路 OOM 的经典形态。降险手段：`spring.servlet.multipart.file-size-threshold: 2MB` 配合流式拷贝，或干脆 §10.5 直传。

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

数组与 `List` 两种签名 feign-form 都支持（每个元素生成一个同名 part）。注意与 HTTP 契约对齐：Tomcat/Spring 接收端把 `files` 的多个 part 聚合成数组/List 的前提是**每个 part 的 name 都叫 files、filename 各自保留**——手写接收 DTO 时用 `MultipartFile[]` 而不是指望 JSON 反序列化；另外批量的内存账按总字节数和，批次大小要有上限约定（一次 100 个文件与一次 100MB 的 zip 是完全不同的风险）。

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

"文件 part + JSON part"的混合形态里，`meta` 会被 SpringFormEncoder 序列化成该 part `Content-Type: application/json` 的内容，接收端 `@RequestPart FileMeta` 靠**part 的 Content-Type** 走消息转换器还原——若接收方写成 `@RequestParam FileMeta` 或漏了 consumes 声明，报的却是"缺少参数"，定位方向容易歪。老浏览器/表单来源不带 part Content-Type 时，一个稳妥的兼容写法是接收 `String metaJson` 手动反序列化；跨团队接口文档里值得把"meta part 必须是 application/json"写进契约。

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

`byte[]` 返回类型让 Decoder 把响应**全量读进堆**（`ByteArrayDecoder` 路径），再 write 出去时又经过 Servlet 缓冲——同一个文件在内存里至少两份。它还静默丢掉了 `Content-Type`/`Content-Disposition` 等元信息（示例硬编码 filename 就是症状）。只适合确定小的文件；文件名动态化、多语言文件名（`filename*=UTF-8''...` 编码）在 byte[] 模式下全部无从获取——要元信息就换 §6.2。

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

推荐它的前提是把两个资源语义处理对：**`feign.Response` 持有底层连接，必须 `try (feign.Response r = ...)` 关闭**——不解 body 直接抛异常的路径（ErrorDecoder 分支）最容易漏关，连接悬挂到超时才回收，表现为下载接口偶发极慢；**头转发要过滤 hop-by-hop 与冲突头**：`Transfer-Encoding`、`Content-Length` 原样转发给已经分块重编码的本层响应，会让客户端拿到长度对不上的报文（示例的全量转发在生产要改成白名单：Content-Type / Content-Disposition / Cache-Control / ETag）。另外返回 `feign.Response` 时 Decoder 不参与，熔断统计仍记成功——业务失败（下游返回 404 页面）要在自己的代码里判 `status()`。

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

流式路径的隐藏前提有两个：**引擎不能中途把响应体缓冲成 byte[]**（Apache/OkHttp 默认流式，但开了 FULL 日志会被 `logAndRebufferResponse` 全量 rebuffer——`10_logging.md` §6，大文件接口所在客户端的日志档位必须停在 BASIC）；**read-timeout 的语义是"两次数据包间隔"**（`11_timeout-connection.md` §4），慢传输大文件不触发它，但总时长没人管——一个吊着不结束的流会长期占用连接与线程，Bulkhead 并发闸（`14_circuit-breaker.md` §8）给下载类客户端单独限额是防拖垮的配套动作。示例的 try-with-resources 只关了流，外层 `response` 同样要关（关流即释放连接，多数引擎下二者等价，但显式 `close` response 更稳）。

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

（还原度说明：真实的 `SpringFormEncoder.encode` 以 `template.headers()` 里是否 multipart 分派；`FormEncoder` 拿到的是 Contract 展开后的**参数 Map**（feign 的 `MapView`），按值类型选择 `MultipartFile` 专用 PojoMultipart 处理器；分派逻辑在 `feign-form` 的 `ToStringExpander`/各种 `MultipartParser` 链。）从源码能回收两个使用推论：分派只看**请求头不看 Java 参数类型**，`consumes` 忘写 multipart 时它会安静地走 delegate 的 JSON 路径（§2 末尾的元数据 JSON 事故正是这条分派线的产物）；文件字节在 encode 阶段全量进模板，**编码器本身无流式选项**——§1.1 的内存账是结构性的，调参调不掉。

# 8. MultipartFile 传递原理

```text
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

图里"读 getBytes()"一步就是 §1.1 内存账的来源；另外 `filename` 与 part 的 `Content-Type` 由 MultipartFile 的 originalFilename 推断，**经 Feign 中转后接收端看到的 filename 是调用方入口收到的原始名**——做文件名安全校验（防路径穿越、扩展名白名单）的必须在**最终处理方**做，不能信任中间任何一跳已清洗过。

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

限额链路上每一跳都要对齐而不是只配这一处：**入口服务的 servlet multipart、中转 Feign 客户端（没有上行大小限制，但有超时/内存）、网关（Nginx `client_max_body_size` 默认 1MB——上传接口 413 的第一嫌疑人，`2_Nginx` 系列有详述）、最终提供方**。超时侧的换算：上传耗时 ≈ 文件大小 / 有效带宽 + 服务端落盘时间，read-timeout 覆盖"发送完成到收到响应"，大文件还要考虑 write 阶段的引擎行为（Apache 对 multipart 全量缓冲后发送，OkHttp 支持 chunked 流式写出）。分片/断点续传的正确载体是**存储层直传协议**（OSS 分片上传、tus），在 Feign 上自研分片协议等于重造一个语义残缺的传输层。

# 10. 最佳实践

1. **大文件使用 `feign.Response` 接收** — 避免一次性加载 byte[] 到内存（配套关闭纪律见 §6.2/§6.3）
2. **上传统一使用 `SpringFormEncoder`** — 不要手动拼接 multipart body（boundary 生成、转义规则手写必错）
3. **设置合理的超时** — 文件传输耗时长，`read-timeout` 应适当放大（并按 §9 换算，不是拍一个 60s）
4. **服务端返回 progress 接口** — 大文件上传配合进度查询接口
5. **考虑使用 OSS** — 超大数据直接上传到 OSS（对象存储），Feign 只传 URL（前端拿签名 URL 直传、后端只收回 URL 校验，文件字节根本不进应用进程——这是架构级的正解，本篇其余全是妥协）
6. **文件接口单独一个 FeignClient** — 与普通 JSON 接口分客户端配置：专属 `SpringFormEncoder`、独立超时档位、BASIC 日志（FULL 会把二进制灌进日志）、单独的 Bulkhead 限额——共享客户端会让文件接口的宽松配置外溢污染普通接口。
