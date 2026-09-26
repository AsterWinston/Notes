
# 1. 概述

Binlog 解析是 Canal 最核心的技术环节。MySQL 所产生的 Binlog 事件是以高效的二进制格式编码的，Canal 需要将其解码为结构化的 Java 对象，还原出用户能够理解的行级数据变更信息。

```
MySQL Binlog (二进制字节流)
        │
        ▼
┌───────────────────────────────────────┐
│        dbsync / open-replicator       │
│                                       │
│  ┌─────────┐   ┌──────────────┐      │
│  │ LogEvent│──>│  LogDecoder  │      │
│  │ Fetcher │   │  (事件解码器) │      │
│  └─────────┘   └──────┬───────┘      │
│                       │               │
│               ┌───────▼───────┐      │
│               │   Event       │      │
│               │   Parsers     │      │
│               └───────────────┘      │
│                       │               │
└───────────────────────┼───────────────┘
                        │
                        ▼
              CanalEntry.Entry
              (结构化数据)
```

# 2. Binlog Event 的二进制格式

## 2.1 Event Header（事件头）

每个 Binlog Event 都有一个公共的 19 字节事件头（V4 格式）：

```
Event Header (19 bytes, V4):

Offset  Size    Description
─────────────────────────────
0       4       timestamp (Unix 时间戳)
4       1       event_type (事件类型)
5       4       server_id (产生事件的服务器 ID)
9       4       event_size (整个事件的大小，包含 header)
13      4       log_pos (下一个事件在 binlog 中的位置)
17      2       flags (事件标志)
        --
        19 bytes total
```

## 2.2 Event Header 解析

```java
public class LogEventHeader {
    
    private long timestamp;       // 4 bytes, little-endian unsigned
    private int eventType;        // 1 byte
    private long serverId;        // 4 bytes, little-endian unsigned
    private long eventLength;     // 4 bytes, little-endian unsigned
    private long nextPosition;    // 4 bytes, little-endian unsigned
    private int flags;            // 2 bytes, little-endian unsigned
    
    /**
     * 从字节流解析 Event Header
     */
    public static LogEventHeader parse(byte[] buffer, int offset) {
        LogEventHeader header = new LogEventHeader();
        
        header.timestamp = ByteHelper.readUnsignedInt(buffer, offset);     // 4 bytes
        offset += 4;
        
        header.eventType = buffer[offset] & 0xFF;                          // 1 byte
        offset += 1;
        
        header.serverId = ByteHelper.readUnsignedInt(buffer, offset);      // 4 bytes
        offset += 4;
        
        header.eventLength = ByteHelper.readUnsignedInt(buffer, offset);   // 4 bytes
        offset += 4;
        
        header.nextPosition = ByteHelper.readUnsignedInt(buffer, offset);  // 4 bytes
        offset += 4;
        
        header.flags = ByteHelper.readUnsignedShort(buffer, offset);       // 2 bytes
        
        return header;
    }
}
```

## 2.3 Event Header Flags

```java
public enum LogEventFlag {
    // 标志位定义
    LOG_EVENT_BINLOG_IN_USE_F           (0x0001),
    LOG_EVENT_FORCED_ROTATE_F           (0x0002),
    LOG_EVENT_THREAD_SPECIFIC_F         (0x0004),
    LOG_EVENT_SUPPRESS_USE_F            (0x0008),
    LOG_EVENT_UPDATE_TABLE_MAP_VERSION_F(0x0010),
    LOG_EVENT_ARTIFICIAL_F              (0x0020),
    LOG_EVENT_RELAY_LOG_F               (0x0040),
    LOG_EVENT_IGNORABLE_F               (0x0080),
    LOG_EVENT_NO_FILTER_F               (0x0100),
    LOG_EVENT_MTS_ISOLATE_F             (0x0200);
}
```

# 3. 各种事件类型的解析

## 3.1 FORMAT_DESCRIPTION_EVENT — 格式描述事件

每个 Binlog 文件的第一个事件，描述 Binlog 版本信息：

