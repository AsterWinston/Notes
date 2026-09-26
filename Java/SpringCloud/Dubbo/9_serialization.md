
# 1. 序列化概述

序列化是将 Java 对象转换为字节流以便网络传输的过程，反序列化是逆向过程。Dubbo 内部使用 SPI 机制支持多种序列化方案。

```java
@SPI("hessian2")  // 默认 Hessian2
public interface Serialization {
    byte getContentTypeId();
    String getContentType();
    ObjectOutput serialize(URL url, OutputStream output) throws IOException;
    ObjectInput deserialize(URL url, InputStream input) throws IOException;
}
```

序列化 ID 与协议帧中的 Flag 字节关联：

```
Flag 字节 (8位)
┌──────────────────────────────────┐
│ [7]  [6-5]   [4-3]    [2-0]     │
│ Req/  TwoWay/  Event/  Serialize │
│ Resp  OneWay   Heartbeat  ID    │
└──────────────────────────────────┘
   Serialize ID:
   2 = Hessian2 (默认)
   3 = Java
   4 = Compacted Java
   6 = FastJSON2
   7 = Native Java
   8 = Kryo
   9 = FST
   11= Protobuf (自定义)
```

# 2. 支持的序列化方案

| 序列化 | ContentTypeId | 特点 | 适用场景 |
|--------|--------------|------|---------|
| **hessian2** | 2 | 跨语言、紧凑、自描述 | 默认选择，通用场景 |
| **fastjson2** | 6 | 极快、文本可读、不跨语言 | 高性能+可观测性 |
| **kryo** | 8 | 极快、需注册Class、二进制 | 极致性能场景 |
| **protobuf** | 11 | 跨语言、极紧凑、需定义IDL | 多语言、云原生 |
| **java** | 3 | JDK自带、兼容性好 | 老系统兼容 |
| **compacted java** | 4 | JDK序列化压缩版 | 过渡方案 |
| **fst** | 9 | 快速、完全兼容JDK | 序列化兼容场景 |
| **avro** | 12 | 跨语言、Schema动态 | 大数据/Hadoop |

# 3. Hessian2（默认）

## 3.1 特点

- 跨语言：支持 Java、C++、Python 等
- 自描述：不带 Schema 也能反序列化（类型信息嵌入数据）
- 紧凑：二进制格式，比 JSON 小
- 稳定：Caucho 公司开发，久经考验

## 3.2 编码示例

```java
// Hessian2 二进制编码示意
// User{id=1, name="test", age=25}
// 编码后（十六进制）:
// 'M'           → Map 类型标记
// 0x00 0x03    → 3个字段
// 'i' 'd'       → key: "id"
// 0x91          → int 1
// 'n' 'a' 'm' 'e' → key: "name"
// 0x04 't' 'e' 's' 't' → string "test"
// 'a' 'g' 'e'   → key: "age"
// 0x64          → int 25
// 'z'           → END
```

## 3.3 配置

```yaml
dubbo:
  protocol:
    serialization: hessian2
```

# 4. FastJSON2

## 4.1 配置

```xml
<dependency>
    <groupId>com.alibaba.fastjson2</groupId>
    <artifactId>fastjson2</artifactId>
</dependency>
```

```yaml
dubbo:
  protocol:
    serialization: fastjson2
```

```java
// 可编程配置
@DubboService(serialization = "fastjson2")
public class UserServiceImpl implements UserService {}
```

## 4.2 特点

- JSON 文本格式，便于调试和日志
- 性能接近 Kryo，远超 Jackson
- 不跨语言（JSON 格式兼容但类型信息为 Java 特有）

## 4.3 序列化示例

```java
// FastJson2ObjectInput / FastJson2ObjectOutput
public class FastJson2ObjectOutput implements ObjectOutput {
    
    @Override
    public void writeObject(Object obj) throws IOException {
        byte[] bytes = JSONB.toBytes(obj);  // JSONB 二进制 JSON
        output.write(bytes);
        output.flush();
    }
}
```

# 5. Kryo

## 5.1 配置

```xml
<dependency>
    <groupId>com.esotericsoftware</groupId>
    <artifactId>kryo</artifactId>
</dependency>
```

```yaml
dubbo:
  protocol:
    serialization: kryo
```

## 5.2 源码

```java
public class KryoObjectOutput implements ObjectOutput {
    private final Kryo kryo;
    private final Output output;

    public KryoObjectOutput(OutputStream outputStream) {
        this.output = new Output(outputStream);
        this.kryo = KryoUtils.get(); // ThreadLocal 复用 Kryo 实例
    }

    @Override
    public void writeObject(Object obj) throws IOException {
        // 无 Class 注册时写入全限定类名
        kryo.writeClassAndObject(output, obj);
        output.flush();
    }
}
```

