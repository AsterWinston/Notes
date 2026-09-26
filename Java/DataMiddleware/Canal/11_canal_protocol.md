
# 1. 协议概述

Canal 的 Client-Server 通信协议基于 **TCP + Protobuf** 实现。客户端通过 TCP 连接到 Canal Server 的指定端口（默认 11111），通过 Protobuf 编码的消息进行交互。

```
┌────────────────────┐                    ┌────────────────────┐
│   Canal Client     │                    │   Canal Server     │
│                    │    TCP/Netty       │                    │
│  ┌──────────────┐  │<─────────────────>│  ┌──────────────┐  │
│  │  Protobuf    │  │   Protocol Buffers │  │  Protobuf    │  │
│  │  Encoder     │  │                    │  │  Decoder     │  │
│  └──────────────┘  │                    │  └──────────────┘  │
└────────────────────┘                    └────────────────────┘
```

# 2. 消息格式

## 2.1 外层 Packet 封装

每个 Canal 协议消息由两层组成：

```
┌──────────────────────────────────────────┐
│            Outer Packet                   │
│                                           │
│  ┌────────────────────────────────────┐  │
│  │  Magic Number (2 bytes): 0xEE 0x11 │  │
│  ├────────────────────────────────────┤  │
│  │  Version (1 byte)                  │  │
│  ├────────────────────────────────────┤  │
│  │  Type (1 byte)                     │  │
│  │  - 0x01: CLIENTAUTHENTICATION      │  │
│  │  - 0x02: CLIENTACK                │  │
│  │  - 0x03: CLIENTROLLBACK           │  │
│  │  - 0x04: SUBSCRIPTION             │  │
│  │  - 0x05: UNSUBSCRIPTION           │  │
│  │  - 0x06: GET                      │  │
│  │  - 0x07: MESSAGES                 │  │
│  │  - 0x08: CLIENTACKWITHNULL        │  │
│  │  - 0x09: HEARTBEAT                │  │
│  │  - 0x0A: DUMP                     │  │
│  ├────────────────────────────────────┤  │
│  │  Compress Flag (1 byte)           │  │
│  │  - 0x00: 未压缩                    │  │
│  │  - 0x01: 已压缩                    │  │
│  ├────────────────────────────────────┤  │
│  │  Fragment Flag (1 byte)           │  │
│  │  - 0x00: 非分片                    │  │
│  │  - 0x01: 第一个分片                │  │
│  │  - 0x02: 中间分片                  │  │
│  │  - 0x03: 最后一个分片              │  │
│  ├────────────────────────────────────┤  │
│  │  Body Length (4 bytes)            │  │
│  ├────────────────────────────────────┤  │
│  │  Body (N bytes)                   │  │
│  │  Protobuf 序列化的消息体            │  │
│  └────────────────────────────────────┘  │
└──────────────────────────────────────────┘
```

## 2.2 Packet 编解码实现

