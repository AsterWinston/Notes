# 1. 概述

Feign 支持对 HTTP 请求体和响应体进行 **Gzip 压缩**，减少网络传输数据量，提升传输效率。压缩对调用方和被调用方基本透明。

```text
请求方 (Feign)                           服务方
  │                                        │
  ├── Body 压缩 (Gzip) ──────────────────→ │
  │   Content-Encoding: gzip               │ 自动解压
  │                                        │
  │   Accept-Encoding: gzip                │
  │ ←────────────────── Body 压缩 (Gzip) ──┤
  │ 自动解压                                │
```

## 1.1 "透明"的实现分工：协商在 Feign，干活在引擎

先建立正确的心智模型，§5 的源码才不会看晕：Spring Cloud OpenFeign 的两个压缩开关本质上只是**两个往请求头里写字段的 RequestInterceptor**——决定"要不要压、能不能压"的元数据（Content-Encoding / Accept-Encoding）由 Feign 层写入，**真正的字节压缩与解压由底层 HTTP 引擎完成**（Apache HttpClient 的压缩实体与自动解压，OkHttp 的透明 gzip 机制）。也就是说 `compression.*.enabled` 不是"Feign 内部有个 gzip 流水线"，而是"Feign 负责和引擎谈好参数"。这个分工解释了本篇几乎所有"配了没生效"问题（§10）。

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

一个执行顺序上的关键点（`7_interceptor.md` §8.1 同族问题）：压缩拦截器检查的是 Encoder 编码后的模板，`min-request-size` 比较的是**序列化后的字节数**——判断"我的批量提交会不会被压缩"要按 JSON 大小估算，不是 Java 对象字段数。另外 `mime-types` 匹配的是请求模板上的 `Content-Type`，没声明 consumes 的接口按默认 JSON 处理（`6_encoder-decoder.md` §2）；表单/multipart 类请求天然不在默认压缩名单里——这是特性不是漏配。

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

`useGzipDecoder` 的存在需要解释（§5.2）：默认路径下响应解压由**引擎**完成，Feign 的 `Decoder` 拿到的已是明文；当引擎的透明解压不可依赖（请求里已有显式 Accept-Encoding、或引擎配置禁用了自动解压）时，`useGzipDecoder: true` 让 `GzipDecoder` 包在真实 Decoder 外面自己做解压。两层机制叠加不会双重解压出错（引擎解压后会剥离 `Content-Encoding` 头，GzipDecoder 看不到该头即直通），但排查"响应到底谁在解压"时要记住链路上有引擎、GzipDecoder、真实 Decoder 三层都可能碰这个流。

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

## 5.1 请求压缩协商 (FeignContentGzipEncodingInterceptor)

```java
@Configuration
@ConditionalOnProperty(value = "spring.cloud.openfeign.compression.request.enabled")
public class FeignContentGzipEncodingAutoConfiguration {

    @Bean
    public FeignContentGzipEncodingInterceptor feignContentGzipEncodingInterceptor() {
        return new FeignContentGzipEncodingInterceptor(properties);
    }
}

public class FeignContentGzipEncodingInterceptor implements RequestInterceptor {

    @Override
    public void apply(RequestTemplate template) {
        // 判断 Content-Type 是否匹配配置的 mime-types
        if (matchesMimeTypes(template) && bodyLargeEnough(template)) {
            // 达到阈值：声明"这个 body 是 gzip 的"
            template.header(HttpEncoding.CONTENT_ENCODING_HEADER,
                    HttpEncoding.GZIP_ENCODING);
            // 实际压缩由底层 HTTP 引擎完成
        }
    }
}
```

（简化还原，真实类结构为 `BaseHttpEncodingInterceptor` 派生 + `HttpEncoding` 工具类。）注意它只**声明**"body 是 gzip"：Apache HttpClient 引擎侧根据 Content-Encoding 把请求实体换成压缩实体，才真正压字节。引擎差异要记住——**OkHttp 不会替你压缩"调用方声明了 Content-Encoding"的请求体**（它的透明压缩只管响应方向），Feign+OkHttp 组合下配了 `request.enabled`，可能出现"头写着 gzip、body 仍是明文"、服务端解出乱码的隐蔽故障；此时应改用 §7 的手动压缩（自己压好字节再写头）或换 Apache 引擎（`19_http-client-engines.md`）。

