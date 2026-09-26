# 1.MinIO Java SDK 概述

## 1.1SDK 基本信息

MinIO Java SDK 是官方提供的 Java 客户端库，用于与 MinIO（以及任何 S3 兼容存储）交互。

| 属性 | 值 |
|------|-----|
| Maven 坐标 | `io.minio:minio` |
| 最新版本 | 8.5.x（与本笔记配套） |
| GitHub | https://github.com/minio/minio-java |
| 最低 Java 版本 | Java 8+（推荐 Java 11+） |
| 依赖 | OkHttp4（HTTP 客户端）、Jackson（JSON）、BouncyCastle（加密） |
| 线程安全 | MinioClient 是线程安全的，推荐作为单例使用 |

## 1.2Maven/Gradle 依赖

```xml
<!-- pom.xml -->
<dependency>
    <groupId>io.minio</groupId>
    <artifactId>minio</artifactId>
    <version>8.5.12</version>
</dependency>
```

```groovy
// build.gradle
implementation 'io.minio:minio:8.5.12'
```

### 完整依赖关系

```text
minio:8.5.x
├── com.squareup.okhttp3:okhttp:4.12.0
│   └── com.squareup.okio:okio:3.6.0
├── com.fasterxml.jackson.core:jackson-databind:2.x
├── com.fasterxml.jackson.core:jackson-annotations:2.x
├── org.bouncycastle:bcprov-jdk18on:1.x
├── org.simpleframework:simple-xml:2.7.1
└── com.carrotsearch.thirdparty:simple-xml-safe:2.7.1
```

---

# 2.MinioClient 初始化与配置

## 2.1基本初始化

```java
import io.minio.MinioClient;

// 方式一：Builder 模式（推荐）
MinioClient minioClient = MinioClient.builder()
    .endpoint("http://localhost:9000")   // MinIO 地址
    .credentials("minioadmin", "minioadmin")  // AccessKey / SecretKey
    .build();

// 方式二：简洁构造（已弃用但广泛使用）
MinioClient minioClient = MinioClient.builder()
    .endpoint("https://s3.amazonaws.com")  // 也可以连接 AWS S3
    .credentials("YOUR_ACCESS_KEY", "YOUR_SECRET_KEY")
    .region("us-east-1")                   // AWS S3 需要指定 region
    .build();
```

## 2.2完整配置项

```java
MinioClient minioClient = MinioClient.builder()
    .endpoint("https://minio.example.com:9000")
    .credentials("AKIAIOSFODNN7EXAMPLE", "wJalrXUtnFEMI/K7MDENG/bPxRfiCYEXAMPLEKEY")
    
    // HTTP 客户端配置
    .httpClient(OkHttpClient httpClient)  // 自定义 OkHttpClient
    
    // 连接池配置（内部默认值）
    // .connectionPool(...)
    
    // 代理配置
    // .proxy(Proxy proxy)
    
    // 区域（AWS S3 需配置，MinIO 可忽略）
    .region("us-east-1")
    
    .build();
```

## 2.3自定义 OkHttpClient（生产环境推荐）

```java
import okhttp3.OkHttpClient;
import okhttp3.ConnectionPool;
import okhttp3.Protocol;
import java.util.Arrays;
import java.util.concurrent.TimeUnit;

public class MinioClientFactory {

    public static MinioClient createProductionClient() {
        OkHttpClient httpClient = new OkHttpClient.Builder()
            // 连接超时
            .connectTimeout(10, TimeUnit.SECONDS)
            // 写入超时（上传大文件时需要足够长）
            .writeTimeout(30, TimeUnit.MINUTES)
            // 读取超时（下载大文件时需要足够长）
            .readTimeout(30, TimeUnit.MINUTES)
            // 调用超时（整个请求的总超时）
            .callTimeout(30, TimeUnit.MINUTES)
            
            // 连接池配置（减少 TLS 握手开销）
            .connectionPool(new ConnectionPool(
                200,                    // 最大空闲连接数
                5, TimeUnit.MINUTES     // 空闲连接保活时间
            ))
            
            // 协议支持
            .protocols(Arrays.asList(Protocol.HTTP_2, Protocol.HTTP_1_1))
            
            // 重试（某些场景下启用）
            .retryOnConnectionFailure(true)
            
            // 添加拦截器（监控/日志）
            .addInterceptor(new MinioLoggingInterceptor())
            
            // TLS 配置（如果使用自签名证书）
            // .sslSocketFactory(sslSocketFactory, trustManager)
            // .hostnameVerifier((hostname, session) -> true)  // 仅在开发环境
            .build();

        return MinioClient.builder()
            .endpoint("https://minio.example.com:9000")
            .credentials("YOUR_ACCESS_KEY", "YOUR_SECRET_KEY")
            .httpClient(httpClient)
            .build();
    }
}
```