```java
public class Packet {
    
    // Magic Number
    public static final short MAGIC = (short) 0xEE11;
    
    // 包类型
    public static final byte TYPE_CLIENTAUTHENTICATION = 0x01;
    public static final byte TYPE_CLIENTACK              = 0x02;
    public static final byte TYPE_CLIENTROLLBACK         = 0x03;
    public static final byte TYPE_SUBSCRIPTION           = 0x04;
    public static final byte TYPE_UNSUBSCRIPTION         = 0x05;
    public static final byte TYPE_GET                    = 0x06;
    public static final byte TYPE_MESSAGES               = 0x07;
    public static final byte TYPE_CLIENTACKWITHNULL      = 0x08;
    public static final byte TYPE_HEARTBEAT              = 0x09;
    public static final byte TYPE_DUMP                   = 0x0A;
    
    private short magic;
    private byte version;
    private byte type;
    private byte compress;
    private byte fragment;
    private int bodyLength;
    private byte[] body;
    
    // ========== 编码 ==========
    
    public byte[] encode() {
        int totalLength = 2 + 1 + 1 + 1 + 1 + 4 + bodyLength;
        ByteBuffer buffer = ByteBuffer.allocate(totalLength);
        
        buffer.putShort(MAGIC);                    // 2 bytes
        buffer.put(version);                        // 1 byte
        buffer.put(type);                           // 1 byte
        buffer.put(compress);                       // 1 byte
        buffer.put(fragment);                       // 1 byte
        buffer.putInt(bodyLength);                  // 4 bytes
        buffer.put(body);                           // N bytes
        
        return buffer.array();
    }
    
    // ========== 解码 ==========
    
    public static Packet decode(byte[] data) {
        ByteBuffer buffer = ByteBuffer.wrap(data);
        
        Packet packet = new Packet();
        packet.magic = buffer.getShort();            // 2 bytes
        packet.version = buffer.get();               // 1 byte
        packet.type = buffer.get();                  // 1 byte
        packet.compress = buffer.get();              // 1 byte
        packet.fragment = buffer.get();              // 1 byte
        packet.bodyLength = buffer.getInt();         // 4 bytes
        packet.body = new byte[packet.bodyLength];   // N bytes
        buffer.get(packet.body);
        
        return packet;
    }
}
```

# 3. Protobuf 协议定义

## 3.1 协议文件结构

Canal 的 Protobuf 定义在 `canal-protocol` 模块中：

```
canal-protocol/src/main/protobuf/
├── CanalProtocol.proto    # 通信协议消息定义
├── EntryProtocol.proto    # 数据 Entry 定义
└── FlatMessage.proto      # 扁平化消息定义
```

## 3.2 CanalProtocol.proto — 通信消息

```protobuf
syntax = "proto3";
package com.alibaba.otter.canal.protocol;

option java_package = "com.alibaba.otter.canal.protocol";
option java_outer_classname = "CanalPacket";

// ========== 数据包类型枚举 ==========

enum PacketType {
    // 兼容旧版本
    HANDHLAKE      = 0;
    CLIENTAUTHENTICATION = 1;
    ACK            = 2;
    SUBSCRIPTION   = 3;
    UNSUBSCRIPTION = 4;
    GET            = 5;
    MESSAGES       = 6;
    CLIENTACK      = 7;
    // 管理命令
    DUMP           = 8;
    HEARTBEAT      = 9;
    CLIENTROLLBACK = 10;
}

// ========== 握手消息 ==========

message Handshake {
    // 通信协议版本
    bytes communication_encoding_version = 1;
    // 支持的序列化方式
    repeated string supported_compressions = 2;
}

// ========== 客户端认证 ==========

message ClientAuth {
    string username = 1;
    bytes  password = 2;   // 现在是明文，以后会加密
    int64  net_read_timeout = 3;
    int64  net_write_timeout = 4;
    string destination = 5;  // 实例名
    string client_id = 6;    // 客户端 ID
    string filter = 7;       // 订阅过滤规则
    int64  start_timestamp = 8;
}

// ========== 认证确认 ==========

message Ack {
    int32  error_code = 1;
    string error_message = 2;
}

// ========== 订阅请求 ==========

message Sub {
    string destination = 1;
    string client_id = 2;
    string filter = 3;      // 过滤规则
}

// ========== 取消订阅 ==========

message Unsub {
    string destination = 1;
    string client_id = 2;
    string filter = 3;
}

// ========== GET 请求 ==========

message Get {
    string destination = 1;
    string client_id = 2;
    int32  batch_size = 3;           // 批量大小
    int64  timeout = 4;              // 超时时间
    int32  unit = 5;                 // 时间单位
    int64  auto_ack = 6;            // 是否自动 ACK
    int64  fetch_size = 7;          // 抓取大小
    
    // 位点信息
    string journal_name = 8;          // binlog 文件名
    int64  position = 9;             // binlog 偏移量
    string gtid = 10;                // GTID 集合
}

// ========== 响应消息 ==========

message Messages {
    int64  batch_id = 1;              // 批次 ID
    repeated bytes messages = 2;     // 序列化后的 Entry 列表
    
    // 扁平化模式（替代 messages 字段）
    repeated FlatMessage flat_messages = 3;
}

// ========== 客户端 ACK ==========

message ClientAck {
    string destination = 1;
    string client_id = 2;
    int64  batch_id = 3;              // ACK 的批次 ID
}

// ========== 客户端 Rollback ==========

message ClientRollback {
    string destination = 1;
    string client_id = 2;
    int64  batch_id = 3;              // 回滚的批次 ID
}

// ========== 心跳消息 ==========

message HeartBeat {
    int64  send_timestamp = 1;        // 发送时间戳
    int64  start_timestamp = 2;       // 启动时间戳
}
```

