
# 1. 概述

Canal 的核心能力之一是**模拟 MySQL Slave（从库）的复制协议**，让 MySQL Master 将其视为一个合法的 Slave，从而持续推送 Binlog 事件流。这个过程涉及 MySQL 客户端-服务器协议中的多个关键命令。

```
┌────────────────┐                          ┌────────────────┐
│  MySQL Master  │                          │  Canal Server   │
│                │                          │  (伪装 Slave)   │
│                │  ① TCP 连接              │                │
│                │<─────────────────────────│                │
│                │                          │                │
│                │  ② 握手认证              │                │
│                │─────────────────────────>│                │
│                │<─────────────────────────│                │
│                │                          │                │
│                │  ③ COM_REGISTER_SLAVE    │                │
│                │<─────────────────────────│                │
│                │                          │                │
│                │  ④ COM_BINLOG_DUMP       │                │
│                │<─────────────────────────│                │
│                │                          │                │
│                │  ⑤ Binlog Event Stream   │                │
│                │─────────────────────────>│                │
│                │─────────────────────────>│                │
│                │─────────────────────────>│  (持续推送)    │
│                │                          │                │
│                │  ⑥ COM_QUERY (心跳)      │                │
│                │<─────────────────────────│                │
│                │─────────────────────────>│                │
└────────────────┘                          └────────────────┘
```

# 2. MySQL 客户端-服务端通信协议基础

## 2.1 协议包结构

MySQL 协议中的每个数据包由一个 4 字节的包头 + 有效载荷组成：

```
┌──────────────────────────────────────────┐
│           MySQL Protocol Packet           │
│                                          │
│  ┌─────────────┬───────────────────────┐ │
│  │ Payload Len │  Sequence ID          │ │
│  │  (3 bytes)  │   (1 byte)            │ │
│  ├─────────────┴───────────────────────┤ │
│  │          Payload Data               │ │
│  │         (Payload Len bytes)         │ │
│  └─────────────────────────────────────┘ │
│                                          │
│  最大 Payload Length: 16MB - 1           │
│  (2^24 - 1 bytes)                        │
└──────────────────────────────────────────┘
```

## 2.2 Packet 编解码

```java
// 写入一个 MySQL 协议包
public class MysqlPacketWriter {
    
    private ByteArrayOutputStream buffer;
    
    public void writePacket(byte[] payload, byte sequenceId) throws IOException {
        int payloadLength = payload.length;
        
        // 写入 3 字节的 payload 长度
        buffer.write(payloadLength & 0xFF);
        buffer.write((payloadLength >> 8) & 0xFF);
        buffer.write((payloadLength >> 16) & 0xFF);
        
        // 写入 1 字节的 sequence ID
        buffer.write(sequenceId);
        
        // 写入 payload
        buffer.write(payload);
    }
}

// 读取一个 MySQL 协议包
public class MysqlPacketReader {
    
    public Packet readPacket(InputStream in) throws IOException {
        // 读取 4 字节包头
        byte[] header = new byte[4];
        readFully(in, header);
        
        // 解析长度
        int payloadLength = (header[0] & 0xFF) 
                          | ((header[1] & 0xFF) << 8) 
                          | ((header[2] & 0xFF) << 16);
        
        // 读取 payload
        byte[] payload = new byte[payloadLength];
        readFully(in, payload);
        
        return new Packet(header[3], payload);  // sequenceId, payload
    }
}
```

# 3. 连接与握手

## 3.1 握手流程

```java
public class MysqlConnector {
    
    /**
     * 建立与 MySQL Master 的连接
     */
    public MysqlConnection connect(InetSocketAddress address, 
                                    String username, String password) 
            throws IOException {
        
        SocketChannel channel = SocketChannel.open(address);
        InputStream in = channel.socket().getInputStream();
        OutputStream out = channel.socket().getOutputStream();
        
        // Step 1: 接收 MySQL 的握手包（HandshakeV10）
        HandshakePacket handshake = readHandshakePacket(in);
        
        // Step 2: 构造认证响应包
        HandshakeResponse41 response = buildAuthResponse(handshake, username, password);
        
        // Step 3: 发送认证响应
        sendPacket(out, response.toBytes(), 1);
        
        // Step 4: 接收认证结果
        AuthResult result = readAuthResult(in);
        
        if (!result.isSuccess()) {
            throw new IOException("Auth failed: " + result.getErrorMessage());
        }
        
        return new MysqlConnection(channel, handshake);
    }
}
```