```java
public class FormatDescriptionLogEvent extends LogEvent {
    
    private int binlogVersion;      // Binlog 版本（固定 4）
    private String serverVersion;   // MySQL 服务器版本
    private long timestamp;         // Binlog 文件创建时间
    private int headerLength;       // Event Header 长度
    private byte[] postHeaderLen;   // 各事件类型的 post-header 长度表
    private int checksumType;       // 校验和类型（0=NONE, 1=CRC32）
    
    @Override
    public void parse(byte[] body) {
        int offset = 0;
        
        // Binlog 版本 (2 bytes)
        binlogVersion = ByteHelper.readUnsignedShort(body, offset);
        offset += 2;
        
        // 服务器版本 (50 bytes, null-terminated)
        serverVersion = new String(body, offset, 50).trim();
        offset += 50;
        
        // 文件创建时间戳 (4 bytes)
        timestamp = ByteHelper.readUnsignedInt(body, offset);
        offset += 4;
        
        // Event Header 长度 (1 byte)
        headerLength = body[offset] & 0xFF;
        offset += 1;
        
        // Post-header 长度表
        // MySQL 5.6+: 固定的 39 个事件类型
        // MySQL 5.7+: 有额外的事件类型
        int postHeaderLenCount = body.length - offset - (hasChecksum() ? 5 : 1);
        postHeaderLen = new byte[postHeaderLenCount];
        System.arraycopy(body, offset, postHeaderLen, 0, postHeaderLenCount);
        offset += postHeaderLenCount;
        
        // 校验和
        if (hasChecksum()) {
            checksumType = body[offset] & 0xFF;
        }
    }
}
```

## 3.2 TABLE_MAP_EVENT — 表映射事件

在 ROW 事件之前，用于描述表的结构信息：

```java
public class TableMapLogEvent extends LogEvent {
    
    private long tableId;           // 表 ID
    private int flags;              // 标志位
    private String database;        // 数据库名
    private int databaseLength;     // 数据库名长度
    private String tableName;       // 表名
    private int tableNameLength;    // 表名长度
    private int columnCount;        // 列数
    private int[] columnTypes;      // 列类型数组
    private int[] columnMetadata;   // 列元数据数组
    private boolean[] nullability;  // 列可空数组
    
    @Override
    public void parse(byte[] body) {
        int offset = 0;
        
        // 表 ID (6 bytes for V4+, 4 bytes for V1)
        tableId = ByteHelper.readLong(body, offset, 6);
        offset += 6;
        
        // Flags (2 bytes)
        flags = ByteHelper.readUnsignedShort(body, offset);
        offset += 2;
        
        // 数据库名长度 (1 byte)
        databaseLength = body[offset] & 0xFF;
        offset += 1;
        
        // 数据库名 (databaseLength bytes, null-terminated)
        database = new String(body, offset, databaseLength);
        offset += databaseLength + 1;
        
        // 表名长度 (1 byte)
        tableNameLength = body[offset] & 0xFF;
        offset += 1;
        
        // 表名
        tableName = new String(body, offset, tableNameLength);
        offset += tableNameLength + 1;
        
        // 列数 (Packed Integer)
        columnCount = ByteHelper.readPackedInteger(body, offset);
        offset = ByteHelper.getPackedIntegerLength();
        
        // 列类型 (1 byte each)
        columnTypes = new int[columnCount];
        for (int i = 0; i < columnCount; i++) {
            columnTypes[i] = body[offset + i] & 0xFF;
        }
        offset += columnCount;
        
        // 列元数据 (变长)
        columnMetadata = readColumnMetadata(body, offset);
        offset += calculateMetadataLength();
        
        // 列可空性 (bitmap)
        nullability = readNullabilityBitmap(body, offset);
    }
}
```

## 3.3 ROWS_EVENT — 行数据事件

这是 Canal 最关注的事件类型，包含实际的数据变更：

