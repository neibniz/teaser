# LevelDB 与 RocksDB：功能、架构设计与核心实现原理

> 目标读者：已经了解基本数据结构、文件系统和后端开发，希望系统理解嵌入式 KV 存储引擎的初学者。
> 学习目标：不只是知道 LevelDB 和 RocksDB 都基于 LSM Tree，而是能理解它们为什么这样设计、读写路径如何工作、Compaction 为什么重要、如何根据业务场景选型与调优。
> 说明：本文依据 LevelDB 官方仓库文档、RocksDB 官方 Wiki 与公开 API 文档整理。具体 API、默认参数和配置项会随版本演进，生产使用以项目当前版本文档和源码为准。

---

## 目录

1. [它们到底解决什么问题](#1-它们到底解决什么问题)
2. [一句话理解 LevelDB 与 RocksDB](#2-一句话理解-leveldb-与-rocksdb)
3. [先建立核心心智模型：磁盘写入为什么难](#3-先建立核心心智模型磁盘写入为什么难)
4. [LSM Tree：把随机写变成顺序写](#4-lsm-tree把随机写变成顺序写)
5. [最重要的三类放大问题](#5-最重要的三类放大问题)
6. [LevelDB 的功能定位](#6-leveldb-的功能定位)
7. [LevelDB 的架构设计](#7-leveldb-的架构设计)
8. [LevelDB 的写入路径](#8-leveldb-的写入路径)
9. [LevelDB 的读取路径](#9-leveldb-的读取路径)
10. [LevelDB 的 SSTable 文件格式](#10-leveldb-的-sstable-文件格式)
11. [LevelDB 的 Compaction 原理](#11-leveldb-的-compaction-原理)
12. [LevelDB 的一致性、快照与恢复](#12-leveldb-的一致性快照与恢复)
13. [RocksDB 的功能定位](#13-rocksdb-的功能定位)
14. [RocksDB 的架构设计](#14-rocksdb-的架构设计)
15. [RocksDB 的核心能力扩展](#15-rocksdb-的核心能力扩展)
16. [RocksDB 的 Compaction 体系](#16-rocksdb-的-compaction-体系)
17. [RocksDB 的内存、缓存与读优化](#17-rocksdb-的内存缓存与读优化)
18. [RocksDB 的写入优化与后台任务](#18-rocksdb-的写入优化与后台任务)
19. [LevelDB 与 RocksDB 横向对比](#19-leveldb-与-rocksdb-横向对比)
20. [如何根据业务场景选型](#20-如何根据业务场景选型)
21. [初学者应该掌握的推理方法](#21-初学者应该掌握的推理方法)
22. [简单使用示例](#22-简单使用示例)
23. [调优思路与常见参数](#23-调优思路与常见参数)
24. [常见误区](#24-常见误区)
25. [学习路线](#25-学习路线)
26. [官方资料](#26-官方资料)

---

## 1. 它们到底解决什么问题

LevelDB 和 RocksDB 都属于嵌入式键值存储引擎。

“嵌入式”意味着：

```text
你的应用进程
  |
  | 直接调用 C++ 库 API
  v
LevelDB / RocksDB
  |
  v
本地磁盘文件
```

它们不是 MySQL、PostgreSQL、Redis 那样的独立服务。没有内置 SQL 层，没有内置网络协议，也没有默认的客户端服务器架构。

它们主要解决的问题是：

```text
如何在本地磁盘上高效、可靠、按 key 有序地保存大量 key-value 数据。
```

典型使用场景包括：

| 场景 | 为什么适合 KV 存储引擎 |
| --- | --- |
| 浏览器 IndexedDB 底层存储 | 需要本地持久化、小型嵌入式库 |
| 区块链节点状态或索引 | 大量 key-value 读写，顺序扫描和随机查询并存 |
| 消息系统本地状态 | 需要高写入吞吐和本地恢复能力 |
| 流处理状态存储 | 状态按 key 读写，崩溃后要恢复 |
| 分布式数据库单机存储引擎 | 上层负责分片、复制、事务协议，下层负责本地持久化 |
| 缓存型持久化索引 | 需要比内存更大容量，同时保留较好读写性能 |

把它们放到系统分层里看：

```text
业务 API / SQL / 分布式协议 / 复制一致性
              |
              v
        本地存储引擎
              |
              v
        文件系统 / SSD / HDD
```

LevelDB 和 RocksDB 主要在“本地存储引擎”这一层。

## 2. 一句话理解 LevelDB 与 RocksDB

| 系统 | 一句话理解 |
| --- | --- |
| LevelDB | Google 开源的轻量级、按 key 有序的嵌入式 KV 存储库，用较少抽象展示 LSM Tree 存储引擎的经典实现 |
| RocksDB | 从 LevelDB 演进而来的高性能嵌入式 KV 存储引擎，面向服务端、SSD、多核和生产运维做了大量增强 |

更形象地说：

```text
LevelDB 像一本结构清晰的 LSM Tree 教材实现。
RocksDB 像一套面向生产环境的 LSM Tree 工业实现。
```

LevelDB 的价值在于：

- 代码量相对小。
- 架构边界清楚。
- 很适合学习 WAL、MemTable、SSTable、Manifest、Compaction 这些基础概念。

RocksDB 的价值在于：

- 性能更强。
- 参数和功能更丰富。
- 更适合高并发、多核、SSD、复杂生产负载。
- 提供 Column Families、Transactions、Merge Operator、Prefix Seek、更多 Compaction 策略、更多缓存和监控能力。

如果学习存储引擎：

```text
先读 LevelDB 理解骨架。
再读 RocksDB 理解工业增强。
```

## 3. 先建立核心心智模型：磁盘写入为什么难

理解 LevelDB 和 RocksDB，首先要理解一个简单事实：

```text
磁盘和 SSD 都更喜欢顺序写，不喜欢大量细碎随机写。
```

假设我们要维护一个有序 Map：

```text
apple  -> 1
banana -> 2
cat    -> 3
dog    -> 4
```

如果每次更新都直接在磁盘上的 B+Tree 或有序文件里原地修改，可能发生：

- 找到 key 所在页面。
- 读取页面。
- 修改页面。
- 写回页面。
- 更新索引。
- 可能发生页面分裂、合并或日志同步。

这种方式读很直观，但写入可能导致很多随机 I/O。

LSM Tree 的思路是反过来：

```text
不要急着把数据写到最终位置。
先把写入追加到日志，再放进内存结构。
内存满了以后，批量写成有序文件。
后台再慢慢合并整理。
```

这是一种典型的工程交换：

| 交换对象 | LSM Tree 的选择 |
| --- | --- |
| 写入 | 尽量顺序写、批量写 |
| 读取 | 可能要查多个层级，需要 Bloom Filter、缓存、索引优化 |
| 空间 | 旧版本和删除标记会暂时存在，需要 Compaction 清理 |
| 后台成本 | Compaction 会持续消耗 CPU、磁盘 I/O 和写放大 |

所以 LevelDB/RocksDB 的设计核心不是“怎么存一个 key-value”，而是：

```text
如何把前台写入变快，同时用后台整理控制读取成本、空间成本和数据版本成本。
```

## 4. LSM Tree：把随机写变成顺序写

LSM Tree 可以从一个很朴素的模型开始理解。

### 4.1 第一层：内存表

新写入先进入内存里的有序结构：

```text
MemTable

apple  -> v3
banana -> v1
cat    -> deleted
dog    -> v2
```

MemTable 通常用跳表、树、向量或哈希加有序结构实现。

它要满足两类需求：

- 写入时能快速插入。
- 读取和 Flush 时能按 key 有序遍历。

### 4.2 第二层：预写日志

内存会丢数据，所以每次写入还要先追加到 WAL：

```text
000001.log

Put apple v3
Put banana v1
Delete cat
Put dog v2
```

WAL 是顺序追加文件。

它的作用是：

```text
进程或机器崩溃后，可以重新播放日志，把 MemTable 恢复出来。
```

注意：

- WAL 保证的是恢复能力。
- 是否每次写都 `fsync` 到稳定存储，取决于写入选项。
- 不同步刷盘时吞吐更高，但机器崩溃可能丢失最近写入。

### 4.3 第三层：不可变有序文件

MemTable 达到阈值后，会被冻结为 Immutable MemTable，然后后台刷成 SSTable：

```text
MemTable 满
    |
    v
Immutable MemTable
    |
    v
000123.ldb / 000123.sst
```

SSTable 的特点：

- 文件写入后基本不再原地修改。
- 文件内部 key 有序。
- 可以通过索引块快速定位数据块。
- 可以带 Bloom Filter，避免无意义磁盘读取。

### 4.4 第四层：多层级结构

磁盘上不是只有一个 SSTable，而是很多个 SSTable，按 level 组织：

```text
L0: [A..K] [C..Z] [B..M]     可能重叠，都是刚 flush 的新文件
L1: [A..F] [G..M] [N..Z]     通常不重叠
L2: [A..D] [E..H] [I..P] [Q..Z]
L3: ...
```

为什么需要 level？

如果只不断生成新 SSTable，不合并，那么读取一个 key 时可能要查很多文件：

```text
Get("apple") -> 查 MemTable -> 查 Immutable MemTable -> 查几十上百个 SSTable
```

这样读会越来越慢。

所以后台要做 Compaction：

```text
把多个有序文件归并成新的有序文件。
丢弃被覆盖的旧版本。
在安全时丢弃删除标记。
让深层文件 key 范围更规整。
```

这就是 LSM Tree 名字里的 Merge。

### 4.5 用归并排序理解 Compaction

假设两个 SSTable 都按 key 有序：

```text
File A:
apple  -> v1
banana -> v1
cat    -> v1

File B:
banana -> v2
dog    -> v1
```

归并时像归并排序一样双指针扫描：

```text
apple  -> v1
banana -> v2   // 新版本覆盖旧版本
cat    -> v1
dog    -> v1
```

如果遇到删除标记：

```text
File A:
cat -> v1

File B:
cat -> tombstone
```

当系统能确认更老层级里不再需要保留 `cat` 的旧值时，就可以把旧值和删除标记都清理掉。

初学者要抓住一句话：

```text
LSM Tree 的写入很像写日志，读取很像查多层索引，维护成本集中在后台归并。
```

## 5. 最重要的三类放大问题

学习 LSM Tree 必须理解三个 amplification。

### 5.1 Write Amplification：写放大

用户写入 1 MB 数据，不代表磁盘只写 1 MB。

在 LSM Tree 中，这 1 MB 可能经历：

```text
写 WAL
写 L0 SSTable
L0 -> L1 Compaction 时再写
L1 -> L2 Compaction 时再写
L2 -> L3 Compaction 时再写
```

如果用户写入 1 MB，底层实际写了 10 MB，那么写放大约为 10。

写放大高会带来：

- SSD 磨损增加。
- 后台 I/O 压力增加。
- 写入尾延迟升高。
- Compaction 跟不上时写入被限速或阻塞。

### 5.2 Read Amplification：读放大

用户查一个 key，不一定只读一个地方。

可能要查：

```text
MemTable
Immutable MemTable
L0 的多个文件
L1 的一个文件
L2 的一个文件
L3 的一个文件
...
```

如果没有 Bloom Filter，很多“不存在”的查询会非常贵。

读放大高会带来：

- 随机读 I/O 增加。
- CPU 解压缩成本增加。
- 缓存命中率下降。
- 查询延迟不稳定。

### 5.3 Space Amplification：空间放大

用户逻辑数据只有 100 GB，磁盘可能占用 130 GB、200 GB 甚至更多。

原因包括：

- 新旧版本同时存在。
- 删除标记暂时存在。
- Compaction 期间输入文件和输出文件同时存在。
- Universal Compaction 等策略可能需要更多临时空间。

空间放大高会带来：

- 磁盘成本增加。
- 磁盘写满风险增加。
- Compaction 空间不足。

### 5.4 三者无法同时最优

LSM Tree 调优的本质是三角平衡：

```text
             低读放大
                /\
               /  \
              /    \
             /      \
      低写放大 ---- 低空间放大
```

通常：

- 更积极 Compaction 可以降低读放大和空间放大，但提高写放大。
- 更少 Compaction 可以降低写放大，但提高读放大和空间放大。
- 更多 Bloom Filter 和缓存可以降低读放大，但增加内存消耗。
- 更大的 MemTable 可以提高写吞吐，但增加内存和恢复时间。

所以不要问：

```text
哪个参数最快？
```

要问：

```text
我的工作负载最怕什么？
随机读延迟？
持续写吞吐？
磁盘空间？
恢复时间？
SSD 写入量？
```

## 6. LevelDB 的功能定位

LevelDB 是一个持久化、有序 key-value 存储库。

官方文档强调的核心能力包括：

| 能力 | 说明 |
| --- | --- |
| 任意字节 key/value | key 和 value 都是 byte array，不要求字符串或固定格式 |
| 按 key 有序存储 | 默认字节序比较，也可以提供自定义 Comparator |
| 基本操作 | `Put`、`Get`、`Delete` |
| 原子批量写 | `WriteBatch` 把多次修改作为一个原子批次提交 |
| 快照 | Snapshot 提供一致性只读视图 |
| 迭代 | 支持正向和反向遍历 |
| 压缩 | 默认使用 Snappy，也支持其他压缩能力，具体以版本为准 |
| Bloom Filter | 通过 FilterPolicy 减少不必要的磁盘读取 |
| Env 抽象 | 文件系统、线程调度等通过 Env 抽象，便于适配环境 |

LevelDB 不提供：

| 不提供什么 | 含义 |
| --- | --- |
| SQL | 没有表、Join、查询优化器 |
| 二级索引 | 需要应用自己维护 |
| 内置网络服务 | 应用要自己封装 RPC 或服务层 |
| 多进程同时打开同一 DB | 同一数据库目录通常只能被一个进程打开 |
| 分布式复制 | 复制、分片、一致性协议由上层系统负责 |

LevelDB 的典型定位：

```text
轻量级、本地、有序、可靠、可嵌入的 KV 存储。
```

它的优势不是功能最多，而是核心设计清楚。

## 7. LevelDB 的架构设计

LevelDB 的主要组件可以这样理解：

```text
应用线程
  |
  | Put / Delete / Write / Get / Iterator
  v
+-------------------------------+
| DBImpl                        |
|                               |
|  MemTable                     |
|  Immutable MemTable           |
|  VersionSet / Version         |
|  TableCache                   |
|  LogWriter / LogReader        |
|  Compaction                   |
+-------------------------------+
  |
  v
文件系统目录
  |
  |-- CURRENT
  |-- MANIFEST-000xxx
  |-- 000xxx.log
  |-- 000xxx.ldb
  |-- LOG
  |-- LOCK
```

### 7.1 MemTable

MemTable 是内存里的有序结构。

LevelDB 经典实现使用 SkipList。

它保存的是内部 key，而不是简单用户 key。

内部 key 可以理解为：

```text
user_key + sequence_number + value_type
```

为什么要加 sequence number？

因为同一个 user key 可能有多个版本：

```text
seq=100 Put    user:1 -> Alice
seq=120 Put    user:1 -> Bob
seq=140 Delete user:1
```

读取时要根据快照版本选择“当前可见的最新版本”。

### 7.2 Immutable MemTable

当 MemTable 达到阈值后，LevelDB 会：

```text
当前 MemTable -> Immutable MemTable
新写入 -> 新 MemTable
后台线程 -> 把 Immutable MemTable flush 成 SSTable
```

这样前台写入不必等待整个刷盘过程完成，除非后台已经跟不上导致写入被限速或阻塞。

### 7.3 WAL Log

WAL 文件保存最近写入。

写入流程中先追加 WAL，再更新 MemTable。

崩溃恢复时：

```text
读取 CURRENT
读取 MANIFEST
找出已有 SSTable
重放仍然需要的 WAL
重建 MemTable 或把日志转换成 L0 SSTable
```

### 7.4 SSTable

SSTable 是磁盘上的有序不可变文件。

LevelDB 文件扩展名常见为 `.ldb`。

它保存：

- 数据块。
- 元数据块。
- 索引块。
- 过滤器块。
- Footer。

### 7.5 Version 与 Manifest

LevelDB 不是直接“扫描目录里的所有文件就是当前状态”。

它用 Version 描述当前数据库由哪些 SSTable 组成：

```text
Level 0: file 101, file 103
Level 1: file 088, file 090
Level 2: file 020, file 021, file 022
```

MANIFEST 是对 Version 变化的日志：

```text
AddFile level=0 file=103 smallest=a largest=z
DeleteFile level=1 file=088
AddFile level=2 file=120 smallest=a largest=m
```

CURRENT 文件保存当前 MANIFEST 文件名。

这样恢复时先读 CURRENT，再读 MANIFEST，就能还原当前版本集合。

### 7.6 TableCache

读取 SSTable 时，如果每次都打开文件、读取索引，会很慢。

TableCache 缓存已打开的 SSTable 句柄和相关 Table 对象。

它降低：

- 文件打开成本。
- 索引读取成本。
- 重复查同一文件的开销。

## 8. LevelDB 的写入路径

以 `Put(key, value)` 为例。

逻辑流程可以理解为：

```text
应用调用 Put
  |
  v
编码成 WriteBatch
  |
  v
追加到 WAL
  |
  v
写入 MemTable
  |
  v
返回成功
```

如果开启同步写：

```text
追加 WAL
  |
  v
fsync / fdatasync
  |
  v
写入 MemTable
  |
  v
返回成功
```

### 8.1 为什么先写 WAL 再写 MemTable

如果先写 MemTable，再写 WAL，崩溃时会出现：

```text
用户以为写成功
MemTable 里有数据
WAL 里没有数据
进程崩溃
数据丢失且无法恢复
```

先写 WAL 的含义是：

```text
只要写入被认为成功，恢复时就能从 WAL 找回来。
```

### 8.2 WriteBatch 的意义

多个修改可以放进一个 WriteBatch：

```text
Delete old_key
Put new_key value
```

它有两个价值：

- 原子性：批次里的修改要么都生效，要么都不生效。
- 性能：多个小写入合并成一次日志追加，降低系统调用和同步刷盘成本。

### 8.3 写入为什么会被阻塞

LevelDB 写入不是永远只追加。

当后台整理跟不上时，前台可能被迫等待。

典型原因：

- MemTable 已满，但 Immutable MemTable 还没 flush 完。
- L0 文件太多，读取成本会变高，需要触发或等待 Compaction。
- 后台 Compaction I/O 跟不上前台写入。

可以这样理解：

```text
前台写入是在借后台 Compaction 的时间。
如果后台还债速度太慢，前台最终会被迫降速。
```

## 9. LevelDB 的读取路径

以 `Get(key)` 为例。

LevelDB 的读取顺序大致是：

```text
1. 查 MemTable
2. 查 Immutable MemTable
3. 查 L0 文件
4. 查 L1、L2、L3... 文件
```

### 9.1 为什么要先查内存

MemTable 和 Immutable MemTable 里的数据最新。

如果磁盘上有：

```text
user:1 -> Alice
```

但 MemTable 里有：

```text
user:1 -> Bob
```

那么读取必须返回 Bob。

所以读取顺序必须从新到旧。

### 9.2 为什么 L0 特殊

L0 文件来自 MemTable flush。

每个 MemTable flush 时都生成一个有序文件，但不同 L0 文件之间 key 范围可能重叠：

```text
L0 file A: apple .. orange
L0 file B: banana .. zebra
L0 file C: cat .. dog
```

查一个 key 时可能需要检查多个 L0 文件。

而 L1 及更深层级通常保持 key 范围不重叠：

```text
L1:
[apple .. cat] [dog .. orange] [pear .. zebra]
```

查某一层时可以根据 key 范围定位到最多一个候选文件。

### 9.3 Bloom Filter 如何减少读放大

没有 Bloom Filter 时：

```text
这个文件可能有 key 吗？
不知道，读一下索引或数据块看看。
```

有 Bloom Filter 后：

```text
这个文件一定没有 key 吗？
如果 Bloom Filter 说一定没有，就跳过。
如果说可能有，再继续查。
```

Bloom Filter 的特点：

- 不会漏掉真实存在的 key。
- 可能误判不存在的 key 为“可能存在”。
- 用更多 bits per key 可以降低误判率，但消耗更多内存或文件空间。

一个常见近似公式是：

```text
false_positive_rate ~= 0.6185 ^ bits_per_key
```

例如：

```text
bits_per_key = 10
false_positive_rate ~= 0.8% 左右
```

这不是所有实现的精确承诺，而是帮助理解 Bloom Filter 代价的近似模型。

### 9.4 Block Cache 如何减少重复读

SSTable 内部按 block 组织。

一次读取通常不是只读一个 key，而是读一个 data block。

如果相邻 key 经常一起访问，或者热点 key 重复访问，Block Cache 可以缓存解压后的 block：

```text
第一次 Get: 磁盘读取 block -> 解压 -> 返回 value -> 放入 cache
第二次 Get: cache 命中 -> 直接返回
```

LevelDB 官方文档也提醒：

- block cache 缓存的是未压缩数据。
- 大范围扫描可以设置不填充 cache，避免扫描把热点数据挤出去。

## 10. LevelDB 的 SSTable 文件格式

SSTable 可以理解成：

```text
[data block 1]
[data block 2]
...
[data block N]
[meta block 1]
...
[meta block K]
[metaindex block]
[index block]
[footer]
```

### 10.1 Data Block

Data Block 保存按 key 排序的 key-value。

为了节省空间，LevelDB 会做前缀压缩。

假设 key 有：

```text
user:10001:name
user:10001:email
user:10001:phone
```

它们有大量公共前缀。

Block 内可以通过 restart point 机制减少重复存储。

### 10.2 Index Block

Index Block 的作用是：

```text
根据 key 找到可能包含它的 data block。
```

它不是对每个 key 都建一个完整索引，而是对 block 建索引。

读取路径大致是：

```text
查 index block
  |
  v
定位 data block 的 offset 和 size
  |
  v
读取 data block
  |
  v
在 block 内查 key
```

### 10.3 Filter Block

如果开启 Bloom Filter，SSTable 会包含 filter block。

Filter block 可以按数据块范围保存过滤信息，让读取在访问 data block 前先判断：

```text
这个 key 是否可能在这个文件或 block 范围内？
```

### 10.4 Footer

Footer 在文件尾部。

它保存：

- metaindex block 的位置。
- index block 的位置。
- magic number。

为什么 Footer 在尾部？

因为写 SSTable 时，前面的数据块数量和位置会逐渐生成，最后才知道索引和元数据在哪里。

打开 SSTable 时，可以先读固定大小 Footer，再找到索引。

## 11. LevelDB 的 Compaction 原理

Compaction 是 LevelDB 的核心。

它做三件事：

```text
1. 把多个有序文件归并成新的有序文件。
2. 丢弃被新版本覆盖的旧版本。
3. 在安全时丢弃删除标记。
```

### 11.1 L0 到 L1

LevelDB 官方实现说明中，日志文件达到阈值后会转换成 sorted table，放入 L0。

当 L0 文件数量超过阈值，会触发 L0 -> L1 Compaction。

因为 L0 文件可能相互重叠，所以 L0 Compaction 可能一次选择多个 L0 文件，再加上 L1 中有重叠范围的文件。

示意：

```text
L0:
F1 [a..m]
F2 [d..z]

L1:
G1 [a..c]
G2 [d..k]
G3 [l..z]

Compaction 输入:
F1, F2, G1, G2, G3
```

### 11.2 L1 到 L2 及更深层

L1 以后通常文件范围不重叠。

当某层总大小超过目标值：

```text
选择该层一个文件
找下一层所有 key 范围重叠文件
归并生成下一层新文件
删除旧输入文件
更新 MANIFEST
```

例如：

```text
L1:
F [h..n]

L2:
G1 [a..g]
G2 [h..k]
G3 [l..p]
G4 [q..z]

Compaction 输入:
F, G2, G3
```

### 11.3 为什么 Compaction 会影响延迟

Compaction 会消耗：

- 读 I/O：读取旧 SSTable。
- 写 I/O：写出新 SSTable。
- CPU：归并、比较 key、压缩、校验。
- 缓存：读取和写入可能扰动 page cache 或 block cache。

如果 Compaction 很大，前台读写可能被影响。

所以 LSM 引擎常见调优目标是：

```text
让 Compaction 足够快，避免 L0 堆积；
又不要太激进，避免后台 I/O 抢占前台请求。
```

### 11.4 删除标记何时能丢

删除不是马上从所有文件里移除。

删除通常写成 tombstone：

```text
Delete user:1
```

它必须在一段时间内保留，因为更深层可能还有旧值：

```text
L0: user:1 -> tombstone
L3: user:1 -> Alice
```

如果过早删除 tombstone，读取可能重新看到旧值。

只有当 Compaction 能确认旧值不会在更深层出现，或者已经归并处理掉，tombstone 才能安全丢弃。

## 12. LevelDB 的一致性、快照与恢复

### 12.1 原子写入

LevelDB 的 WriteBatch 提供批量原子更新。

如果一个批次里有：

```text
Put a 1
Put b 2
Delete c
```

读取不会看到“只写入 a 但 b 没写入”的中间状态。

### 12.2 Snapshot

Snapshot 是某个 sequence number 对应的一致性视图。

假设：

```text
seq=100 Put user:1 Alice
seq=120 Snapshot S
seq=130 Put user:1 Bob
```

用 Snapshot S 读取 `user:1`，应该看到 Alice。

不用 Snapshot 的普通读取，则会看到 Bob。

Snapshot 的代价是：

```text
它会让一些旧版本暂时不能被 Compaction 清理。
```

如果长时间持有 Snapshot，可能导致空间放大升高。

### 12.3 崩溃恢复

恢复路径可以理解为：

```text
1. 读取 LOCK，确保单进程打开。
2. 读取 CURRENT，找到 MANIFEST。
3. 读取 MANIFEST，恢复 VersionSet。
4. 找出需要重放的 WAL。
5. 重放 WAL，恢复最近写入。
6. 清理不再被引用的旧文件。
```

### 12.4 Sync 写与异步写

LevelDB 默认写入不一定每次都同步到稳定存储。

可以通过写选项开启同步写：

```cpp
leveldb::WriteOptions options;
options.sync = true;
db->Put(options, "k", "v");
```

这意味着写入返回前会更努力地把日志刷到持久介质。

代价是：

- 延迟显著增加。
- 吞吐下降。

工程上常见策略：

- 关键元数据用同步写。
- 批量导入可以异步写，失败后重跑。
- 多条更新放入 WriteBatch，用一次同步写摊薄成本。

## 13. RocksDB 的功能定位

RocksDB 是从 LevelDB 演进而来的嵌入式 KV 存储引擎。

它的设计目标更偏向：

```text
服务端生产负载 + 多核 CPU + SSD/Flash + 高可调性 + 可观测性。
```

RocksDB 仍然是嵌入式库，不是独立数据库服务。

它提供：

| 能力 | 说明 |
| --- | --- |
| 有序 key-value | key/value 是字节流，按 Comparator 排序 |
| 点查与范围扫描 | `Get`、`MultiGet`、`Iterator` |
| 原子写批次 | WriteBatch 支持多个 key 原子更新 |
| Column Families | 一个 DB 实例中划分多个逻辑列族 |
| Transactions | 支持乐观和悲观事务模式 |
| Snapshots | 一致性时间点视图 |
| Prefix Seek | 利用 prefix extractor 和 Bloom Filter 优化前缀范围读取 |
| Merge Operator | 把读改写操作延迟到 Compaction 或读取时合并 |
| 多种 Compaction | Leveled、Universal、FIFO，以及自定义/手动 Compaction 能力 |
| 多线程后台任务 | Flush 和 Compaction 可并行 |
| 丰富压缩 | 支持多种压缩算法和按层配置 |
| 缓存体系 | Block Cache、Table Cache、可选压缩块缓存等 |
| 工具与观测 | `ldb`、`sst_dump`、统计、日志、Perf Context、IO Stats |
| 备份与 Checkpoint | 支持本地备份和检查点能力 |

RocksDB 适合：

- 写入量很大。
- 读写混合复杂。
- 需要细粒度调优。
- 需要多列族或事务。
- 需要和上层分布式数据库、流处理系统、消息系统深度集成。

## 14. RocksDB 的架构设计

RocksDB 的高层结构仍然是 LSM：

```text
应用线程
  |
  | Put / Delete / Merge / Get / MultiGet / Iterator / Transaction
  v
+--------------------------------------------------+
| RocksDB                                          |
|                                                  |
|  DBImpl                                          |
|  ColumnFamilyData                                |
|  MemTable / Immutable MemTables                  |
|  WAL                                             |
|  VersionSet / MANIFEST                           |
|  SST Files                                       |
|  Block Cache / Table Cache                       |
|  Flush Scheduler / Compaction Picker             |
|  Background Thread Pools                         |
|  Env / FileSystem / RateLimiter / Statistics     |
+--------------------------------------------------+
  |
  v
本地或自定义文件系统
```

### 14.1 Column Family

Column Family 可以理解为：

```text
同一个 RocksDB 实例里的多个逻辑有序 KV 空间。
```

示例：

```text
default
user_profile
order_index
session_state
```

每个 Column Family 可以有自己的：

- MemTable。
- SSTable 集合。
- Comparator。
- Compaction 配置。
- 压缩配置。
- Bloom Filter 配置。

但它们可以共享：

- WAL。
- Block Cache。
- 后台线程池。
- Env。
- RateLimiter。

Column Family 的价值是：

```text
把不同访问模式的数据拆开调优，而不必开多个 RocksDB 实例。
```

### 14.2 MemTable 与 Immutable MemTable

RocksDB 的 MemTable 作用和 LevelDB 类似：

```text
新写入进入 MemTable；
读取先查 MemTable；
MemTable 满后变成 Immutable MemTable；
后台 Flush 成 L0 SST。
```

不同点是 RocksDB 支持更丰富的 MemTable 策略。

常见类型包括：

| MemTable 类型 | 适用场景 |
| --- | --- |
| SkipList | 通用读写和范围扫描，默认常用 |
| Vector | 批量导入、写多读少，Flush 时排序 |
| Hash-based | 前缀查询明显、按 prefix 访问的数据 |

RocksDB 还支持多个 Immutable MemTable 进入 flush pipeline，以提高写入吞吐。

### 14.3 WAL

RocksDB 的 WAL 也是顺序日志。

写入可以配置是否写 WAL、是否同步刷盘。

WAL 的关键价值：

```text
MemTable 还没 flush 成 SST 前，如果进程崩溃，可以靠 WAL 恢复。
```

RocksDB 还允许把 WAL 放在和 SST 不同的目录。

这对某些架构有意义：

```text
SST 放在快但可能非持久的介质；
WAL 放在较慢但更持久的介质。
```

### 14.4 SST 文件

RocksDB 默认 SST 格式是 BlockBasedTable。

它继承了 LevelDB 的核心思想，同时扩展了更多元数据和格式能力：

- data block。
- index block。
- filter block。
- properties block。
- compression dictionary block。
- range deletion block。
- footer。

BlockBasedTable 的设计目标是：

```text
让点查、范围扫描、过滤、缓存、压缩、统计都能围绕 block 组织。
```

### 14.5 Version 与 MANIFEST

RocksDB 也用 MANIFEST 记录数据库状态变化。

Compaction 生成新文件、删除旧文件后，需要把这些变化持久化到 MANIFEST。

否则崩溃恢复时会不知道：

```text
哪些 SST 文件属于当前数据库版本？
哪些文件已经被废弃？
哪些 Compaction 输出是有效的？
```

## 15. RocksDB 的核心能力扩展

RocksDB 相比 LevelDB 的重要增强，不只是“更快”，而是围绕生产系统增加了很多控制面。

### 15.1 MultiGet

如果应用要一次查多个 key：

```text
Get k1
Get k2
Get k3
...
```

逐个调用可能重复做很多工作。

MultiGet 可以批量处理多个 key，减少调用开销，并有机会合并内部读取。

### 15.2 Transactions

RocksDB 支持多操作事务。

可以粗略分为：

- Optimistic Transaction：提交时检测冲突。
- Pessimistic Transaction：通过锁避免冲突。

这适合上层需要在多个 key 之间维护局部一致性的场景。

但要注意：

```text
RocksDB 的事务是本地存储引擎事务，不等于分布式事务。
```

跨机器、跨分片的一致性仍然需要上层协议。

### 15.3 Merge Operator

很多应用会做读改写：

```text
old = Get(counter)
new = old + 1
Put(counter, new)
```

这会产生一次读和一次写。

Merge Operator 的思想是：

```text
先记录“我要加 1”这个意图；
之后在读取或 Compaction 时再把多个意图合并。
```

例如：

```text
Merge counter +1
Merge counter +1
Merge counter +1
```

最后可以合并为：

```text
counter = old + 3
```

适合：

- 计数器。
- 集合追加。
- 聚合状态。

不适合：

- 合并逻辑复杂且不可结合。
- 每次读取都必须立即看到精确物化值，且合并成本高。

### 15.4 Prefix Seek

很多业务 key 有自然前缀：

```text
user:1001:profile
user:1001:orders
user:1001:settings
user:1002:profile
```

如果查询总是在同一个 prefix 内扫描：

```text
prefix = user:1001:
```

RocksDB 可以通过 prefix extractor 和 prefix Bloom Filter 优化。

这避免扫描时检查大量不相关文件。

### 15.5 DeleteRange 与 SingleDelete

`Delete` 删除单个 key。

`DeleteRange` 用于删除一个 key 范围：

```text
DeleteRange("user:1001:", "user:1002:")
```

这比逐个 key 删除更适合范围清理。

`SingleDelete` 是针对“某个 key 只被 Put 一次后删除”的优化语义。使用它必须满足约束，否则结果可能不符合预期。

### 15.6 Backup 与 Checkpoint

生产系统经常需要：

- 在线备份。
- 快速制作一致性副本。
- 用副本做迁移或恢复。

RocksDB 提供 BackupEngine 和 Checkpoint 能力，帮助上层系统在不停止服务的情况下管理本地数据副本。

## 16. RocksDB 的 Compaction 体系

RocksDB 的 Compaction 比 LevelDB 丰富得多。

### 16.1 Leveled Compaction

Leveled 是默认常见策略。

结构：

```text
L0: 多个可能重叠的新文件
L1: 一个按 key 范围切分的 sorted run
L2: 一个更大的 sorted run
L3: 一个更大的 sorted run
...
```

特点：

- 深层 level 通常不重叠。
- 读放大较低。
- 空间放大相对可控。
- 写放大可能较高，因为数据会逐层重写。

适合：

- 点查多。
- 范围扫描多。
- 希望空间占用稳定。
- 可接受一定写放大。

### 16.2 Universal Compaction

Universal 更接近 tiered compaction。

它倾向于把多个 sorted runs 一次性合并。

特点：

- 写放大较低。
- 临时空间需求可能更高。
- 读放大和空间放大可能高于 Leveled。

适合：

- 写入极重。
- 数据生命周期相对短。
- 可以接受更多空间波动。

### 16.3 FIFO Compaction

FIFO 更适合缓存或时序类过期数据。

它的核心思想是：

```text
当总大小超过阈值，删除最老文件。
```

适合：

- 只关心最近数据。
- 旧数据可以按文件粒度淘汰。
- 类似本地持久化缓存。

不适合：

- 需要永久保留所有 key。
- 需要精细删除语义。

### 16.4 Subcompaction

当一次 Compaction 的 key 范围很大时，可以把范围切开并行执行：

```text
Compaction range: [a..z]

thread 1: [a..f]
thread 2: [g..m]
thread 3: [n..t]
thread 4: [u..z]
```

这能更好利用多核和 SSD 带宽。

但并行越多不一定越好，因为会增加：

- I/O 竞争。
- CPU 压缩竞争。
- 输出文件管理复杂度。
- 对前台请求的干扰。

### 16.5 Compaction Filter

Compaction Filter 允许应用在 Compaction 时处理 key-value。

例如：

```text
如果 value 已过期，丢弃它。
如果 value 需要脱敏，改写它。
如果 key 属于某类旧数据，删除它。
```

它的优势是：

```text
顺着后台归并流程做数据清理，不需要额外全库扫描。
```

但要谨慎：

- 过滤逻辑必须高效。
- 不能引入不稳定外部依赖。
- 要考虑 Snapshot 和一致性语义。

## 17. RocksDB 的内存、缓存与读优化

RocksDB 读性能很大程度上取决于内存分配。

主要内存去向：

```text
MemTable
Block Cache
Index / Filter
Table Cache
Iterator pinned blocks
Compaction buffers
WAL / write buffers
```

### 17.1 Block Cache

Block Cache 缓存从 SST 读取出来的 block。

常见读取路径：

```text
Get key
  |
  v
查 block cache
  |
  |-- 命中 -> 返回
  |
  |-- 未命中 -> 读 SST block -> 解压 -> 放入 cache -> 返回
```

RocksDB 支持 LRU 类缓存，也支持面向并发优化的其他缓存实现。

工程上要注意：

- Block Cache 太小，随机读会频繁打磁盘。
- Block Cache 太大，会挤压 MemTable、系统 page cache 或业务内存。
- 大范围扫描可能污染缓存，需要设置不填充 cache。

### 17.2 Index 和 Filter 的缓存

点查一个 key 时，通常要先判断文件是否可能包含它，再定位数据块。

所以 index/filter 元数据非常重要。

如果 index/filter 不在内存中，点查可能多一次 I/O。

常见优化：

- cache index and filter blocks。
- 为高优先级元数据预留 cache 空间。
- pin L0 filter/index blocks。
- 使用 partitioned index/filter 降低大索引常驻内存压力。

### 17.3 Bloom Filter

RocksDB 的 Bloom Filter 对点查和 prefix seek 非常关键。

适合开启：

- 大量随机 Get。
- 很多 key 不存在。
- LSM 层数较多。
- 数据集大于内存。

收益不明显的场景：

- 几乎都是全量顺序扫描。
- 数据集完全在 cache 中。
- key 查询总能命中很新的 MemTable。

### 17.4 Prefix Bloom

Prefix Bloom 针对这种查询：

```text
查 user:1001:* 下的所有 key
```

它要求 key 设计能稳定提取 prefix。

如果 prefix extractor 配错，可能出现：

- 读不到预期数据。
- 扫描退化。
- Bloom Filter 效果差。

所以 prefix 优化必须和 key 编码一起设计。

## 18. RocksDB 的写入优化与后台任务

RocksDB 面向服务端负载做了大量写入优化。

### 18.1 写入批处理

多个线程同时写入时，RocksDB 可以把多个写请求合并成组，减少 WAL 写入和同步成本。

直观理解：

```text
10 个线程各写 1 条
    |
    v
一次 group commit 写入 WAL
    |
    v
分别插入 MemTable
```

这对高并发小写入很重要。

### 18.2 并发 MemTable 写

RocksDB 的 skiplist memtable 支持并发插入能力。

这降低多线程写入时在 MemTable 上的串行瓶颈。

### 18.3 Flush 线程

MemTable 满后要 flush 到 L0。

如果所有后台线程都在做大 Compaction，Flush 没有线程执行，写入会很快卡住。

所以 RocksDB 支持把 flush 和 compaction 的后台资源做更细致配置。

### 18.4 写入停顿

写入停顿通常来自：

- L0 文件过多。
- Immutable MemTable 过多。
- WAL 过多，需要推动旧 MemTable flush。
- Compaction 速度低于写入速度。
- 磁盘空间不足。
- 后台错误。

表现为：

```text
写入延迟突然升高
吞吐下降
日志出现 stall 信息
L0 file count 上升
pending compaction bytes 上升
```

解决方向不是盲目加线程，而是判断瓶颈：

| 现象 | 可能原因 | 方向 |
| --- | --- | --- |
| CPU 满 | 压缩或比较器成本高 | 调整压缩、优化 key comparator、增加 CPU |
| 磁盘写满 | Compaction 输出太多 | 降低写放大、调整 compaction、增加磁盘 |
| L0 堆积 | Flush/Compaction 跟不上 | 增加后台能力、调大写缓冲、调整 level 参数 |
| 读延迟高 | 读放大或 cache 不足 | Bloom、Block Cache、index/filter 缓存 |
| 空间涨得快 | 旧版本和 tombstone 堆积 | 检查长快照、compaction、删除策略 |

## 19. LevelDB 与 RocksDB 横向对比

| 维度 | LevelDB | RocksDB |
| --- | --- | --- |
| 项目定位 | 轻量级嵌入式 KV | 面向生产服务端的高性能嵌入式 KV |
| 代码复杂度 | 相对简单，适合学习 | 更复杂，功能更完整 |
| 维护状态 | 官方仓库说明维护较有限 | 持续面向生产场景演进 |
| 基础结构 | WAL + MemTable + SSTable + Leveled Compaction | 同样基于 LSM，但扩展大量组件 |
| Column Family | 不支持 | 支持 |
| 事务 | WriteBatch 原子写 | 支持 WriteBatch，也支持事务库 |
| Compaction 策略 | 经典 leveled 思路 | Leveled、Universal、FIFO、自定义/手动能力 |
| 并行能力 | 较有限 | 多线程 Flush/Compaction/Subcompaction |
| 缓存能力 | Block Cache、Table Cache | 更丰富的 Block Cache、Index/Filter 缓存、Table Cache |
| MemTable 类型 | 经典 SkipList | SkipList、Vector、Hash-based 等 |
| 调优参数 | 少，简单 | 多，适合复杂负载 |
| 工具与观测 | 基础属性和日志 | 丰富统计、工具、Perf Context、IO Stats |
| 适合用途 | 学习、轻量嵌入、本地简单 KV | 高吞吐生产系统、复杂状态存储、数据库底层引擎 |

一个简单判断：

```text
想学习 LSM Tree 的基本骨架，优先看 LevelDB。
想构建高负载生产系统，优先评估 RocksDB。
```

## 20. 如何根据业务场景选型

### 20.1 选择 LevelDB 的情况

可以考虑 LevelDB：

- 数据量不大。
- 单机嵌入式场景。
- 功能需求简单。
- 希望依赖少、行为容易理解。
- 主要目的是学习或做轻量本地持久化。

不适合 LevelDB：

- 高并发服务端写入。
- 需要多列族。
- 需要丰富事务。
- 需要大量调优和监控。
- 需要面向 SSD 的复杂后台并行能力。

### 20.2 选择 RocksDB 的情况

可以考虑 RocksDB：

- 数据量大。
- 写入压力高。
- 读写混合复杂。
- 需要精细控制内存、压缩、Compaction。
- 需要 Column Families、Transactions、Merge Operator。
- 上层系统能承受 RocksDB 调优复杂度。

不适合 RocksDB：

- 团队没有能力理解 LSM 调优。
- 只是存少量配置。
- 需要现成 SQL 和网络服务。
- 不希望处理本地文件、备份、恢复、监控等问题。

### 20.3 一个选型表

| 场景 | 推荐倾向 | 原因 |
| --- | --- | --- |
| 写一个教学型 KV 存储研究项目 | LevelDB | 架构小，便于阅读 |
| 桌面应用本地缓存 | LevelDB 或 RocksDB | 看功能复杂度 |
| 高吞吐流处理状态存储 | RocksDB | 后台并行、调优和状态能力更强 |
| 分布式数据库底层存储 | RocksDB | 生产特性更完整 |
| 简单 key-value 配置文件 | 两者都可能太重 | 普通文件或 SQLite 可能更合适 |
| 需要 SQL 查询 | 不直接使用二者 | 需要 SQL 层或关系数据库 |

## 21. 初学者应该掌握的推理方法

学习 LevelDB/RocksDB 不应靠背术语，而要形成可推导的模型。

### 21.1 任何问题先画数据路径

遇到问题先问：

```text
写入路径是什么？
读取路径是什么？
崩溃恢复路径是什么？
后台整理路径是什么？
```

例如写入慢：

```text
Put
 -> WAL append
 -> WAL sync?
 -> MemTable insert
 -> MemTable full?
 -> Flush needed?
 -> L0 too many?
 -> Compaction pending?
 -> disk bandwidth enough?
```

只说“RocksDB 慢”没有意义。

要定位是：

- WAL 慢。
- MemTable 锁竞争。
- Flush 慢。
- Compaction 慢。
- Cache miss 多。
- 磁盘满。
- Comparator 太贵。
- value 太大。

### 21.2 任何优化都问三种放大

调参数时问：

```text
这个参数会降低读放大、写放大还是空间放大？
它会牺牲什么？
```

例如：

```text
增大 write_buffer_size
  好处：减少 flush 次数，提高写入吞吐
  代价：更多内存，更长恢复时间，flush 输出更大
```

```text
开启 Bloom Filter
  好处：减少不存在 key 的磁盘查找
  代价：增加内存/文件空间，构建 filter 有成本
```

```text
使用 Universal Compaction
  好处：降低写放大
  代价：可能增加读放大和空间波动
```

### 21.3 任何 key 设计都问访问模式

KV 引擎不会自动理解业务。

key 的编码直接决定性能。

好的 key 设计要考虑：

- 是否需要按用户扫描。
- 是否需要按时间扫描。
- 是否有热点前缀。
- 是否会导致写入集中到某个范围。
- 是否需要 prefix Bloom。
- 是否需要自定义 Comparator。

例子：

```text
order:{user_id}:{timestamp}
```

适合按用户查订单。

但如果要按全局时间扫描，可能不合适。

另一个设计：

```text
order_by_time:{timestamp}:{order_id}
```

适合按时间扫描。

如果两个访问模式都重要，可能需要维护两个 key 空间：

```text
order:{order_id} -> order_detail
user_order_index:{user_id}:{timestamp}:{order_id} -> empty
time_order_index:{timestamp}:{order_id} -> empty
```

这就是应用自己维护二级索引。

### 21.4 任何长尾延迟都检查后台任务

LSM 引擎里，长尾延迟经常来自后台任务。

要看：

- L0 文件数。
- Pending compaction bytes。
- Flush 是否排队。
- Compaction 是否过大。
- 是否有长 Iterator 或 Snapshot 阻止清理。
- 是否有大范围删除产生大量 tombstone。
- 磁盘 I/O 是否被后台任务打满。

不要只看平均延迟。

## 22. 简单使用示例

### 22.1 LevelDB 基本读写

```cpp
#include <cassert>
#include <iostream>
#include <string>

#include "leveldb/db.h"

int main() {
  leveldb::DB* db = nullptr;
  leveldb::Options options;
  options.create_if_missing = true;

  leveldb::Status status = leveldb::DB::Open(options, "/tmp/example_leveldb", &db);
  assert(status.ok());

  status = db->Put(leveldb::WriteOptions(), "user:1", "Alice");
  assert(status.ok());

  std::string value;
  status = db->Get(leveldb::ReadOptions(), "user:1", &value);
  if (status.ok()) {
    std::cout << value << std::endl;
  }

  status = db->Delete(leveldb::WriteOptions(), "user:1");
  assert(status.ok());

  delete db;
  return 0;
}
```

这段代码背后发生的是：

```text
Put -> 写 WAL -> 写 MemTable -> 未来 Flush/Compaction
Get -> 查 MemTable -> 查 SSTable -> 返回
Delete -> 写 tombstone -> 未来 Compaction 清理
```

### 22.2 LevelDB 原子批量写

```cpp
#include "leveldb/db.h"
#include "leveldb/write_batch.h"

void MoveValue(leveldb::DB* db, const std::string& from, const std::string& to) {
  std::string value;
  leveldb::Status s = db->Get(leveldb::ReadOptions(), from, &value);
  if (!s.ok()) {
    return;
  }

  leveldb::WriteBatch batch;
  batch.Delete(from);
  batch.Put(to, value);

  leveldb::WriteOptions write_options;
  write_options.sync = true;
  s = db->Write(write_options, &batch);
}
```

这里 `Delete` 和 `Put` 在同一个 batch 中。

如果进程崩溃，不应该出现只删除旧 key 但没写新 key 的中间状态。

### 22.3 RocksDB 基本读写

```cpp
#include <cassert>
#include <iostream>
#include <string>

#include "rocksdb/db.h"
#include "rocksdb/options.h"

int main() {
  rocksdb::DB* db = nullptr;
  rocksdb::Options options;
  options.create_if_missing = true;

  rocksdb::Status s = rocksdb::DB::Open(options, "/tmp/example_rocksdb", &db);
  assert(s.ok());

  s = db->Put(rocksdb::WriteOptions(), "user:1", "Alice");
  assert(s.ok());

  std::string value;
  s = db->Get(rocksdb::ReadOptions(), "user:1", &value);
  if (s.ok()) {
    std::cout << value << std::endl;
  }

  delete db;
  return 0;
}
```

### 22.4 RocksDB Column Family 思路示例

伪代码结构：

```cpp
rocksdb::ColumnFamilyHandle* users = nullptr;
rocksdb::ColumnFamilyHandle* orders = nullptr;

db->Put(rocksdb::WriteOptions(), users, "user:1", "...profile...");
db->Put(rocksdb::WriteOptions(), orders, "order:9", "...order...");
```

设计动机：

```text
users 和 orders 可以拥有不同的 compaction、compression、prefix、cache 策略。
```

不要因为 Column Family 存在就滥用。

如果只是 key 前缀不同，但访问模式和调优需求完全一样，用普通 key 前缀可能更简单。

## 23. 调优思路与常见参数

调优不要从参数开始，要从问题开始。

### 23.1 先描述工作负载

先回答：

```text
数据量多大？
key 平均多大？
value 平均多大？
写入 QPS 多少？
随机读 QPS 多少？
范围扫描多不多？
不存在 key 查询多不多？
读写比例是多少？
是否有热点 key？
是否有 TTL 或批量删除？
是否需要崩溃后零丢失？
SSD 还是 HDD？
内存预算多少？
```

没有这些信息，参数讨论很容易变成猜测。

### 23.2 LevelDB 常见调节点

| 参数/机制 | 影响 |
| --- | --- |
| `write_buffer_size` | MemTable 大小；影响 flush 频率、内存、恢复时间 |
| `max_file_size` | SSTable 目标文件大小；影响文件数量和 Compaction 粒度 |
| `block_size` | 数据块大小；影响点查、扫描、压缩效率 |
| `block_cache` | 未压缩 block 缓存；影响随机读 |
| `filter_policy` | Bloom Filter；影响随机点查和不存在 key 查询 |
| `compression` | 压缩算法；影响 CPU、磁盘空间和 I/O |
| `WriteOptions::sync` | 持久性与写入延迟之间的取舍 |
| `ReadOptions::fill_cache` | 大扫描是否污染缓存 |

### 23.3 RocksDB 常见调节点

| 参数/机制 | 影响 |
| --- | --- |
| `write_buffer_size` | 单个 MemTable 大小 |
| `max_write_buffer_number` | 可积累的 MemTable 数量 |
| `min_write_buffer_number_to_merge` | Flush 前合并多个 MemTable，可能降低写放大 |
| `max_background_jobs` | 后台 Flush/Compaction 总并行度 |
| `level0_file_num_compaction_trigger` | L0 文件数达到多少触发 Compaction |
| `level0_slowdown_writes_trigger` | L0 文件过多时开始写入限速 |
| `level0_stop_writes_trigger` | L0 文件过多时停止写入 |
| `target_file_size_base` | SST 文件目标大小 |
| `max_bytes_for_level_base` | L1 目标大小 |
| `max_bytes_for_level_multiplier` | level 大小增长倍数 |
| `compaction_style` | Leveled、Universal、FIFO |
| `compression_per_level` | 不同层使用不同压缩 |
| `block_cache` | 数据块缓存 |
| `cache_index_and_filter_blocks` | 是否把 index/filter 放入 block cache 管理 |
| `prefix_extractor` | 前缀查询优化 |
| `filter_policy` | Bloom Filter |
| `rate_limiter` | 限制后台 I/O，降低对前台干扰 |
| `stats` / `perf_context` | 观测内部耗时和事件 |

### 23.4 三类典型负载

#### 写多读少

目标：

```text
降低写放大，让后台跟上。
```

可能方向：

- 增大 MemTable。
- 增加后台任务能力。
- 评估 Universal Compaction。
- 使用较快压缩或降低压缩。
- 批量写入。
- 避免每条写都 sync。

#### 随机读多

目标：

```text
降低读放大，提高 cache 命中。
```

可能方向：

- 增大 Block Cache。
- 开启 Bloom Filter。
- 缓存 index/filter。
- 避免 L0 文件堆积。
- 选择 Leveled Compaction。
- 优化 key 大小和 Comparator。

#### 范围扫描多

目标：

```text
让相邻访问的数据在 key 空间相邻，并减少扫描过程中的无效读取。
```

可能方向：

- 重新设计 key 顺序。
- 使用合适 block size。
- 大扫描设置不填充 cache。
- 避免过多 tombstone。
- 控制 Compaction，减少文件碎片。

## 24. 常见误区

### 24.1 误区：LSM Tree 写入快，所以一定比 B+Tree 好

不一定。

LSM Tree 把随机写变成顺序写，但代价是：

- 读取可能查多层。
- Compaction 会写放大。
- 空间暂时放大。
- 调优更复杂。

B+Tree 对某些读多、更新局部性好、事务复杂的场景仍然很强。

### 24.2 误区：RocksDB 是数据库

RocksDB 是存储引擎库。

它不直接提供：

- SQL。
- 网络协议。
- 用户权限。
- 分布式复制。
- 自动分片。
- 查询优化器。

很多系统把 RocksDB 作为底层引擎，但“上层数据库能力”需要另外实现。

### 24.3 误区：Compaction 只是清理垃圾

Compaction 不只是清理。

它同时决定：

- 读放大。
- 写放大。
- 空间放大。
- 删除何时真正生效。
- 数据在 level 中的形态。
- 前台写入是否会 stall。

Compaction 是 LSM 的核心成本中心。

### 24.4 误区：Bloom Filter 能让所有读取变快

Bloom Filter 主要优化：

- 点查。
- 不存在 key 查询。
- prefix seek。

它不擅长优化：

- 全量扫描。
- 大范围顺序读取。
- 已经完全命中缓存的数据。

### 24.5 误区：参数越大越好

例如 MemTable 越大：

- Flush 次数可能越少。
- 写吞吐可能更好。
- 但内存更多。
- 崩溃恢复可能更久。
- 单次 Flush/Compaction 可能更重。

任何参数都要回到工作负载。

### 24.6 误区：长 Snapshot 没有副作用

长时间持有 Snapshot 或 Iterator 可能阻止旧版本清理。

结果：

- 磁盘空间上涨。
- Compaction 无法丢弃旧数据。
- 读写性能下降。

### 24.7 误区：key 只是字符串

在有序 KV 引擎中，key 是数据布局。

key 决定：

- 排序。
- 扫描范围。
- 热点分布。
- Bloom Filter 效果。
- Compaction 局部性。
- 二级索引维护成本。

设计 key 就是在设计存储模型。

## 25. 学习路线

### 25.1 第一阶段：使用者视角

掌握：

- `Put`、`Get`、`Delete`。
- `WriteBatch`。
- `Iterator`。
- `Snapshot`。
- 同步写和异步写。
- 基本错误处理。

能回答：

```text
一次写入什么时候算成功？
崩溃后哪些数据能恢复？
Iterator 看到的是不是一致视图？
```

### 25.2 第二阶段：LSM 结构视角

掌握：

- WAL。
- MemTable。
- Immutable MemTable。
- SSTable。
- Level。
- Bloom Filter。
- Block Cache。
- Compaction。

能画出：

```text
写入路径
读取路径
Flush 路径
Compaction 路径
恢复路径
```

### 25.3 第三阶段：源码视角

读 LevelDB：

- `db/db_impl.*`
- `db/memtable.*`
- `db/version_set.*`
- `db/log_writer.*`
- `table/table_builder.*`
- `table/block_builder.*`
- `util/cache.*`

目标不是背代码，而是把模块和路径对应起来。

建议先跟踪：

```text
DB::Open
DBImpl::Write
DBImpl::Get
DBImpl::BackgroundCompaction
VersionSet::LogAndApply
```

### 25.4 第四阶段：RocksDB 工业能力视角

再读 RocksDB：

- Column Family。
- Flush Scheduler。
- Compaction Picker。
- BlockBasedTable。
- WriteThread。
- TransactionDB。
- Cache。
- Statistics。
- RateLimiter。

重点不是一次读完，而是带着问题读：

```text
RocksDB 如何让多线程写入扩展？
如何控制 L0 堆积？
如何把 index/filter 放入 cache？
如何让不同 Column Family 共享资源？
如何观测一次读慢在哪里？
```

### 25.5 第五阶段：实践视角

做几个实验：

1. 写 1000 万个随机 key，观察 SST 文件和 level 变化。
2. 开关 Bloom Filter，比较不存在 key 查询延迟。
3. 改变 block cache 大小，观察随机读命中率。
4. 制造长 Snapshot，观察磁盘空间变化。
5. 调整后台任务数，观察 L0 文件堆积和写入 stall。
6. 对比 Leveled 与 Universal Compaction 的写放大和空间波动。

## 26. 官方资料

LevelDB：

- [LevelDB GitHub README](https://github.com/google/leveldb)
- [LevelDB library documentation](https://github.com/google/leveldb/blob/main/doc/index.md)
- [LevelDB implementation notes](https://github.com/google/leveldb/blob/main/doc/impl.md)
- [LevelDB table format](https://github.com/google/leveldb/blob/main/doc/table_format.md)
- [LevelDB DB API header](https://github.com/google/leveldb/blob/main/include/leveldb/db.h)
- [LevelDB options header](https://github.com/google/leveldb/blob/main/include/leveldb/options.h)

RocksDB：

- [RocksDB Overview](https://github.com/facebook/rocksdb/wiki/RocksDB-Overview)
- [RocksDB Getting Started](https://rocksdb.org/docs/getting-started.html)
- [RocksDB Leveled Compaction](https://github.com/facebook/rocksdb/wiki/Leveled-Compaction)
- [RocksDB Universal Compaction](https://github.com/facebook/rocksdb/wiki/Universal-Compaction)
- [RocksDB Write Ahead Log](https://github.com/facebook/rocksdb/wiki/Write-Ahead-Log-%28WAL%29)
- [RocksDB MemTable](https://github.com/facebook/rocksdb/wiki/MemTable)
- [RocksDB Block Cache](https://github.com/facebook/rocksdb/wiki/Block-Cache)
- [RocksDB BlockBasedTable Format](https://github.com/facebook/rocksdb/wiki/Rocksdb-BlockBasedTable-Format)

延伸阅读：

- [LSM-based Storage Techniques: A Survey](https://arxiv.org/abs/1812.07527)
