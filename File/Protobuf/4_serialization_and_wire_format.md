很多人用 Protobuf 用了几年，也没看过它到底怎么编码。这不算错，但一旦遇到"改类型后数据错乱""负数怎么变大 10 字节""packed 和不 packed 解析不一致"，不了解线格式就只能靠猜。本篇把编码拆到字节级。

# 1. 为什么要懂线格式

## 1.1 三个实际收益

- **判断兼容性**：知道 wire type 才知道哪种类型互换是安全的。
- **估算体积**：知道 varint 才知道为什么负数不划算、字符串占多少。
- **排查乱码**：抓到十六进制后能手动解出字段。

```text
一段常见的 Protobuf 字节：
08 96 01 12 05 61 6C 69 63 65

拆开看：
08        → tag：字段 1，wire type 0（varint）
96 01     → varint：150
12        → tag：字段 2，wire type 2（长度前缀）
05        → 长度 5
61...65   → "alice"
```

★ **关键认知**：Protobuf 的线格式只有一条主干：**tag + payload**。tag 里同时编码了字段编号和 wire type，payload 的形态由 wire type 决定。掌握这条主干，任何消息都能手解。

## 1.2 与 JSON 的根本差异

⚠ **关键认知**：Protobuf **不自描述**。同一段字节，没有 schema 你无法知道 2 号字段是 `string` 还是 `bytes`，也无法知道 3 号字段是 `int32` 还是 `sint32`。字段名和类型都不在字节里，schema 是解码的必要条件。

# 2. tag 与 wire type

## 2.1 tag 的组成

```text
tag = (field_number << 3) | wire_type

字段 1，varint：  (1 << 3) | 0 = 8   → 0x08
字段 2，length：  (2 << 3) | 2 = 18  → 0x12
```

## 2.2 六种 wire type

| 值 | 名称 | 用于 |
|----|------|------|
| 0 | VARINT | `int32/int64/uint32/uint64/sint32/sint64/bool/enum` |
| 1 | I64 | `fixed64/sfixed64/double` |
| 2 | LEN | `string/bytes/message/repeated packed` |
| 3 | SGROUP | 已废弃（proto2 group） |
| 4 | EGROUP | 已废弃 |
| 5 | I32 | `fixed32/sfixed32/float` |

★ **关键认知**：wire type 只有 6 种，**字段编号才决定语义**。这就是为什么 `int32` 换 `uint32` 有时能读（同是 VARINT），而 `int32` 换 `fixed32` 一定错（VARINT 对 I32）。判断兼容性时，先比 wire type，再比取值范围和编码细节。

## 2.3 tag 占几个字节

tag 本身也是 varint：字段 1~15 只占 1 字节，16~2047 占 2 字节，2048 以上更多。

⚠ **关键认知**：`16` 是第一个需要 2 字节 tag 的编号。所以"1~15 留给高频字段"不是玄学，是实打实的每字段每消息省 1 字节。

# 3. varint 编码

## 3.1 原理

varint 用每个字节的最高位（MSB）当**继续位**：1 表示后面还有字节，0 表示结束。剩下 7 位存数据，低位在前。

```text
150 = 1001 0110
拆 7 位一组（低位在前）：0000001  0010110
加继续位：             1 0000001  0 0010110
写成字节：             0x96       0x01
```

```python
def encode_varint(n: int) -> bytes:
    out = bytearray()
    while True:
        b = n & 0x7F          # 取低 7 位
        n >>= 7
        if n:
            out.append(b | 0x80)   # 还有后续，置继续位
        else:
            out.append(b)          # 最后一字节
            return bytes(out)

print(encode_varint(150).hex())   # 9601
print(encode_varint(1).hex())     # 01
print(encode_varint(300).hex())   # ac02
```

| 值 | 编码 | 字节数 |
|----|------|--------|
| 127 | `7F` | 1 |
| 128 | `80 01` | 2 |
| 300 | `AC 02` | 2 |
| 16384 | `80 80 01` | 3 |