## 2.4Spring Boot 集成配置

```java
@Configuration
public class MinioConfig {

    @Value("${minio.endpoint}")
    private String endpoint;

    @Value("${minio.access-key}")
    private String accessKey;

    @Value("${minio.secret-key}")
    private String secretKey;

    @Bean
    public MinioClient minioClient() {
        return MinioClient.builder()
            .endpoint(endpoint)
            .credentials(accessKey, secretKey)
            .build();
    }
}
```

```yaml
# application.yml
minio:
  endpoint: http://localhost:9000
  access-key: minioadmin
  secret-key: minioadmin
```

---

# 3.Bucket 操作

## 3.1创建 Bucket

```java
// 基本创建（默认 region: us-east-1）
minioClient.makeBucket(
    MakeBucketArgs.builder()
        .bucket("my-bucket")
        .build()
);

// 创建时指定 region 和对象锁
minioClient.makeBucket(
    MakeBucketArgs.builder()
        .bucket("my-locked-bucket")
        .region("cn-north-1")
        .objectLock(true)          // 启用对象锁（创建后不可关闭）
        .build()
);
```

## 3.2检查 Bucket 是否存在

```java
boolean exists = minioClient.bucketExists(
    BucketExistsArgs.builder()
        .bucket("my-bucket")
        .build()
);
System.out.println("Bucket 存在: " + exists);
```

## 3.3列举所有 Bucket

```java
List<Bucket> buckets = minioClient.listBuckets();
for (Bucket bucket : buckets) {
    System.out.println(bucket.name() + " - " + bucket.creationDate());
}
```

## 3.4删除 Bucket

```java
// Bucket 必须为空才能删除
minioClient.removeBucket(
    RemoveBucketArgs.builder()
        .bucket("my-bucket")
        .build()
);

// 先清空 Bucket，再删除
public void forceRemoveBucket(MinioClient client, String bucketName) throws Exception {
    // 列举并删除所有对象
    Iterable<Result<Item>> results = client.listObjects(
        ListObjectsArgs.builder()
            .bucket(bucketName)
            .recursive(true)
            .build()
    );
    for (Result<Item> result : results) {
        Item item = result.get();
        client.removeObject(
            RemoveObjectArgs.builder()
                .bucket(bucketName)
                .object(item.objectName())
                .build()
        );
    }
    // 删除Bucket
    client.removeBucket(RemoveBucketArgs.builder().bucket(bucketName).build());
}
```

---

# 4.Object 上传操作

## 4.1基本上传（小文件）

```java
// 方式一：从 InputStream 上传
try (InputStream inputStream = new FileInputStream("/path/to/file.jpg")) {
    minioClient.putObject(
        PutObjectArgs.builder()
            .bucket("my-bucket")
            .object("photos/avatar.jpg")
            .stream(inputStream, fileSize, -1)  // -1 表示未知分片大小，由 SDK 自动决定
            .contentType("image/jpeg")
            .build()
    );
}

// 方式二：从文件路径直接上传
minioClient.uploadObject(
    UploadObjectArgs.builder()
        .bucket("my-bucket")
        .object("photos/avatar.jpg")
        .filename("/path/to/file.jpg")
        .contentType("image/jpeg")
        .build()
);
```

## 4.2设置对象元数据

```java
Map<String, String> userMetadata = new HashMap<>();
userMetadata.put("author", "张三");
userMetadata.put("department", "研发部");
userMetadata.put("content-language", "zh-CN");

Map<String, String> tags = new HashMap<>();
tags.put("status", "published");
tags.put("year", "2024");

minioClient.putObject(
    PutObjectArgs.builder()
        .bucket("my-bucket")
        .object("reports/annual-2024.pdf")
        .stream(inputStream, fileSize, -1)
        .contentType("application/pdf")
        
        // 用户自定义元数据
        .userMetadata(userMetadata)
        
        // 对象标签
        .tags(tags)
        
        // 标准 HTTP 头
        .headers(Map.of(
            "Content-Disposition", "attachment; filename=\"annual-2024.pdf\"",
            "Cache-Control", "max-age=3600",
            "Content-Encoding", "gzip"
        ))
        .build()
);
```

## 4.3上传时指定存储类（Storage Class）

```java
// MinIO 支持的存储类（需根据实际部署情况）
minioClient.putObject(
    PutObjectArgs.builder()
        .bucket("my-bucket")
        .object("data/archive.zip")
        .stream(inputStream, fileSize, -1)
        .storageClass("STANDARD")       // STANDARD | REDUCED_REDUNDANCY | GLACIER
        .build()
);
```

## 4.4上传时启用服务端加密（SSE）