```java
public class RowsLogEvent extends LogEvent {
    
    private long tableId;
    private int tableFlags;
    private int extraDataLength;
    private byte[] extraData;
    private int columnCount;
    private boolean[] columnsPresent;       // 列存在 bitmap
    private boolean[] columnsPresentForUpdate; // UPDATE 的 before 列 bitmap
    
    // 行数据（解析后的结果）
    private List<Row> beforeRows;
    private List<Row> afterRows;
    
    @Override
    public void parse(byte[] body) {
        int offset = 0;
        
        // 表 ID (6 bytes for V2, 可变)
        tableId = readTableId(body, offset);
        offset += getTableIdLength();
        
        // Flags (2 bytes)
        offset += 2;
        
        // 额外数据长度 (v2: 2 bytes)
        if (isVersion2()) {
            extraDataLength = ByteHelper.readUnsignedShort(body, offset);
            offset += 2;
            extraData = new byte[extraDataLength];
            System.arraycopy(body, offset, extraData, 0, extraDataLength);
            offset += extraDataLength;
        }
        
        // 列数 (Packed Integer)
        columnCount = ByteHelper.readPackedInteger(body, offset);
        offset = ByteHelper.getPackedIntegerLength();
        
        // 列存在 bitmap
        int bitmapSize = (columnCount + 7) / 8;
        columnsPresent = readBitmap(body, offset, bitmapSize);
        offset += bitmapSize;
        
        // UPDATE 事件的 before 列 bitmap
        if (getEventType() == LogEvent.UPDATE_ROWS_EVENT_V2) {
            columnsPresentForUpdate = readBitmap(body, offset, bitmapSize);
            offset += bitmapSize;
        }
        
        // ===== 行数据解析 =====
        beforeRows = new ArrayList<>();
        afterRows = new ArrayList<>();
        
        while (offset < body.length) {
            switch (getEventType()) {
                case LogEvent.WRITE_ROWS_EVENT_V2:
                    // INSERT: 只有 after image
                    Row after = parseRow(body, offset, columnsPresent);
                    afterRows.add(after);
                    offset += calculateRowLength(after);
                    break;
                    
                case LogEvent.DELETE_ROWS_EVENT_V2:
                    // DELETE: 只有 before image
                    Row before = parseRow(body, offset, columnsPresent);
                    beforeRows.add(before);
                    offset += calculateRowLength(before);
                    break;
                    
                case LogEvent.UPDATE_ROWS_EVENT_V2:
                    // UPDATE: 有 before 和 after image
                    Row beforeUpdate = parseRow(body, offset, columnsPresentForUpdate);
                    offset += calculateRowLength(beforeUpdate);
                    Row afterUpdate = parseRow(body, offset, columnsPresent);
                    afterRows.add(afterUpdate);
                    beforeRows.add(beforeUpdate);
                    offset += calculateRowLength(afterUpdate);
                    break;
            }
        }
    }
    
    /**
     * 解析一行数据
     */
    private Row parseRow(byte[] body, int offset, boolean[] columnsPresent) {
        Row row = new Row();
        row.setColumnCount(columnCount);
        row.setColumns(new ColumnValue[columnCount]);
        
        // 读取 null bitmap
        int nullBitmapSize = (columnCount + 7) / 8;
        boolean[] nullBitmap = readNullBitmap(body, offset, nullBitmapSize);
        offset += nullBitmapSize;
        
        // 读取每列的值
        for (int i = 0; i < columnCount; i++) {
            if (!columnsPresent[i]) {
                // 列不在本事件中
                row.getColumns()[i] = ColumnValue.ABSENT;
                continue;
            }
            
            if (nullBitmap[i]) {
                // NULL 值
                row.getColumns()[i] = ColumnValue.NULL;
                continue;
            }
            
            // 根据列类型读取实际值
            int columnType = getTableMap().getColumnTypes()[i];
            int columnMeta = getTableMap().getColumnMetadata()[i];
            
            byte[] value = readColumnValue(body, offset, columnType, columnMeta);
            row.getColumns()[i] = new ColumnValue(value, columnType);
            offset += value.length;
        }
        
        return row;
    }
}
```

# 4. 各列数据类型的解码

## 4.1 数值类型解码

