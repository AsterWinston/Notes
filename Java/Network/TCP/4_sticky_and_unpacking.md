写完 NIO 服务端后，几乎所有人都会撞上同一个问题：客户端明明发了两条消息，服务端有时一次收到两条，有时只收到半条。这不是 bug，而是 TCP 字节流的本质。本篇讲清粘包与拆包的成因，给出定长、分隔符、长度字段三种划界方案，并用 `ByteBuffer` 手写一个能复用的解码器。

# 1. 现象与成因

## 1.1 TCP 是字节流，没有消息边界

TCP 协议只承诺"字节按序到达"，不承诺"一次 `read()` 对应一条消息"。发送方调用两次 `write()`，内核可能合并成一个报文发出；接收方一次 `read()` 可能读到一条、两条、半条，甚至一条半。

```text
发送方应用            TCP 层                接收方应用
┌──────────┐      ┌──────────┐          ┌──────────┐
│ write(M1)│ ───> │ 发送缓冲区 │ ──────> │ read()   │
│ write(M2)│ ───> │ 合并成报文 │ ──────> │          │
└──────────┘      └──────────┘          └──────────┘
```

按接收结果分类：

| 现象 | 表现 | 原因 |
|------|------|------|
| 粘包 | 一次读到多条消息 | 发送方合并写、Nagle 算法、接收缓冲区一次取走多段 |
| 拆包 | 一条消息分多次读到 | 消息大于 MSS、发送缓冲区边界、读取缓冲区偏小 |
| 粘拆混合 | 半条 + 一条 + 半条 | 上述情况叠加，真实场景最常见 |

**注意**："粘包"这个词容易误导，好像 TCP 做错了什么。准确说法是**应用层没有定义消息边界**，TCP 只是忠实地传递字节流。解决方式只能是在应用层加边界信息，调 `TCP_NODELAY` 只能减少合并，无法消除。

## 1.2 产生合并与拆分的位置

数据从应用到网络要经过多层缓冲，任何一层都可能改变"一次调用"与"一个报文"的对应关系：

- **应用层**：`BufferedWriter` 攒够一批才交给内核。
- **传输层**：Nagle 算法把多个小包合并，直到收到 ACK 或凑够 MSS。
- **网络层**：超过 MTU 的 IP 包会被分片。
- **接收侧**：内核接收缓冲区把连续到达的字节放在一起，`read()` 按请求长度交付。

**设计考量**：Nagle 算法本意是减少小包、提高带宽利用率，但对延迟敏感的场景（游戏、RPC）有害，通常配 `TCP_NODELAY` 关闭。即便如此，边界问题依然存在，因为接收端的读取长度天然与发送端的写入长度解耦。

## 1.3 为什么 HTTP 没这个问题

HTTP/1.1 用 `Content-Length` 或 `Transfer-Encoding: chunked` 明确消息长度，头部以空行结束。这就是"长度字段"加"分隔符"的组合。任何自定义协议都必须自己定义类似的规则。

# 2. 三种划界协议

## 2.1 定长

每条消息长度固定，接收方按固定长度切分。

```text
┌──────────┬──────────┬──────────┐
│  100 字节 │  100 字节 │  100 字节 │
└──────────┴──────────┴──────────┘
```

```java
// 定长协议：每条消息 8 字节，不足补空格
private static final int FIXED_LEN = 8;

public static List<String> decodeFixed(ByteBuffer buf) {
    List<String> messages = new ArrayList<>();
    buf.flip();
    while (buf.remaining() >= FIXED_LEN) {
        byte[] data = new byte[FIXED_LEN];
        buf.get(data);
        messages.add(new String(data, StandardCharsets.UTF_8).trim());
    }
    buf.compact();   // 不足一条的残留留到下次
    return messages;
}
```

| 优点 | 缺点 |
|------|------|
| 解析最简单，无需扫描 | 浪费带宽，短消息要填充 |
| 长度可预判，便于分配 | 变长业务不适合，超长消息要拆多条 |

定长适合**遥测采样、定长帧**这类数据本身就等长的场景，不适合聊天、文件传输。

## 2.2 分隔符

用特殊字符或字节序列标记消息结束，常见的有换行 `\n`、`\r\n`、`\0`。

```java
// 分隔符协议：以换行为界
public static List<String> decodeLine(ByteBuffer buf) {
    List<String> messages = new ArrayList<>();
    buf.flip();
    int start = buf.position();
    for (int i = buf.position(); i < buf.limit(); i++) {
        if (buf.get(i) == '\n') {
            int len = i - start;
            byte[] data = new byte[len];
            buf.get(data);
            buf.get();                    // 跳过换行符
            messages.add(new String(data, StandardCharsets.UTF_8));
            start = buf.position();
        }
    }
    buf.compact();
    return messages;
}
```