## 3.2 握手包解析

```java
public class HandshakePacket {
    
    private int protocolVersion;      // 协议版本（固定 10）
    private String serverVersion;     // MySQL 版本号
    private long threadId;            // 连接线程 ID
    private String scramblePart1;     // 挑战码第1部分（8字节）
    private int serverCapabilities;   // 服务器能力标志（低16位）
    private int characterSet;         // 字符集
    private int statusFlags;          // 服务器状态标志
    private int serverCapabilities2;  // 服务器能力标志（高16位）
    private String scramblePart2;     // 挑战码第2部分（12字节）
    private String authPluginName;    // 认证插件名
    
    /**
     * 从输入流解析握手包
     */
    public static HandshakePacket parse(InputStream in) throws IOException {
        HandshakePacket handshake = new HandshakePacket();
        
        PacketReader reader = new PacketReader(in);
        
        // 协议版本
        handshake.protocolVersion = reader.readByte();
        
        // 服务器版本（以 null 结尾的字符串）
        handshake.serverVersion = reader.readNullTerminatedString();
        
        // 连接线程 ID
        handshake.threadId = reader.readInt(4);
        
        // 挑战码第1部分（8字节）
        handshake.scramblePart1 = reader.readString(8);
        
        // 保留字节
        reader.skip(1);
        
        // 服务器能力标志（低16位）
        handshake.serverCapabilities = reader.readInt(2);
        
        // 字符集
        handshake.characterSet = reader.readByte();
        
        // 服务器状态标志
        handshake.statusFlags = reader.readInt(2);
        
        // 服务器能力标志（高16位）
        handshake.serverCapabilities2 = reader.readInt(2);
        
        // 认证插件数据长度
        int authPluginDataLength = reader.readByte();
        
        // 保留字节
        reader.skip(10);
        
        // 挑战码第2部分
        handshake.scramblePart2 = reader.readString(Math.max(13, authPluginDataLength - 8));
        
        // 认证插件名
        if (handshake.serverCapabilities & CLIENT_PLUGIN_AUTH) {
            handshake.authPluginName = reader.readNullTerminatedString();
        }
        
        return handshake;
    }
}
```

## 3.3 认证响应构建

```java
public class HandshakeResponse41 {
    
    private int clientCapabilities;     // 客户端能力标志
    private int maxPacketSize;          // 最大包大小
    private int characterSet;           // 字符集
    private String username;            // 用户名
    private byte[] authResponse;        // 认证响应（加密的密码）
    private String database;            // 默认数据库
    private String authPluginName;      // 认证插件名
    
    /**
     * 构建认证响应包
     */
    public static HandshakeResponse41 build(HandshakePacket handshake,
                                             String username, String password) {
        HandshakeResponse41 response = new HandshakeResponse41();
        
        // 设置客户端能力标志
        response.clientCapabilities = CLIENT_LONG_PASSWORD 
                                    | CLIENT_FOUND_ROWS 
                                    | CLIENT_LONG_FLAG
                                    | CLIENT_CONNECT_WITH_DB
                                    | CLIENT_PROTOCOL_41
                                    | CLIENT_SECURE_CONNECTION
                                    | CLIENT_PLUGIN_AUTH;
        
        response.maxPacketSize = 16 * 1024 * 1024;  // 16MB
        response.characterSet = 33;                  // UTF-8
        response.username = username;
        
        // 构造认证响应
        String authPlugin = handshake.getAuthPluginName();
        if ("mysql_native_password".equals(authPlugin)) {
            response.authResponse = mysqlNativePassword(
                password,
                handshake.getScramble()
            );
        } else if ("caching_sha2_password".equals(authPlugin)) {
            response.authResponse = cachingSha2Password(
                password,
                handshake.getScramble()
            );
        }
        
        response.authPluginName = authPlugin;
        
        return response;
    }
    
    /**
     * mysql_native_password 认证算法
     * SHA1(password) XOR SHA1(scramble + SHA1(SHA1(password)))
     */
    private static byte[] mysqlNativePassword(String password, byte[] scramble) 
            throws NoSuchAlgorithmException {
        MessageDigest sha1 = MessageDigest.getInstance("SHA-1");
        
        byte[] stage1 = sha1.digest(password.getBytes());
        sha1.reset();
        byte[] stage2 = sha1.digest(stage1);
        sha1.reset();
        
        sha1.update(scramble);
        sha1.update(stage2);
        byte[] stage3 = sha1.digest();
        
        byte[] result = new byte[stage1.length];
        for (int i = 0; i < stage1.length; i++) {
            result[i] = (byte) (stage1[i] ^ stage3[i]);
        }
        
        return result;
    }
}
```