```java
public class NumericTypeDecoder {
    
    /**
     * 解码整数类型
     */
    public static long decodeInt(byte[] buffer, int offset, int length, boolean unsigned) {
        switch (length) {
            case 1:
                return unsigned ? (buffer[offset] & 0xFF) : buffer[offset];
            case 2:
                return unsigned ? ByteHelper.readUnsignedShort(buffer, offset)
                                : ByteHelper.readShort(buffer, offset);
            case 3:
                return unsigned ? ByteHelper.readUnsignedInt24(buffer, offset)
                                : ByteHelper.readInt24(buffer, offset);
            case 4:
                return unsigned ? ByteHelper.readUnsignedInt(buffer, offset)
                                : ByteHelper.readInt(buffer, offset);
            case 8:
                return ByteHelper.readLong(buffer, offset);
            default:
                throw new IllegalArgumentException("unsupported int length: " + length);
        }
    }
    
    /**
     * 解码浮点数
     */
    public static float decodeFloat(byte[] buffer, int offset) {
        int bits = ByteHelper.readInt(buffer, offset);
        return Float.intBitsToFloat(bits);
    }
    
    /**
     * 解码双精度浮点数
     */
    public static double decodeDouble(byte[] buffer, int offset) {
        long bits = ByteHelper.readLong(buffer, offset);
        return Double.longBitsToDouble(bits);
    }
    
    /**
     * 解码 DECIMAL 类型
     */
    public static String decodeDecimal(byte[] buffer, int offset, int precision, int scale) {
        // DECIMAL 使用 Binary-Encoded Decimal 格式
        // 每 9 位十进制数编码为 4 字节
        int integral = precision - scale;
        int fractional = scale;
        
        int integralSize = (integral + 8) / 9;
        int fractionalSize = (fractional + 8) / 9;
        int totalSize = integralSize + fractionalSize;
        
        int[] parts = new int[totalSize * 4];
        for (int i = 0; i < totalSize * 4; i++) {
            parts[i] = buffer[offset + i] & 0xFF;
        }
        
        return BigDecimalDecoder.decode(parts, precision, scale).toString();
    }
    
    /**
     * 解码 BIT 类型
     */
    public static long decodeBit(byte[] buffer, int offset, int length) {
        long value = 0;
        for (int i = 0; i < length; i++) {
            value = (value << 8) | (buffer[offset + i] & 0xFF);
        }
        return value;
    }
}
```

## 4.2 字符串类型解码

```java
public class StringTypeDecoder {
    
    /**
     * 解码变长字符串 (VARCHAR, VARBINARY)
     */
    public static String decodeVarString(byte[] buffer, int offset, 
                                          int metadata, String charset) {
        // metadata 包含最大长度
        int maxLength = metadata;
        int length;
        
        if (maxLength > 255) {
            // 使用 2 字节长度前缀
            length = ByteHelper.readUnsignedShort(buffer, offset);
            offset += 2;
        } else {
            // 使用 1 字节长度前缀
            length = buffer[offset] & 0xFF;
            offset += 1;
        }
        
        return new String(buffer, offset, length, Charset.forName(charset));
    }
    
    /**
     * 解码 BLOB/TEXT 类型
     */
    public static String decodeBlob(byte[] buffer, int offset, int metadata, 
                                     String charset) {
        // metadata 的第1字节表示长度前缀大小
        int lengthSize = metadata & 0xFF;
        int length;
        
        switch (lengthSize) {
            case 1:
                length = buffer[offset] & 0xFF;
                offset += 1;
                break;
            case 2:
                length = ByteHelper.readUnsignedShort(buffer, offset);
                offset += 2;
                break;
            case 3:
                length = ByteHelper.readUnsignedInt24(buffer, offset);
                offset += 3;
                break;
            case 4:
                length = ByteHelper.readInt(buffer, offset);
                offset += 4;
                break;
            default:
                throw new IllegalArgumentException("unsupported blob length size: " + lengthSize);
        }
        
        // BLOB 类型返回十六进制表示
        if (isBinaryType(metadata)) {
            return ByteHelper.toHexString(buffer, offset, length);
        }
        
        // TEXT 类型返回字符串
        return new String(buffer, offset, length, Charset.forName(charset));
    }
}
```

## 4.3 时间类型解码

