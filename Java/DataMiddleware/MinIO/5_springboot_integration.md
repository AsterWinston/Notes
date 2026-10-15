# 1. Spring Boot 整合 MinIO — 基础篇

## 1.1 创建项目与引入依赖

```xml
<parent>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-parent</artifactId>
    <version>3.3.4</version>
</parent>

<properties>
    <java.version>21</java.version>
</properties>

<dependencies>
    <!-- Spring Boot Web -->
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-web</artifactId>
    </dependency>

    <!-- MinIO Java SDK -->
    <dependency>
        <groupId>io.minio</groupId>
        <artifactId>minio</artifactId>
        <version>8.5.12</version>
    </dependency>

    <!-- Lombok -->
    <dependency>
        <groupId>org.projectlombok</groupId>
        <artifactId>lombok</artifactId>
        <optional>true</optional>
    </dependency>

    <!-- 参数校验 -->
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-validation</artifactId>
    </dependency>

    <!-- 测试 -->
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-test</artifactId>
        <scope>test</scope>
    </dependency>
</dependencies>
```

## 1.2 application.yml 配置

```yaml
spring:
  servlet:
    multipart:
      max-file-size: 200MB        # 单个文件最大大小
      max-request-size: 210MB     # 整个请求最大大小
      enabled: true

minio:
  endpoint: http://localhost:9000
  access-key: minioadmin
  secret-key: minioadmin
  bucket-name: my-app-bucket
```

## 1.3 MinioClient 配置类

```java
import io.minio.MinioClient;
import org.springframework.boot.context.properties.ConfigurationProperties;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;

@Configuration
public class MinioConfig {

    @Bean
    @ConfigurationProperties(prefix = "minio")
    public MinioProperties minioProperties() {
        return new MinioProperties();
    }

    @Bean
    public MinioClient minioClient(MinioProperties properties) {
        return MinioClient.builder()
            .endpoint(properties.getEndpoint())
            .credentials(properties.getAccessKey(), properties.getSecretKey())
            .build();
    }
}
```

```java
import lombok.Data;
import org.springframework.boot.context.properties.ConfigurationProperties;

@Data
@ConfigurationProperties(prefix = "minio")
public class MinioProperties {
    private String endpoint;
    private String accessKey;
    private String secretKey;
    private String bucketName;
}
```

---

# 2. 文件服务封装 — 基础版本

## 2.1 通用文件服务接口

```java
public interface FileService {

    /**
     * 上传文件
     * @return 对象的 URL（或 Key）
     */
    String uploadFile(MultipartFile file, String objectName) throws Exception;

    /**
     * 下载文件
     */
    InputStream downloadFile(String objectName) throws Exception;

    /**
     * 删除文件
     */
    void deleteFile(String objectName) throws Exception;

    /**
     * 获取文件访问 URL（预签名）
     */
    String getFileUrl(String objectName, long expiryMinutes) throws Exception;
}
```

## 2.2 接口实现

```java
import io.minio.*;
import io.minio.http.Method;
import org.springframework.stereotype.Service;
import org.springframework.web.multipart.MultipartFile;

import java.io.InputStream;
import java.util.concurrent.TimeUnit;

@Service
public class MinioFileService implements FileService {

    private final MinioClient minioClient;
    private final MinioProperties minioProperties;

    public MinioFileService(MinioClient minioClient, MinioProperties minioProperties) {
        this.minioClient = minioClient;
        this.minioProperties = minioProperties;
        // 启动时确保 Bucket 存在
        createBucketIfNotExists();
    }

    private void createBucketIfNotExists() {
        try {
            String bucket = minioProperties.getBucketName();
            boolean exists = minioClient.bucketExists(
                BucketExistsArgs.builder().bucket(bucket).build()
            );
            if (!exists) {
                minioClient.makeBucket(MakeBucketArgs.builder().bucket(bucket).build());
            }
        } catch (Exception e) {
            throw new RuntimeException("初始化 MinIO Bucket 失败", e);
        }
    }

    @Override
    public String uploadFile(MultipartFile file, String objectName) throws Exception {
        minioClient.putObject(
            PutObjectArgs.builder()
                .bucket(minioProperties.getBucketName())
                .object(objectName)
                .stream(file.getInputStream(), file.getSize(), -1)
                .contentType(file.getContentType())
                .build()
        );
        return objectName;
    }

    @Override
    public InputStream downloadFile(String objectName) throws Exception {
        return minioClient.getObject(
            GetObjectArgs.builder()
                .bucket(minioProperties.getBucketName())
                .object(objectName)
                .build()
        );
    }

    @Override
    public void deleteFile(String objectName) throws Exception {
        minioClient.removeObject(
            RemoveObjectArgs.builder()
                .bucket(minioProperties.getBucketName())
                .object(objectName)
                .build()
        );
    }

    @Override
    public String getFileUrl(String objectName, long expiryMinutes) throws Exception {
        return minioClient.getPresignedObjectUrl(
            GetPresignedObjectUrlArgs.builder()
                .method(Method.GET)
                .bucket(minioProperties.getBucketName())
                .object(objectName)
                .expiry(expiryMinutes, TimeUnit.MINUTES)
                .build()
        );
    }
}
```