★ **关键认知**：varint 对**小正整数**极友好，但对**大数**和**负数**不友好。`int32` 的负数是按 64 位补码编码的，所以 `-1` 会变成 `FF FF FF FF FF FF FF FF FF 01`，占 10 字节。这就是负数要用 `sint` 的原因。

## 3.2 ZigZag：给负数用的映射

ZigZag 把有符号整数映射成无符号，让绝对值小的负数也变小：

```text
0 → 0
-1 → 1
1 → 2
-2 → 3
2 → 4
```

公式：

```text
编码： (n << 1) ^ (n >> 31)     // 32 位
解码： (n >> 1) ^ -(n & 1)
```

```python
def zigzag32(n: int) -> int:
    return (n << 1) ^ (n >> 31)

for v in [-1, 1, -100]:
    print(v, zigzag32(v), encode_varint(zigzag32(v)).hex())
# 输出：-1 1 01 / 1 2 02 / -100 199 c701
```

⚠ **关键认知**：`sint32` 和 `int32` 的 wire type 都是 0（VARINT），**它们字节层面能互相读，但值会错**。`int32` 存 `-1` 是 10 字节的 `FF...01`，`sint32` 读到会解成很大的数。这就是"wire type 相同但仍不兼容"的典型例子。

# 4. 各类型的编码

## 4.1 定长类型

`fixed32/sfixed32/float` 固定 4 字节，`fixed64/sfixed64/double` 固定 8 字节，**小端序**。

```text
字段 1，fixed32，值 1：0D 01 00 00 00
                      │  └──────────┘
                      tag  4 字节小端
字段 2，double，值 1.0：11 00 00 00 00 00 00 F0 3F
```

★ **关键认知**：定长类型的优点是**解析不需要扫描**，直接按偏移取；缺点是**小值也占满字节**。适合哈希、时间戳（毫秒）、大整数这类"反正都大"的字段。

## 4.2 长度前缀类型

`string`、`bytes`、嵌套 `message` 都用 LEN：先写长度（varint），再写内容。

假设 `User { string name = 1; Address addr = 2; }`，其中 `Address { string city = 1; }`。

```text
name = "alice"，addr = {city: "NY"}：

0A 05 61 6C 69 63 65
12 04 0A 02 4E 59
│  │  └─ 嵌套消息："0A 02 4E 59"
│  └──── 长度 4
└─────── tag：字段 2，LEN
```

⚠ **关键认知**：嵌套消息的长度前缀是**整个子消息的字节数**，不是字段数。解析器靠这个长度知道子消息在哪结束，所以**不需要知道子消息的 schema 也能跳过它**——这也是未知字段能被保留的基础。

## 4.3 repeated 与 packed

proto3 里标量 `repeated` 默认 packed：所有元素共用一个 tag，长度前缀后连续排列。

```protobuf
message M {
  repeated int32 nums = 1;   // [1, 2, 300]
}
```

```text
packed：    0A 04 01 02 AC 02
            │  │  └─ 三个 varint 紧挨着
            │  └──── 总长度 4
            └─────── tag：字段 1，LEN

非 packed： 08 01 08 02 08 AC 02
            │     │     └─ 每个元素都带 tag
```

| 编码 | 特点 | 适用 |
|------|------|------|
| packed | 省 tag，体积小 | proto3 标量 repeated 默认 |
| 非 packed | 可流式读，元素可交错 | proto2 默认、消息 repeated |

★ **关键认知**：packed 与不 packed 的**线上字节完全不同**。解析器一般两种都接受，但如果你手写二进制或跨版本对接，必须显式约定。消息类型的 `repeated` **永远不 packed**，因为每个元素是 LEN，本身已带长度。

# 5. 完整编码示例

## 5.1 给定消息

```protobuf
message Person {
  int32  id    = 1;
  string name  = 2;
  bool   vip   = 3;
  repeated int32 scores = 4;
}
```

赋值：`id = 150`、`name = "alice"`、`vip = true`、`scores = [1, 2]`。

## 5.2 逐字段编码