```java
public class TimeTypeDecoder {
    
    /**
     * 解码 DATE 类型 (3 bytes)
     */
    public static String decodeDate(byte[] buffer, int offset) {
        int value = ByteHelper.readUnsignedInt24(buffer, offset);
        
        // value = year * 16 * 32 + month * 32 + day
        int day = value % 32;
        value >>= 5;
        int month = value % 16;
        value >>= 4;
        int year = value;
        
        return String.format("%04d-%02d-%02d", year, month, day);
    }
    
    /**
     * 解码 TIME 类型 (3 bytes)
     */
    public static String decodeTime(byte[] buffer, int offset) {
        int value = ByteHelper.readUnsignedInt24(buffer, offset);
        
        // value = hours * 10000 + minutes * 100 + seconds
        int hours = value / 10000;
        value %= 10000;
        int minutes = value / 100;
        int seconds = value % 100;
        
        return String.format("%s%02d:%02d:%02d",
            hours < 0 ? "-" : "",
            Math.abs(hours), minutes, seconds);
    }
    
    /**
     * 解码 TIME2 类型 (MySQL 5.6+)
     * 支持微秒精度
     */
    public static String decodeTime2(byte[] buffer, int offset, int fractionalSeconds) {
        // big-endian 编码
        long value = ByteHelper.readBigEndianLong(buffer, offset, 3);
        offset += 3;
        
        boolean negative = (value & 0x800000) != 0;
        if (negative) value = ~value & 0x7FFFFF;
        
        int hours = (int) (value / 10000);
        value %= 10000;
        int minutes = (int) (value / 100);
        int seconds = (int) (value % 100);
        
        StringBuilder result = new StringBuilder();
        result.append(negative ? "-" : "");
        result.append(String.format("%02d:%02d:%02d", hours, minutes, seconds));
        
        // 微秒部分
        if (fractionalSeconds > 0) {
            int fraction = ByteHelper.readBigEndianInt(buffer, offset, 
                (fractionalSeconds + 1) / 2);
            result.append(".").append(String.format("%0" + fractionalSeconds + "d", fraction));
        }
        
        return result.toString();
    }
    
    /**
     * 解码 DATETIME2 类型 (MySQL 5.6+)
     */
    public static String decodeDatetime2(byte[] buffer, int offset, int fractionalSeconds) {
        // 5 bytes, big-endian
        long value = ByteHelper.readBigEndianLong(buffer, offset, 5);
        offset += 5;
        
        // value = year_month * 2^22 + day_hour * 2^17 + minutes_seconds * 2^12 + unused
        int yearMonth = (int) (value >> 22);
        value &= 0x3FFFFF;
        
        int dayHour = (int) (value >> 17);
        value &= 0x1FFFF;
        
        int minutesSeconds = (int) (value >> 12);
        
        int year = yearMonth / 13;
        int month = yearMonth % 13;
        int day = dayHour / 24;
        int hour = dayHour % 24;
        int minute = minutesSeconds / 60;
        int second = minutesSeconds % 60;
        
        StringBuilder result = new StringBuilder();
        result.append(String.format("%04d-%02d-%02d %02d:%02d:%02d",
            year, month, day, hour, minute, second));
        
        // 微秒部分
        if (fractionalSeconds > 0) {
            int fraction = ByteHelper.readBigEndianInt(buffer, offset,
                (fractionalSeconds + 1) / 2);
            result.append(".").append(String.format("%0" + fractionalSeconds + "d", fraction));
        }
        
        return result.toString();
    }
    
    /**
     * 解码 TIMESTAMP2 类型 (MySQL 5.6+)
     */
    public static String decodeTimestamp2(byte[] buffer, int offset, int fractionalSeconds) {
        // 4 bytes, big-endian (Unix 时间戳)
        long unixTimestamp = ByteHelper.readBigEndianUnsignedInt(buffer, offset);
        offset += 4;
        
        // 将 Unix 时间戳转换为日期时间
        Instant instant = Instant.ofEpochSecond(unixTimestamp);
        LocalDateTime dateTime = LocalDateTime.ofInstant(instant, ZoneOffset.UTC);
        
        StringBuilder result = new StringBuilder();
        result.append(dateTime.format(DateTimeFormatter.ofPattern("yyyy-MM-dd HH:mm:ss")));
        
        // 微秒部分
        if (fractionalSeconds > 0) {
            int fraction = ByteHelper.readBigEndianInt(buffer, offset,
                (fractionalSeconds + 1) / 2);
            result.append(".").append(String.format("%0" + fractionalSeconds + "d", fraction));
        }
        
        return result.toString();
    }
}
```

# 5. 编码与字符集处理