## 2.3 文件上传 Controller

```java
import org.springframework.http.ResponseEntity;
import org.springframework.web.bind.annotation.*;
import org.springframework.web.multipart.MultipartFile;

import java.util.Map;

@RestController
@RequestMapping("/api/files")
public class FileController {

    private final FileService fileService;

    public FileController(FileService fileService) {
        this.fileService = fileService;
    }

    @PostMapping("/upload")
    public ResponseEntity<Map<String, String>> upload(@RequestParam("file") MultipartFile file) {
        try {
            String originalFilename = file.getOriginalFilename();
            String objectName = "uploads/" + System.currentTimeMillis() + "_" + originalFilename;
            String result = fileService.uploadFile(file, objectName);
            return ResponseEntity.ok(Map.of("objectName", result));
        } catch (Exception e) {
            return ResponseEntity.internalServerError()
                .body(Map.of("error", e.getMessage()));
        }
    }

    @GetMapping("/download/{objectName}")
    public ResponseEntity<byte[]> download(@PathVariable String objectName) {
        try {
            InputStream inputStream = fileService.downloadFile(objectName);
            byte[] data = inputStream.readAllBytes();
            return ResponseEntity.ok()
                .header("Content-Disposition", "attachment; filename=\"" + objectName + "\"")
                .body(data);
        } catch (Exception e) {
            return ResponseEntity.internalServerError().build();
        }
    }

    @DeleteMapping("/{objectName}")
    public ResponseEntity<Map<String, String>> delete(@PathVariable String objectName) {
        try {
            fileService.deleteFile(objectName);
            return ResponseEntity.ok(Map.of("message", "删除成功"));
        } catch (Exception e) {
            return ResponseEntity.internalServerError()
                .body(Map.of("error", e.getMessage()));
        }
    }

    @GetMapping("/url/{objectName}")
    public ResponseEntity<Map<String, String>> getUrl(@PathVariable String objectName) {
        try {
            String url = fileService.getFileUrl(objectName, 60);
            return ResponseEntity.ok(Map.of("url", url));
        } catch (Exception e) {
            return ResponseEntity.internalServerError().build();
        }
    }
}
```

---

# 3. 增强型文件服务 — 生产级别

## 3.1 对象 Key 命名策略

```java
import java.time.LocalDate;
import java.time.format.DateTimeFormatter;
import java.util.UUID;

public class ObjectKeyGenerator {

    /**
     * 生成按日期分组的对象 Key
     * 示例: files/2024/01/15/a1b2c3d4-avatar.jpg
     */
    public static String generateDateBased(String prefix, String originalFilename) {
        String dateDir = LocalDate.now().format(DateTimeFormatter.ISO_LOCAL_DATE);
        String ext = getExtension(originalFilename);
        String uuid = UUID.randomUUID().toString().substring(0, 8);
        return String.format("%s/%s/%s.%s", prefix, dateDir, uuid, ext);
    }

    /**
     * 生成按用户分组的对象 Key
     * 示例: user/1001/avatars/a1b2c3d4.png
     */
    public static String generateUserBased(String userId, String category, String originalFilename) {
        String ext = getExtension(originalFilename);
        String uuid = UUID.randomUUID().toString().substring(0, 8);
        return String.format("user/%s/%s/%s.%s", userId, category, uuid, ext);
    }

    private static String getExtension(String filename) {
        int lastDot = filename.lastIndexOf('.');
        return lastDot > 0 ? filename.substring(lastDot + 1) : "bin";
    }
}
```

## 3.2 增强型文件服务