```text
id = 150：
  tag  = (1 << 3) | 0 = 0x08
  val  = varint(150)  = 0x96 0x01
  → 08 96 01

name = "alice"：
  tag  = (2 << 3) | 2 = 0x12
  len  = 5 = 0x05
  val  = 61 6C 69 63 65
  → 12 05 61 6C 69 63 65

vip = true：
  tag  = (3 << 3) | 0 = 0x18
  val  = 0x01
  → 18 01

scores = [1, 2]：
  tag  = (4 << 3) | 2 = 0x22
  len  = 2 = 0x02
  val  = 01 02
  → 22 02 01 02
```

拼起来：

```text
08 96 01 12 05 61 6C 69 63 65 18 01 22 02 01 02   # 最终字节
```

验证：

```bash
echo "089601120561 6C6963651801 22020102" | xxd -r -p > person.bin
protoc --decode=Person person.proto < person.bin
# 输出 id: 150 / name: "alice" / vip: true / scores: 1 / scores: 2
```

★ **关键认知**：字段顺序在编码时通常按编号升序写出，但**解析时不依赖顺序**。因为每个字段都自带 tag，解析器可以任意顺序读取，遇到不认识的编号就按 wire type 跳过。这就是 Protobuf 能演进的根本机制。

# 6. 大小与性能

## 6.1 和 JSON 对比

同一个 `Person`：

```text
JSON（约 60 字节）：
{"id":150,"name":"alice","vip":true,"scores":[1,2]}

Protobuf（16 字节）：
08 96 01 12 05 61 6C 69 63 65 18 01 22 02 01 02
```

省字节的来源很直接：没有字段名，小整数用 varint，没有 `{}[],":` 这些标点，默认值不写。

## 6.2 什么时候 Protobuf 不一定更小

如果消息里主要是**长字符串**，Protobuf 和 JSON 的体积差距会缩小到 10%~20%，因为字符串本身占大头，字段名省下的比例变小。

⚠ **关键认知**：Protobuf 的体积优势主要来自"**大量小字段 + 数值 + 高频重复结构**"。如果一条消息里有 3 个几 KB 的文本字段，为了省那几十字节字段名而引入 schema 和代码生成，收益很低。

## 6.3 性能来自哪里

★ **关键认知**：性能优势的大头不是"字节少"，而是**解析路径短且可预测**。生成代码里每个字段的读取都是固定偏移或一次 varint 扫描，没有字符串比较，也没有动态类型判断。

# 7. 常见误解

| 误解 | 事实 |
|------|------|
| "Protobuf 一定比 JSON 小" | 长字符串场景差距很小 |
| "字段顺序重要" | 解析不依赖顺序，编码通常升序 |
| "int32 和 sint32 可以互换" | wire type 同，但值会错 |
| "零值也会被编码" | 零值被跳过，除非 `optional` |
| "未知字段会被丢弃" | proto3 会保留在 unknown fields |

# 8. 练习

1. 手算 `id = 300`（`int32`，字段 1）的编码字节，再用 Python 的 `encode_varint` 验证。
2. 给定字节 `0A 03 61 62 63 10 7B`，schema 为 `message M { string a = 1; int32 b = 2; }`，手动解出字段值。
3. 把 `message M { repeated int32 v = 1; }` 的值 `[1, 2, 3]` 分别按 packed 和非 packed 编码，写出两种字节，并说明解析器如何区分。

# 9. 小结

| 概念 | 要点 |
|------|------|
| tag | `(编号 << 3) \| wire_type`，编号 1~15 只占 1 字节 |
| wire type | 只有 6 种，决定 payload 形态 |
| varint | 7 位一组，MSB 是继续位，小整数省字节 |
| 负数 | `int32` 负数占 10 字节，用 `sint` 走 ZigZag |
| 定长 | fixed/float/double 固定 4 或 8 字节小端 |
| LEN | string/bytes/message 用长度前缀 |
| packed | proto3 标量 repeated 默认打包 |
| 解析 | 靠 tag 跳转，不依赖字段顺序 |

下一篇进入安全与坑点，讲未知字段怎么被保留、反序列化如何被超大消息和深递归拖垮、类型混淆如何产生，以及敏感数据在 Protobuf 里的处理方式。