| 优点 | 缺点 |
|------|------|
| 可读性好，便于调试 | 消息体不能包含分隔符，需要转义 |
| 实现简单 | 需要逐字节扫描，大消息效率一般 |

文本协议常用分隔符，如 Redis 的 RESP、SMTP。二进制消息体可能恰好包含分隔符字节，因此必须配合转义机制，反而变复杂。

## 2.3 长度字段

在消息头里写明消息体长度，接收方先读长度、再读正文。这是二进制协议的主流做法。

```text
┌──────────────┬─────────────────────────┐
│ 4 字节长度 L  │       L 字节的消息体       │
└──────────────┴─────────────────────────┘
```

```java
// 长度字段协议：4 字节大端长度 + 正文
private static final int MAX_FRAME = 1024 * 1024;   // 单帧上限，防内存打爆

public static List<String> decodeLength(ByteBuffer buf) {
    List<String> messages = new ArrayList<>();
    buf.flip();
    while (buf.remaining() >= 4) {
        buf.mark();
        int len = buf.getInt();
        if (len < 0 || len > MAX_FRAME) {
            throw new IllegalArgumentException("非法帧长度: " + len);
        }
        if (buf.remaining() < len) {
            buf.reset();      // 正文未到齐，回退到长度字段之前
            break;
        }
        byte[] body = new byte[len];
        buf.get(body);
        messages.add(new String(body, StandardCharsets.UTF_8));
    }
    buf.compact();
    return messages;
}
```

## 2.4 三种方案对比

| 方案 | 边界信息 | 优点 | 缺点 | 典型协议 |
|------|----------|------|------|----------|
| 定长 | 隐含在长度约定 | 解析最快 | 浪费、不灵活 | 定长传感器帧 |
| 分隔符 | 特殊字节 | 可读、简单 | 正文需转义 | HTTP 头、SMTP、Redis |
| 长度字段 | 头部数字 | 精确、高效、支持二进制 | 需先读头再读体 | Dubbo、RocketMQ、gRPC |

**设计考量**：长度字段方案天然支持"先分配、再读取"，还能在头部塞版本号、序列号、压缩标志。几乎所有现代 RPC 框架都选它。代价是必须限制最大长度，否则一个伪造的超大长度就能让服务端 OOM。

> **行业实践**：成熟 RPC 协议通常把"魔数 + 版本 + 序列化方式 + 长度"一起放进帧头。魔数用于识别协议并快速拒绝非法连接，版本号用于灰度升级；只写长度的协议一旦要演进就无处落脚。

# 3. 用 ByteBuffer 手写解码

## 3.1 累积缓冲区

单次 `read()` 不一定拿到完整消息，所以每条连接要维护一个持久缓冲区。流程固定为三步：

```text
read 追加 ──> flip 解析循环 ──> compact 保留残片
```

```java
public class LengthFieldDecoder {
    private final ByteBuffer buffer = ByteBuffer.allocateDirect(8192);
    private static final int MAX_FRAME = 1024 * 1024;

    public List<byte[]> decode(ByteBuffer incoming) {
        List<byte[]> frames = new ArrayList<>();
        // 1. 把新读到的数据追加到累积缓冲区（incoming 刚从 Channel 读出，处于写模式）
        incoming.flip();
        buffer.put(incoming);
        // 2. 切读模式，循环取完整帧
        buffer.flip();
        while (buffer.remaining() >= 4) {
            buffer.mark();
            int len = buffer.getInt();
            if (len < 0 || len > MAX_FRAME) {
                throw new IllegalStateException("帧长度非法: " + len);
            }
            if (buffer.remaining() < len) {
                buffer.reset();
                break;                    // 半包，等下一批
            }
            byte[] body = new byte[len];
            buffer.get(body);
            frames.add(body);
        }
        // 3. 保留未解析完的残片，切回写模式
        buffer.compact();
        return frames;
    }
}
```

**注意**：累积缓冲区满时 `buffer.put(incoming)` 会抛 `BufferOverflowException`。当对端发来一个声明长度合法、实际却迟迟不补齐的帧时，残片会一直堆积。解决办法是设一个"最大待解析字节数"，超过就断开连接，防止慢速攻击（slowloris）。

## 3.2 一次读与多次读的对比

| 做法 | 是否正确 | 说明 |
|------|----------|------|
| 每次 `read` 后清空缓冲区再解析 | 错 | 丢失上一次的半包 |
| 只解析一次就 `clear` | 错 | 一次可能读到多条，只处理一条会丢 |
| `while` 循环解析 + `compact` | 对 | 一次读多帧也能全部取出 |
| 用 `read()` 返回值判断消息结束 | 错 | 返回值只是本次读到的字节数 |

# 4. 编解码器抽象

## 4.1 Decoder 与 Encoder

把"边界处理"从业务代码里抽出来，就是编解码器。一个合格的分帧解码器只需要两个方法：