```java
import io.minio.*;
import io.minio.http.Method;
import io.minio.messages.DeleteObject;
import lombok.RequiredArgsConstructor;
import lombok.extern.slf4j.Slf4j;
import org.springframework.stereotype.Service;
import org.springframework.web.multipart.MultipartFile;

import java.io.*;
import java.security.MessageDigest;
import java.util.*;
import java.util.concurrent.TimeUnit;

@Slf4j
@Service
@RequiredArgsConstructor
public class EnhancedFileService {

    private final MinioClient minioClient;
    private final MinioProperties minioProperties;

    /**
     * 上传文件（带类型检查）
     */
    public UploadResult uploadFile(MultipartFile file, String objectName,
                                    List<String> allowedContentTypes) throws Exception {
        // 类型检查
        if (allowedContentTypes != null && !allowedContentTypes.isEmpty()) {
            String contentType = file.getContentType();
            if (!allowedContentTypes.contains(contentType)) {
                throw new IllegalArgumentException("不允许的文件类型: " + contentType);
            }
        }

        // 上传
        minioClient.putObject(
            PutObjectArgs.builder()
                .bucket(minioProperties.getBucketName())
                .object(objectName)
                .stream(file.getInputStream(), file.getSize(), -1)
                .contentType(file.getContentType())
                .userMetadata(Map.of(
                    "original-filename", file.getOriginalFilename(),
                    "upload-time", String.valueOf(System.currentTimeMillis()),
                    "file-size", String.valueOf(file.getSize())
                ))
                .build()
        );

        return UploadResult.builder()
            .objectName(objectName)
            .originalName(file.getOriginalFilename())
            .size(file.getSize())
            .contentType(file.getContentType())
            .build();
    }

    /**
     * 上传文件并返回可访问的 URL
     */
    public UploadResult uploadAndGetUrl(MultipartFile file, String objectName) throws Exception {
        UploadResult result = uploadFile(file, objectName, null);
        result.setUrl(getPresignedUrl(objectName, 24, TimeUnit.HOURS));
        return result;
    }

    /**
     * 获取预签名 URL
     */
    public String getPresignedUrl(String objectName, long duration, TimeUnit unit) 
            throws Exception {
        return minioClient.getPresignedObjectUrl(
            GetPresignedObjectUrlArgs.builder()
                .method(Method.GET)
                .bucket(minioProperties.getBucketName())
                .object(objectName)
                .expiry(Math.toIntExact(unit.toSeconds(duration)), TimeUnit.SECONDS)
                .build()
        );
    }

    /**
     * 批量删除
     */
    public int deleteFiles(List<String> objectNames) throws Exception {
        List<DeleteObject> objects = objectNames.stream()
            .map(DeleteObject::new)
            .toList();

        int deletedCount = 0;
        Iterable<Result<DeleteError>> results = minioClient.removeObjects(
            RemoveObjectsArgs.builder()
                .bucket(minioProperties.getBucketName())
                .objects(objects)
                .build()
        );

        for (Result<DeleteError> result : results) {
            try {
                result.get();  // 触发实际执行
                deletedCount++;
            } catch (Exception e) {
                log.error("删除失败: {}", e.getMessage());
            }
        }
        return deletedCount;
    }

    /**
     * 下载文件到本地
     */
    public void downloadToFile(String objectName, File targetFile) throws Exception {
        minioClient.downloadObject(
            DownloadObjectArgs.builder()
                .bucket(minioProperties.getBucketName())
                .object(objectName)
                .filename(targetFile.getAbsolutePath())
                .overwrite(true)
                .build()
        );
    }

    /**
     * 获取对象信息
     */
    public ObjectInfo getObjectInfo(String objectName) throws Exception {
        StatObjectResponse stat = minioClient.statObject(
            StatObjectArgs.builder()
                .bucket(minioProperties.getBucketName())
                .object(objectName)
                .build()
        );

        return ObjectInfo.builder()
            .objectName(objectName)
            .size(stat.size())
            .etag(stat.etag())
            .contentType(stat.contentType())
            .lastModified(stat.lastModified())
            .userMetadata(stat.userMetadata())
            .build();
    }
}
```

## 3.3 DTO 定义

```java
import lombok.Builder;
import lombok.Data;
import java.util.Map;

@Data
@Builder
public class UploadResult {
    private String objectName;
    private String originalName;
    private long size;
    private String contentType;
    private String url;
}

@Data
@Builder
public class ObjectInfo {
    private String objectName;
    private long size;
    private String etag;
    private String contentType;
    private ZonedDateTime lastModified;
    private Map<String, String> userMetadata;
}
```

## 3.4 增强型 Controller