# 4. COM_REGISTER_SLAVE — 注册为从库

## 4.1 命令格式

```
COM_REGISTER_SLAVE (0x15)

Payload:
  ┌──────────────────────────────────────────────────────────┐
  │ server_id (4 bytes)                                      │
  │ slave_hostname_length (1 byte) + hostname (N bytes)      │
  │ slave_user_length (1 byte) + username (N bytes)          │
  │ slave_password_length (1 byte) + password (N bytes)      │
  │ slave_mysql_port (2 bytes)                               │
  │ replication_rank (4 bytes)                               │
  │ master_id (4 bytes)                                      │
  └──────────────────────────────────────────────────────────┘
```

## 4.2 Canal 实现

```java
public class RegisterSlaveCommand {
    
    private static final byte COM_REGISTER_SLAVE = 0x15;
    
    public static byte[] build(long slaveId) {
        ByteArrayOutputStream buffer = new ByteArrayOutputStream();
        
        try {
            // 命令类型
            buffer.write(COM_REGISTER_SLAVE);
            
            // server_id (4 bytes, little-endian)
            writeInt4(buffer, slaveId);
            
            // hostname (canal 发送空字符串)
            buffer.write(0);
            
            // username (空字符串)
            buffer.write(0);
            
            // password (空字符串)
            buffer.write(0);
            
            // port (0 = 不关心)
            writeInt2(buffer, 0);
            
            // replication rank (0 = 不参与选举)
            writeInt4(buffer, 0);
            
            // master_id (0 = 不需要)
            writeInt4(buffer, 0);
            
        } catch (IOException e) {
            throw new RuntimeException(e);
        }
        
        return buffer.toByteArray();
    }
}

// 在 Canal 中调用
public void registerSlave(MysqlConnection connection, long slaveId) {
    byte[] command = RegisterSlaveCommand.build(slaveId);
    connection.sendCommand(command);
    
    // 读取响应
    Packet response = connection.readPacket();
    
    if (response.getPayload()[0] == 0xFF) {
        // 错误
        throw new IOException("COM_REGISTER_SLAVE failed: " + response.getErrorMessage());
    }
    
    logger.info("successfully registered as slave, server_id={}", slaveId);
}
```

## 4.3 注册的意义

```
mysql> SHOW SLAVE HOSTS;
+-----------+------+------+-----------+--------------------------------------+
| Server_id | Host | Port | Master_id | Slave_UUID                           |
+-----------+------+------+-----------+--------------------------------------+
|      1234 |      |    0 |         0 | 3e11fa47-71ca-11e1-9e33-c80aa9429562 |
+-----------+------+------+-----------+--------------------------------------+
```

注册为 Slave 后：
- MySQL Master 会在 `SHOW SLAVE HOSTS` 中显示
- 如果启用半同步复制，Master 会等待 Slave 的 ACK
- 用于 GTID 模式下追踪每个 Slave 的执行进度

# 5. COM_BINLOG_DUMP — 订阅 Binlog

## 5.1 命令格式

