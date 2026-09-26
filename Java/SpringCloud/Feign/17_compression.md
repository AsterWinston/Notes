
# 1. 概述

Feign 支持对 HTTP 请求体和响应体进行 **Gzip 压缩**，减少网络传输数据量，提升传输效率。压缩对调用方和被调用方基本透明。

```
请求方 (Feign)                           服务方
  │                                        │
  ├── Body 压缩 (Gzip) ──────────────────→ │
  │   Content-Encoding: gzip               │ 自动解压
  │                                        │
  │   Accept-Encoding: gzip                │
  │ ←────────────────── Body 压缩 (Gzip) ──┤
  │ 自动解压                                │
```

# 2. 配置请求压缩

```yaml
spring:
  cloud:
    openfeign:
      compression:
        request:
          enabled: true
          mime-types: application/json,application/xml,text/plain
          min-request-size: 2048          # 最小压缩阈值 (bytes)
```

**说明**：
- `enabled: true` — 开启请求 Gzip 压缩
- `mime-types` — 对哪些 Content-Type 进行压缩
- `min-request-size` — 只有请求体大小超过此阈值才压缩（小请求压缩得不偿失）

# 3. 配置响应压缩

```yaml
spring:
  cloud:
    openfeign:
      compression:
        response:
          enabled: true
          useGzipDecoder: true             # 使用 Gzip decoder
```

- 请求头会添加 `Accept-Encoding: gzip, deflate`
- 如果服务端返回 `Content-Encoding: gzip`，Feign 自动解压

# 4. 完整压缩配置示例

```yaml
spring:
  cloud:
    openfeign:
      compression:
        request:
          enabled: true
          mime-types: application/json,application/xml
          min-request-size: 2048
        response:
          enabled: true
          useGzipDecoder: true

  # 同时配置底层 HTTP 引擎压缩（可选，针对特定引擎）
  # OkHttp 自带压缩支持
  # Apache HttpClient 也支持压缩
```

# 5. Feign 压缩源码分析

## 5.1 请求压缩 (FeignContentGzipEncodingInterceptor)

```java
@Configuration
@ConditionalOnProperty(value = "spring.cloud.openfeign.compression.request.enabled")
public class FeignAcceptGzipEncodingAutoConfiguration {

    @Bean
    public FeignAcceptGzipEncodingInterceptor feignAcceptGzipEncodingInterceptor() {
        return new FeignAcceptGzipEncodingInterceptor();
    }
}

public class FeignContentGzipEncodingInterceptor implements RequestInterceptor {

    @Override
    public void apply(RequestTemplate template) {
        // 判断 Content-Type 是否匹配配置的 mime-types
        if (shouldCompress(template)) {
            // 判断请求体大小是否 >= min-request-size
            byte[] body = template.body();
            if (body != null && body.length >= minRequestSize) {
                template.header(HttpEncoding.CONTENT_ENCODING_HEADER,
                        HttpEncoding.GZIP_ENCODING, HttpEncoding.DEFLATE_ENCODING);
                // 实际压缩由底层 HTTP 引擎完成
            }
        }
    }
}
```

## 5.2 响应解压 (FeignAcceptGzipEncodingInterceptor)

```java
public class FeignAcceptGzipEncodingInterceptor implements RequestInterceptor {
    @Override
    public void apply(RequestTemplate template) {
        // 添加 Accept-Encoding 头，告知服务端支持压缩
        template.header(HttpEncoding.ACCEPT_ENCODING_HEADER,
                HttpEncoding.GZIP_ENCODING, HttpEncoding.DEFLATE_ENCODING);
    }
}
```

```java
// 响应解压由 FeignContentGzipEncodingAutoConfiguration 配置 GzipDecoder
// 或由底层 HTTP 客户端自动处理
```

# 6. 底层 HTTP 引擎压缩

## 6.1 OkHttp 自动压缩

OkHttp 默认开启了透明 Gzip 压缩，无需额外配置。如果服务端返回 gzip 响应，OkHttp 自动解压。

如果想要控制：

```java
@Bean
public OkHttpClient okHttpClient() {
    return new OkHttpClient.Builder()
            .addInterceptor(chain -> {
                Request request = chain.request().newBuilder()
                        .header("Accept-Encoding", "gzip")
                        .build();
                return chain.proceed(request);
            })
            .build();
}
```

## 6.2 Apache HttpClient 压缩

```java
@Bean
public HttpClient httpClient() {
    return HttpClientBuilder.create()
            .addInterceptorFirst((HttpRequestInterceptor) (request, context) -> {
                if (!request.containsHeader("Accept-Encoding")) {
                    request.addHeader("Accept-Encoding", "gzip");
                }
            })
            .addInterceptorFirst((HttpResponseInterceptor) (response, context) -> {
                if (response.getEntity() != null
                        && response.getEntity().getContentEncoding() != null) {
                    String encoding = response.getEntity().getContentEncoding().getValue();
                    if ("gzip".equalsIgnoreCase(encoding)) {
                        response.setEntity(new GzipDecompressingEntity(response.getEntity()));
                    }
                }
            })
            .build();
}
```

# 7. 手动请求压缩

如果不使用 Feign 内置压缩配置，可以手动在 Interceptor 或 Encoder 中做：

```java
@Component
public class GzipEncodingInterceptor implements RequestInterceptor {

    @Override
    public void apply(RequestTemplate template) {
        byte[] body = template.body();
        if (body != null && body.length > 1024) {
            try {
                ByteArrayOutputStream bos = new ByteArrayOutputStream();
                try (GZIPOutputStream gos = new GZIPOutputStream(bos)) {
                    gos.write(body);
                }
                template.body(bos.toByteArray());
                template.header("Content-Encoding", "gzip");
            } catch (IOException e) {
                throw new RuntimeException("Gzip compress failed", e);
            }
        }
    }
}
```

# 8. 压缩效果与取舍

| 数据类型 | 压缩比 | 示例 |
|----------|--------|------|
| JSON（大量文本） | 5:1 ~ 10:1 | 10KB → 1-2KB |
| XML | 3:1 ~ 8:1 | 10KB → 1.2-3KB |
| 二进制（图片/文件） | 1:1 | 已经压缩，无需再压 |
| 小对象 (< 1KB) | 可能更大 | Gzip header 开销约 20 bytes |

**建议**：
- 请求体 > 2KB 时才开启压缩
- 二进制传输（文件上传）不压缩
- 内网延迟低（< 1ms）场景，压缩收益有限，反而增加 CPU 开销

# 9. 服务端配合

压缩需要**服务端也支持**。对于 Spring Boot 服务端：

```yaml
server:
  compression:
    enabled: true
    mime-types: application/json,application/xml
    min-response-size: 2048
```

服务端收到 `Content-Encoding: gzip` 的请求时会自动解压；根据 `Accept-Encoding: gzip` 决定是否压缩响应。

# 10. 常见问题

**Q：开启了压缩配置但没有生效？**

1. 确认 `spring.cloud.openfeign.compression.request.enabled: true`
2. 检查请求体大小是否 >= `min-request-size`
3. 检查 Content-Type 是否在 `mime-types` 列表中
4. 确认服务端也支持 gzip（返回 `Content-Encoding: gzip`）

**Q：压缩后请求变大了？**

小请求 Gzip 反而增大（header 开销）。设置 `min-request-size: 2048` 可避免。

**Q：二进制流要压缩吗？**

不要。图片、音视频等已经是压缩格式，再 Gzip 只会增加 CPU 开销且无压缩收益。