```java
import io.minio.ServerSideEncryption;
import io.minio.ServerSideEncryptionCustomerKey;

// SSE-S3: MinIO 管理密钥（最简单）
minioClient.putObject(
    PutObjectArgs.builder()
        .bucket("my-bucket")
        .object("encrypted-data.bin")
        .stream(inputStream, fileSize, -1)
        .sse(ServerSideEncryption.atMinio())   // SSE-S3
        .build()
);

// SSE-C: 客户提供密钥（密钥不存储在服务端）
String customerKey = "my-32-byte-encryption-key-here!!";  // 必须是 32 字节
ServerSideEncryptionCustomerKey sseC =
    new ServerSideEncryptionCustomerKey(customerKey.getBytes(StandardCharsets.UTF_8));

minioClient.putObject(
    PutObjectArgs.builder()
        .bucket("my-bucket")
        .object("sensitive-data.bin")
        .stream(inputStream, fileSize, -1)
        .sse(sseC)
        .build()
);

// SSE-KMS: 通过 KES 密钥管理服务（需要部署 KES）
// 使用 KeyId 引用 KES 中的密钥
minioClient.putObject(
    PutObjectArgs.builder()
        .bucket("my-bucket")
        .object("kms-data.bin")
        .stream(inputStream, fileSize, -1)
        .sse(ServerSideEncryption.withManagedKeys("my-kms-key-id"))
        .build()
);
```

---

# 5.Object 下载与读取

## 5.1基本下载

```java
// 方式一：下载到 OutputStream
try (OutputStream outputStream = new FileOutputStream("/path/to/download.jpg")) {
    minioClient.downloadObject(
        DownloadObjectArgs.builder()
            .bucket("my-bucket")
            .object("photos/avatar.jpg")
            .filename("/path/to/download.jpg")
            .overwrite(true)    // 是否覆盖本地文件
            .build()
    );
}

// 方式二：获取 InputStream（流式读取）
try (InputStream stream = minioClient.getObject(
        GetObjectArgs.builder()
            .bucket("my-bucket")
            .object("photos/avatar.jpg")
            .build()
    )) {
    // 处理流...
    byte[] bytes = stream.readAllBytes();
}

// 方式三：获取字节数组（小文件）
try (InputStream stream = minioClient.getObject(
        GetObjectArgs.builder()
            .bucket("my-bucket")
            .object("config.json")
            .build()
    )) {
    byte[] data = stream.readAllBytes();
    String content = new String(data, StandardCharsets.UTF_8);
}
```

## 5.2范围下载（Range Get）

```java
// 只下载文件的第 1024~2047 字节（共 1024 字节）
try (InputStream stream = minioClient.getObject(
        GetObjectArgs.builder()
            .bucket("my-bucket")
            .object("large-file.bin")
            .offset(1024L)        // 起始位置
            .length(1024L)        // 读取长度
            .build()
    )) {
    byte[] chunk = stream.readAllBytes();
    System.out.println("读取了 " + chunk.length + " 字节");
}

// 常见场景：断点续传、视频拖拽
// Range: bytes=0-1048575   ← 读取前 1MB
// Range: bytes=1048576-    ← 从第 1MB 到文件末尾
```

## 5.3获取对象元数据（不下载数据）

```java
StatObjectResponse stat = minioClient.statObject(
    StatObjectArgs.builder()
        .bucket("my-bucket")
        .object("photos/avatar.jpg")
        .build()
);

System.out.println("文件大小: " + stat.size());
System.out.println("ETag: " + stat.etag());
System.out.println("Content-Type: " + stat.contentType());
System.out.println("最后修改时间: " + stat.lastModified());
System.out.println("用户元数据: " + stat.userMetadata());
System.out.println("存储类: " + stat.storageClass());

// 检查对象是否存在（通过 stat）
public boolean objectExists(MinioClient client, String bucket, String object) {
    try {
        client.statObject(StatObjectArgs.builder()
            .bucket(bucket).object(object).build());
        return true;
    } catch (ErrorResponseException e) {
        if ("NoSuchKey".equals(e.errorResponse().code())) {
            return false;
        }
        throw new RuntimeException(e);
    }
}
```

---

# 6.Object 删除与批量操作

## 6.1删除单个对象

```java
minioClient.removeObject(
    RemoveObjectArgs.builder()
        .bucket("my-bucket")
        .object("photos/old-avatar.jpg")
        .build()
);
```

## 6.2批量删除