这是 Canal 最核心的命令，向 MySQL Master 请求订阅 Binlog 事件流：

```
COM_BINLOG_DUMP (0x12)

Payload (传统模式):
  ┌──────────────────────────────────────────────────────────┐
  │ binlog_position (4 bytes)                                │
  │ flags (2 bytes)                                          │
  │ server_id (4 bytes)                                      │
  │ binlog_filename (N bytes, null-terminated)               │
  └──────────────────────────────────────────────────────────┘

Payload (GTID 模式):
  ┌──────────────────────────────────────────────────────────┐
  │ binlog_position (4 bytes): 固定为 4                      │
  │ flags (2 bytes): 固定为 0x04 (BINLOG_THROUGH_GTID)       │
  │ server_id (4 bytes)                                      │
  │ binlog_filename (N bytes, null-terminated): 长度固定为 0 │
  │ gtid_set (N bytes, null-terminated)                     │
  └──────────────────────────────────────────────────────────┘
```

## 5.2 Canal 实现

```java
public class BinlogDumpCommand {
    
    private static final byte COM_BINLOG_DUMP = 0x12;
    private static final short BINLOG_DUMP_NON_BLOCK = 0x01;
    private static final short BINLOG_THROUGH_GTID = 0x04;
    
    /**
     * 构造传统模式的 COM_BINLOG_DUMP 命令
     */
    public static byte[] build(String binlogFilename, long binlogPosition, 
                                long slaveId) {
        ByteArrayOutputStream buffer = new ByteArrayOutputStream();
        
        try {
            // 命令类型
            buffer.write(COM_BINLOG_DUMP);
            
            // binlog 位置（4 bytes）
            writeInt4(buffer, binlogPosition);
            
            // flags（0 = 阻塞模式，一直推送）
            writeInt2(buffer, 0);
            
            // server_id（4 bytes）
            writeInt4(buffer, slaveId);
            
            // binlog 文件名（null-terminated）
            buffer.write(binlogFilename.getBytes());
            buffer.write(0);  // null terminator
            
        } catch (IOException e) {
            throw new RuntimeException(e);
        }
        
        return buffer.toByteArray();
    }
    
    /**
     * 构造 GTID 模式的 COM_BINLOG_DUMP 命令
     */
    public static byte[] buildWithGtid(String gtidSet, long slaveId) {
        ByteArrayOutputStream buffer = new ByteArrayOutputStream();
        
        try {
            // 命令类型
            buffer.write(COM_BINLOG_DUMP);
            
            // binlog 位置（GTID 模式下固定为 4）
            writeInt4(buffer, 4);
            
            // flags（BINLOG_THROUGH_GTID）
            writeInt2(buffer, BINLOG_THROUGH_GTID);
            
            // server_id
            writeInt4(buffer, slaveId);
            
            // binlog 文件名（GTID 模式下长度为 0）
            buffer.write(0);  // 空字符串
            
            // GTID set（null-terminated）
            buffer.write(gtidSet.getBytes());
            buffer.write(0);  // null terminator
            
        } catch (IOException e) {
            throw new RuntimeException(e);
        }
        
        return buffer.toByteArray();
    }
}
```

## 5.3 Canal 中的调用

```java
// MysqlConnection 中发送 Binlog Dump 命令
public void sendBinlogDump(String binlogFilename, long binlogPosition,
                            String gtidSet, long slaveId) {
    
    byte[] command;
    
    if (StringUtils.isNotEmpty(gtidSet)) {
        // GTID 模式
        command = BinlogDumpCommand.buildWithGtid(gtidSet, slaveId);
        logger.info("send binlog dump with GTID: {}", gtidSet);
    } else {
        // 传统模式
        command = BinlogDumpCommand.build(binlogFilename, binlogPosition, slaveId);
        logger.info("send binlog dump: {}@{}", binlogFilename, binlogPosition);
    }
    
    // 发送命令
    sendCommand(command);
    
    // 等待第一个响应（成功或失败）
    Packet response = readPacket();
    
    if (response.getPayload()[0] == (byte) 0xFF) {
        // 错误响应
        String errorMsg = response.getErrorMessage();
        throw new BinlogDumpException("Binlog dump failed: " + errorMsg);
    }
    
    // 如果返回 OK 包，之后 MySQL 将持续推送 Binlog Event
    logger.info("binlog dump started successfully");
}
```