```java
public class CharsetHandler {
    
    /**
     * MySQL 字符集编号 → Java Charset 映射
     */
    private static final Map<Integer, String> MYSQL_TO_JAVA_CHARSET = new HashMap<>();
    
    static {
        MYSQL_TO_JAVA_CHARSET.put(8, "LATIN1");        // latin1
        MYSQL_TO_JAVA_CHARSET.put(33, "UTF-8");        // utf8mb3
        MYSQL_TO_JAVA_CHARSET.put(45, "UTF-8");        // utf8mb4 (Canonical)
        MYSQL_TO_JAVA_CHARSET.put(46, "UTF-8");        // utf8mb4
        MYSQL_TO_JAVA_CHARSET.put(63, "UTF-8");        // utf8 (alias)
        MYSQL_TO_JAVA_CHARSET.put(28, "GBK");          // gbk
        MYSQL_TO_JAVA_CHARSET.put(30, "GB2312");       // gb2312
        MYSQL_TO_JAVA_CHARSET.put(27, "BIG5");         // big5
        MYSQL_TO_JAVA_CHARSET.put(22, "EUC-KR");       // euckr
        MYSQL_TO_JAVA_CHARSET.put(95, "Cp932");        // cp932 (Shift-JIS)
    }
    
    public static String getJavaCharset(int mysqlCharsetId) {
        return MYSQL_TO_JAVA_CHARSET.getOrDefault(mysqlCharsetId, "UTF-8");
    }
}
```

# 6. Binlog Checksum 校验

## 6.1 校验和格式

MySQL 5.6+ 支持 Binlog 校验和，确保数据完整性：

```
Binlog Event 末尾附加：

┌──────────────────────────────────────────┐
│ Event Body                               │
├──────────────────────────────────────────┤
│ Checksum (4 bytes, CRC32)  ← 可选       │
│ - 如果 checksum_type = 0 (NONE):  没有   │
│ - 如果 checksum_type = 1 (CRC32):  4字节 │
└──────────────────────────────────────────┘
```

## 6.2 Checksum 验证

```java
public class ChecksumValidator {
    
    private static final int CHECKSUM_CRC32_LENGTH = 4;
    
    /**
     * 验证并处理校验和
     */
    public static byte[] validateChecksum(byte[] eventData, int eventLength, 
                                           int checksumType) {
        
        if (checksumType == CHECKSUM_TYPE_NONE) {
            return eventData;
        }
        
        if (checksumType == CHECKSUM_TYPE_CRC32) {
            int dataLength = eventLength - CHECKSUM_CRC32_LENGTH;
            
            // 计算 CRC32
            CRC32 crc32 = new CRC32();
            crc32.update(eventData, 0, dataLength);
            long expected = crc32.getValue();
            
            // 读取实际校验和
            long actual = ByteHelper.readUnsignedInt(eventData, dataLength);
            
            if (expected != actual) {
                throw new ChecksumException(
                    String.format("CRC32 checksum mismatch! expected=%d, actual=%d",
                        expected, actual));
            }
            
            // 截掉校验和部分
            return Arrays.copyOf(eventData, dataLength);
        }
        
        throw new IllegalArgumentException("unknown checksum type: " + checksumType);
    }
}
```

# 7. 大事件处理（拆包与组包）

当 Binlog Event 的大小超过 MySQL 协议的最大包大小（16MB - 1）时，会被拆分为多个包：

```java
public class LargeEventHandler {
    
    private static final int MAX_PACKET_SIZE = 0xFFFFFF;  // 16MB - 1
    
    /**
     * 处理大于 16MB 的 Event（拆包接收）
     */
    public static byte[] readLargeEvent(PacketReader reader, long eventLength) 
            throws IOException {
        
        ByteArrayOutputStream buffer = new ByteArrayOutputStream();
        long remaining = eventLength;
        
        while (remaining > 0) {
            Packet packet = reader.readPacket();
            
            int payloadLength = packet.getPayloadLength();
            byte[] payload = packet.getPayload();
            
            buffer.write(payload);
            remaining -= payloadLength;
            
            if (payloadLength < MAX_PACKET_SIZE && remaining == 0) {
                // 完整接收
                break;
            }
        }
        
        return buffer.toByteArray();
    }
}
```

# 8. LogFetcher — 底层 Binlog 流读取器