# 4. FlatMessage — 扁平化消息

Canal 1.1.4+ 引入了 FlatMessage 格式，将嵌套的 Protobuf 结构扁平化为更易处理的 JSON 格式：

```protobuf
message FlatMessage {
    
    // === 基础信息 ===
    int64  id = 1;              // 唯一 ID
    string database = 2;        // 库名
    string table = 3;           // 表名
    repeated string pkNames = 4;// 主键列名
    
    // === 变更信息 ===
    bool   isDdl = 5;           // 是否 DDL
    string type = 6;            // INSERT/UPDATE/DELETE/CREATE/ALTER/...
    
    // === 时间信息 ===
    int64  es = 7;              // binlog 执行时间戳（毫秒）
    int64  ts = 8;              // Canal 处理时间戳（毫秒）
    
    // === 变更数据 ===
    string sql = 9;             // DDL SQL
    map<string, string> sqlType = 10;    // 列名 -> MySQL 类型
    map<string, string> mysqlType = 11; // 列名 -> 更具体的 MySQL 类型
    
    // === 行数据 ===
    repeated map<string, string> data = 12;  // 变更后数据（INSERT 和 UPDATE）
    repeated map<string, string> old = 13;   // 变更前数据（UPDATE 和 DELETE）
    
    // === 位点信息 ===
    int64  gtid = 14;           // GTID（兼容字段，实际 gtid 是字符串放这里）
    int64  seq_id = 15;        // 序列号
    
    // === 扩展属性 ===
    map<string, string> props = 16;   // 扩展属性
}
```

## 4.1 FlatMessage JSON 示例

**INSERT**：
```json
{
  "id": 1,
  "database": "test",
  "table": "users",
  "pkNames": ["id"],
  "isDdl": false,
  "type": "INSERT",
  "es": 1672545600000,
  "ts": 1672545600010,
  "sql": "",
  "sqlType": {"id": "4", "name": "12", "email": "12", "status": "-6"},
  "mysqlType": {"id": "bigint(20)", "name": "varchar(50)", "email": "varchar(100)", "status": "tinyint(4)"},
  "data": [
    {"id": "1", "name": "Alice", "email": "alice@example.com", "status": "1"}
  ],
  "old": []
}
```

**UPDATE**：
```json
{
  "id": 2,
  "database": "test",
  "table": "users",
  "pkNames": ["id"],
  "isDdl": false,
  "type": "UPDATE",
  "es": 1672545601000,
  "ts": 1672545601010,
  "data": [
    {"id": "1", "name": "Alice", "status": "0"}
  ],
  "old": [
    {"status": "1"}
  ]
}
```

# 5. 通信交互流程

## 5.1 标准通信流程