# 6. Binlog Event 流接收

## 6.1 Event 包结构

发送 `COM_BINLOG_DUMP` 后，MySQL Master 将持续推送 Binlog Event，每个 Event 的包结构如下：

```
┌──────────────────────────────────────────────────────────┐
│  MySQL Packet Header (4 bytes)                            │
│  ├─ Payload Length (3 bytes)                              │
│  └─ Sequence ID (1 byte)                                  │
├──────────────────────────────────────────────────────────┤
│  Binlog Event Payload                                     │
│  ┌───────────────────────────────────────────────────┐   │
│  │ OK Byte (1 byte): 固定 0x00                        │   │
│  │ Event Data (N bytes): Binlog Event 序列化数据       │   │
│  └───────────────────────────────────────────────────┘   │
└──────────────────────────────────────────────────────────┘
```

## 6.2 Event 接收循环

```java
public class BinlogEventReceiver {
    
    private MysqlConnection connection;
    private PacketReader packetReader;
    
    /**
     * 循环接收 Binlog Event
     */
    public void startReceive(EventHandler handler) {
        
        while (running) {
            try {
                // 1. 读取包头
                PacketHeader header = packetReader.readHeader();
                
                if (header.getPayloadLength() == 0) {
                    // 可能是心跳包
                    continue;
                }
                
                // 2. 读取 Payload
                byte[] payload = packetReader.readPayload(header.getPayloadLength());
                
                // 3. 检查第一个字节
                // 0x00: Binlog Event
                // 0xFE: EOF（Binlog 文件结束）
                // 0xFF: Error
                switch (payload[0]) {
                    case (byte) 0x00:
                        // 正常的 Binlog Event
                        BinlogEvent event = parseBinlogEvent(payload, 1);
                        handler.onEvent(event);
                        break;
                        
                    case (byte) 0xFE:
                        // Binlog 文件结束，等待 ROTATE_EVENT
                        handler.onEndOfFile();
                        break;
                        
                    case (byte) 0xFF:
                        // 错误
                        String error = new String(payload, 1, payload.length - 1);
                        handler.onError(new BinlogDumpException(error));
                        return;
                        
                    default:
                        logger.warn("unknown payload type: {}", payload[0]);
                }
                
            } catch (SocketTimeoutException e) {
                // 读取超时，可能是空闲状态
                handler.onIdle();
            } catch (IOException e) {
                handler.onError(e);
                break;
            }
        }
    }
}
```

# 7. 半同步复制支持

## 7.1 半同步复制原理

```
标准复制 (异步)：
Master 写 Binlog ──> 返回 Client OK
                └──> 推送 Binlog 给 Slave（异步，可能丢失）

半同步复制 (Semi-Sync)：
Master 写 Binlog ──> 推送 Binlog 给 Slave
                            │
                    等待 Slave ACK  (超时后降级为异步)
                            │
                    返回 Client OK
```

## 7.2 Canal 的半同步 ACK

```java
public class SemiSyncAckSender {
    
    private static final byte COM_SEMI_SYNC_ACK = (byte) 0xEF;
    
    /**
     * 向 Master 发送半同步复制 ACK
     */
    public void sendAck(MysqlConnection connection, long binlogPosition) {
        
        ByteArrayOutputStream buffer = new ByteArrayOutputStream();
        
        // 半同步 ACK 命令
        buffer.write(COM_SEMI_SYNC_ACK);
        
        // ACK 的 binlog 位置 (8 bytes)
        writeLong8(buffer, binlogPosition);
        
        byte[] payload = buffer.toByteArray();
        connection.sendPacket(payload, SEMI_SYNC_SEQUENCE_ID);
    }
}
```

# 8. 心跳保持

当 MySQL Master 没有数据变更时，Canal 需要定期发送查询来保持连接：