```java
import lombok.RequiredArgsConstructor;
import org.springframework.http.ResponseEntity;
import org.springframework.web.bind.annotation.*;
import org.springframework.web.multipart.MultipartFile;

import java.util.*;

@RestController
@RequestMapping("/api/v2/files")
@RequiredArgsConstructor
public class EnhancedFileController {

    private final EnhancedFileService fileService;

    /**
     * 上传单个文件
     */
    @PostMapping("/upload")
    public ResponseEntity<UploadResult> upload(
            @RequestParam("file") MultipartFile file,
            @RequestParam(value = "category", defaultValue = "common") String category) {

        String objectName = ObjectKeyGenerator.generateDateBased(category,
            file.getOriginalFilename());

        List<String> allowedTypes = List.of("image/jpeg", "image/png", "image/gif",
            "image/webp", "application/pdf");
        UploadResult result = fileService.uploadFile(file, objectName, allowedTypes);
        result.setUrl(fileService.getPresignedUrl(objectName, 12, TimeUnit.HOURS));

        return ResponseEntity.ok(result);
    }

    /**
     * 批量上传
     */
    @PostMapping("/upload/batch")
    public ResponseEntity<List<UploadResult>> uploadBatch(
            @RequestParam("files") MultipartFile[] files,
            @RequestParam(value = "category", defaultValue = "common") String category) {

        List<UploadResult> results = new ArrayList<>();
        for (MultipartFile file : files) {
            String objectName = ObjectKeyGenerator.generateDateBased(category,
                file.getOriginalFilename());
            results.add(fileService.uploadFile(file, objectName, null));
        }
        return ResponseEntity.ok(results);
    }

    /**
     * 获取文件信息
     */
    @GetMapping("/info/{objectName}")
    public ResponseEntity<ObjectInfo> getInfo(@PathVariable String objectName) {
        return ResponseEntity.ok(fileService.getObjectInfo(objectName));
    }

    /**
     * 获取文件列表（分页）
     */
    @GetMapping("/list")
    public ResponseEntity<List<Map<String, Object>>> list(
            @RequestParam(value = "prefix", defaultValue = "") String prefix,
            @RequestParam(value = "maxKeys", defaultValue = "50") int maxKeys,
            @RequestParam(value = "startAfter", required = false) String startAfter) {

        List<Map<String, Object>> items = new ArrayList<>();
        Iterable<Result<Item>> results = minioClient.listObjects(
            ListObjectsArgs.builder()
                .bucket(minioProperties.getBucketName())
                .prefix(prefix)
                .maxKeys(maxKeys)
                .startAfter(startAfter != null ? startAfter : "")
                .build()
        );

        for (Result<Item> result : results) {
            Item item = result.get();
            if (!item.isDir()) {
                Map<String, Object> map = new LinkedHashMap<>();
                map.put("objectName", item.objectName());
                map.put("size", item.size());
                map.put("lastModified", item.lastModified());
                map.put("etag", item.etag());
                items.add(map);
            }
        }
        return ResponseEntity.ok(items);
    }

    /**
     * 批量删除
     */
    @DeleteMapping("/batch")
    public ResponseEntity<Map<String, Object>> deleteBatch(
            @RequestBody List<String> objectNames) {
        int count = fileService.deleteFiles(objectNames);
        return ResponseEntity.ok(Map.of("deleted", count, "total", objectNames.size()));
    }
}
```

---

# 4. 大文件分片上传 — 前后端完整方案

## 4.1 后端：分片上传 API