```java
// 构建要删除的对象列表
List<DeleteObject> objects = Arrays.asList(
    new DeleteObject("photos/old-1.jpg"),
    new DeleteObject("photos/old-2.jpg"),
    new DeleteObject("documents/draft.pdf")
);

// 批量删除
Iterable<Result<DeleteError>> results = minioClient.removeObjects(
    RemoveObjectsArgs.builder()
        .bucket("my-bucket")
        .objects(objects)
        .build()
);

// 处理删除失败的
for (Result<DeleteError> result : results) {
    DeleteError error = result.get();
    System.out.println("删除失败: " + error.objectName() + " - " + error.message());
}
```

## 6.3删除带版本控制的对象

```java
// 如果 Bucket 启用了版本控制
// 删除指定版本的对象
minioClient.removeObject(
    RemoveObjectArgs.builder()
        .bucket("my-bucket")
        .object("data.txt")
        .versionId("a1b2c3d4-e5f6-...")  // 指定要删除的版本
        .build()
);

// 如果只删除对象（不指定版本ID），会创建一个删除标记（DeleteMarker）
// 对象的最新版本变为不可见，但历史版本保留
```

---

# 7.复制对象

```java
// 同 Bucket 内复制
minioClient.copyObject(
    CopyObjectArgs.builder()
        .bucket("my-bucket")
        .object("photos/avatar-backup.jpg")
        .source(CopySource.builder()
            .bucket("my-bucket")
            .object("photos/avatar.jpg")
            .build())
        .build()
);

// 跨 Bucket 复制
minioClient.copyObject(
    CopyObjectArgs.builder()
        .bucket("backup-bucket")
        .object("archive/avatar-2024.jpg")
        .source(CopySource.builder()
            .bucket("my-bucket")
            .object("photos/avatar.jpg")
            .build())
        .build()
);

// 复制时修改元数据
minioClient.copyObject(
    CopyObjectArgs.builder()
        .bucket("my-bucket")
        .object("photos/preview.jpg")
        .source(CopySource.builder()
            .bucket("my-bucket")
            .object("photos/avatar.jpg")
            .build())
        .headers(Map.of("Content-Type", "image/webp"))
        .metadataDirective(Directive.REPLACE)
        .userMetadata(Map.of("purpose", "preview"))
        .build()
);
```

**注意事项**：
- COPY 是服务端操作，数据不经过客户端
- COPY 本质上是通过 MinIO 内部读取源对象并写入目标位置（服务器端复制）
- 复制大对象可能需要时间，此时 `CopyObjectResult` 包含 ETag 和最后修改时间

---

# 8.列举对象（ListObjects）

## 8.1基本列举

```java
// 列举 Bucket 中所有对象（注意：如果对象数量很大，可能耗时较长）
Iterable<Result<Item>> results = minioClient.listObjects(
    ListObjectsArgs.builder()
        .bucket("my-bucket")
        .recursive(true)    // 递归列举（不按目录层级分组）
        .build()
);

for (Result<Item> result : results) {
    Item item = result.get();
    System.out.println(item.objectName() + " - " + item.size() + " bytes");
}
```

## 8.2分页列举（模拟目录树）

```java
// 按目录层级列举（非递归）
Iterable<Result<Item>> results = minioClient.listObjects(
    ListObjectsArgs.builder()
        .bucket("my-bucket")
        .prefix("photos/2024/")    // 前缀过滤
        .delimiter("/")            // 目录分隔符
        .maxKeys(100)              // 每页最多返回数
        .build()
);

for (Result<Item> result : results) {
    Item item = result.get();
    if (item.isDir()) {
        System.out.println("[目录] " + item.objectName());
    } else {
        System.out.println("[文件] " + item.objectName() + " (" + item.size() + "字节)");
    }
}
```

## 8.3使用 startAfter 实现游标分页

```java
// 使用 V2 API 实现分页（推荐）
String startAfter = null;
boolean hasMore = true;

while (hasMore) {
    ListObjectsArgs.Builder builder = ListObjectsArgs.builder()
        .bucket("my-bucket")
        .prefix("logs/")
        .maxKeys(50);
    
    if (startAfter != null) {
        builder.startAfter(startAfter);
    }

    Iterable<Result<Item>> results = minioClient.listObjects(builder.build());
    
    int count = 0;
    for (Result<Item> result : results) {
        Item item = result.get();
        if (!item.isDir()) {
            System.out.println(item.objectName());
            startAfter = item.objectName();  // 记录最后一个 Key
            count++;
        }
    }
    
    hasMore = (count == 50);  // 如果返回数等于 maxKeys，可能还有更多
}
```

## 8.4包含版本信息的列举

```java
// 如果 Bucket 启用了版本控制，可以列举所有版本
Iterable<Result<Item>> results = minioClient.listObjects(
    ListObjectsArgs.builder()
        .bucket("my-bucket")
        .prefix("data.txt")
        .includeVersions(true)  // 包含所有版本
        .build()
);

for (Result<Item> result : results) {
    Item item = result.get();
    System.out.println(item.objectName() +
        " | versionId: " + item.versionId() +
        " | 是最新版本: " + item.isLatest() +
        " | 是删除标记: " + item.isDeleteMarker());
}
```