```java
public class HeartbeatKeeper {
    
    private static final byte COM_QUERY = 0x03;
    private static final String HEARTBEAT_SQL = "SELECT 1";
    
    private long lastEventTime;
    private long heartbeatInterval = 30000;  // 30s
    
    /**
     * 心跳检测与发送
     */
    public void checkAndSendHeartbeat(MysqlConnection connection) {
        
        long now = System.currentTimeMillis();
        
        if (now - lastEventTime > heartbeatInterval) {
            // 发送心跳查询
            sendHeartbeat(connection);
            lastEventTime = now;
        }
    }
    
    private void sendHeartbeat(MysqlConnection connection) {
        ByteArrayOutputStream buffer = new ByteArrayOutputStream();
        buffer.write(COM_QUERY);
        buffer.write(HEARTBEAT_SQL.getBytes());
        
        connection.sendCommand(buffer.toByteArray());
        
        // 接收心跳响应
        Packet response = connection.readPacket();
        
        // 更新最后事件时间
        lastEventTime = System.currentTimeMillis();
    }
}
```

# 9. 完整的连接生命周期

```java
public class MysqlConnection implements CanalLifeCycle {

    private SocketChannel channel;
    private OutputStream out;
    private InputStream in;
    
    // 认证信息
    private String username;
    private String password;
    private long slaveId;
    
    /**
     * 完整的连接生命周期
     */
    public void fullLifecycle() {
        
        try {
            // ====== 1. 建立 TCP 连接 ======
            channel = SocketChannel.open(new InetSocketAddress(host, port));
            out = Channels.newOutputStream(channel);
            in = Channels.newInputStream(channel);
            
            // ====== 2. 接收握手包 ======
            HandshakePacket handshake = HandshakePacket.parse(in);
            logger.info("connected to MySQL {}, capabilities=0x{}",
                handshake.getServerVersion(),
                Integer.toHexString(handshake.getServerCapabilities()));
            
            // ====== 3. 发送认证响应 ======
            HandshakeResponse41 auth = HandshakeResponse41.build(
                handshake, username, password);
            sendPacket(auth.toBytes(), 1);
            
            // ====== 4. 验证认证结果 ======
            Packet authResult = readPacket();
            if (!isAuthSuccess(authResult)) {
                throw new IOException("Auth failed");
            }
            logger.info("authentication success");
            
            // ====== 5. 注册为 Slave ======
            sendCommand(RegisterSlaveCommand.build(slaveId), 0);
            Packet regResult = readPacket();
            if (!isOk(regResult)) {
                throw new IOException("Register slave failed");
            }
            logger.info("registered as slave, id={}", slaveId);
            
            // ====== 6. 发送 Binlog Dump ======
            sendBinlogDump(binlogFile, binlogPos, gtidSet, slaveId);
            
            // ====== 7. 循环接收 Binlog Event ======
            while (running) {
                Packet eventPacket = readPacket();
                processBinlogEvent(eventPacket);
            }
            
        } catch (IOException e) {
            logger.error("connection error", e);
            // 触发重连
            reconnect();
        } finally {
            close();
        }
    }
}
```

# 10. dbsync 驱动

Canal 的底层 MySQL 协议模拟使用的是 `dbsync` 模块，它基于 `open-replicator` 项目演化而来：