```java
import io.minio.*;
import io.minio.messages.Part;
import lombok.RequiredArgsConstructor;
import lombok.extern.slf4j.Slf4j;
import org.springframework.http.ResponseEntity;
import org.springframework.web.bind.annotation.*;

import java.util.*;
import java.util.concurrent.ConcurrentHashMap;

@Slf4j
@RestController
@RequestMapping("/api/chunk")
@RequiredArgsConstructor
public class ChunkUploadController {

    private final MinioClient minioClient;
    private final MinioProperties minioProperties;

    /**
     * 步骤1：初始化分片上传
     */
    @PostMapping("/init")
    public ResponseEntity<Map<String, String>> initUpload(
            @RequestParam String objectName,
            @RequestParam(required = false) String contentType) {

        try {
            Map<String, String> userMetadata = new HashMap<>();
            userMetadata.put("upload-start-time", String.valueOf(System.currentTimeMillis()));

            CreateMultipartUploadResponse response = minioClient.createMultipartUpload(
                minioProperties.getBucketName(),
                null, objectName, userMetadata, null
            );
            return ResponseEntity.ok(Map.of("uploadId", response.result().uploadId()));
        } catch (Exception e) {
            log.error("初始化分片上传失败", e);
            return ResponseEntity.internalServerError().build();
        }
    }

    /**
     * 步骤2：上传单个分片
     */
    @PostMapping("/upload")
    public ResponseEntity<Map<String, String>> uploadPart(
            @RequestParam String objectName,
            @RequestParam String uploadId,
            @RequestParam int partNumber,
            @RequestParam byte[] data) {

        try {
            UploadPartResponse response = minioClient.uploadPart(
                minioProperties.getBucketName(),
                null, objectName,
                data, data.length,
                uploadId, partNumber, null, null
            );
            return ResponseEntity.ok(Map.of("etag", response.etag()));
        } catch (Exception e) {
            log.error("上传分片失败: part {}", partNumber, e);
            return ResponseEntity.internalServerError().build();
        }
    }

    /**
     * 步骤3：完成分片上传
     */
    @PostMapping("/complete")
    public ResponseEntity<Map<String, String>> completeUpload(
            @RequestParam String objectName,
            @RequestParam String uploadId,
            @RequestBody List<PartInfo> parts) {

        try {
            Part[] partArray = parts.stream()
                .map(p -> new Part(p.partNumber, p.etag))
                .toArray(Part[]::new);

            minioClient.completeMultipartUpload(
                minioProperties.getBucketName(),
                null, objectName, uploadId,
                partArray, null, null
            );
            return ResponseEntity.ok(Map.of("message", "上传完成"));
        } catch (Exception e) {
            log.error("完成分片上传失败", e);
            return ResponseEntity.internalServerError().build();
        }
    }

    /**
     * 列举已上传的分片（用于断点续传）
     */
    @GetMapping("/parts")
    public ResponseEntity<List<PartInfo>> listParts(
            @RequestParam String objectName,
            @RequestParam String uploadId) {

        try {
            ListPartsResponse response = minioClient.listParts(
                minioProperties.getBucketName(),
                null, objectName, 1000, 0, uploadId, null, null
            );
            List<PartInfo> parts = response.result().partList().stream()
                .map(p -> new PartInfo(p.partNumber(), p.etag(), p.partSize()))
                .toList();
            return ResponseEntity.ok(parts);
        } catch (Exception e) {
            log.error("列举分片失败", e);
            return ResponseEntity.internalServerError().build();
        }
    }

    /**
     * 取消分片上传
     */
    @PostMapping("/abort")
    public ResponseEntity<Map<String, String>> abortUpload(
            @RequestParam String objectName,
            @RequestParam String uploadId) {

        try {
            minioClient.abortMultipartUpload(
                minioProperties.getBucketName(),
                null, objectName, uploadId, null, null
            );
            return ResponseEntity.ok(Map.of("message", "已取消"));
        } catch (Exception e) {
            log.error("取消上传失败", e);
            return ResponseEntity.internalServerError().build();
        }
    }

    // 分片信息 DTO
    public record PartInfo(int partNumber, String etag, long size) {}
}
```

## 4.2 前端：分片上传实现（JavaScript）

```javascript
const CHUNK_SIZE = 5 * 1024 * 1024; // 每片 5MB

class ChunkUploader {
    constructor(file) {
        this.file = file;
        this.objectName = `uploads/${Date.now()}_${file.name}`;
        this.uploadId = null;
        this.uploadedParts = [];
    }

    // 计算文件 MD5（用于完整性校验）
    async calculateMD5(file) {
        const buffer = await file.arrayBuffer();
        const hashBuffer = await crypto.subtle.digest('SHA-256', buffer);
        const hashArray = Array.from(new Uint8Array(hashBuffer));
        return hashArray.map(b => b.toString(16).padStart(2, '0')).join('');
    }

    // 开始分片上传
    async start() {
        // 1. 初始化
        const initResp = await fetch('/api/chunk/init', {
            method: 'POST',
            headers: { 'Content-Type': 'application/x-www-form-urlencoded' },
            body: new URLSearchParams({
                objectName: this.objectName,
                contentType: this.file.type
            })
        });
        const { uploadId } = await initResp.json();
        this.uploadId = uploadId;

        // 2. 分片上传
        const totalParts = Math.ceil(this.file.size / CHUNK_SIZE);
        for (let i = 0; i < totalParts; i++) {
            const start = i * CHUNK_SIZE;
            const end = Math.min(start + CHUNK_SIZE, this.file.size);
            const chunk = this.file.slice(start, end);
            const data = await chunk.arrayBuffer();

            const formData = new FormData();
            formData.append('objectName', this.objectName);
            formData.append('uploadId', this.uploadId);
            formData.append('partNumber', i + 1);
            formData.append('data', new Blob([data]));

            const resp = await fetch('/api/chunk/upload', {
                method: 'POST',
                body: formData
            });
            const { etag } = await resp.json();
            this.uploadedParts.push({ partNumber: i + 1, etag });

            this.onProgress((i + 1) / totalParts * 100);
        }

        // 3. 完成
        await fetch('/api/chunk/complete', {
            method: 'POST',
            headers: { 'Content-Type': 'application/json' },
            body: JSON.stringify({
                objectName: this.objectName,
                uploadId: this.uploadId,
                parts: this.uploadedParts
            })
        });

        return this.objectName;
    }

    // 进度回调
    onProgress(percent) {
        console.log(`上传进度: ${percent.toFixed(1)}%`);
    }
}

// 使用示例
document.getElementById('fileInput').addEventListener('change', async (e) => {
    const file = e.target.files[0];
    const uploader = new ChunkUploader(file);
    uploader.onProgress = (p) => {
        document.getElementById('progress').style.width = p + '%';
    };
    const objectName = await uploader.start();
    console.log('上传完成:', objectName);
});
```