---

# 9.Presigned URL（预签名 URL）

Presigned URL 是 MinIO 中非常实用的特性，允许生成有时效的临时访问链接。

## 9.1生成预签名下载 URL

```java
// GET 预签名 URL（默认有效期 7 天）
String downloadUrl = minioClient.getPresignedObjectUrl(
    GetPresignedObjectUrlArgs.builder()
        .method(Method.GET)
        .bucket("my-bucket")
        .object("photos/avatar.jpg")
        .expiry(1, TimeUnit.HOURS)     // 有效期 1 小时
        .build()
);

// 返回示例：http://localhost:9000/my-bucket/photos/avatar.jpg
//   ?X-Amz-Algorithm=AWS4-HMAC-SHA256
//   &X-Amz-Credential=minioadmin/20240115/us-east-1/s3/aws4_request
//   &X-Amz-Date=20240115T120000Z
//   &X-Amz-Expires=3600
//   &X-Amz-SignedHeaders=host
//   &X-Amz-Signature=abc123...
```

## 9.2生成预签名上传 URL

```java
// PUT 预签名 URL（允许客户端直接上传，不经过服务端中转）
Map<String, String> reqParams = new HashMap<>();
reqParams.put("Content-Type", "image/jpeg");

String uploadUrl = minioClient.getPresignedObjectUrl(
    GetPresignedObjectUrlArgs.builder()
        .method(Method.PUT)
        .bucket("my-bucket")
        .object("uploads/user-1001/avatar.jpg")
        .expiry(30, TimeUnit.MINUTES)
        .extraQueryParams(reqParams)  // 额外的请求参数
        .build()
);

// 前端可以使用此 URL 直接上传：
// fetch(uploadUrl, {
//   method: 'PUT',
//   body: file,
//   headers: { 'Content-Type': 'image/jpeg' }
// });
```

## 9.3使用 POST Policy 表单上传（浏览器直传）

POST Policy 是更安全的浏览器端直传方式，支持上传限制：

```java
import io.minio.PostPolicy;

// 创建 POST 策略
PostPolicy policy = new PostPolicy(
    "my-bucket",
    "uploads/" + UUID.randomUUID() + ".jpg",
    ZonedDateTime.now().plusMinutes(30)  // 过期时间
);

// 添加限制条件
policy.addEqualsCondition("key", "uploads/user-1001/avatar.jpg");  // 限制只能上传到指定 Key
policy.addStartsWithCondition("Content-Type", "image/");           // 限制只能上传图片
policy.addContentLengthRangeCondition(1, 10 * 1024 * 1024);        // 限制大小 1B~10MB

// 生成策略和签名
Map<String, String> formData = minioClient.getPresignedPostFormData(policy);

// formData 中包含：
//   x-amz-signature: 签名
//   x-amz-algorithm: AWS4-HMAC-SHA256
//   x-amz-credential: 凭证
//   x-amz-date: 日期
//   policy: Base64 编码的策略文档
//   key: 对象的 Key
```

对应的前端 HTML 表单：

```html
<form action="http://localhost:9000/my-bucket" method="post" enctype="multipart/form-data">
    <!-- 隐藏字段：从后端获取的 formData -->
    <input type="hidden" name="key" value="uploads/user-1001/avatar.jpg">
    <input type="hidden" name="x-amz-algorithm" value="AWS4-HMAC-SHA256">
    <input type="hidden" name="x-amz-credential" value="...">
    <input type="hidden" name="x-amz-date" value="...">
    <input type="hidden" name="policy" value="...">
    <input type="hidden" name="x-amz-signature" value="...">
    
    <!-- 用户选择的文件 -->
    <input type="file" name="file">
    <button type="submit">上传</button>
</form>
```

---

# 10.分片上传（Multipart Upload）详解

分片上传是上传大文件的必备技术，也是 MinIO Java SDK 中最重要的特性之一。

## 10.1何时使用分片上传

- 文件大小 > 16MB（MinIO 内部会强制使用分片上传）
- 文件大小未知（如流式数据）
- 需要上传进度监控
- 需要断点续传
- 需要并发上传提高速度

## 10.2手动控制分片上传（完整流程）