```java
// dbsync 模块的核心依赖
public class MysqlConnection extends ErosaConnection {
    
    // 基于 Netty 或直接 Socket 实现
    private Channel channel;
    
    public MysqlConnection(InetSocketAddress address, String username, String password) {
        this.address = address;
        this.username = username;
        this.password = password;
    }
    
    @Override
    public void connect() throws IOException {
        // 建立连接
        Socket socket = new Socket();
        socket.connect(address, 30000);
        
        this.channel = new BioSocketChannel(socket);
        
        // 认证
        negotiate(channel);
    }
    
    /**
     * MySQL 协议协商
     */
    public void negotiate(Channel channel) throws IOException {
        
        // 1. 创建 MySQL 协议协商器
        MysqlConnector connector = new MysqlConnector();
        
        // 2. 执行协商
        MysqlConnectionConfig config = new MysqlConnectionConfig();
        config.setUsername(username);
        config.setPassword(password);
        config.setSlaveId(slaveId);
        
        connector.connect(channel, config);
    }
    
    /**
     * 发送 Binlog Dump 请求
     */
    public void dump(String binlogFilename, Long binlogPosition, Long slaveId) 
            throws IOException {
        
        // 使用 dbsync 提供的 dump 方法
        DirectLogFetcher fetcher = new DirectLogFetcher(channel);
        fetcher.open(new DirectLogFetcher.DumpConfig(binlogFilename, binlogPosition, slaveId));
        
        // 接收 binlog 数据
        while (fetcher.fetch()) {
            LogEvent event = fetcher.getEvent();
            // 解析事件...
        }
    }
}
```

# 11. 协议的兼容性处理

## 11.1 MySQL 版本兼容

```java
public class MysqlVersionAdaptor {
    
    /**
     * 根据 MySQL 版本选择不同的认证方式
     */
    public static AuthMethod selectAuthMethod(String serverVersion) {
        Version version = Version.parse(serverVersion);
        
        if (version.compareTo(new Version("8.0.4")) >= 0) {
            return AuthMethod.CACHING_SHA2_PASSWORD;  // MySQL 8.0.4+ 默认
        } else if (version.compareTo(new Version("5.7.0")) >= 0) {
            return AuthMethod.MYSQL_NATIVE_PASSWORD;   // MySQL 5.7 默认
        } else {
            return AuthMethod.MYSQL_OLD_PASSWORD;       // 旧版本
        }
    }
    
    /**
     * 根据版本判断是否支持 GTID
     */
    public static boolean supportsGtid(String serverVersion) {
        Version version = Version.parse(serverVersion);
        return version.compareTo(new Version("5.6.5")) >= 0;
    }
}
```

## 11.2 阿里云 RDS 的特殊处理

```java
public class RdsConnectionAdaptor {
    
    /**
     * 对阿里云 RDS 的特殊处理
     */
    public static MysqlConnection createRdsConnection(String accessKey, 
                                                       String secretKey,
                                                       String instanceId) {
        
        // RDS 不支持直接的 COM_BINLOG_DUMP
        // 需要通过 RDS API 获取 Binlog 备份文件
        // 或者在 RDS 控制台开启 Binlog 订阅功能
        
        RDSClient client = new RDSClient(accessKey, secretKey);
        
        // 获取可用的 binlog 文件列表
        List<BinlogFile> binlogFiles = client.describeBinlogFiles(instanceId);
        
        // 下载最新的 binlog 文件
        BinlogFile latestFile = binlogFiles.get(binlogFiles.size() - 1);
        String downloadUrl = client.getBinlogDownloadUrl(instanceId, latestFile);
        
        // 返回一个封装后的连接
        return new RdsMysqlConnection(client, downloadUrl);
    }
}
```

# 12. 认证插件支持

Canal 支持多种 MySQL 认证插件：

```java
public enum AuthPlugin {
    
    MYSQL_NATIVE_PASSWORD("mysql_native_password") {
        @Override
        public byte[] scramble(String password, byte[] challenge) {
            // SHA1(password) XOR SHA1(challenge + SHA1(SHA1(password)))
            return mysqlNativePasswordScramble(password, challenge);
        }
    },
    
    CACHING_SHA2_PASSWORD("caching_sha2_password") {
        @Override
        public byte[] scramble(String password, byte[] challenge) {
            // SHA256(password) XOR SHA256(challenge + SHA256(SHA256(password)))
            return cachingSha2PasswordScramble(password, challenge);
        }
    },
    
    MYSQL_CLEAR_PASSWORD("mysql_clear_password") {
        @Override
        public byte[] scramble(String password, byte[] challenge) {
            // 明文密码 + null terminator
            return (password + "\0").getBytes();
        }
    };
    
    public abstract byte[] scramble(String password, byte[] challenge);
}
```