---

# 5. 图片处理与缩略图方案

## 5.1 图片上传 + 缩略图生成（服务端处理）

```java
import org.springframework.scheduling.annotation.Async;
import org.springframework.stereotype.Service;
import javax.imageio.ImageIO;
import java.awt.*;
import java.awt.image.BufferedImage;
import java.io.*;

@Slf4j
@Service
@RequiredArgsConstructor
public class ImageService {

    private final MinioClient minioClient;
    private final MinioProperties minioProperties;

    /**
     * 上传原图并异步生成缩略图
     */
    public UploadResult uploadImage(MultipartFile file, String category) throws Exception {
        // 1. 上传原图
        String originalName = ObjectKeyGenerator.generateDateBased(category,
            file.getOriginalFilename());
        minioClient.putObject(PutObjectArgs.builder()
            .bucket(minioProperties.getBucketName())
            .object(originalName)
            .stream(file.getInputStream(), file.getSize(), -1)
            .contentType(file.getContentType())
            .build());

        // 2. 异步生成缩略图
        generateThumbnailAsync(file.getInputStream(), file.getOriginalFilename(),
            originalName, category);

        return UploadResult.builder()
            .objectName(originalName)
            .originalName(file.getOriginalFilename())
            .size(file.getSize())
            .build();
    }

    @Async
    public void generateThumbnailAsync(InputStream inputStream, String originalFile,
                                        String originalObjectName, String category) {
        try {
            // 复制流（因为原流已被消费）
            byte[] data = inputStream.readAllBytes();

            // 生成缩略图
            BufferedImage original = ImageIO.read(new ByteArrayInputStream(data));
            int thumbWidth = 300;
            int thumbHeight = (int) ((double) thumbWidth / original.getWidth() * original.getHeight());

            BufferedImage thumbnail = new BufferedImage(thumbWidth, thumbHeight,
                BufferedImage.TYPE_INT_RGB);
            Graphics2D g2d = thumbnail.createGraphics();
            g2d.setRenderingHint(RenderingHints.KEY_INTERPOLATION,
                RenderingHints.VALUE_INTERPOLATION_BILINEAR);
            g2d.drawImage(original, 0, 0, thumbWidth, thumbHeight, null);
            g2d.dispose();

            // 输出缩略图到字节数组
            ByteArrayOutputStream baos = new ByteArrayOutputStream();
            String formatName = getImageFormat(originalFile);
            ImageIO.write(thumbnail, formatName, baos);
            byte[] thumbBytes = baos.toByteArray();

            // 上传缩略图（thumb/ 前缀区分）
            String thumbObjectName = "thumb/" + originalObjectName;
            minioClient.putObject(PutObjectArgs.builder()
                .bucket(minioProperties.getBucketName())
                .object(thumbObjectName)
                .stream(new ByteArrayInputStream(thumbBytes), thumbBytes.length, -1)
                .contentType("image/" + formatName)
                .build());

            log.info("缩略图生成完成: {}", thumbObjectName);
        } catch (Exception e) {
            log.error("生成缩略图失败", e);
        }
    }

    private String getImageFormat(String filename) {
        String name = filename.toLowerCase();
        if (name.endsWith(".png")) return "png";
        if (name.endsWith(".gif")) return "gif";
        if (name.endsWith(".webp")) return "webp";
        if (name.endsWith(".bmp")) return "bmp";
        return "jpg";
    }
}
```

## 5.2 缩略图访问 Controller