```java
public class DirectLogFetcher extends LogFetcher {
    
    private SocketChannel channel;
    private InputStream input;
    
    // 缓冲区
    private byte[] buffer;
    private int bufferPosition;
    private int bufferLimit;
    
    // Binlog 事件头缓存
    private LogEventHeader currentHeader;
    
    /**
     * 从流中获取下一个 Binlog Event
     */
    @Override
    public boolean fetch() throws IOException {
        
        while (true) {
            // 1. 读取 Event Header (19 bytes)
            readFully(LOG_EVENT_HEADER_LEN);
            
            // 2. 解析 Event Header
            currentHeader = LogEventHeader.parse(buffer, bufferPosition);
            
            // 3. 计算 Event Body 长度
            int bodyLength = (int) currentHeader.getEventLength() - LOG_EVENT_HEADER_LEN;
            
            // 4. 读取 Event Body
            readFully(bodyLength);
            
            // 5. 返回成功
            return true;
        }
    }
    
    /**
     * 获取当前 Event 的类型
     */
    public int getEventType() {
        return currentHeader.getEventType();
    }
    
    /**
     * 获取当前 Event 的数据
     */
    public byte[] getEventData() {
        int length = (int) currentHeader.getEventLength();
        byte[] data = new byte[length];
        System.arraycopy(buffer, bufferPosition, data, 0, length);
        return data;
    }
    
    /**
     * 确保缓冲区有足够的数据
     */
    private void ensureCapacity(int required) throws IOException {
        if (bufferLimit - bufferPosition < required) {
            // 紧凑缓冲区
            System.arraycopy(buffer, bufferPosition, buffer, 0, bufferLimit - bufferPosition);
            bufferLimit -= bufferPosition;
            bufferPosition = 0;
            
            // 读取更多数据
            int read = input.read(buffer, bufferLimit, buffer.length - bufferLimit);
            if (read == -1) {
                throw new EOFException("unexpected end of stream");
            }
            bufferLimit += read;
        }
    }
    
    private void readFully(int length) throws IOException {
        ensureCapacity(length);
        bufferPosition += length;
    }
}
```

# 9. LogDecoder — 事件工厂与路由

```java
public class LogDecoder {
    
    // 事件解析器注册表
    private final Map<Integer, Class<? extends LogEvent>> eventParsers;
    
    public LogDecoder() {
        eventParsers = new HashMap<>();
        
        // 注册各事件类型对应的解析器
        eventParsers.put(LogEvent.QUERY_EVENT, QueryLogEvent.class);
        eventParsers.put(LogEvent.XID_EVENT, XidLogEvent.class);
        eventParsers.put(LogEvent.TABLE_MAP_EVENT, TableMapLogEvent.class);
        eventParsers.put(LogEvent.WRITE_ROWS_EVENT_V1, WriteRowsLogEvent.class);
        eventParsers.put(LogEvent.UPDATE_ROWS_EVENT_V1, UpdateRowsLogEvent.class);
        eventParsers.put(LogEvent.DELETE_ROWS_EVENT_V1, DeleteRowsLogEvent.class);
        eventParsers.put(LogEvent.WRITE_ROWS_EVENT_V2, WriteRowsLogEvent.class);
        eventParsers.put(LogEvent.UPDATE_ROWS_EVENT_V2, UpdateRowsLogEvent.class);
        eventParsers.put(LogEvent.DELETE_ROWS_EVENT_V2, DeleteRowsLogEvent.class);
        eventParsers.put(LogEvent.FORMAT_DESCRIPTION_EVENT, FormatDescriptionLogEvent.class);
        eventParsers.put(LogEvent.ROTATE_EVENT, RotateLogEvent.class);
        eventParsers.put(LogEvent.GTID_LOG_EVENT, GtidLogEvent.class);
        eventParsers.put(LogEvent.ANONYMOUS_GTID_LOG_EVENT, GtidLogEvent.class);
        eventParsers.put(LogEvent.PREVIOUS_GTIDS_LOG_EVENT, PreviousGtidsLogEvent.class);
        eventParsers.put(LogEvent.HEARTBEAT_LOG_EVENT, HeartbeatLogEvent.class);
    }
    
    /**
     * 解码一个 Binlog Event
     */
    public LogEvent decode(byte[] eventData, FormatDescriptionLogEvent formatDescription) {
        
        // 1. 解析 Event Header
        LogEventHeader header = LogEventHeader.parse(eventData, 0);
        
        // 2. 提取 Event Body
        int bodyLength = (int) header.getEventLength() - formatDescription.getHeaderLength();
        byte[] body = new byte[bodyLength];
        System.arraycopy(eventData, formatDescription.getHeaderLength(), 
                         body, 0, bodyLength);
        
        // 3. 查找对应的解析器类
        Class<? extends LogEvent> parserClass = eventParsers.get(header.getEventType());
        
        if (parserClass == null) {
            // 未知事件类型，使用通用 Event
            return new UnknownLogEvent(header, body, formatDescription);
        }
        
        // 4. 实例化并解析
        try {
            LogEvent event = parserClass.newInstance();
            event.setHeader(header);
            event.setFormatDescription(formatDescription);
            event.parse(body);
            return event;
        } catch (Exception e) {
            throw new RuntimeException("failed to decode event type " + header.getEventType(), e);
        }
    }
}
```