```
Client                                  Server
  │                                       │
  │──── TCP Connect ────────────────────>│
  │                                       │
  │<─── HANDSHAKE ──────────────────────│
  │     (协议版本、压缩支持)              │
  │                                       │
  │──── CLIENTAUTH ────────────────────>│
  │     (用户名、密码、客户端ID、过滤规则)│
  │                                       │
  │<─── ACK ────────────────────────────│
  │     (认证成功/失败)                    │
  │                                       │
  │──── SUBSCRIPTION ──────────────────>│
  │     (订阅确认，触发 ACK 位点同步)      │
  │                                       │
  │<─── ACK ────────────────────────────│
  │     (订阅成功)                         │
  │                                       │
  │  ╔══════ 消费循环 ═══════╗          │
  │  ║                         ║          │
  │  ║──── GET ──────────────>║          │
  │  ║                         ║          │
  │  ║<─── MESSAGES ──────────║          │
  │  ║     (一批数据)           ║          │
  │  ║                         ║          │
  │  ║  业务处理...            ║          │
  │  ║                         ║          │
  │  ║──── CLIENTACK ────────>║          │
  │  ║     (确认消费)           ║          │
  │  ║                         ║          │
  │  ╚═════════════════════════╝          │
  │                                       │
  │──── UNSUBSCRIPTION ────────────────>│
  │                                       │
  │──── TCP Disconnect ────────────────>│
  │                                       │
```

## 5.2 Netty Handler 实现

```java
// Server 端 Netty Handler
@Sharable
public class CanalServerHandler extends SimpleChannelInboundHandler<Packet> {
    
    private CanalServerWithNetty canalServer;
    
    @Override
    protected void channelRead0(ChannelHandlerContext ctx, Packet packet) {
        
        switch (packet.getType()) {
            case Packet.TYPE_CLIENTAUTHENTICATION:
                handleClientAuth(ctx, packet);
                break;
                
            case Packet.TYPE_SUBSCRIPTION:
                handleSubscribe(ctx, packet);
                break;
                
            case Packet.TYPE_UNSUBSCRIPTION:
                handleUnsubscribe(ctx, packet);
                break;
                
            case Packet.TYPE_GET:
                handleGet(ctx, packet);
                break;
                
            case Packet.TYPE_CLIENTACK:
                handleClientAck(ctx, packet);
                break;
                
            case Packet.TYPE_CLIENTROLLBACK:
                handleClientRollback(ctx, packet);
                break;
                
            case Packet.TYPE_HEARTBEAT:
                handleHeartbeat(ctx, packet);
                break;
                
            default:
                logger.warn("unknown packet type: {}", packet.getType());
        }
    }
    
    private void handleClientAuth(ChannelHandlerContext ctx, Packet packet) {
        ClientAuth auth = ClientAuth.parseFrom(packet.getBody());
        
        // 认证逻辑
        boolean success = canalServer.auth(auth.getUsername(), auth.getPassword());
        
        if (!success) {
            // 发送错误 ACK
            Ack ack = Ack.newBuilder()
                .setErrorCode(400)
                .setErrorMessage("auth failed")
                .build();
            sendPacket(ctx, Packet.TYPE_CLIENTACK, ack.toByteArray());
            return;
        }
        
        // 绑定客户端到 Channel
        ClientIdentity identity = new ClientIdentity(
            auth.getDestination(), auth.getClientId(), auth.getFilter()
        );
        ctx.channel().attr(CLIENT_IDENTITY_KEY).set(identity);
        
        // 发送成功 ACK
        Ack ack = Ack.newBuilder()
            .setErrorCode(200)
            .build();
        sendPacket(ctx, Packet.TYPE_CLIENTACK, ack.toByteArray());
    }
    
    private void handleSubscribe(ChannelHandlerContext ctx, Packet packet) {
        Sub sub = Sub.parseFrom(packet.getBody());
        
        canalServer.subscribe(sub.getDestination(), sub.getClientId(), sub.getFilter());
        
        // 回滚到上次 ACK 的位置
        canalServer.rollback(sub.getDestination(), sub.getClientId());
        
        Ack ack = Ack.newBuilder().setErrorCode(200).build();
        sendPacket(ctx, Packet.TYPE_CLIENTACK, ack.toByteArray());
    }
    
    private void handleGet(ChannelHandlerContext ctx, Packet packet) {
        Get get = Get.parseFrom(packet.getBody());
        
        // 从对应的 Instance 获取数据
        Message message = canalServer.get(
            get.getDestination(),
            get.getClientId(),
            get.getBatchSize(),
            get.getTimeout(),
            TimeUnit.values()[get.getUnit()]
        );
        
        // 封装为 Messages 响应
        Messages messages = Messages.newBuilder()
            .setBatchId(message.getId())
            .addAllMessages(message.getProtobufEntries())
            .build();
        
        sendPacket(ctx, Packet.TYPE_MESSAGES, messages.toByteArray());
    }
    
    private void handleClientAck(ChannelHandlerContext ctx, Packet packet) {
        ClientAck clientAck = ClientAck.parseFrom(packet.getBody());
        
        canalServer.ack(
            clientAck.getDestination(),
            clientAck.getClientId(),
            clientAck.getBatchId()
        );
    }
    
    private void sendPacket(ChannelHandlerContext ctx, byte type, byte[] body) {
        Packet packet = new Packet();
        packet.setType(type);
        packet.setBody(body);
        packet.setBodyLength(body.length);
        
        ctx.writeAndFlush(packet);
    }
}
```