```java
@RestController
@RequestMapping("/api/images")
@RequiredArgsConstructor
public class ImageController {

    private final MinioClient minioClient;
    private final MinioProperties minioProperties;

    /**
     * 获取带水印/尺寸参数的图片 URL
     * 注：MinIO 本身不处理图片，需要配合图片处理服务（如 imgproxy）或 Nginx image_filter
     */
    @GetMapping("/{objectName}")
    public ResponseEntity<String> getImageUrl(
            @PathVariable String objectName,
            @RequestParam(defaultValue = "original") String size) {

        String targetObject = "original".equals(size)
            ? objectName
            : "thumb/" + objectName;

        try {
            String url = minioClient.getPresignedObjectUrl(
                GetPresignedObjectUrlArgs.builder()
                    .method(Method.GET)
                    .bucket(minioProperties.getBucketName())
                    .object(targetObject)
                    .expiry(1, TimeUnit.HOURS)
                    .build()
            );
            return ResponseEntity.ok(url);
        } catch (Exception e) {
            return ResponseEntity.internalServerError().build();
        }
    }
}
```

---

# 6. 文件访问控制与防盗链

## 6.1 基于预签名 URL 的访问控制

```java
@Service
@RequiredArgsConstructor
public class SecureFileService {

    private final MinioClient minioClient;
    private final MinioProperties minioProperties;

    /**
     * 生成带权限检查的访问 URL
     * 只有登录用户且具有相应权限才能获取 URL
     */
    public String getSecuredFileUrl(String objectName, String userId,
                                     List<String> userRoles, long expirySeconds) {
        // 1. 权限检查（示例逻辑）
        if (!hasAccess(userId, userRoles, objectName)) {
            throw new AccessDeniedException("无权访问该文件");
        }

        // 2. 生成短期预签名 URL
        try {
            return minioClient.getPresignedObjectUrl(
                GetPresignedObjectUrlArgs.builder()
                    .method(Method.GET)
                    .bucket(minioProperties.getBucketName())
                    .object(objectName)
                    .expiry((int) expirySeconds, TimeUnit.SECONDS)
                    .build()
            );
        } catch (Exception e) {
            throw new RuntimeException("生成访问链接失败", e);
        }
    }

    private boolean hasAccess(String userId, List<String> roles, String objectName) {
        // 私有文件：只允许上传者或管理员访问
        if (objectName.startsWith("private/")) {
            String ownerId = extractOwnerId(objectName);
            return userId.equals(ownerId) || roles.contains("ADMIN");
        }
        // 公共文件：所有登录用户可访问
        if (objectName.startsWith("public/")) {
            return true;
        }
        return false;
    }

    private String extractOwnerId(String objectName) {
        // 从 objectName 中提取用户 ID
        // 例如: private/1001/documents/file.pdf → 1001
        String[] parts = objectName.split("/");
        return parts.length > 1 ? parts[1] : null;
    }
}
```

---

# 7. 文件上传完整校验

```java
import org.springframework.stereotype.Component;
import org.springframework.web.multipart.MultipartFile;

import java.security.MessageDigest;
import java.util.Arrays;
import java.util.HashSet;
import java.util.Set;

@Component
public class FileValidator {

    // 允许的文件类型
    private static final Set<String> ALLOWED_CONTENT_TYPES = new HashSet<>(Arrays.asList(
        "image/jpeg", "image/png", "image/gif", "image/webp",
        "application/pdf",
        "text/plain",
        "application/msword",
        "application/vnd.openxmlformats-officedocument.wordprocessingml.document",
        "application/vnd.ms-excel",
        "application/vnd.openxmlformats-officedocument.spreadsheetml.sheet"
    ));

    // 最大文件大小（100MB）
    private static final long MAX_FILE_SIZE = 100 * 1024 * 1024;

    // 禁止的文件扩展名（防脚本上传）
    private static final Set<String> FORBIDDEN_EXTENSIONS = new HashSet<>(Arrays.asList(
        "exe", "bat", "sh", "cmd", "com", "msi", "dll",
        "php", "jsp", "asp", "aspx", "jspx",
        "py", "rb", "pl", "cgi"
    ));

    public void validate(MultipartFile file) {
        // 1. 空文件检查
        if (file.isEmpty()) {
            throw new IllegalArgumentException("文件不能为空");
        }

        // 2. 大小检查
        if (file.getSize() > MAX_FILE_SIZE) {
            throw new IllegalArgumentException("文件大小不能超过 100MB");
        }

        // 3. Content-Type 检查
        String contentType = file.getContentType();
        if (contentType == null || !ALLOWED_CONTENT_TYPES.contains(contentType)) {
            throw new IllegalArgumentException("不支持的文件类型: " + contentType);
        }

        // 4. 扩展名检查
        String originalFilename = file.getOriginalFilename();
        if (originalFilename != null) {
            String extension = originalFilename.substring(
                originalFilename.lastIndexOf('.') + 1).toLowerCase();
            if (FORBIDDEN_EXTENSIONS.contains(extension)) {
                throw new IllegalArgumentException("禁止的文件扩展名: " + extension);
            }
        }

        // 5. 文件魔数检查（推荐：检查文件头部字节而非依赖 Content-Type）
        // validateMagicNumber(file);
    }

    // 可选的魔数校验（用于更严格的安全控制）
    @SuppressWarnings("unused")
    private void validateMagicNumber(MultipartFile file) throws Exception {
        byte[] header = new byte[8];
        file.getInputStream().read(header);

        String contentType = file.getContentType();
        if ("image/jpeg".equals(contentType)) {
            // JPEG: FF D8 FF
            if ((header[0] & 0xFF) != 0xFF || (header[1] & 0xFF) != 0xD8) {
                throw new IllegalArgumentException("文件不是有效的 JPEG 格式");
            }
        } else if ("image/png".equals(contentType)) {
            // PNG: 89 50 4E 47
            if (header[0] != (byte) 0x89 || header[1] != 'P' || header[2] != 'N' || header[3] != 'G') {
                throw new IllegalArgumentException("文件不是有效的 PNG 格式");
            }
        }
    }
}
```