# 10. JSON 数据类型处理

MySQL 5.7+ 引入了 JSON 数据类型，其在 Binlog 中的存储格式特殊：

```java
public class JsonTypeDecoder {
    
    /**
     * 解码 JSON 类型的 Binlog 值
     * JSON 在 Binlog 中以二进制 JSON 格式存储
     */
    public static String decodeJson(byte[] buffer, int offset, int length) {
        
        // JSON 二进制格式：
        // byte 0: 类型标志
        // bytes 1+: JSON 文档的二进制表示
        
        int jsonType = buffer[offset] & 0xFF;
        
        switch (jsonType) {
            case JSONB_TYPE_SMALL_DOCUMENT:
                return parseSmallDocument(buffer, offset + 1, length - 1);
            case JSONB_TYPE_LARGE_DOCUMENT:
                return parseLargeDocument(buffer, offset + 1, length - 1);
            case JSONB_TYPE_SMALL_ARRAY:
                return parseSmallArray(buffer, offset + 1, length - 1);
            case JSONB_TYPE_LARGE_ARRAY:
                return parseLargeArray(buffer, offset + 1, length - 1);
            default:
                return new String(buffer, offset, length, StandardCharsets.UTF_8);
        }
    }
    
    /**
     * 解析小型 JSON 文档
     */
    private static String parseSmallDocument(byte[] buffer, int offset, int length) {
        // JSONB 格式解析
        // element_count (2 bytes)
        // bytes (2 bytes) - 可变部分的起始偏移
        int elementCount = ByteHelper.readUnsignedShort(buffer, offset);
        offset += 2;
        int valueOffset = ByteHelper.readUnsignedShort(buffer, offset);
        offset += 2;
        
        // 读取 key-entry 和 value-entry
        // 每个 entry 包含 type(1B), key_offset(2B)/key_length(2B), value_info
        
        StringBuilder json = new StringBuilder("{");
        for (int i = 0; i < elementCount; i++) {
            if (i > 0) json.append(",");
            
            // 解析 key
            int keyInfo = ByteHelper.readUnsignedShort(buffer, offset + 1);
            String key = readString(buffer, offset, keyInfo);
            
            // 解析 value
            int valueType = buffer[offset + 3] & 0xFF;
            int valueInfo = ByteHelper.readUnsignedShort(buffer, offset + 4);
            String value = readValue(buffer, offset + 6, valueType, valueInfo);
            
            json.append("\"").append(key).append("\":").append(value);
        }
        json.append("}");
        
        return json.toString();
    }
}
```

# 11. GEOMETRY 类型处理

```java
public class GeometryTypeDecoder {
    
    /**
     * 解码 GEOMETRY 类型
     * 使用 WKB (Well-Known Binary) 格式
     */
    public static String decodeGeometry(byte[] buffer, int offset, int length) {
        // WKB 格式:
        // byte 0: byte order (01 = little-endian)
        // bytes 1-4: geometry type
        // bytes 5+: coordinates
        
        byte byteOrder = buffer[offset];
        int geometryType = ByteHelper.readInt(buffer, offset + 1, 4, byteOrder == 1);
        
        String typeName = getGeometryTypeName(geometryType);
        
        // 对于简单类型，输出 WKT (Well-Known Text)
        return String.format("%s(%s)", typeName, parseCoordinates(buffer, offset + 5, length - 5));
    }
    
    private static String getGeometryTypeName(int type) {
        switch (type) {
            case 1: return "POINT";
            case 2: return "LINESTRING";
            case 3: return "POLYGON";
            case 4: return "MULTIPOINT";
            case 5: return "MULTILINESTRING";
            case 6: return "MULTIPOLYGON";
            case 7: return "GEOMETRYCOLLECTION";
            default: return "UNKNOWN";
        }
    }
}
```