# 6. 压缩支持

对于大数据量的消息，Canal 支持压缩传输：

```java
public class PacketCompressor {
    
    // 压缩阈值：超过此大小才压缩
    private static final int COMPRESS_THRESHOLD = 1024 * 64;  // 64KB
    
    public static Packet compressIfNeeded(Packet packet) {
        if (packet.getBodyLength() < COMPRESS_THRESHOLD) {
            return packet;  // 太小，不压缩
        }
        
        try {
            // 使用 GZIP 压缩
            ByteArrayOutputStream baos = new ByteArrayOutputStream();
            try (GZIPOutputStream gzip = new GZIPOutputStream(baos)) {
                gzip.write(packet.getBody());
            }
            
            byte[] compressed = baos.toByteArray();
            
            // 只有压缩后确实变小了才使用压缩
            if (compressed.length < packet.getBodyLength()) {
                packet.setCompress((byte) 1);
                packet.setBody(compressed);
                packet.setBodyLength(compressed.length);
            }
            
        } catch (IOException e) {
            logger.warn("compress failed, use original", e);
        }
        
        return packet;
    }
    
    public static Packet decompressIfNeeded(Packet packet) throws IOException {
        if (packet.getCompress() == 0) {
            return packet;
        }
        
        ByteArrayInputStream bais = new ByteArrayInputStream(packet.getBody());
        try (GZIPInputStream gzip = new GZIPInputStream(bais)) {
            byte[] decompressed = IOUtils.toByteArray(gzip);
            packet.setCompress((byte) 0);
            packet.setBody(decompressed);
            packet.setBodyLength(decompressed.length);
        }
        
        return packet;
    }
}
```

# 7. 数据分片

当单条消息过大时（如包含大量行数据的批量操作），Canal 会对消息进行分片：

```java
public class PacketFragmenter {
    
    private static final int MAX_FRAGMENT_SIZE = 1024 * 512;  // 512KB
    
    /**
     * 分片发送
     */
    public static List<Packet> fragment(Packet packet) {
        if (packet.getBodyLength() <= MAX_FRAGMENT_SIZE) {
            return Collections.singletonList(packet);
        }
        
        byte[] body = packet.getBody();
        int totalFragments = (body.length + MAX_FRAGMENT_SIZE - 1) / MAX_FRAGMENT_SIZE;
        List<Packet> fragments = new ArrayList<>(totalFragments);
        
        for (int i = 0; i < totalFragments; i++) {
            int offset = i * MAX_FRAGMENT_SIZE;
            int length = Math.min(MAX_FRAGMENT_SIZE, body.length - offset);
            
            Packet fragment = new Packet();
            fragment.setType(packet.getType());
            fragment.setVersion(packet.getVersion());
            fragment.setBody(Arrays.copyOfRange(body, offset, offset + length));
            fragment.setBodyLength(length);
            
            if (i == 0) {
                fragment.setFragment((byte) 0x01);  // 第一个分片
            } else if (i == totalFragments - 1) {
                fragment.setFragment((byte) 0x03);  // 最后一个分片
            } else {
                fragment.setFragment((byte) 0x02);  // 中间分片
            }
            
            fragments.add(fragment);
        }
        
        return fragments;
    }
}
```