```java
import io.minio.*;
import io.minio.messages.Part;

import java.io.*;
import java.util.ArrayList;
import java.util.HashMap;
import java.util.List;
import java.util.Map;

public class MultipartUploadDemo {

    private final MinioClient minioClient;

    public MultipartUploadDemo(MinioClient minioClient) {
        this.minioClient = minioClient;
    }

    public void uploadLargeFile(String bucket, String objectName, File file) throws Exception {
        long fileSize = file.length();
        // 每片 5MB（MinIO 要求的最小分片大小，最后一片可以小于此值）
        long partSize = 5 * 1024 * 1024; 

        // 步骤1：初始化分片上传
        Map<String, String> userMetadata = new HashMap<>();
        userMetadata.put("original-filename", file.getName());

        CreateMultipartUploadResponse initResponse = minioClient.createMultipartUpload(
            bucket,
            null,                  // region
            objectName,
            userMetadata,          // 元数据
            null                   // extra headers
        );
        String uploadId = initResponse.result().uploadId();
        System.out.println("开始分片上传 uploadId: " + uploadId);

        // 步骤2：上传各分片
        List<Part> parts = new ArrayList<>();
        try (BufferedInputStream bis = new BufferedInputStream(new FileInputStream(file))) {
            int partNumber = 1;
            byte[] buffer = new byte[(int) partSize];
            int bytesRead;

            while ((bytesRead = readFully(bis, buffer)) > 0) {
                // 上传当前分片
                UploadPartResponse partResponse = minioClient.uploadPart(
                    bucket,
                    null,          // region
                    objectName,
                    buffer,
                    bytesRead,
                    uploadId,
                    partNumber,
                    null,          // extra headers
                    null           // SSE
                );

                Part part = new Part(partNumber, partResponse.etag());
                parts.add(part);

                System.out.printf("分片 %d 上传完成, 大小: %d bytes, ETag: %s%n",
                    partNumber, bytesRead, partResponse.etag());

                partNumber++;
            }
        }

        // 步骤3：完成分片上传
        minioClient.completeMultipartUpload(
            bucket,
            null,          // region
            objectName,
            uploadId,
            parts.toArray(new Part[0]),
            null,          // extra headers
            null           // extra query params
        );

        System.out.println("分片上传完成！总片数: " + parts.size());
    }

    // 辅助方法：读取完整缓冲区
    private int readFully(InputStream in, byte[] buffer) throws IOException {
        int totalRead = 0;
        while (totalRead < buffer.length) {
            int bytesRead = in.read(buffer, totalRead, buffer.length - totalRead);
            if (bytesRead == -1) break;
            totalRead += bytesRead;
        }
        return totalRead;
    }

    // 取消分片上传（清理未完成的分片）
    public void abortMultipartUpload(String bucket, String objectName, String uploadId) 
            throws Exception {
        minioClient.abortMultipartUpload(
            bucket, null, objectName, uploadId, null, null
        );
        System.out.println("已取消分片上传: " + uploadId);
    }

    // 列举正在进行的分片上传
    public void listIncompleteUploads(String bucket) throws Exception {
        ListMultipartUploadsResponse response = minioClient.listMultipartUploads(
            bucket, null, null, null, null, 1000, null, null
        );
        for (Upload upload : response.result().uploads()) {
            System.out.println("Key: " + upload.objectName() + 
                               ", UploadId: " + upload.uploadId() +
                               ", 创建时间: " + upload.initiated());
        }
    }
}
```

## 10.3断点续传实现

```java
public class ResumableUploadDemo {

    private final MinioClient minioClient;

    public ResumableUploadDemo(MinioClient minioClient) {
        this.minioClient = minioClient;
    }

    // 断点续传：检查已上传的分片，只上传缺失的分片
    public void resumeUpload(String bucket, String objectName, 
                              File file, String uploadId) throws Exception {
        // 1. 列举已上传的分片
        ListPartsResponse partsResponse = minioClient.listParts(
            bucket, null, objectName, 1000, 0, uploadId, null, null
        );

        // 构建已上传分片的 Map（partNumber → Part）
        Map<Integer, Part> uploadedParts = new HashMap<>();
        for (Part part : partsResponse.result().partList()) {
            uploadedParts.put(part.partNumber(), part);
        }

        long fileSize = file.length();
        long partSize = 5 * 1024 * 1024;
        int totalParts = (int) Math.ceil((double) fileSize / partSize);

        List<Part> allParts = new ArrayList<>(uploadedParts.values());

        // 2. 只上传缺失的分片
        try (RandomAccessFile raf = new RandomAccessFile(file, "r")) {
            for (int partNumber = 1; partNumber <= totalParts; partNumber++) {
                if (uploadedParts.containsKey(partNumber)) {
                    System.out.println("分片 " + partNumber + " 已上传，跳过");
                    continue;
                }

                // 计算分片的起始位置和大小
                long startPos = (long) (partNumber - 1) * partSize;
                int currentPartSize = (int) Math.min(partSize, fileSize - startPos);

                byte[] buffer = new byte[currentPartSize];
                raf.seek(startPos);
                raf.readFully(buffer);

                UploadPartResponse partResponse = minioClient.uploadPart(
                    bucket, null, objectName,
                    buffer, currentPartSize,
                    uploadId, partNumber,
                    null, null
                );

                Part uploaded = new Part(partNumber, partResponse.etag());
                allParts.add(uploaded);
                System.out.printf("断点续传: 分片 %d/%d 完成%n", partNumber, totalParts);
            }
        }

        // 3. 完成分片上传
        minioClient.completeMultipartUpload(
            bucket, null, objectName, uploadId,
            allParts.toArray(new Part[0]), null, null
        );
        System.out.println("断点续传完成！");
    }
}
```