## 5.2 响应压缩协商 (FeignAcceptGzipEncodingInterceptor)

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
// 响应解压由底层 HTTP 客户端自动处理；
// useGzipDecoder=true 时改由 GzipDecoder 包装真实 Decoder 解压
```

它无判断、无条件加头——比 §5.1 简单一个数量级，因为"对方压不压"由服务端看头决定，我方只需声明能力。坑在 OkHttp：它的透明解压前提是"**Accept-Encoding 是我自己隐式添加的**"——一旦请求里已有显式 Accept-Encoding（本拦截器写的），OkHttp 视为你接管编码协商、**不再自动解压也不剥离响应头**，明文解析的 Jackson 会直接在 gzip 字节上抛 `Unexpected end-of-input` / 非法首字符类错误。解法二选一：关闭 `response.enabled` 信任 OkHttp 默认行为，或开启 `useGzipDecoder: true` 让 Feign 侧解压补位——两边都关（既不声明又不解压）才是坏状态。

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

**慎用这段"控制"代码**：手动加 Accept-Encoding 恰恰触发了 §5.2 说的"OkHttp 停止透明解压"——这段示例只在"响应侧另有解压手段（GzipDecoder）"的语境下才成立，否则它就是故障制造器。OkHttp 的正确用法是**什么都不配**：它自己声明、自己解压、解压后剥头，对上层完全透明。

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

事实上 Apache HttpClient 的 `HttpClientBuilder` **默认就注册了**内容解压拦截器（`ContentEncodingInterceptor`，除非 `disableContentCompression()`），这段手写等价物主要用于理解机制；真要改的是行为细节，比如 `decompressingEntity` 对 `deflate` 的支持差异。请求方向则没有默认压缩：Apache 侧要靠 Spring Cloud OpenFeign 引擎适配层对 Content-Encoding 的识别（§5.1）或 §7 手动压缩。

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

这是唯一"**字节我自己压**"的路线（前两条只谈协商），因此它不依赖引擎行为，OkHttp 组合下也能保证正确——§5.1 故障的推荐修复就是它。但有两个必须自补的配套：压完之后 **`Content-Length` 必须按压缩后的字节数重算**（模板 `body(byte[])` 会处理，用 `body(String)` 重载则要当心编码）；重试场景要防**双重压缩**——拦截器每次尝试都执行（`7_interceptor.md` §8.1），第二次进来 `template.body()` 已是 gzip 字节，不加守卫会再压一层，服务端解一层得到另一层 gzip。守卫写法：`template.headers().containsKey("Content-Encoding")` 时直接返回。

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

## 8.1 收益公式：什么时候压缩是负资产

压缩的账 = 传输时间节省 −（两端 CPU 压/解压耗时 + 内存缓冲）。粗算：JSON 压到 1/8，在 100Mbps 链路传 100KB 省约 70ms 网络时间；同量级 Gzip 在两颗现代核上约 3-5ms——值得。把带宽换成内网 10Gbps（100KB 传输不到 1ms），同样 3-5ms 的 CPU 就变成了纯亏；把数据换成 10MB 大响应，网络省的 800ms 又压倒 CPU。**决策变量是"带宽 × 报文大小"而不是"要不要开压缩"这个布尔值**：内网小报文默认关、跨机房/公网大报文默认开、网关到服务这段通常由网关统一处理（§10 的重复压缩问题），逐跳决策才能既省又不赔。

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

**请求体的自动解压是 Servlet 容器/框架层能力，不是默认都有的**：Tomcat 对**响应**按 `server.compression` 压缩是开箱即用，但**解压 gzip 请求体**依赖具体框架（Spring Cloud Gateway、Nginx `gunzip` 模块、或自定义 Filter；裸 Spring MVC 收到 `Content-Encoding: gzip` 的 POST 并不会自动解压，Jackson 会在压缩字节上失败）。调用方开 `request.enabled` 之前，先确认提供方的解压路径真实存在——这是"开了请求压缩后对方 400"的标准成因。

# 10. 常见问题

**Q：开启了压缩配置但没有生效？**

1. 确认 `spring.cloud.openfeign.compression.request.enabled: true`
2. 检查请求体大小是否 >= `min-request-size`
3. 检查 Content-Type 是否在 `mime-types` 列表中
4. 确认服务端也支持 gzip（返回 `Content-Encoding: gzip`）
5. 按 §1.1 分工核对**引擎**：Feign 只写头，请求体压缩依赖 Apache 引擎适配或 §7 手动路线；响应方向 OkHttp 可能因显式 Accept-Encoding 关掉了透明解压（§5.2）

**Q：压缩后请求变大了？**

小请求 Gzip 反而增大（header 开销）。设置 `min-request-size: 2048` 可避免。

**Q：二进制流要压缩吗？**

不要。图片、音视频等已经是压缩格式，再 Gzip 只会增加 CPU 开销且无压缩收益。

**Q：请求经过网关/负载均衡后收到双重压缩或解压错乱？**

逐跳都要看头说话：网关已解压再转发时**必须剥掉或重写 `Content-Encoding` 与 `Content-Length`**，否则下一层对着明文解 gzip。经验规则是"链路上只安排一跳负责压缩协商"——要么端到端透传压缩字节（网关不碰 body），要么网关统一解压/压缩，混着来必然错位；Nginx 侧对应 `proxy_set_header Accept-Encoding ""`（强制上游返回明文由 Nginx 压给客户端）这类单点接管写法。

**Q：压缩和 Feign FULL 日志同时开，日志里是乱码？**

正常现象，级别决定谁看到什么：`logAndRebufferResponse` 在引擎解压**之后**打日志（引擎已还原明文则日志正常）；若 §5.2 场景下解压发生在 GzipDecoder（比日志晚），日志缓冲的就是压缩字节。排障时先关一个，别拿乱码日志定位业务问题。