# 8. FlatMessage 转换器

```java
public class FlatMessageConverter {
    
    /**
     * 将 Entry 列表转换为 FlatMessage 列表
     */
    public static List<FlatMessage> convert(String destination, List<Entry> entries) {
        
        List<FlatMessage> result = new ArrayList<>();
        
        for (Entry entry : entries) {
            if (entry.getEntryType() != EntryType.ROWDATA) continue;
            
            FlatMessage msg = new FlatMessage();
            Header header = entry.getHeader();
            
            msg.setDatabase(header.getSchemaName());
            msg.setTable(header.getTableName());
            msg.setEs(header.getExecuteTime());
            msg.setTs(System.currentTimeMillis());
            
            try {
                RowChange rowChange = RowChange.parseFrom(entry.getStoreValue());
                
                // DDL 处理
                if (rowChange.getIsDdl()) {
                    msg.setIsDdl(true);
                    msg.setType("DDL");
                    msg.setSql(rowChange.getSql());
                    result.add(msg);
                    continue;
                }
                
                // DML 处理
                EventType eventType = rowChange.getEventType();
                msg.setIsDdl(false);
                msg.setType(eventType.name());
                
                for (RowData rowData : rowChange.getRowDatasList()) {
                    // 提取主键名
                    if (msg.getPkNames().isEmpty()) {
                        extractPkNames(rowData, msg.getPkNames());
                    }
                    
                    // 提取 sqlType / mysqlType
                    if (msg.getSqlType().isEmpty()) {
                        extractColumnTypes(rowData, msg.getSqlType(), msg.getMysqlType());
                    }
                    
                    // 构建 data 和 old
                    if (eventType == EventType.INSERT) {
                        msg.getData().add(toMap(rowData.getAfterColumnsList()));
                    } else if (eventType == EventType.UPDATE) {
                        msg.getData().add(toMap(rowData.getAfterColumnsList()));
                        msg.getOld().add(toMap(rowData.getBeforeColumnsList()));
                    } else if (eventType == EventType.DELETE) {
                        msg.getData().add(toMap(rowData.getBeforeColumnsList()));
                    }
                }
                
                result.add(msg);
                
            } catch (InvalidProtocolBufferException e) {
                throw new RuntimeException("parse entry failed", e);
            }
        }
        
        return result;
    }
    
    private static Map<String, String> toMap(List<Column> columns) {
        Map<String, String> map = new LinkedHashMap<>();
        for (Column col : columns) {
            map.put(col.getName(), col.getIsNull() ? null : col.getValue());
        }
        return map;
    }
}
```

# 9. 安全性 — 认证扩展

Canal 1.1.4+ 支持用户名/密码认证：

```properties
# canal.properties — 服务端配置
canal.user = admin
canal.passwd = E10ADC3949BA59ABBE56E057F20F883E   # MD5

# 客户端连接时需提供
connector = CanalConnectors.newSingleConnector(
    new InetSocketAddress("127.0.0.1", 11111),
    "example",
    "admin",
    "E10ADC3949BA59ABBE56E057F20F883E"
);
```

```java
// Server 端认证处理
public class CanalAuthService {
    
    private Map<String, String> users = new HashMap<>();
    
    public CanalAuthService(Properties properties) {
        String username = properties.getProperty("canal.user");
        String password = properties.getProperty("canal.passwd");
        if (username != null) {
            users.put(username, password);
        }
    }
    
    public boolean authenticate(String username, String password) {
        if (users.isEmpty()) {
            return true;  // 未配置认证，允许所有连接
        }
        
        String expectedPassword = users.get(username);
        if (expectedPassword == null) {
            return false;
        }
        
        // 密码比对（支持 MD5）
        return DigestUtils.md5Hex(password).equals(expectedPassword);
    }
}
```