```java
public interface FrameDecoder {
    /** 追加数据，返回所有已完整的消息 */
    List<byte[]> decode(ByteBuffer incoming);
    /** 连接关闭前调用，处理残留的字节 */
    default void onEndOfStream(List<byte[]> out) {
        // 默认不做处理；严格协议可在此报"帧被截断"
    }
}

public interface FrameEncoder {
    /** 把一条业务消息编码成可发送的字节 */
    ByteBuffer encode(byte[] message);
}
```

编码端负责加长度头：

```java
public class LengthFieldEncoder implements FrameEncoder {
    @Override
    public ByteBuffer encode(byte[] message) {
        // 4 字节长度 + 正文，一次分配，避免多次拷贝
        ByteBuffer out = ByteBuffer.allocate(4 + message.length);
        out.putInt(message.length);
        out.put(message);
        out.flip();       // 交给 Channel 前必须 flip
        return out;
    }
}
```

## 4.2 Netty 的对应实现

手写编解码器容易在边界上出错，Netty 内置了几个成熟的实现：

| 协议形态 | Netty 解码器 | 关键参数 |
|----------|--------------|----------|
| 定长 | `FixedLengthFrameDecoder` | 帧长度 |
| 分隔符 | `LineBasedFrameDecoder`、`DelimiterBasedFrameDecoder` | 最大长度、分隔符 |
| 长度字段 | `LengthFieldBasedFrameDecoder` | 最大帧长、偏移、长度字节数、跳过字节数 |

```java
// 协议：4 字节长度 + 4 字节版本 + 正文
ch.pipeline().addLast(new LengthFieldBasedFrameDecoder(
        1024 * 1024,   // maxFrameLength：单帧上限，必设
        0,             // lengthFieldOffset：长度字段在帧中的偏移
        4,             // lengthFieldLength：长度字段占几字节
        4,             // lengthAdjustment：长度字段之后、正文之前的字节数
        0))            // initialBytesToStrip：解码后是否剥掉头部
  .addLast(new StringDecoder(StandardCharsets.UTF_8))
  .addLast(new BusinessHandler());
```

参数组合是最容易配错的地方，记住一个换算：

```text
正文长度 = 长度字段的值 + lengthAdjustment
```

如果长度字段的值表示"整帧长度（含头）"，那么 `lengthAdjustment = -(lengthFieldOffset + lengthFieldLength)`。配错的表现通常是解码器一直等数据，或者一次吐出带垃圾头的帧。

# 5. 常见错误

| 错误 | 症状 | 正确做法 |
|------|------|----------|
| 假设一次 `read` 一条消息 | 偶发丢消息或解析异常 | 累积缓冲 + 循环解析 |
| 解析后忘记 `compact` | 半包丢失，消息错位 | `compact` 保留残片 |
| 不限制帧长度 | 伪造长度导致 OOM | 设置 `MAX_FRAME` 并校验 |
| 分隔符不做转义 | 正文含分隔符时错误切分 | 转义或改用长度字段 |
| 长度字段大小端不一致 | 长度解析成天文数字 | 两端统一大端，写入协议文档 |
| `encode` 后忘记 `flip` | 发送 0 字节或读越界 | 写完立即 `flip` |
| 编码/解码用不同字符集 | 中文乱码、长度不符 | 统一 UTF-8，长度按字节算 |

**注意**：长度字段一定按**字节数**算，不是字符数。`"你好".length()` 是 2，但 UTF-8 编码后是 6 字节。用字符数当长度，接收端会少读 4 字节，后续全部错位。

# 6. 练习与自检

**练习一**：实现一个分隔符协议，要求正文中的分隔符用 `\` 转义。发送 `a\nb\n` 与 `a\\\nb`，验证解码正确。

**练习二**：把 `LengthFieldDecoder` 的累积缓冲区设为 16 字节，发送一条 100 字节的消息，确认它会分多次 `read` 并在最后正确拼出完整帧。

**练习三**：故意把 `LengthFieldBasedFrameDecoder` 的 `lengthAdjustment` 配错，观察现象：是卡住不吐帧，还是吐出带垃圾的帧？解释原因。

自检问题：粘包是 TCP 的缺陷吗？为什么长度字段必须按字节算？`compact` 与 `clear` 在半包场景下差在哪？

# 7. 小结

| 主题 | 要点 |
|------|------|
| 根因 | TCP 是字节流，消息边界必须由应用层定义 |
| 定长 | 最简单，适合等长帧，浪费带宽 |
| 分隔符 | 可读性好，正文需转义 |
| 长度字段 | 二进制协议主流，必须限制最大长度 |
| 解码套路 | 追加 → flip → 循环解析 → compact |
| 编码 | 写头写体 → flip → 发送，长度按字节算 |

消息边界解决后，下一个问题随之而来：连接看起来还在，对端其实早就不可达了。下一篇讲心跳、空闲检测与断线重连，让长连接真正"活"起来。