## 10.4使用 composeObject 进行分片合并上传

`composeObject` 允许将多个已存在的对象合并为一个新对象，适合以下场景：
- 合并多个分片文件
- 在服务端拼接文件而不经过客户端

```java
// 将两个文件在服务端合并为一个
List<ComposeSource> sources = Arrays.asList(
    ComposeSource.builder()
        .bucket("my-bucket")
        .object("part-1.csv")
        .build(),
    ComposeSource.builder()
        .bucket("my-bucket")
        .object("part-2.csv")
        .build()
);

minioClient.composeObject(
    ComposeObjectArgs.builder()
        .bucket("my-bucket")
        .object("merged.csv")
        .sources(sources)
        .contentType("text/csv")
        .build()
);

// 合并后可以删除源分片
minioClient.removeObject(RemoveObjectArgs.builder()
    .bucket("my-bucket").object("part-1.csv").build());
minioClient.removeObject(RemoveObjectArgs.builder()
    .bucket("my-bucket").object("part-2.csv").build());
```

---

# 11.对象锁（Object Lock）操作

需要对 Bucket 启用对象锁后，才能对对象设置保留期限和依法保留。

```java
// 上传对象并设置保留模式
minioClient.putObject(
    PutObjectArgs.builder()
        .bucket("my-locked-bucket")
        .object("legal_document.pdf")
        .stream(inputStream, fileSize, -1)
        
        // 保留模式
        .retention(Retention.create(
            RetentionMode.COMPLIANCE,              // 合规模式（不可被任何人删除）
            ZonedDateTime.now().plusYears(7)        // 保留 7 年
        ))
        
        // 依法保留
        .legalHold(io.minio.messages.LegalHold.create(true))
        
        .build()
);
```

```java
// 单独设置对象保留期限
minioClient.setObjectRetention(
    SetObjectRetentionArgs.builder()
        .bucket("my-locked-bucket")
        .object("document.pdf")
        .retention(Retention.create(
            RetentionMode.GOVERNANCE,               // 治理模式（有权限的用户可删除）
            ZonedDateTime.now().plusYears(5)         // 保留 5 年
        ))
        .bypassGovernanceRetention(true)             // 如果需要绕过治理模式
        .build()
);

// 设置依法保留
minioClient.setObjectLegalHold(
    SetObjectLegalHoldArgs.builder()
        .bucket("my-locked-bucket")
        .object("document.pdf")
        .legalHold(io.minio.messages.LegalHold.create(true))
        .build()
);
```

---

# 12.对象标签管理

```java
// 获取对象标签
Tags tags = minioClient.getObjectTags(
    GetObjectTagsArgs.builder()
        .bucket("my-bucket")
        .object("photo.jpg")
        .build()
);
System.out.println("标签: " + tags.get());

// 设置对象标签（覆盖已有标签）
Map<String, String> newTags = Map.of(
    "status", "archived",
    "retention", "5years"
);
minioClient.setObjectTags(
    SetObjectTagsArgs.builder()
        .bucket("my-bucket")
        .object("photo.jpg")
        .tags(newTags)
        .build()
);

// 删除所有对象标签
minioClient.deleteObjectTags(
    DeleteObjectTagsArgs.builder()
        .bucket("my-bucket")
        .object("photo.jpg")
        .build()
);
```

---

# 13.错误处理

## 13.1常见错误类型

