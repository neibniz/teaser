# Protobuf 与 FlatBuffers 原理详解：从字节编码到工程选型

> **目标读者**：第一次系统学习二进制序列化，或会调用生成代码但不理解底层格式的开发者。
>
> **学习目标**：能够从 schema 推导核心字节布局；解释 Protobuf 为什么紧凑、FlatBuffers 为什么能够原地读取；独立设计可演进的数据协议；根据读写比例、内存、延迟和生态约束选择方案，而不是背诵“Protobuf 小、FlatBuffers 快”。
>
> **版本说明**：本文以 Protobuf Editions（重点参考 Edition 2024）、Protobuf 二进制 wire format，以及 FlatBuffers 官方 schema、internals、evolution 和 C++ 文档为依据。不同语言的生成 API 会随版本变化，具体命令和接口以项目锁定版本的官方文档为准；本文重点讲稳定的格式原理与设计方法。

官方资料入口：

- [Protobuf Encoding](https://protobuf.dev/programming-guides/encoding/)
- [Protobuf Editions Overview](https://protobuf.dev/editions/overview/)
- [Protobuf Field Presence](https://protobuf.dev/programming-guides/field_presence/)
- [Protobuf Best Practices](https://protobuf.dev/best-practices/dos-donts/)
- [FlatBuffers Internals](https://flatbuffers.dev/internals/)
- [FlatBuffers Schema Evolution](https://flatbuffers.dev/evolution/)
- [FlatBuffers C++ Guide](https://flatbuffers.dev/languages/cpp/)

---

## 目录

0. [先抓住主线：序列化是在设计另一种内存](#0-先抓住主线序列化是在设计另一种内存)
1. [序列化到底解决什么问题](#1-序列化到底解决什么问题)
2. [评价格式的七个维度](#2-评价格式的七个维度)
3. [Schema、编译器、运行库和生成代码](#3-schema编译器运行库和生成代码)
4. [Protobuf 的核心模型：带标签的记录流](#4-protobuf-的核心模型带标签的记录流)
5. [Varint：小整数为什么只占少数字节](#5-varint小整数为什么只占少数字节)
6. [ZigZag、定长整数与浮点数](#6-zigzag定长整数与浮点数)
7. [Tag 与六种 wire type](#7-tag-与六种-wire-type)
8. [字符串、子消息、repeated、map 与 oneof](#8-字符串子消息repeatedmap-与-oneof)
9. [逐字节手算一个 Protobuf 消息](#9-逐字节手算一个-protobuf-消息)
10. [Protobuf 解析器如何工作](#10-protobuf-解析器如何工作)
11. [字段存在性、默认值与合并语义](#11-字段存在性默认值与合并语义)
12. [未知字段为何支撑协议演进](#12-未知字段为何支撑协议演进)
13. [Protobuf schema 演进规则](#13-protobuf-schema-演进规则)
14. [确定性不等于规范化，消息也不自带边界](#14-确定性不等于规范化消息也不自带边界)
15. [Protobuf 生成代码、对象模型与 Arena](#15-protobuf-生成代码对象模型与-arena)
16. [Protobuf C++ 端到端示例](#16-protobuf-c-端到端示例)
17. [FlatBuffers 的出发点：让序列化结果可直接读取](#17-flatbuffers-的出发点让序列化结果可直接读取)
18. [FlatBuffers schema：table 与 struct](#18-flatbuffers-schematable-与-struct)
19. [FlatBuffer 的整体内存布局](#19-flatbuffer-的整体内存布局)
20. [vtable：可演进与随机访问的关键](#20-vtable可演进与随机访问的关键)
21. [Offset、String、Vector 与 Union](#21-offsetstringvector-与-union)
22. [为什么 Builder 从后向前构造](#22-为什么-builder-从后向前构造)
23. [读取、生命周期、验证与并发](#23-读取生命周期验证与并发)
24. [修改、Object API 与反射](#24-修改object-api-与反射)
25. [FlatBuffers schema 演进规则](#25-flatbuffers-schema-演进规则)
26. [FlatBuffers C++ 端到端示例](#26-flatbuffers-c-端到端示例)
27. [两种格式的成本模型](#27-两种格式的成本模型)
28. [Protobuf 与 FlatBuffers 系统对比](#28-protobuf-与-flatbuffers-系统对比)
29. [如何按场景做选型](#29-如何按场景做选型)
30. [如何做可信的性能测试](#30-如何做可信的性能测试)
31. [安全、可靠性与生产检查清单](#31-安全可靠性与生产检查清单)
32. [常见误区与反例](#32-常见误区与反例)
33. [从零实现时应抓住的最小算法](#33-从零实现时应抓住的最小算法)
34. [练习题与答案线索](#34-练习题与答案线索)
35. [学习路线与官方资料](#35-学习路线与官方资料)

---

## 0. 先抓住主线：序列化是在设计另一种内存

初学者常把序列化理解成“把对象存成文件”。这个说法没有错，但太浅。更有解释力的定义是：

> **序列化是在字节数组中设计一种跨进程、跨机器、跨版本的数据表示；反序列化是在本地程序中重新解释或重建这种表示。**

一个 C++ 对象不能直接当作长期协议，原因包括：

- 指针只在当前进程地址空间有效；
- `std::string` 通常保存指针、长度和容量，而不是把字符内联在对象里；
- 编译器可能插入 padding；
- ABI、字节序、类型宽度和对齐规则可能不同；
- 新旧程序对“这个对象有哪些字段”的认识可能不同；
- 原始内存可能包含未初始化 padding，既不稳定也可能泄露数据。

所以我们真正要解决的是一个映射问题：

```text
本地对象 O  --encode-->  字节序列 B
本地对象 O' <--decode--  字节序列 B
```

理想情况下，`O'` 与 `O` 在协议语义上相等。但“语义相等”不要求：

- 两端使用同一种编程语言；
- 两端对象具有相同的内存布局；
- 相同语义一定产生完全相同的字节序列；
- 新旧两端认识完全相同的字段集合。

Protobuf 与 FlatBuffers 对这个问题做出了不同选择：

```text
Protobuf：
  字节是紧凑的记录流
  -> 通常先扫描并解析
  -> 在本地生成易修改的对象

FlatBuffers：
  字节本身就是受约束的可遍历对象图
  -> 先验证边界
  -> 通过 offset 和 accessor 原地读取
```

![Protobuf 与 FlatBuffers 的统一数据路径](https://oss.euler.icu/teaser/docs/data-serialization/protobuf-flatbuffers/01-unified-data-path.png?v=20260713-final)

### 0.1 全文反复使用的四个问题

看到任何序列化设计，先问：

1. **边界在哪里**：解析器怎样知道一个值或一条消息何时结束？
2. **类型在哪里**：字节本身携带多少类型信息，多少信息必须由 schema 提供？
3. **地址怎样表达**：嵌套对象靠顺序、长度、field number，还是相对 offset 连接？
4. **版本怎样错开**：旧读者遇到新字段时，怎样跳过、保留或返回默认值？

Protobuf 的答案主要是 `tag + wire type + length`；FlatBuffers 的答案主要是 `root offset + vtable + relative offset + alignment`。

### 0.2 不要先问“谁更快”

“快”必须带宾语：

- 编码快还是读取快？
- 读取全部字段，还是只读取两个字段？
- 第一次读取，还是同一对象重复读取一万次？
- CPU 时间，还是端到端延迟？
- 是否包含分配、网络、压缩、校验和业务对象转换？

脱离访问模式谈格式性能，结论通常没有可迁移性。

---

## 1. 序列化到底解决什么问题

### 1.1 空间隔离

两个进程不能直接共享普通指针：

```text
进程 A: 0x7f001000 -> User
进程 B: 0x7f001000 -> 可能未映射，也可能是完全不同的数据
```

因此跨进程数据必须使用与地址空间无关的表示。常见方法包括：

- 按顺序排列值；
- 用长度描述后续区域；
- 用 ID 代替指针；
- 用相对于缓冲区位置的 offset 代替绝对地址。

### 1.2 平台差异

整数 `0x12345678` 在小端机器中的四个字节通常是：

```text
地址从低到高：78 56 34 12
```

协议必须规定字节序，不能依赖发送端 CPU。Protobuf 的 `fixed32/fixed64` 和 FlatBuffers 标量都规定为 little-endian；生成的 accessor 在大端平台负责转换。

### 1.3 版本错位

现实发布不是：

```text
所有生产者 V1 -> 同一瞬间 -> 所有消费者 V2
```

而是：

```text
V1 客户端、V2 服务端、回滚后的 V1.5 服务端、历史文件 V0
在一段时间内同时存在。
```

一个可演进协议必须把“版本错位”当作常态，而不是异常。

### 1.4 数据边界

TCP 是字节流，不保留应用消息边界。即使序列化库知道字段边界，也未必知道整条消息边界：

```text
[message A][message B]
             ^
如果没有外层长度或 framing，接收方不知道 A 在哪里结束。
```

因此“消息格式”和“传输 framing”是两个层次。gRPC、长度前缀、文件索引、消息队列 record 都可以提供外层边界。

### 1.5 安全边界

不可信字节可能伪造：

- 极大长度；
- 越界 offset；
- 过深嵌套；
- 巨量重复字段；
- 非法 UTF-8；
- 让解析器耗尽 CPU 或内存的输入。

“格式能解析”不等于“输入值得信任”。生产系统还要限制消息大小、深度、总对象数和业务字段范围。

---

## 2. 评价格式的七个维度

### 2.1 编码体积

体积来自：

```text
有效数据
+ 字段定位信息
+ 长度或 offset
+ 对齐 padding
+ 类型或 schema 元数据
+ 外层 framing
```

Protobuf 使用 field number 和变长整数，稀疏小整数通常很紧凑。FlatBuffers 使用固定宽度 offset、vtable 和对齐，可能牺牲一些体积来换取原地随机访问。这里是“可能”，不是无条件结论。

### 2.2 编码成本

需要观察：

- 是否必须先计算大小；
- 是否需要临时对象；
- 是否会扩容和复制 builder 缓冲区；
- 字符串和子对象是否复制；
- 能否批量或流式输出。

### 2.3 首次读取成本

Protobuf 通常扫描记录流并填充生成对象。FlatBuffers 通常先验证，再按需访问字段。若只读少量字段，按需访问可以少碰很多字节；若完整遍历并转成本地对象，优势可能缩小。

### 2.4 重复访问成本

已经解析好的 Protobuf 对象适合频繁修改和重复遍历。FlatBuffers 每次访问 table 字段通常都要经过 vtable 间接寻址，但避免了对象树分配，并保持数据集中在一个缓冲区中。

### 2.5 内存峰值

要区分：

```text
输入 buffer
+ 解析后对象树
+ 分配器元数据
+ 临时构建空间
+ 业务转换副本
```

只比较文件大小看不到运行时内存峰值。

### 2.6 演进能力

演进不是一个布尔值。至少要分别判断：

- 新代码读旧数据；
- 旧代码读新数据；
- 旧代码读新数据后再写出，未知字段是否还在；
- 二进制、JSON、文本格式是否具有相同兼容性；
- 默认值变化是否改变历史数据语义。

### 2.7 生态与可运维性

生产选型还包括：

- 支持语言与平台；
- RPC、注册中心、代码生成和构建系统集成；
- schema lint、breaking-change 检查；
- 调试、反射、JSON 转换；
- 团队经验和长期维护成本。

格式微基准领先，不足以抵消生态不匹配。

---

## 3. Schema、编译器、运行库和生成代码

两种系统都有四个角色：

```text
schema 文件
   |
   v
schema compiler: protoc / flatc
   |
   +--> 生成的类型、accessor、builder、parser
   |
应用代码 + runtime library
   |
   v
稳定定义的二进制格式
```

### 3.1 Schema 是协议契约，不只是代码模板

schema 同时规定：

- 字段的逻辑含义；
- 字段的稳定身份；
- 二进制解释方式；
- 缺失字段的默认行为；
- 哪些演进是兼容的。

生成代码可以重新生成，线上历史字节不能被重新生成。因此 field number、field slot 和默认值都应被当作持久数据的一部分审查。

### 3.2 编译期类型信息与运行时字节信息

二进制通常不会保存完整字段名：

- Protobuf 保存 field number 和 wire type，不保存 `device_id` 这个名字；
- FlatBuffers table 通过 vtable slot 定位字段，不保存字段名；
- 解码端必须拥有兼容 schema 或反射描述，才能恢复业务含义。

这就是 schema 驱动格式比 JSON 紧凑的重要原因之一。

### 3.3 生成代码不是可选装饰

生成代码把协议规则固化为高效操作：

- Protobuf 生成字段访问器、解析和序列化逻辑、presence 状态与 descriptor；
- FlatBuffers 生成相对 offset 的类型化 accessor、builder 和 verifier；
- 它们还把很多错误提前到编译期，例如字段类型不匹配。

手写解析器适合学习和诊断，不适合替代成熟 runtime。

---

## 4. Protobuf 的核心模型：带标签的记录流

一个 Protobuf message 的 wire format 可以抽象为：

```text
message := record*
record  := tag payload
tag     := varint((field_number << 3) | wire_type)
```

它不是 C struct 的内存快照，也不是“字段按 schema 顺序连续排列”。它是一串可按规则跳过的记录。

### 4.1 为什么只写 field number，不写字段名

假设 schema 是：

```proto
message User {
  uint64 id = 1;
  string name = 2;
}
```

wire format 只关心 `1` 和 `2`：

```text
field 1 -> id
field 2 -> name
```

字段名用于源码可读性；field number 才是线上稳定身份。重命名字段通常不改变二进制格式，复用旧号码却会让旧字节被解释成新含义。

### 4.2 为什么 tag 还需要 wire type

旧解析器可能不知道新字段的业务类型，但它必须知道怎样跳过该字段。wire type 提供“机械边界”：

- 这是一个 varint，读到 continuation bit 为 0；
- 这是固定 4 或 8 字节；
- 这是 length-delimited，先读长度，再跳过对应字节数。

因此 wire type 不是完整类型。`string`、`bytes`、子消息和 packed repeated 都共享 wire type 2；真正含义仍由 schema 决定。

### 4.3 字段顺序不是语义

解析器必须接受任意字段顺序：

```text
field 2, field 1
```

与：

```text
field 1, field 2
```

通常可表达相同消息。不要通过“第几个字段”理解 Protobuf，要通过 field number 理解。

---

## 5. Varint：小整数为什么只占少数字节

### 5.1 每个字节分成两部分

Varint 的每个字节：

```text
bit 7       bits 6..0
continuation payload
```

- 最高位 `1`：后面还有字节；
- 最高位 `0`：这是最后一个字节；
- 低 7 位：有效载荷；
- 各组 payload 按低位组在前的顺序组合。

因此一个字节可表示 `0..127`，两个字节可表示到 `16383`。

### 5.2 手算 150

`150 = 22 + 1 * 128`，把它拆成 base-128 数位：

```text
低 7 位：22 = 0010110
高 7 位： 1 = 0000001
```

低位组后面还有数据，所以 continuation bit 设为 1：

```text
第一字节：1 0010110 = 10010110 = 0x96
第二字节：0 0000001 = 00000001 = 0x01
```

最终：

```text
150 -> 96 01
```

还原公式为：

```text
(0x96 & 0x7f) << 0
+ (0x01 & 0x7f) << 7
= 22 + 128
= 150
```

### 5.3 编码算法

```cpp
void writeVarint64(uint64_t value, std::vector<uint8_t>& out) {
    while (value >= 0x80) {
        out.push_back(static_cast<uint8_t>(value) | 0x80);
        value >>= 7;
    }
    out.push_back(static_cast<uint8_t>(value));
}
```

为什么 `static_cast<uint8_t>(value) | 0x80` 可行？转换只保留最低 8 位，而 payload 需要最低 7 位；最高位随后被设成 continuation bit。写得更显式也可以：

```cpp
out.push_back(static_cast<uint8_t>(value & 0x7f) | 0x80);
```

### 5.4 解码算法与防御

```cpp
bool readVarint64(std::span<const uint8_t> input,
                  std::size_t& cursor,
                  uint64_t& value) {
    value = 0;
    for (unsigned i = 0; i < 10; ++i) {
        if (cursor >= input.size()) return false;
        const uint8_t byte = input[cursor++];

        // uint64 的第 10 个字节最多只能携带最低 1 bit，且不能再续写。
        if (i == 9 && byte > 0x01) return false;

        value |= static_cast<uint64_t>(byte & 0x7f) << (i * 7);
        if ((byte & 0x80) == 0) return true;
    }
    return false;  // 超长或溢出的恶意 varint
}
```

不能只写 `while (byte & 0x80)` 而不限制次数。攻击者可以提供永不终止的 continuation sequence，或在第 10 个字节放入超过 1 bit 的有效载荷。注意：教学函数在失败时可能已经推进 `cursor`；生产解析器通常通过受限子流或事务式游标统一管理失败恢复。

### 5.5 什么时候 Varint 省空间

对 `uint64`：

| 数值范围 | Varint 字节数 |
| --- | ---: |
| `0..127` | 1 |
| `128..16,383` | 2 |
| `16,384..2,097,151` | 3 |
| 更大 | 每增加 7 个有效 bit，多 1 字节 |
| 最大 `uint64` | 10 |

如果数值几乎总是随机 64 位，`fixed64` 固定 8 字节可能比 9 或 10 字节 varint 更小且更容易解码。类型选择应来自数据分布，不是名字偏好。

---

## 6. ZigZag、定长整数与浮点数

### 6.1 为什么负 `int32` 可能占 10 字节

普通 `int32/int64` 使用补码值再按无符号 varint 解释。负数的高位大多为 1，扩展到 64 位后通常需要完整 10 字节。因此“值的绝对值很小”不等于普通 `int32` 编码很小。

### 6.2 ZigZag 把符号折叠到最低位

`sint32/sint64` 先做 ZigZag：

```text
 0 -> 0
-1 -> 1
 1 -> 2
-2 -> 3
 2 -> 4
```

32 位编码公式：

```text
(n << 1) XOR (n >> 31)
```

直觉是：

- 非负数映射到偶数；
- 负数映射到奇数；
- 接近 0 的正负数都映射到接近 0 的无符号数。

所以温度变化、坐标增量、误差值这类围绕 0 波动的数据通常适合 `sint32/sint64`。

### 6.3 反向 ZigZag

若编码值为 `u`：

```text
n = (u >> 1) XOR -(u & 1)
```

当最低位为 0，右侧掩码为 0；当最低位为 1，右侧形成全 1 补码，从而恢复负数。

### 6.4 定长数字

| 逻辑类型 | wire type | 长度 | 字节序 |
| --- | ---: | ---: | --- |
| `fixed32`, `sfixed32`, `float` | 5 | 4 | little-endian |
| `fixed64`, `sfixed64`, `double` | 1 | 8 | little-endian |

选择思路：

- 小且偏斜的非负整数：`uint32/uint64`；
- 接近 0 的有符号整数：`sint32/sint64`；
- 大值分布广、常超过 varint 紧凑区：考虑 fixed；
- 浮点数：使用 `float/double`，但要单独定义 NaN、无穷和比较语义。

### 6.5 不要用“可 wire-compatible”代替“语义正确”

某些整数类型在 wire 上能够互相解析，不代表修改字段类型是安全演进。例如同一串 bit 在 `int32` 与 `uint32` 下的业务含义不同，溢出、语言转换和 JSON 映射也可能不同。生产规则应更保守：创建新 field number，双写、迁移、再停写旧字段。

---

## 7. Tag 与六种 wire type

Tag 的公式是：

```text
tag_value = (field_number << 3) | wire_type
```

低 3 bit 存 wire type，其余 bit 存 field number。tag_value 本身再使用 varint 编码。

![Protobuf 的 tag、Varint 与 length-delimited 记录](https://oss.euler.icu/teaser/docs/data-serialization/protobuf-flatbuffers/02-protobuf-wire-record.png?v=20260713-final)

### 7.1 六种 wire type

| ID | 官方名称 | payload 边界 | 常见逻辑类型 |
| ---: | --- | --- | --- |
| 0 | `VARINT` | 读到最高位为 0 | `int*`, `uint*`, `sint*`, `bool`, `enum` |
| 1 | `I64` | 固定 8 字节 | `fixed64`, `sfixed64`, `double` |
| 2 | `LEN` | varint 长度 + 对应字节 | `string`, `bytes`, message, packed repeated |
| 3 | `SGROUP` | 到配对结束 tag | 已废弃 group 的开始 |
| 4 | `EGROUP` | 无 payload | 已废弃 group 的结束 |
| 5 | `I32` | 固定 4 字节 | `fixed32`, `sfixed32`, `float` |

不要在新 schema 中使用 group，但解析器仍需理解 wire type 3/4，因为 wire format 保留了它们。

### 7.2 手算 tag

字段 `device_id = 1`，类型 `uint32`，wire type 为 0：

```text
(1 << 3) | 0 = 8 = 0x08
```

字段 `state = 3`，类型 `string`，wire type 为 2：

```text
(3 << 3) | 2 = 26 = 0x1a
```

### 7.3 小 field number 更省 tag 空间

field number `1..15` 配合常用 wire type 时，tag 通常只占一个字节；`16..2047` 通常占两个字节。因此高频字段可以优先使用较小号码。但不要为了节省一字节而重新编号已发布字段，兼容性价值远大于微小体积收益。

### 7.4 解析器如何跳过未知字段

```text
读 tag
  -> 分离 field number 与 wire type
  -> schema 不认识这个 field number
  -> 根据 wire type 读完或跳过 payload
  -> 继续下一个 tag
```

这段机械规则是 Protobuf 前向兼容的底座。

---

## 8. 字符串、子消息、repeated、map 与 oneof

### 8.1 Length-delimited 的统一外壳

wire type 2 的形状：

```text
[tag varint][length varint][length 个 payload 字节]
```

例如字段 3 的字符串 `"OK"`：

```text
tag    = 0x1a
length = 0x02
UTF-8  = 0x4f 0x4b

结果：1a 02 4f 4b
```

### 8.2 子消息不是特殊容器

子消息先独立编码，再作为父字段的 length-delimited payload：

```text
父字段 tag
+ 子消息字节长度
+ 子消息的 record stream
```

这使解析器可以在不知道子消息字段含义时整体跳过它。

### 8.3 Packed repeated

可打包的标量 repeated 不必为每个元素重复 tag：

```text
未打包： [tag][v1] [tag][v2] [tag][v3]
打包：   [tag LEN][payload length][v1][v2][v3]
```

字符串和 message 不能用这种 packed 形式，因为它们本身需要各自边界。解析器应能接受兼容的 packed/expanded 表示，不能假设同一字段只出现一次。

### 8.4 Map 是语法糖

```proto
map<string, uint32> counts = 5;
```

概念上等价于：

```proto
message CountsEntry {
  string key = 1;
  uint32 value = 2;
}
repeated CountsEntry counts = 5;
```

所以 map 的 wire 表示是一组 length-delimited entry 子消息。不要依赖 map 的序列化顺序。

### 8.5 Oneof 的 wire 上没有额外容器

`oneof` 成员像普通字段一样编码。冲突处理主要发生在解析和对象 API：若同一个 oneof 的多个成员都出现，通常后出现者胜出。由此得到两个结论：

- 字段顺序在普通字段上通常不重要，但会影响 duplicate/oneof 的最终值；
- 不要手写会同时发出多个 oneof 成员的编码器。

### 8.6 Duplicate 的合并规则

- singular 标量或字符串：通常 last one wins；
- repeated：追加；
- singular 子消息：按 message merge 语义合并；
- packed repeated 可拆成多段，解析后拼接。

这解释了一个有用但不应滥用的性质：两个同类型 Protobuf 消息字节串的连接，解析效果可对应于 message merge。

---

## 9. 逐字节手算一个 Protobuf 消息

采用 Edition 2024 schema：

```proto
edition = "2024";

package tutorial.telemetry;

message Telemetry {
  uint32 device_id = 1;
  sint32 delta = 2;
  string state = 3;
  repeated uint32 samples = 4;
}
```

逻辑值：

```text
device_id = 150
delta     = -2
state     = "OK"
samples   = [1, 2, 300]
```

### 9.1 字段 1：device_id

```text
tag   = (1 << 3) | 0 = 8 -> 08
150   = varint 96 01
record = 08 96 01
```

### 9.2 字段 2：delta

`sint32` 先 ZigZag：

```text
-2 -> 3
tag = (2 << 3) | 0 = 16 -> 10
3   -> 03
record = 10 03
```

### 9.3 字段 3：state

```text
tag       = (3 << 3) | 2 = 26 -> 1a
UTF-8 长度 = 2 -> 02
"OK"      = 4f 4b
record    = 1a 02 4f 4b
```

### 9.4 字段 4：samples

每个元素的 varint：

```text
1   -> 01
2   -> 02
300 -> ac 02
```

packed payload 共 4 字节：

```text
01 02 ac 02
```

字段 tag 和长度：

```text
tag    = (4 << 3) | 2 = 34 -> 22
length = 4 -> 04
record = 22 04 01 02 ac 02
```

### 9.5 拼接完整消息

```text
08 96 01
10 03
1a 02 4f 4b
22 04 01 02 ac 02
```

合成一行：

```text
08 96 01 10 03 1a 02 4f 4b 22 04 01 02 ac 02
```

共 15 字节。这里没有字段名、message 名，也没有“总长度”。外层若要连续传输多条消息，仍需 framing。

### 9.6 反向解析演练

从 `08` 开始：

```text
field_number = 0x08 >> 3 = 1
wire_type    = 0x08 & 7  = 0
```

于是读一个 varint，得到 150。游标到 `10`，继续拆出 field 2、wire type 0。schema 告诉解析器它是 `sint32`，所以 varint 3 还要做 ZigZag inverse，得到 -2。

注意：wire type 只说“这是 varint”，不能独自区分 `uint32` 与 `sint32`。没有 schema，`03` 可以解释成无符号 3；有 schema 才知道它代表 -2。

---

## 10. Protobuf 解析器如何工作

### 10.1 主循环

概念解析器如下：

```text
while cursor < message_end:
    tag = read_varint()
    field_number = tag >> 3
    wire_type = tag & 7

    if field_number 是已知字段:
        检查 wire_type 是否可接受
        按 schema 解析 payload
        写入对象或追加 repeated
    else:
        按 wire_type 读取并保存/跳过未知 payload
```

正确实现还要处理：

- 非法 field number 0；
- 截断输入；
- 过长 varint；
- length 超过剩余 buffer；
- 嵌套深度限制；
- duplicate 与 merge；
- packed/expanded 两种形式；
- group 的配对；
- 总消息大小限制。

### 10.2 为什么解析通常是线性的

记录流没有 table-of-contents。要建立完整对象，解析器通常至少扫描一次消息：

```text
时间约为 O(B)
```

其中 `B` 是实际触碰的字节数。变长整数又意味着边界依赖当前字节，难以像固定数组那样直接按索引跳转。

### 10.3 解析后的访问

一旦生成对象建立完成，字段 getter 通常不再扫描 wire bytes。此时访问由对象布局决定，可近似看作 O(1)。所以：

```text
Protobuf 的成本偏向“先付解析费，再得到普通对象体验”。
```

### 10.4 Parse 与业务校验不是一回事

下面的消息在 wire 上可能完全合法：

```text
user_id = 0
page_size = 4,000,000,000
email = "not-an-email"
```

解析器只验证格式规则。业务层还必须验证值域、长度、枚举组合、权限和跨字段约束。

---

## 11. 字段存在性、默认值与合并语义

### 11.1 三个容易混淆的状态

以数值字段为例：

```text
A. 字段从未设置
B. 字段被显式设置为 0
C. getter 返回 0
```

在 implicit presence 下，A 与 B 在 API 和重新序列化时可能无法区分；C 既可能来自缺失字段的默认值，也可能来自实际记录。

### 11.2 Explicit presence

显式 presence 除了保存值，还保存“是否出现/是否设置”的状态：

```text
value = 0, present = false
value = 0, present = true
```

官方 field presence 指南建议 proto3 基础类型使用 `optional`，以获得显式 presence，并更平滑地迁移到 Editions；Editions 默认行为则由 edition feature 控制。

### 11.3 为什么 PATCH 接口尤其需要 presence

假设更新请求：

```text
volume = 0
```

它可能表示：

- 把音量设置为 0；
- 客户端没有提供 volume，不要修改。

若 API 无法区分 present 与 default，就无法表达这两个动作。可用 explicit presence、`FieldMask` 或专门的 update operation 建模。

### 11.4 默认值通常不在 wire 中

缺失字段的默认值主要由读端生成代码提供。它不是“发送端把默认值写入了消息”。因此修改默认值会导致：同一份历史字节被新旧程序解释成不同值。这就是为什么默认值属于兼容性契约。

### 11.5 Merge 不等于赋值

Protobuf merge 的一般思路：

- present 的 singular 标量覆盖目标值；
- repeated 追加；
- 子消息递归合并；
- 未出现字段不影响目标。

这对配置叠加很有用，但若业务想表达“清空列表”或“显式恢复默认值”，需要额外 presence 或操作语义。

---

## 12. 未知字段为何支撑协议演进

假设 V1：

```proto
message User {
  uint64 id = 1;
}
```

V2 新增：

```proto
message User {
  uint64 id = 1;
  string nickname = 2;
}
```

V1 代码读取 V2 字节时：

1. 认识 field 1，写入 `id`；
2. 不认识 field 2；
3. 根据 wire type 2 知道它有长度前缀；
4. 保存为 unknown field，或至少安全跳过；
5. 继续解析后续字段。

![Protobuf 通过字段号与未知字段实现滚动升级](https://oss.euler.icu/teaser/docs/data-serialization/protobuf-flatbuffers/03-protobuf-schema-evolution.png?v=20260713-final)

### 12.1 Round-trip preservation

许多官方二进制 runtime 会在 parse -> serialize 时保留 unknown fields：

```text
V2 bytes
  -> V1 parser（nickname 进入 unknown set）
  -> V1 修改 id
  -> V1 serialize
  -> V2 parser 仍可能看到 nickname
```

但不能把它理解为跨所有转换都无条件保留。以下操作可能丢失未知字段：

- 转成 JSON 再转回二进制；
- 按已知字段逐个复制到新 message；
- 使用不保留 unknown fields 的特殊 runtime 或配置；
- 把嵌套消息误当普通 `bytes` 重建。

### 12.2 “旧代码能跳过”不等于“旧代码理解语义”

旧消费者不会因为兼容就自动支持新业务状态。例如 V2 新增支付状态 `REVIEWING`，V1 即使不崩溃，也可能按默认状态执行错误业务。wire compatibility 只解决结构读取，业务兼容仍需产品和状态机设计。

### 12.3 删除字段后为什么要 reserved

历史日志、缓存、数据库和离线文件仍可能包含旧 field 7。若未来把 field 7 复用为另一含义，旧字节会被新代码当成新字段，形成静默数据污染。正确做法：

```proto
message User {
  reserved 7;
  reserved legacy_name;
}
```

上例采用 Editions 语法；proto2/proto3 的保留字段名写作字符串字面量，例如 `reserved "legacy_name";`。两种语法都表达“这个名字不得再次分配”，不要跨语法版本机械复制。

reserved 是“永不回收协议身份”的声明。

---

## 13. Protobuf schema 演进规则

### 13.1 推荐的安全变更

| 变更 | 二进制层面 | 工程说明 |
| --- | --- | --- |
| 使用新号码添加 optional 字段 | 通常安全 | 旧代码跳过；新代码读旧数据得到默认/缺失 |
| 使用新号码添加 repeated 字段 | 通常安全 | 注意业务端对空列表的语义 |
| 重命名字段 | 二进制通常不变 | 会影响源码、JSON/TextFormat 和外部工具 |
| 删除字段并 reserved 号码和名字 | 推荐 | 先停止写入，确认读者迁移，再删除声明 |
| 添加 enum 值 | wire 可读取 | 业务代码必须处理未知 enum |

### 13.2 高风险或破坏性变更

- 复用 field number；
- 改变字段的业务含义；
- 改变默认值；
- 从 repeated 改成 singular；
- 把已发布 optional 改成 required，或新增 required；
- 不经迁移直接改变字段类型；
- 在 oneof 内外移动字段而不分析历史组合；
- 假设 JSON 与 binary 具有完全相同的兼容性。

### 13.3 保守的字段类型迁移模式

不要原地把 field 5 从 `string` 改成 `int64`。使用新字段：

```proto
message Order {
  string legacy_amount = 5 [deprecated = true];
  int64 amount_minor_units = 12;
}
```

滚动步骤：

1. 读新字段，缺失时回退旧字段；
2. 生产者双写；
3. 回填持久数据；
4. 所有读者切到新字段；
5. 停写旧字段；
6. 保留旧号码，最终删除声明并 `reserved 5`。

### 13.4 Enum 的零值应表示 unspecified

推荐：

```proto
enum JobState {
  JOB_STATE_UNSPECIFIED = 0;
  JOB_STATE_PENDING = 1;
  JOB_STATE_RUNNING = 2;
  JOB_STATE_DONE = 3;
}
```

不要让 0 表示一个强业务动作，例如 `DELETE_IMMEDIATELY`。缺失字段、旧客户端未知值和未初始化值都会与零值发生关系，`UNSPECIFIED` 能迫使业务层显式判断。

### 13.5 RPC message 与存储 message 分离

官方最佳实践建议不要让 API message 与长期存储 message 永久共用。两者演进压力不同：

```text
API：客户端兼容、权限、分页、部分更新
存储：索引、回填、历史读取、内部压缩、迁移
```

增加一个翻译层的成本，通常小于未来让外部 API 和历史数据被同一 schema 锁死的成本。

---

## 14. 确定性不等于规范化，消息也不自带边界

### 14.1 语义相同可以有不同字节

可能原因包括：

- 字段顺序不同；
- packed repeated 被拆成多段；
- unknown field 的顺序不同；
- map 迭代顺序不同；
- 实现语言或 runtime 版本不同；
- duplicate singular 记录在解析后合并成同一值。

因此不要默认：

```text
semantic_equal(a, b) => serialize(a) == serialize(b)
```

### 14.2 Deterministic serialization 的边界

确定性模式通常保证同一个二进制实现对同一对象产生稳定输出，但官方明确不承诺跨构建、跨版本、跨语言的规范化字节。它适合减少单进程中的随机性，不足以直接定义全球一致的内容哈希。

需要签名、共识或长期 hash key 时，应：

1. 明确定义应用层 canonical form；
2. 规范 map 排序、浮点 NaN、unknown fields、默认值和 Unicode；
3. 对 canonical bytes 签名；
4. 用跨语言测试向量锁定结果。

### 14.3 Protobuf message 没有总长度

message 内的 string 和 submessage 有长度，不代表顶层 message 自带总长度。常见 framing：

```text
[fixed32 length][protobuf bytes]
[varint length][protobuf bytes]
[message type][length][protobuf bytes][checksum]
```

必须规定：

- length 的字节序和上限；
- 是否包含 header 自身；
- 断流与半包处理；
- 压缩和校验发生在 framing 的哪一层。

### 14.4 gRPC framing 不是 Protobuf wire format

gRPC 常使用 Protobuf 作为 message serializer，但 RPC frame、压缩标志、HTTP/2/HTTP/3 传输与 Protobuf field encoding 是不同层。替换 serializer 不等于替换传输协议，反之亦然。

---

## 15. Protobuf 生成代码、对象模型与 Arena

### 15.1 普通对象模型的成本

生成对象通常包含：

- 标量字段或压缩的 presence bit；
- 字符串、bytes、子消息的拥有关系；
- repeated 容器；
- unknown field storage；
- descriptor/reflection 支持所需元数据。

解析嵌套消息时，普通堆分配可能产生许多小对象。此成本不体现在 wire bytes 中，却会影响延迟、碎片和内存峰值。

### 15.2 Arena 的核心思想

Arena 把一批生命周期相近的对象放进同一个区域：

```text
传统：new child1, new child2, new string... -> 分别 delete
Arena：从大块区域顺序分配 -> 整个 arena 一次释放
```

收益可能包括：

- 减少 allocator 调用；
- 改善局部性；
- 批量释放，省去逐对象析构管理。

代价包括：

- 生命周期被绑定到 arena；
- 跨 arena ownership 和交换更复杂；
- 长生命周期对象可能拖住整块内存；
- Arena 不会减少 wire parse 本身，也不自动消除字符串内容复制。

### 15.3 何时使用 Arena

适合：

- 一次 RPC 请求中的整棵 message tree；
- 批处理阶段结束后整体销毁；
- profiling 已确认分配是瓶颈。

不适合盲目全局化。一个永不释放的全局 arena 只是换了一种内存增长方式。

### 15.4 反射与 generated fast path

生成 getter/setter 是静态类型路径；descriptor + reflection 允许运行时按字段描述遍历，适合网关、调试器、通用转换工具。反射提供通用性，但通常带来更多分支和间接访问。业务 hot path 应优先使用生成 API，除非通用性是核心需求。

---

## 16. Protobuf C++ 端到端示例

### 16.1 Schema

保存为 `telemetry.proto`：

```proto
edition = "2024";

package tutorial.telemetry;

option cc_enable_arenas = true;

message Telemetry {
  uint32 device_id = 1;
  sint32 delta = 2;
  string state = 3;
  repeated uint32 samples = 4;
}
```

生成 C++ 代码：

```bash
protoc --cpp_out=generated telemetry.proto
```

实际工程应固定 `protoc` 与 runtime 的兼容版本，不要在开发机上随意混用不同 major 版本的生成代码和 runtime。

### 16.2 构造与序列化

```cpp
#include <cstdint>
#include <iostream>
#include <string>

#include "generated/telemetry.pb.h"

int main() {
    tutorial::telemetry::Telemetry source;
    source.set_device_id(150);
    source.set_delta(-2);
    source.set_state("OK");
    source.add_samples(1);
    source.add_samples(2);
    source.add_samples(300);

    std::string bytes;
    if (!source.SerializeToString(&bytes)) {
        std::cerr << "serialize failed\n";
        return 1;
    }

    tutorial::telemetry::Telemetry decoded;
    if (!decoded.ParseFromString(bytes)) {
        std::cerr << "parse failed\n";
        return 1;
    }

    std::cout << decoded.device_id() << ' '
              << decoded.delta() << ' '
              << decoded.state() << '\n';
}
```

### 16.3 不可信输入要先限定外层长度

`ParseFromString` 知道 `std::string` 的边界，但系统仍应在读入这个 string 前限制 frame size。不要先按攻击者声明的 20 GiB 长度分配，再希望 Protobuf parser 拒绝。

### 16.4 使用 Arena 的局部生命周期

```cpp
#include <google/protobuf/arena.h>

void handleRequest() {
    google::protobuf::Arena arena;
    auto* message = google::protobuf::Arena::Create<
        tutorial::telemetry::Telemetry>(&arena);

    message->set_device_id(150);
    message->add_samples(1);
    // 函数结束时 arena 整体释放。
}
```

Arena API 与生成代码选项可能随版本变化，工程中应以锁定版本文档为准。

### 16.5 CMake 结构示意

现代 CMake 项目通常让生成步骤成为 target 的一部分，而不是提交不同机器生成出的临时文件。概念结构：

```cmake
find_package(Protobuf CONFIG REQUIRED)

add_executable(proto_demo main.cpp telemetry.proto)
target_link_libraries(proto_demo PRIVATE protobuf::libprotobuf)

# 具体 helper 名称取决于 Protobuf 包和构建方式；
# 核心要求是生成文件、include 目录和 protoc 版本都由构建系统追踪。
```

不要把示意 helper 当作跨所有发行版通用的完整 CMakeLists；先检查项目所用 Protobuf CMake 包导出的函数与 target。

---

## 17. FlatBuffers 的出发点：让序列化结果可直接读取

传统反序列化常形成两份表示：

```text
serialized bytes
    |
    | parse + allocate + copy
    v
native object tree
```

FlatBuffers 的目标是让 buffer 自身满足受约束的对象布局：

```text
verified serialized bytes
    |
    | generated accessor + relative offset
    v
field inside the same buffer
```

### 17.1 “直接读取”的三个前提

1. 缓冲区在访问期间一直存活；
2. offset、长度和对齐已通过验证，或数据来自完全可信的构建链；
3. 通过生成 accessor 读取，而不是把任意位置强转成业务 struct。

### 17.2 它优化的主要矛盾

FlatBuffers 官方白皮书把注意力放在内存：

- 避免构建临时对象树；
- 避免大量小分配；
- 只访问需要的字段；
- 让 vector 和 struct 连续布局；
- 用统一字节序和对齐规则跨平台。

这特别适合读取频繁、消息较大、只读部分字段、内存敏感的场景，例如游戏资源、移动端配置、只读索引和低延迟数据分发。

### 17.3 它没有消灭的成本

- 数据仍要从磁盘或网络进入内存；
- 加密、压缩后必须先解密、解压；
- builder 写字符串和 vector 时仍会复制数据；
- 不可信输入仍应 verifier 遍历结构；
- 若调用 Object API `UnPack()`，仍会分配并复制为本地对象；
- 业务若需要任意修改，往往要重建 buffer。

因此更准确的说法是“避免传统对象化反序列化”，不是“所有路径都零复制”。

---

## 18. FlatBuffers schema：table 与 struct

示例 schema：

```fbs
namespace Tutorial.Telemetry;

enum Status : byte {
  Unknown = 0,
  Online = 1,
  Offline = 2
}

struct Vec3 {
  x:float;
  y:float;
  z:float;
}

table Telemetry {
  device_id:uint;
  position:Vec3;
  name:string;
  samples:[ushort];
  status:Status = Unknown;
}

root_type Telemetry;
file_identifier "TLM1";
```

### 18.1 Table：为演进而设计

table 的字段：

- 通过 vtable slot 间接定位；
- 未标为 `required` 时可以缺失；
- 缺失时 accessor 返回 schema 默认值或 null；
- 新字段可在兼容规则下加入；
- 实例只存实际写入的字段，加上布局元数据。

table 适合大多数会演进的业务对象。

### 18.2 Struct：为固定布局而设计

struct：

- 字段固定、全部内联；
- 按明确对齐规则排列；
- 没有每实例 vtable；
- 可以高效形成连续 vector；
- 基本不能做兼容的字段增删。

`Vec3` 很适合 struct，因为三个坐标是不可分的固定值。用户资料、订单、配置项通常不适合 struct，因为它们会演进。

### 18.3 Table 与 Struct 的决策题

问自己：

```text
这个类型五年后是否仍能保证字段集合和布局完全不变？
```

若答案不是明确的“是”，优先 table。不要为了少一个 vtable 就把演进成本推给未来。

### 18.4 默认值与缺失

当 table 标量等于默认值时，builder 通常可不写该字段。accessor 看见 vtable entry 为 0，就返回生成代码中的默认值。因此修改默认值会让同一历史 buffer 在新旧代码下产生不同解释，仍是破坏性操作。

FlatBuffers 还区分三种缺失策略：

| schema 写法 | 字段缺失时 | 适用思路 |
| --- | --- | --- |
| `count:uint;` 或显式标量默认值 | accessor 返回默认值 | 最常见，缺失与“等于默认值”通常不可区分 |
| `count:uint = null;` | 支持该能力的语言返回 optional/null | 需要区分“未提供”与数值 0 |
| `child:Child (required);` | builder/Verifier 将缺失视为错误 | 只适合真正不可缺少、且不会破坏历史数据的字段 |

不要在已经发布的 table 上新增 `required` 字段：旧 buffer 不可能包含它，新 verifier 会把原本有效的历史数据判为无效。格式层 required 也不能替代业务校验，例如“字符串必须非空”仍由应用负责。

---

## 19. FlatBuffer 的整体内存布局

一个典型 buffer 从低地址开始可抽象为：

```text
[root uoffset]
[optional file identifier]
[vtable / other children / padding]
[root table inline area]
[string / vector / child tables]
```

具体对象顺序不是唯一的，builder 可以因创建顺序和对齐产生不同布局。重要的是所有引用都能通过相对 offset 找到目标。

![FlatBuffer 的 root、vtable、table、vector 与 string 内存布局](https://oss.euler.icu/teaser/docs/data-serialization/protobuf-flatbuffers/04-flatbuffers-memory-layout.png?v=20260713-final)

### 19.1 Root offset

buffer 开头通常是一个 32 位无符号相对 offset，指向 root table：

```text
root_address = buffer_start + read_uoffset(buffer_start)
```

因此 root 不必紧跟在 header 后。

### 19.2 Table 起点的 vtable offset

table 起点的第一个值是有符号 offset，通常向后指向 vtable：

```text
vtable_address = table_address - read_soffset(table_address)
```

官方 internals 的常见编码中，该值表达 table 到 vtable 的距离。使用生成 accessor，而不是在业务代码中硬编码公式和宽度。

### 19.3 Vtable 的结构

概念上：

```text
uint16 vtable_size
uint16 object_inline_size
uint16 field0_offset
uint16 field1_offset
...
```

字段 entry：

- `0`：本实例未写该字段；
- 非 0：从 table 起点到字段数据的 byte offset。

### 19.4 Table inline area

table inline area 保存：

- 标量值；
- 内联 struct；
- 指向 string、vector、子 table 的 offset；
- 为满足对齐而插入的 padding。

它不是按 schema 声明顺序简单紧密排列。生成器可以根据对齐安排字段，vtable 负责提供位置。

### 19.5 相对 offset 为什么可搬运

绝对指针随加载地址变化；相对距离不变：

```text
target = address_of_offset_word + offset_value
```

整个 buffer 从地址 A 搬到地址 B 后，`target - offset_word` 的距离保持不变。因此同一 buffer 可在文件映射、共享内存或网络接收区中读取。

---

## 20. vtable：可演进与随机访问的关键

### 20.1 Getter 的概念算法

读取 table 的字段 `k`：

```text
1. 从 table 起点找到 vtable
2. 计算字段 k 对应的 vtable entry 位置
3. entry 超出 vtable_size -> 旧数据没有该字段 -> 返回默认值
4. entry == 0 -> 本实例省略该字段 -> 返回默认值
5. 否则 table + entry -> 读取字段
```

这段逻辑同时实现：

- 按字段 O(1) 定位；
- 新代码读取旧 buffer；
- 实例级可选字段；
- 默认值省略。

### 20.2 为什么旧代码能读新 buffer

旧代码只查询自己知道的 vtable entry。新字段拥有后续 slot，旧 accessor 根本不会访问它。整个 buffer 保持原样，所以新字段也不会因为“被旧代码读取过”而消失。

### 20.3 为什么新代码能读旧 buffer

新 accessor 请求后续 slot，但旧 vtable 较短：

```text
entry_position >= vtable_size
```

于是 accessor 返回该字段默认值或 null，而不是越界。

### 20.4 Vtable 可共享

多个 table 实例若拥有相同字段布局，builder 可能复用 vtable，减少空间。共享的是布局描述，不是 table 字段值。

### 20.5 间接访问的代价

读取 table 字段通常包含：

- 读取 table -> vtable 的 offset；
- 检查 slot；
- 读取 table -> field；
- 若字段是对象，再跟随一次 uoffset。

它比读取普通 C++ 成员多一些依赖加载，但可换来无需对象化解析和良好演进能力。最终性能取决于 cache、访问字段数量和重复次数。

---

## 21. Offset、String、Vector 与 Union

### 21.1 Offset 是“从自己到目标”的距离

对 table 中指向 string/vector/child table 的 offset：

```text
target_address = offset_word_address + read_uoffset(offset_word_address)
```

常规对象 offset 指向前方；vtable 使用另一种有符号后向关系。不要把二者混成“所有 offset 都从 buffer 开头算”。

### 21.2 String

FlatBuffers string 的典型布局：

```text
[uint32 byte_length][UTF-8 bytes][NUL terminator][padding]
```

`byte_length` 不包含末尾 NUL。长度按字节，不按 Unicode code point。即使有 NUL terminator，业务仍应使用长度，因为字符串内容和语言 API 的约束可能不同。

### 21.3 Vector

```text
[uint32 element_count][element 0][element 1]...[padding]
```

但 element 的物理含义取决于类型：

- scalar vector：标量连续排列；
- struct vector：固定布局 struct 连续内联；
- table/string vector：连续排列的是 offset，不是对象本体；
- union vector：通常还需要对应的 type/discriminant vector，具体支持看目标语言。

### 21.4 为什么 vector 适合批量读取

scalar/struct vector 的数据连续，适合 cache line、SIMD 或图形 API。但前提是直接使用底层表示且平台条件满足。若把每个元素转换成本地对象，仍会引入额外成本。

### 21.5 Union

FlatBuffers union 在 wire 中概念上由两部分构成：

```text
type field: discriminant enum
value field: offset to selected object
```

`NONE = 0` 通常表示未设置。演进 union 时要稳定 discriminant 数值，新增 variant 放在末尾；旧代码看到不认识的 variant 时，业务层必须有明确降级策略。

### 21.6 File identifier 与 size prefix

`file_identifier "TLM1"` 可在 root offset 后放四字节标识，用于快速拒绝明显错误类型。它不是密码学认证，也不能替代 verifier。

某些存储/传输场景可使用 size-prefixed buffer，让 buffer 前带整体大小。是否使用、长度是否可信、如何做上限检查，必须在外层协议中统一。

---

## 22. 为什么 Builder 从后向前构造

### 22.1 相对 offset 带来的依赖

父 table 要写一个 offset 指向 child。若 child 已经在 buffer 中，距离立即可知：

```text
child address 已知
offset word address 已知
distance = child - offset_word
```

若从前向后写父对象，而 child 尚未创建，就需要预留位置和回填。回填会增加 bookkeeping，并可能在 buffer 扩容搬家时变复杂。

### 22.2 反向构建

FlatBufferBuilder 从 buffer 高地址向低地址生长：

```text
先创建叶子：string、vector、child table
再创建引用它们的 parent table
最后 Finish(root)
```

![FlatBufferBuilder 从叶子到根反向构造相对 offset](https://oss.euler.icu/teaser/docs/data-serialization/protobuf-flatbuffers/05-flatbuffers-backward-builder.png?v=20260713-final)

### 22.3 一个逻辑构建顺序

```cpp
flatbuffers::FlatBufferBuilder builder;

auto name = builder.CreateString("sensor-a");
std::vector<uint16_t> raw{10, 20, 30};
auto samples = builder.CreateVector(raw);
Tutorial::Telemetry::Vec3 position(1.0f, 2.0f, 3.0f);

auto root = Tutorial::Telemetry::CreateTelemetry(
    builder,
    42,
    &position,
    name,
    samples,
    Tutorial::Telemetry::Status_Online);

Tutorial::Telemetry::FinishTelemetryBuffer(builder, root);
```

`Vec3` 是 inline struct，生成的 table 构造函数在调用期间复制它；`name` 和 `samples` 则是已经创建好的 child offset。核心顺序是先创建 child，再创建引用它们的 parent。

### 22.4 “反向”不改变最终读取方向

反向只是 builder 的实现策略。`GetBufferPointer()` 返回的是最终 buffer 起点，读者仍从 root offset 开始按引用遍历。不要把“反向构建”误解为字节需要逆序传输。

### 22.5 扩容仍可能发生

Builder 的底层数组容量不足时仍要扩容和移动已有字节。offset 是相对位置，加上 builder 的内部修正策略，使移动可管理。若已知典型消息大小，可给 builder 合理初始容量，但应通过 profiling 决定，而不是随意设成巨大值。

---

## 23. 读取、生命周期、验证与并发

### 23.1 Getter 返回的是 buffer 内视图

```cpp
const auto* root = Tutorial::Telemetry::GetTelemetry(buffer.data());
const auto* name = root->name();
```

`root` 和 `name` 指向或引用 `buffer` 内部。以下操作会使它们悬空：

- `buffer` 被销毁；
- `std::vector<uint8_t>` 扩容；
- memory mapping 被解除；
- 接收缓冲区被连接池复用；
- builder 被 `Clear()` 或销毁。

必须让 buffer 生命周期覆盖所有 accessor/view。

### 23.2 GetRoot 不等于 Verify

`GetRoot<T>()` 只根据 root offset 计算指针，不会完整证明所有后续 offset 安全。对不可信输入，应先：

```cpp
flatbuffers::Verifier verifier(buffer.data(), buffer.size());
if (!Tutorial::Telemetry::VerifyTelemetryBuffer(verifier)) {
    return InvalidData();
}
const auto* root = Tutorial::Telemetry::GetTelemetry(buffer.data());
```

Verifier 检查：

- offset 和字段访问是否落在 buffer 内；
- vector/string 长度是否可容纳；
- string 是否正确终止；
- 对齐与嵌套结构；
- 可配置的深度和 table 数量上限。

### 23.3 File identifier 只做快速类型检查

四字节 identifier 可能被伪造，也可能碰撞。正确顺序可以是：

```text
先检查外层 frame size
-> 检查 identifier，快速拒绝类型错误
-> verifier
-> 业务字段校验
```

### 23.4 并发读取

只读 FlatBuffer 不修改原 buffer，可由多线程并发访问，无需为 accessor 自身加锁。前提仍是：

- buffer 生命周期稳定；
- 没有线程通过 mutable API 改写；
- 持有 buffer 的容器不会扩容或释放。

### 23.5 Builder 不能跨线程共享写入

`FlatBufferBuilder` 保存构建状态，不是线程安全对象。推荐每线程或每任务独立 builder，必要时做对象池，但归还前清理状态并控制保留容量，避免少数超大消息让池中每个 builder 永久占用大内存。

---

## 24. 修改、Object API 与反射

### 24.1 原地修改为何受限

若一个标量字段已存在并且宽度不变，可以在启用 mutable 生成选项后原地写新值。但下面的操作通常会改变布局：

- 给原本省略的字段增加存储；
- 把字符串从 4 字节改成 100 字节；
- 给 vector 增加元素；
- 替换为更大的 child table。

这些操作可能推动后续对象并使大量 relative offset 失效，因此基础 API 不把 FlatBuffer 当作任意可变对象树。

### 24.2 Object API

使用 `--gen-object-api` 可生成 native object：

```text
FlatBuffer --UnPack--> NativeTable / std::string / std::vector
Native object --Pack--> 新 FlatBuffer
```

它让修改更自然，但重新引入：

- 对象分配；
- 字符串和 vector 拷贝；
- 完整遍历；
- 新 buffer 构建。

Object API 是便利性模式，不应仍宣称为零拷贝读取。

### 24.3 Reflection

FlatBuffers 可把 schema 编译成 binary schema（`.bfbs`），运行时据此通用遍历。适合：

- inspector；
- 数据迁移工具；
- 通用网关；
- schema-aware 日志和调试。

反射路径比静态生成 accessor 更通用，也更复杂。核心业务 hot path 通常应保留静态路径。

### 24.4 Resizing 的边界

反射库提供某些 resize 能力，但它可能移动数据和修正 offset，不等于普通 pointer mutation。使用前应检查目标语言、版本、复杂度和性能，并对旧 view 全部失效做明确处理。

---

## 25. FlatBuffers schema 演进规则

![Protobuf 与 FlatBuffers 的 schema 演进规则对照](https://oss.euler.icu/teaser/docs/data-serialization/protobuf-flatbuffers/06-schema-evolution-rules.png?v=20260713-final)

### 25.1 Table 新字段默认追加到末尾

```fbs
// V1
table User {
  id:ulong;
  name:string;
}

// V2：安全方向
table User {
  id:ulong;
  name:string;
  nickname:string;
}
```

字段 slot 由声明顺序决定。追加使已有 slot 保持不变。若要自由重排声明，应从一开始就在该 table 的所有字段上显式使用连续 `id`，并遵守官方规则；不能只给部分字段加 `id`。

### 25.2 不删除 slot，使用 deprecated

错误：

```fbs
// 删除第一个字段会让后面 slot 左移。
table User {
  name:string;
}
```

正确方向：

```fbs
table User {
  id:ulong (deprecated);
  name:string;
}
```

deprecated 让新生成代码不再暴露该字段，但保留 slot 身份。

### 25.3 不改变默认值

省略字段的值来自读端 schema。默认值从 0 改为 1 后，同一旧 buffer 会被解释成不同值。即使你控制当前服务，也要考虑历史文件、缓存、离线任务和回滚二进制。

### 25.4 类型修改要极度保守

官方 evolution 文档提到某些相同宽度类型更改可能可行，但符号、浮点解释、语言 API 和业务语义都可能破坏。工程上优先新增字段并迁移，不要把“物理宽度相同”当作“协议相同”。

### 25.5 Struct 不能像 Table 那样演进

struct 是固定 inline layout。增加字段会改变大小和后续 offset/stride，旧代码无法通过 vtable 返回默认值，因为 struct 没有 vtable。已发布 struct 应视为冻结格式；需要演进时新增一个 struct/table 类型和新字段。

### 25.6 Rename 的二进制与文本影响不同

字段名通常不写入二进制，所以 rename 不改变 slot。但它会影响：

- 生成 API；
- JSON 输入输出；
- 反射名称；
- 配置和脚本；
- 源码兼容。

“二进制兼容”不是“零迁移成本”。

### 25.7 Union 与 Enum

- 新值追加到末尾，保持旧数值；
- 不复用已删除的 discriminant；
- 旧代码遇到未知 variant 时不能盲目取 value；
- 业务逻辑提供 unknown/default 分支。

### 25.8 旧代码重建 buffer 时的未知字段

FlatBuffers 常见读取路径不解析重写，所以未知字段仍在原 buffer 中。但若旧代码 `UnPack()` 为只认识旧 schema 的 native object，再 `Pack()` 新 buffer，它无法重建不认识的字段，未知数据会丢失。

这与 Protobuf unknown field set 的 round-trip 机制不同。设计中要明确中间节点是“透明转发原 bytes”，还是“读取后重建”。

---

## 26. FlatBuffers C++ 端到端示例

### 26.1 Schema

保存为 `telemetry.fbs`：

```fbs
namespace Tutorial.Telemetry;

enum Status : byte {
  Unknown = 0,
  Online = 1,
  Offline = 2
}

struct Vec3 {
  x:float;
  y:float;
  z:float;
}

table Telemetry {
  device_id:uint;
  position:Vec3;
  name:string;
  samples:[ushort];
  status:Status = Unknown;
}

root_type Telemetry;
file_identifier "TLM1";
```

生成 C++ 头文件：

```bash
flatc --cpp telemetry.fbs
```

### 26.2 构建 buffer

```cpp
#include <cstdint>
#include <iostream>
#include <span>
#include <string_view>
#include <vector>

#include "flatbuffers/flatbuffers.h"
#include "telemetry_generated.h"

std::vector<uint8_t> buildTelemetry() {
    flatbuffers::FlatBufferBuilder builder(256);

    const auto name = builder.CreateString("sensor-a");
    const std::vector<uint16_t> values{10, 20, 30};
    const auto samples = builder.CreateVector(values);
    const Tutorial::Telemetry::Vec3 position(1.0f, 2.0f, 3.0f);

    const auto root = Tutorial::Telemetry::CreateTelemetry(
        builder,
        42,
        &position,
        name,
        samples,
        Tutorial::Telemetry::Status_Online);

    Tutorial::Telemetry::FinishTelemetryBuffer(builder, root);

    const uint8_t* begin = builder.GetBufferPointer();
    return std::vector<uint8_t>(begin, begin + builder.GetSize());
}
```

将 builder 内容复制到 `std::vector` 是为了让函数返回后 bytes 继续存活。若直接返回 accessor 指针，它会在局部 builder 销毁后悬空。

### 26.3 验证并读取

```cpp
bool printTelemetry(std::span<const uint8_t> bytes) {
    flatbuffers::Verifier verifier(bytes.data(), bytes.size());
    if (!Tutorial::Telemetry::VerifyTelemetryBuffer(verifier)) {
        return false;
    }

    const auto* message =
        Tutorial::Telemetry::GetTelemetry(bytes.data());

    std::cout << "id=" << message->device_id() << '\n';

    if (const auto* name = message->name()) {
        const std::string_view nameView(name->c_str(), name->size());
        std::cout << "name=" << nameView << '\n';
    }

    if (const auto* samples = message->samples()) {
        for (flatbuffers::uoffset_t i = 0; i < samples->size(); ++i) {
            std::cout << samples->Get(i) << ' ';
        }
        std::cout << '\n';
    }
    return true;
}
```

`String::c_str()/size()` 与 `Vector::Get()` 直接表达了底层 view 的边界。不要只依赖 NUL 终止来推导字符串长度，也不要把这些 view 保存到其 buffer 生命周期之外。

### 26.4 完整调用

```cpp
int main() {
    const auto bytes = buildTelemetry();
    if (!printTelemetry(bytes)) {
        std::cerr << "invalid FlatBuffer\n";
        return 1;
    }
}
```

### 26.5 文件映射的真正价值

若一个大型只读 FlatBuffer 文件通过 `mmap` 映射：

```text
文件页 -> 进程虚拟地址 -> accessor 按需触页
```

可以避免“先读完整文件，再构造完整对象树”。但：

- 首次访问仍可能 page fault；
- 映射必须保持有效；
- 文件仍需 verifier 或可信构建链；
- 随机访问模式可能造成差的 page locality。

所谓零拷贝收益要结合 OS page cache 和访问局部性理解。

---

## 27. 两种格式的成本模型

设：

- `B`：消息字节数；
- `F`：字段数；
- `K`：实际访问字段数；
- `N`：嵌套对象/容器数；
- `R`：同一消息重复遍历次数。

### 27.1 Protobuf 的粗略模型

```text
首次解析成本 ≈ scan(B) + allocate(N) + copy(strings/vectors)
后续访问成本 ≈ R * object_access(K)
峰值内存 ≈ input B + native object tree + allocator overhead
```

可通过 Arena、对象复用、流式接口和避免不必要业务转换降低常数，但整体仍偏向 eager materialization。

### 27.2 FlatBuffers 的粗略模型

```text
首次读取成本 ≈ verify(reachable structure) + access(K)
后续访问成本 ≈ R * vtable/offset_access(K)
峰值内存 ≈ input B + small views
```

如果省略 verifier，首次成本更低但只适用于可信 buffer。若 `UnPack()`：

```text
成本重新接近 scan + allocate + copy
峰值内存也重新出现 native object tree
```

### 27.3 交叉点在哪里

![Protobuf 对象化读取与 FlatBuffers 按需读取的成本交叉](https://oss.euler.icu/teaser/docs/data-serialization/protobuf-flatbuffers/07-read-cost-crossover.png?v=20260713-final)

典型趋势：

- `K << F`、只读一次：FlatBuffers 更有机会占优；
- 完整读取且大量业务转换：差距缩小；
- 解析一次、反复复杂修改：Protobuf 对象模型通常更自然；
- 消息很小：框架、函数调用和网络成本可能淹没格式差异；
- 极小整数很多：Protobuf varint 的体积优势可能显著；
- 大型 scalar/struct vector：FlatBuffers 连续原地访问可能显著。

这是推理模型，不是 benchmark 结论。

---

## 28. Protobuf 与 FlatBuffers 系统对比

| 维度 | Protobuf | FlatBuffers |
| --- | --- | --- |
| 核心表示 | tag-value 记录流 | offset 连接的可遍历 buffer |
| 字段定位 | 解析时扫描 tag | table 通过 vtable slot 定位 |
| 常规读取 | 先解析为生成对象 | 验证后按需 accessor |
| 常规修改 | 对象 API 自然 | 原地修改受限，复杂修改常重建 |
| 小整数体积 | Varint 常很紧凑 | 标量固定宽度并有对齐 |
| 大向量 | 解析进容器 | scalar/struct vector 可连续原地读 |
| 内存分配 | 子对象和容器可能较多，Arena 可缓解 | 基础读取不创建对象树 |
| 演进身份 | field number | table field slot 或显式 `id` |
| 缺失字段 | presence/default 规则 | vtable 缺失 entry 返回默认/null |
| 未知字段 | 二进制 runtime 常可保存并 round-trip | 原 buffer 自然保留；旧 schema 重建会丢 |
| 字节规范化 | 不保证 canonical | 构建顺序/布局也不应当作 canonical |
| 不可信输入 | 解析器 + size/depth limit | 外层 size limit + Verifier + 业务校验 |
| 典型优势 | RPC 生态、紧凑传输、可变对象、跨语言成熟 | 读路径低分配、部分访问、mmap/只读数据 |

### 28.1 不能简单说“FlatBuffers 没有反序列化”

更准确：FlatBuffers 基础 API 不把整个 buffer 物化为 native object tree。它仍需要：

- 将 bytes 放在可访问内存中；
- 验证不可信结构；
- 解释 little-endian 标量和 offset；
- 可能进行字符串/业务类型转换。

### 28.2 不能简单说“Protobuf 不能零拷贝”

某些 runtime 提供 zero-copy input/output stream，含义通常是减少 I/O buffer 之间的额外复制，并不等于解析后没有对象或字符串拥有成本。术语必须说明它消除了哪一次 copy。

### 28.3 两者都不是数据库

它们不自动提供：

- 事务；
- 索引；
- 并发控制；
- 日志恢复；
- 权限；
- 数据生命周期管理。

FlatBuffer 可以被 mmap，不代表它自动拥有数据库查询能力；Protobuf 可用于 RPC，不代表它自动定义服务语义。

---

## 29. 如何按场景做选型

### 29.1 优先考虑 Protobuf

- RPC/API 消息，需要成熟跨语言生态；
- 消息较小，小整数和稀疏字段很多；
- 接收后会修改、合并、校验并构造业务对象；
- 需要成熟的 descriptor、reflection、gRPC 和工具链；
- 同一消息会解析一次后在对象层重复复杂使用；
- 团队更重视协议治理和生态标准化。

### 29.2 优先考虑 FlatBuffers

- 大型只读数据，读取远多于写入；
- 经常只访问消息中的少量字段；
- 内存分配和峰值是已测量的瓶颈；
- 数据可保持为连续 buffer 或 memory mapping；
- 大量 scalar/struct vector 需要原地遍历；
- 游戏、移动端、资源包或低延迟只读路径。

### 29.3 可能两者都不合适

- 人工频繁编辑：JSON/YAML/TOML 可能更合适；
- 分析型列扫描：Parquet/Arrow 类列式格式可能更合适；
- schema-less 动态数据：考虑 CBOR、MessagePack、FlexBuffers 等，但仍需治理；
- 需要随机更新、事务和索引：使用数据库或专门存储格式；
- 共享内存中的可变并发对象：需要专门并发布局和同步协议。

### 29.4 决策树

```text
是否需要读取前避免完整对象化？
  否 -> Protobuf 通常是稳健默认项
  是
  |
  +-- buffer 能否在访问期稳定存活？
  |     否 -> 先解决 ownership；FlatBuffers 优势难成立
  |
  +-- 数据是否以读取为主，修改可通过重建完成？
  |     否 -> Protobuf/native object 可能更自然
  |
  +-- profiling 是否显示 parse/allocation 是真实瓶颈？
        否 -> 先测量，不因口号迁移
        是 -> 用真实数据做 FlatBuffers 原型
```

### 29.5 混合架构

系统可以按边界使用不同格式：

```text
外部 RPC：Protobuf
  -> 业务验证和转换
内部只读资源：FlatBuffers
  -> mmap / shared immutable buffer
长期分析：Parquet
```

不要为了“全公司统一一个格式”而忽略不同数据路径的目标。统一治理规则比统一所有物理表示更重要。

---

## 30. 如何做可信的性能测试

### 30.1 先写 workload 假设

至少记录：

- 消息大小分布，而不只是平均值；
- 字段数量与缺失率；
- 整数数值分布；
- 字符串和 vector 长度；
- 嵌套深度；
- 读取字段比例 `K/F`；
- 每条消息重复访问次数；
- 生产者与消费者线程数；
- 是否压缩、加密、校验、mmap；
- buffer ownership 和分配器策略。

### 30.2 分开测量阶段

```text
1. build/encode
2. bytes copy / transport simulation
3. verify
4. parse or first access
5. selected-field access
6. full traversal
7. conversion to domain object
8. destruction/release
```

把所有阶段混成一个总时间，会无法解释优化来自哪里。

### 30.3 至少报告这些指标

- p50/p95/p99 延迟；
- throughput；
- serialized size 分位数；
- allocation 次数与字节；
- peak RSS；
- CPU cycles/instructions；
- cache miss 和 branch miss（平台允许时）；
- warm cache 与 cold cache；
- 编译器、优化级别、库版本和硬件。

### 30.4 防止编译器消除工作

benchmark 必须消费输出并检查结果。否则编译器可能删掉完整解析或访问。使用框架提供的 `DoNotOptimize`/`ClobberMemory`，并在 benchmark 外验证语义正确性。

### 30.5 使用真实 schema 和数据

一个只有两个整数的 microbenchmark 不能代表包含字符串、嵌套列表和 optional 字段的生产消息。至少准备：

- 小消息典型样本；
- p95 大消息；
- 极端合法消息；
- 只读 2 个字段；
- 完整遍历；
- 解析后重复访问；
- 不可信输入校验路径。

### 30.6 公平比较 ownership

错误比较：

```text
Protobuf：从 socket bytes 解析并复制
FlatBuffers：直接读取预热的静态内存
```

公平比较要么都从同一输入 buffer 开始，要么明确报告端到端路径。若 FlatBuffers 借助 mmap，这是有效架构优势，但必须把 page fault 和 verifier 纳入相应场景。

---

## 31. 安全、可靠性与生产检查清单

### 31.1 接收任何格式之前

- 外层 frame length 有硬上限；
- 在分配前检查 length；
- 整数加法和 `offset + length` 使用防溢出检查；
- 设置连接级和请求级资源预算；
- 超时与取消能中断大消息处理；
- 压缩数据有解压后大小和压缩比上限。

### 31.2 Protobuf

- 使用官方 parser，不手写生产解析器；
- 限制总消息大小和递归深度；
- 对 repeated/map 长度做业务上限；
- 不把 unknown fields 当作已授权数据；
- 解析后验证 enum、字符串、ID 和跨字段约束；
- 不用序列化 bytes 直接做稳定 hash/签名；
- 不把 TextFormat 当作不可信交换格式。

### 31.3 FlatBuffers

- 网络、用户文件和可变存储先跑 verifier；
- verifier 之前先检查外层 buffer 大小；
- 需要时收紧 max depth/max tables；
- accessor 生命周期不超过 buffer；
- 不对未验证 bytes 调用 `GetRoot` 后直接遍历；
- file identifier 不能替代 verifier 和鉴权；
- 共享 buffer 时明确只读性和解除映射时机。

### 31.4 Schema 治理

- schema 进入版本控制；
- CI 执行格式/lint 和 breaking-change 检查；
- 发布生成器与 runtime 的版本矩阵；
- 删除字段前先统计生产者/消费者；
- 保留跨版本 golden bytes；
- 同时测试 `old reader <- new writer` 与 `new reader <- old writer`；
- 测试 read-modify-write 是否保留需要的数据。

### 31.5 数据真实性

二进制格式不是安全协议。需要时在外层增加：

- TLS 或传输认证；
- MAC/数字签名；
- checksum 检测非恶意损坏；
- replay protection；
- schema/type 标识；
- 租户和权限校验。

---

## 32. 常见误区与反例

### 32.1 误区：二进制一定比 JSON 小

极小消息中，固定 header、offset、vtable 和 padding 可能占较大比例；短字段名的 JSON 经过压缩后也可能很小。通常趋势不能代替真实样本测量。

### 32.2 误区：Protobuf 字段按声明顺序编码

wire format 不保证字段顺序。依赖顺序会在跨语言、unknown fields、map 或 runtime 升级时出错。

### 32.3 误区：开启 deterministic 就可做内容寻址

确定性不等于跨版本 canonical。应用必须定义自己的规范化规则和测试向量。

### 32.4 误区：FlatBuffers 就是把 C struct 写进文件

table 有 vtable，引用使用相对 offset，字符串和 vector 有长度，对齐由格式控制。只有 FlatBuffers struct 的部分布局接近固定 C struct，也仍受 little-endian 和生成规则约束。

### 32.5 误区：FlatBuffers 完全不解析，所以不需要验证

正因为 accessor 会直接跟随 offset，恶意 offset 更需要 verifier。跳过验证是可信数据路径的性能决策，不是格式自动安全。

### 32.6 误区：零拷贝意味着没有任何 copy

问清是哪一段：

```text
NIC -> kernel
kernel -> user buffer
decompression -> output
buffer -> native object
native object -> business object
```

FlatBuffers 主要消除最后几类中的“完整对象化”路径；其他 copy 取决于系统架构。

### 32.7 误区：兼容格式保证兼容业务

旧代码能跳过新字段，不代表它能正确处理新状态。支付、权限、状态机和风控必须设计 unknown 行为。

### 32.8 误区：字段删了几年后号码就能复用

历史备份、离线日志和极少升级的客户端可能永久存在。协议身份原则上不回收。

### 32.9 误区：FlatBuffers 一定比 Protobuf 大

FlatBuffers 有 offset、vtable 和 padding；Protobuf 有重复 tag、长度和 varint。实际大小取决于字段数、缺失率、整数分布、vector、字符串和对象复用。结论必须来自 schema 与样本。

### 32.10 误区：一种格式应该覆盖所有数据

RPC、资源文件、分析数据和数据库页的访问模式不同。系统设计目标是稳定边界与可控转换，不是追求格式宗教式统一。

---

## 33. 从零实现时应抓住的最小算法

自己实现最小版本的目的，是建立机械理解，不是替代官方库。

### 33.1 最小 Protobuf reader

需要实现：

1. bounded varint reader；
2. `tag >> 3` 和 `tag & 7`；
3. I32/I64 的小端读取；
4. LEN 的长度和边界检查；
5. unknown field skip；
6. ZigZag inverse；
7. 顶层 message end 由调用者传入。

核心 skip：

```cpp
bool skipField(uint32_t wireType,
               std::span<const uint8_t> input,
               std::size_t& cursor) {
    if (cursor > input.size()) return false;

    uint64_t length = 0;
    switch (wireType) {
    case 0:
        return readVarint64(input, cursor, length);
    case 1:
        if (input.size() - cursor < 8) return false;
        cursor += 8;
        return true;
    case 2:
        if (!readVarint64(input, cursor, length)) return false;
        if (length > input.size() - cursor) return false;
        cursor += static_cast<std::size_t>(length);
        return true;
    case 5:
        if (input.size() - cursor < 4) return false;
        cursor += 4;
        return true;
    default:
        return false;  // 教学版省略 group；生产中不能这样替代官方 parser。
    }
}
```

这里使用 `length > size - cursor`，而不是 `cursor + length > size`，是为了避免加法溢出。

### 33.2 最小 FlatBuffers reader

教学版只读一个已验证、结构固定的 table，核心步骤：

1. 读取 root uoffset；
2. 找 table 起点；
3. 读取 table 的 soffset，找到 vtable；
4. 检查 vtable size；
5. 读取字段 entry；
6. entry 非 0 时读取 field；
7. offset field 从 offset word 自身位置加相对距离。

但要注意：完整 verifier 还必须递归检查所有边界、对齐、vector、string、union 和深度。几行教学代码无法安全处理攻击者输入。

### 33.3 两个最小实现揭示的差异

```text
Protobuf reader 的核心循环：顺序发现“下一个记录是什么”。
FlatBuffers reader 的核心动作：从已知 slot 计算“目标字段在哪里”。
```

这就是记录流与可寻址布局的根本差别。

---

## 34. 练习题与答案线索

### 34.1 Protobuf 字节题

给定：

```proto
message M {
  uint32 a = 1;
  sint32 b = 2;
  string c = 4;
}
```

值为 `a=300, b=-1, c="A"`。写出 hex。

答案线索：

```text
a tag 08, 300 -> ac 02
b tag 10, ZigZag(-1) -> 1
c tag (4 << 3) | 2 -> 22, length 1, ASCII 41
```

### 34.2 Unknown field 题

V1 只认识 field 1，V2 新增 field 9 的 string。V1 如何跳过 field 9？

答案线索：先从 tag 得到 wire type 2，再读 varint length，检查边界后移动游标；业务类型不需要知道。

### 34.3 Presence 题

设计“把重试次数设置为 0”和“不修改重试次数”两种请求。为什么 implicit scalar 不够？可用哪些方式？

答案线索：显式 `optional`/Editions presence，或 `FieldMask`，或命令式 operation。

### 34.4 FlatBuffers 地址题

某 offset word 位于 buffer 地址 100，值为 24。目标地址是多少？如果整个 buffer 搬到新基址后，该相对值为何仍有效？

答案线索：目标 124；offset word 与目标一起平移，差保持 24。

### 34.5 Vtable 题

新代码请求 slot 5，但旧 buffer 的 vtable 到 slot 3 就结束。正确 accessor 应怎样处理？

答案线索：先比较 entry 位置与 vtable size，不读取越界，返回 schema 默认值/null。

### 34.6 Builder 题

为什么先创建 string，再创建引用它的 table？

答案线索：父对象写 offset 时，子对象位置已经确定；反向 builder 可避免未决引用和回填。

### 34.7 选型题

一个 50 MiB 地图资源包含百万个固定坐标，启动后 mmap，只读取可见区域；另一个接口每次传 1 KiB 请求并在服务端反复修改。分别倾向什么格式？

答案线索：前者值得用 FlatBuffers struct vector 做原型；后者通常倾向 Protobuf。最终仍需真实 benchmark 和生态评估。

### 34.8 兼容迁移题

如何把已发布的 `string amount = 5` 迁移为整数最小货币单位？

答案线索：新增号码、读回退、双写、回填、停旧写、reserved；不要原地改 type。

---

## 35. 学习路线与官方资料

### 35.1 第一阶段：会手算

完成：

- 手算 150 和 300 的 varint；
- 写出 tag 公式；
- 区分 varint、I32、I64、LEN；
- 画出 root -> vtable -> field 和 offset -> child；
- 解释 table 与 struct 的差别。

### 35.2 第二阶段：会使用

- 用 `protoc` 和 `flatc` 生成同一业务模型；
- 分别完成 build、serialize/read、错误处理；
- dump hex 并与手算对应；
- 对 FlatBuffer 故意破坏一个 offset，观察 verifier；
- 对 Protobuf 增加未知字段，测试旧 reader round-trip。

### 35.3 第三阶段：会演进

- 准备 V1/V2 两套生成代码；
- 测 `V1 writer -> V2 reader`；
- 测 `V2 writer -> V1 reader`；
- 测 `V2 -> V1 read-modify-write -> V2`；
- 分别经过 JSON/Object API，观察未知数据是否保留；
- 把兼容测试放进 CI。

### 35.4 第四阶段：会测量

- 真实 schema、真实数据分布；
- 分离 encode、verify、parse、access、UnPack；
- 比较少字段访问与完整遍历；
- 记录 allocation、RSS 和 tail latency；
- 用结果解释成本来自哪里。

### 35.5 官方资料

Protobuf：

- [Encoding / Wire Format](https://protobuf.dev/programming-guides/encoding/)
- [Language Guide: Editions](https://protobuf.dev/programming-guides/editions/)
- [Edition 2024 Language Specification](https://protobuf.dev/reference/protobuf/edition-2024-spec/)
- [Editions Overview](https://protobuf.dev/editions/overview/)
- [Field Presence](https://protobuf.dev/programming-guides/field_presence/)
- [Proto Best Practices](https://protobuf.dev/best-practices/dos-donts/)
- [Proto Serialization Is Not Canonical](https://protobuf.dev/programming-guides/serialization-not-canonical/)
- [Proto Limits](https://protobuf.dev/programming-guides/proto-limits/)
- [Cross-Version Runtime Guarantee](https://protobuf.dev/support/cross-version-runtime-guarantee/)
- [C++ Tutorial](https://protobuf.dev/getting-started/cpptutorial/)

FlatBuffers：

- [Overview](https://flatbuffers.dev/)
- [Schema](https://flatbuffers.dev/schema/)
- [Internals](https://flatbuffers.dev/internals/)
- [Schema Evolution](https://flatbuffers.dev/evolution/)
- [White Paper](https://flatbuffers.dev/white_paper/)
- [Using flatc](https://flatbuffers.dev/flatc/)
- [C++ Language Guide](https://flatbuffers.dev/languages/cpp/)
- [Official Repository](https://github.com/google/flatbuffers)

### 35.6 最终自检

如果能够不看资料回答下面六问，就已经掌握了主干：

1. Protobuf 为何能跳过未知字段？
2. `sint32` 为何适合接近 0 的负数？
3. deterministic serialization 为何不等于 canonical serialization？
4. FlatBuffers 的 vtable 如何同时实现可选字段和新旧版本读取？
5. “零拷贝读取”依赖哪些 buffer 生命周期与安全前提？
6. 给出一个工作负载时，如何从 `B、K/F、N、R` 和 ownership 推导选型？

真正的掌握不是记住结论，而是能够从字节边界、字段身份、地址表达和版本错位四个问题重新推导结论。