## 5.3 优化：类注册

默认 Kryo 不注册类时会在数据中写入类名，增加体积。注册类后使用 ID 替代类名：

```java
public class KryoFactory {
    public static Kryo createKryo() {
        Kryo kryo = new Kryo();
        // 注册常用类，使用 int ID 替代全限定类名
        kryo.register(User.class, 1001);
        kryo.register(Order.class, 1002);
        return kryo;
    }
}
```

SPI 配置文件 `META-INF/dubbo/org.apache.dubbo.common.serialize.Serialization`：

```
kryo=org.apache.dubbo.common.serialize.kryo.KryoSerialization
```

# 6. Protobuf（Protocol Buffers）

## 6.1 配置

```xml
<dependency>
    <groupId>com.google.protobuf</groupId>
    <artifactId>protobuf-java</artifactId>
</dependency>
```

```yaml
dubbo:
  protocol:
    serialization: protobuf
```

## 6.2 IDL 定义

```protobuf
syntax = "proto3";

message User {
    int64 id = 1;
    string name = 2;
    int32 age = 3;
}
```

## 6.3 编解码

```java
public class ProtobufObjectOutput implements ObjectOutput {

    @Override
    public void writeObject(Object obj) throws IOException {
        if (obj instanceof MessageLite) {
            // protobuf 原生消息 → 直接序列化
            ((MessageLite) obj).writeTo(outputStream);
        } else {
            // POJO → 通过 ProtobufMessageBuilder 转换
            MessageLite.Builder builder = getBuilder(obj.getClass());
            ProtobufMessageBuilder.setValue(builder, obj);
            builder.build().writeTo(outputStream);
        }
    }
}
```

# 7. 自定义序列化

```java
@SPI
public interface Serialization {
    byte getContentTypeId();
    String getContentType();
    ObjectOutput serialize(URL url, OutputStream output) throws IOException;
    ObjectInput deserialize(URL url, InputStream input) throws IOException;
}

// 实现
public class CustomSerialization implements Serialization {
    @Override
    public byte getContentTypeId() { return 20; }

    @Override
    public String getContentType() { return "x-application/custom"; }

    @Override
    public ObjectOutput serialize(URL url, OutputStream output) throws IOException {
        return new CustomObjectOutput(output);
    }

    @Override
    public ObjectInput deserialize(URL url, InputStream input) throws IOException {
        return new CustomObjectInput(input);
    }
}
```

SPI 注册（`META-INF/dubbo/org.apache.dubbo.common.serialize.Serialization`）：

```
custom=com.example.CustomSerialization
```

使用：

```yaml
dubbo:
  protocol:
    serialization: custom
```

# 8. 序列化性能对比

```
维度\方案     Hessian2    FastJSON2    Kryo     Protobuf    Java
────────────────────────────────────────────────────────────
体积          小          中           极小      极小        大
速度          快          很快         极快      极快        慢
跨语言        支持        JSON可读     不支持     原生支持    不支持
对象兼容性    好（自描述）  好          需要注册   需要IDL      好
空字段        省略        包含         省略      省略        包含
子类多态      支持        限制         支持      限制        支持
集合泛型      部分丢失    JSON限制    需要注册   需要map     支持
```

**选择建议**：

- 内部服务：Kryo > Hessian2 > FastJSON2
- 跨语言：Protobuf（Proto IDL）或 Triple 协议
- 调试：FastJSON2（日志可读）
- 默认：Hessian2（稳定、兼容性好）

# 9. 安全考虑

## 9.1 反序列化风险

```java
// 如果不限制反序列化类型，存在安全风险
// 攻击者可能利用特定 Class 的构造/反序列化逻辑执行恶意代码
```

Dubbo 的安全措施：

```yaml
dubbo:
  protocol:
    # 限制可反序列化的类（白名单）
    serialization-security:
      check-serializable: true
      trusted-classes:
        - com.example.api.*
        - java.util.*
        - java.lang.*
```

## 9.2 源码层面的安全增强

```java
// Dubbo 3.2+ 内置了反序列化安全检查
public class SerializeSecurityManager {
    private final Set<String> allowedPrefixes = new HashSet<>();
    private final Set<String> disallowedPrefixes = new HashSet<>();

    public void checkSerialization(Class<?> clazz) {
        String name = clazz.getName();
        for (String prefix : disallowedPrefixes) {
            if (name.startsWith(prefix)) {
                throw new RpcException("Class " + name + " is not allowed");
            }
        }
        boolean allowed = allowedPrefixes.isEmpty();
        for (String prefix : allowedPrefixes) {
            if (name.startsWith(prefix)) {
                allowed = true;
                break;
            }
        }
        if (!allowed) {
            throw new RpcException("Class " + name + " is not trusted");
        }
    }
}
```