```java
import io.minio.errors.*;

public class MinioErrorHandling {

    public void handleErrors(MinioClient client, String bucket, String object) {
        try {
            client.statObject(StatObjectArgs.builder()
                .bucket(bucket).object(object).build());
        } catch (ErrorResponseException e) {
            // S3 级别的错误响应（服务端返回的错误）
            ErrorResponse errorResponse = e.errorResponse();
            String errorCode = errorResponse.code();
            String errorMessage = errorResponse.message();
            String resource = errorResponse.resource();
            String requestId = errorResponse.requestId();
            int statusCode = e.response().code();

            switch (errorCode) {
                case "NoSuchBucket":
                    System.out.println("Bucket 不存在: " + bucket);
                    break;
                case "NoSuchKey":
                    System.out.println("对象不存在: " + object);
                    break;
                case "AccessDenied":
                    System.out.println("访问被拒绝: 权限不足");
                    break;
                case "BucketAlreadyExists":
                    System.out.println("Bucket 已存在");
                    break;
                case "EntityTooLarge":
                    System.out.println("上传文件超过大小限制");
                    break;
                default:
                    System.out.println("未知错误: " + errorCode + " - " + errorMessage);
            }
        } catch (InsufficientDataException e) {
            // 读取的数据不足（EOF before expected length）
            System.out.println("数据不足: " + e.getMessage());
        } catch (InternalException e) {
            // XML/HTTP 解析错误
            System.out.println("内部错误: " + e.getMessage());
        } catch (InvalidKeyException e) {
            // 无效的 AccessKey/SecretKey
            System.out.println("凭证无效: " + e.getMessage());
        } catch (InvalidResponseException e) {
            // 服务端返回无效响应
            System.out.println("无效响应: " + e.getMessage());
        } catch (IOException e) {
            // 网络连接错误
            System.out.println("网络错误: " + e.getMessage());
        } catch (NoSuchAlgorithmException e) {
            // TLS 加密相关
            System.out.println("加密错误: " + e.getMessage());
        } catch (ServerException e) {
            // 服务端内部错误（500系列）
            System.out.println("服务端错误: " + e.getMessage());
        } catch (XmlParserException e) {
            // XML 解析错误
            System.out.println("XML解析错误: " + e.getMessage());
        } catch (Exception e) {
            System.out.println("未预期错误: " + e.getMessage());
        }
    }
}
```

## 13.2重试机制

```java
public class RetryableMinioOperation {

    private final MinioClient minioClient;
    private final int maxRetries = 3;
    private final long retryDelayMs = 1000;

    public RetryableMinioOperation(MinioClient minioClient) {
        this.minioClient = minioClient;
    }

    public void uploadWithRetry(String bucket, String object, 
                                 InputStream stream, long size) throws Exception {
        int attempt = 0;
        Exception lastException = null;

        while (attempt < maxRetries) {
            try {
                minioClient.putObject(PutObjectArgs.builder()
                    .bucket(bucket).object(object)
                    .stream(stream, size, -1).build());
                return;  // 成功，退出
            } catch (IOException e) {
                // 网络相关错误可重试
                lastException = e;
                attempt++;
                System.out.printf("上传失败 (第%d次重试): %s%n", attempt, e.getMessage());
                Thread.sleep(retryDelayMs * attempt);  // 指数退避
            } catch (ErrorResponseException e) {
                if (e.response().code() >= 500) {
                    // 服务端错误可重试
                    lastException = e;
                    attempt++;
                    Thread.sleep(retryDelayMs * attempt);
                } else {
                    throw e;  // 客户端错误不重试
                }
            }
        }

        throw new RuntimeException("上传重试 " + maxRetries + " 次后仍失败", lastException);
    }
}
```

## 13.3S3 错误码速查表

| 错误码 | HTTP 状态码 | 说明 |
|--------|------------|------|
| `AccessDenied` | 403 | 无权限访问 |
| `AccountProblem` | 403 | 账户问题 |
| `BucketAlreadyExists` | 409 | Bucket 已存在 |
| `BucketNotEmpty` | 409 | Bucket 不为空，无法删除 |
| `EntityTooLarge` | 400 | 上传对象超过大小限制 |
| `EntityTooSmall` | 400 | 上传对象小于最小大小限制 |
| `InternalError` | 500 | 服务端内部错误 |
| `InvalidAccessKeyId` | 403 | AK 无效 |
| `InvalidBucketName` | 400 | Bucket 名称无效 |
| `InvalidDigest` | 400 | Content-MD5 不正确 |
| `InvalidObjectState` | 403 | 对象状态不符合操作要求 |
| `InvalidPart` | 400 | 分片信息无效 |
| `InvalidPartOrder` | 400 | 分片顺序不正确 |
| `MalformedXML` | 400 | XML 格式错误 |
| `MethodNotAllowed` | 405 | HTTP 方法不允许 |
| `MissingContentLength` | 411 | 缺少 Content-Length 头 |
| `NoSuchBucket` | 404 | Bucket 不存在 |
| `NoSuchKey` | 404 | 对象不存在 |
| `NoSuchUpload` | 404 | 指定的分片上传不存在 |
| `PreconditionFailed` | 412 | 前置条件不满足 |
| `RequestTimeout` | 400 | 请求超时 |
| `SignatureDoesNotMatch` | 403 | 请求签名不匹配 |
| `SlowDown` | 503 | 请求频率过高 |
| `XMinioServerNotInitialized` | 503 | 服务端集群未初始化完成 |