---

# 8. 全局异常处理

```java
import io.minio.errors.ErrorResponseException;
import lombok.extern.slf4j.Slf4j;
import org.springframework.http.HttpStatus;
import org.springframework.http.ResponseEntity;
import org.springframework.web.bind.annotation.ExceptionHandler;
import org.springframework.web.bind.annotation.RestControllerAdvice;
import org.springframework.web.multipart.MaxUploadSizeExceededException;

import java.util.Map;

@Slf4j
@RestControllerAdvice
public class GlobalExceptionHandler {

    @ExceptionHandler(IllegalArgumentException.class)
    public ResponseEntity<Map<String, String>> handleIllegalArgument(IllegalArgumentException e) {
        return ResponseEntity.badRequest()
            .body(Map.of("error", e.getMessage()));
    }

    @ExceptionHandler(MaxUploadSizeExceededException.class)
    public ResponseEntity<Map<String, String>> handleMaxUploadSize(MaxUploadSizeExceededException e) {
        return ResponseEntity.status(HttpStatus.PAYLOAD_TOO_LARGE)
            .body(Map.of("error", "上传文件超过大小限制"));
    }

    @ExceptionHandler(ErrorResponseException.class)
    public ResponseEntity<Map<String, String>> handleMinioError(ErrorResponseException e) {
        String errorCode = e.errorResponse().code();
        log.error("MinIO 错误: {} - {}", errorCode, e.errorResponse().message());

        HttpStatus status = switch (errorCode) {
            case "NoSuchBucket", "NoSuchKey" -> HttpStatus.NOT_FOUND;
            case "AccessDenied" -> HttpStatus.FORBIDDEN;
            case "EntityTooLarge" -> HttpStatus.PAYLOAD_TOO_LARGE;
            default -> HttpStatus.INTERNAL_SERVER_ERROR;
        };

        return ResponseEntity.status(status)
            .body(Map.of("error", e.errorResponse().message()));
    }

    @ExceptionHandler(Exception.class)
    public ResponseEntity<Map<String, String>> handleGeneral(Exception e) {
        log.error("未预期的错误", e);
        return ResponseEntity.internalServerError()
            .body(Map.of("error", "服务器内部错误"));
    }
}
```

---

# 9. 集成测试

```java
import org.junit.jupiter.api.*;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.boot.test.autoconfigure.web.servlet.AutoConfigureMockMvc;
import org.springframework.boot.test.context.SpringBootTest;
import org.springframework.mock.web.MockMultipartFile;
import org.springframework.test.web.servlet.MockMvc;

import static org.springframework.test.web.servlet.request.MockMvcRequestBuilders.*;
import static org.springframework.test.web.servlet.result.MockMvcResultMatchers.*;

@SpringBootTest
@AutoConfigureMockMvc
class FileControllerTest {

    @Autowired
    private MockMvc mockMvc;

    @Test
    void testUploadAndGetInfo() throws Exception {
        // 上传文件
        MockMultipartFile file = new MockMultipartFile(
            "file", "test.jpg", "image/jpeg",
            "test image content".getBytes()
        );

        String response = mockMvc.perform(multipart("/api/files/upload").file(file))
            .andExpect(status().isOk())
            .andReturn().getResponse().getContentAsString();
        // 提取 objectName 并验证后续操作
    }

    @Test
    void testUploadEmptyFile_ShouldReturnError() throws Exception {
        MockMultipartFile emptyFile = new MockMultipartFile(
            "file", "empty.jpg", "image/jpeg", new byte[0]
        );

        mockMvc.perform(multipart("/api/files/upload").file(emptyFile))
            .andExpect(status().isBadRequest());
    }
}
```
