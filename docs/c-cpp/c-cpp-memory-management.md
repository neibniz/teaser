# C/C++ 内存管理：内存池、对象池、多线程分配与 CSAPP malloc

> 目标读者：已经会 C/C++ 基础语法，希望系统理解堆内存、内存池、对象池、多线程内存分配，以及《深入理解计算机系统》中动态内存分配器思想的学习者。  
> 学习目标：不只会调用 `malloc/free`、`new/delete`，而是理解分配器如何组织堆、如何减少碎片、如何提高多线程性能、如何设计对象生命周期。  
> 说明：本文对《深入理解计算机系统》中的动态内存分配器思想做原理化整理，不是原书摘录。

---

## 目录

1. [C/C++ 内存管理总览](#1-cc-内存管理总览)
2. [堆分配器要解决的问题](#2-堆分配器要解决的问题)
3. [C 与 C++ 内存管理接口](#3-c-与-c-内存管理接口)
4. [内存池的核心思想](#4-内存池的核心思想)
5. [固定大小块内存池](#5-固定大小块内存池)
6. [可变大小内存池](#6-可变大小内存池)
7. [对象池的经典方案](#7-对象池的经典方案)
8. [C++ allocator 与 PMR](#8-c-allocator-与-pmr)
9. [多线程内存分配的挑战](#9-多线程内存分配的挑战)
10. [多线程分配器经典架构](#10-多线程分配器经典架构)
11. [多线程对象管理方案](#11-多线程对象管理方案)
12. [CSAPP malloc 实现方案](#12-csapp-malloc-实现方案)
13. [碎片、对齐与安全问题](#13-碎片对齐与安全问题)
14. [工程实践选择指南](#14-工程实践选择指南)
15. [学习路线与实现练习](#15-学习路线与实现练习)

---

## 1. C/C++ 内存管理总览

C/C++ 程序运行时常见内存区域：

| 区域 | 存放内容 | 生命周期 |
| --- | --- | --- |
| 栈 | 局部变量、函数调用帧 | 进入/退出作用域自动创建销毁 |
| 堆 | `malloc/new` 动态分配对象 | 手动释放或由封装对象释放 |
| 全局/静态区 | 全局变量、静态变量 | 程序整个生命周期 |
| 代码区 | 程序指令 | 程序整个生命周期 |
| 常量区 | 字符串字面量、只读常量 | 程序整个生命周期 |

内存管理主要讨论堆。

因为堆内存有几个特点：

- 大小运行时才知道。
- 生命周期不一定跟作用域一致。
- 分配和释放顺序不可预测。
- 多线程可能同时分配释放。
- 错误释放会导致严重 bug。

### 1.1 一句话理解动态内存

动态内存管理是在做：

> 从一大块可用内存中切出合适大小的块给用户，用完后收回来，并尽量减少浪费和竞争。

这句话包含三个关键目标：

- 找到合适大小。
- 正确回收复用。
- 性能和碎片之间做权衡。

### 1.2 初学者常见误解

误解一：`malloc` 直接向操作系统要内存。  
更准确地说，多数分配器会先向操作系统申请较大内存区域，再在用户态内部切分管理。

误解二：`free` 立刻把内存还给操作系统。  
很多时候 `free` 只是把块放回分配器的空闲结构，等待复用。

误解三：`new` 和 `malloc` 只是名字不同。  
`new` 会分配内存并调用构造函数；`malloc` 只分配原始字节。

误解四：内存池一定比系统分配器快。  
如果场景不匹配，内存池可能造成更多浪费、复杂度和 bug。

---

## 2. 堆分配器要解决的问题

一个通用分配器至少要处理：

1. 对齐。
2. 查找空闲块。
3. 切分大块。
4. 合并相邻空闲块。
5. 管理元数据。
6. 减少内部碎片和外部碎片。
7. 多线程同步。

### 2.1 对齐

CPU 访问某些类型时要求地址按特定边界对齐。

例如：

```text
int      常见需要 4 字节对齐
double   常见需要 8 字节对齐
指针      64 位系统常见需要 8 字节对齐
```

分配器通常把用户请求大小向上取整。

例如用户请求 13 字节，按 8 字节对齐：

```text
实际分配 16 字节
```

### 2.2 内部碎片

内部碎片指：

> 已经分配给用户，但用户没有真正使用的空间。

例如用户请求 13 字节，实际分配 16 字节，多出的 3 字节就是内部碎片。

### 2.3 外部碎片

外部碎片指：

> 总空闲内存足够，但被分散成很多小块，无法满足一个大请求。

例如空闲块：

```text
8 字节 + 8 字节 + 8 字节 + 8 字节
```

总共 32 字节，但如果用户请求连续 24 字节，可能无法满足。

### 2.4 元数据

分配器需要知道每个块的大小、是否空闲、前后关系等信息。

常见做法是在用户内存前面放一个 header：

```text
[ block header ][ user payload ]
```

用户拿到的是 `payload` 指针，不应该碰 header。

---

## 3. C 与 C++ 内存管理接口

### 3.1 C 接口

常见 C 接口：

```c
void* malloc(size_t size);
void free(void* ptr);
void* calloc(size_t count, size_t size);
void* realloc(void* ptr, size_t new_size);
```

含义：

- `malloc`：分配未初始化内存。
- `calloc`：分配并清零。
- `realloc`：调整已分配块大小。
- `free`：释放。

### 3.2 C++ 接口

常见 C++ 接口：

```cpp
T* p = new T(args);
delete p;

T* arr = new T[n];
delete[] arr;
```

`new T(args)` 做两件事：

1. 调用 `operator new` 分配原始内存。
2. 在这块内存上调用 `T` 的构造函数。

`delete p` 也做两件事：

1. 调用析构函数。
2. 调用 `operator delete` 释放原始内存。

### 3.3 placement new

placement new 在已有内存上构造对象：

```cpp
void* mem = std::malloc(sizeof(T));
T* obj = new (mem) T(args);

obj->~T();
std::free(mem);
```

这在对象池里非常重要。

对象池常常先准备原始内存，真正使用时再构造对象，用完时显式析构但不一定马上释放内存。

### 3.4 RAII

C++ 推荐用 RAII 管理资源：

```cpp
std::unique_ptr<T> p = std::make_unique<T>();
```

RAII 的核心：

> 资源生命周期绑定对象生命周期。

这能减少忘记释放、异常路径泄漏等问题。

---

## 4. 内存池的核心思想

内存池的基本思想：

> 一次向系统申请一大块内存，之后在这块内存里快速分配小块。

适用场景：

- 大量小对象。
- 分配释放非常频繁。
- 对象大小相同或大小类别有限。
- 生命周期有规律。
- 希望减少系统分配器开销。
- 希望改善缓存局部性。

不适用场景：

- 对象大小差异极大。
- 生命周期完全随机。
- 总量很小，系统分配器已经足够。
- 需要极强通用性。
- 没有清晰所有权边界。

### 4.1 内存池的典型收益

1. 减少 `malloc/free` 调用次数。
2. 分配速度接近指针弹出。
3. 释放速度接近指针压入。
4. 减少元数据开销。
5. 改善局部性。
6. 可按业务生命周期整体释放。

### 4.2 内存池的典型风险

1. 释放错误更隐蔽。
2. 内存峰值可能更高。
3. 池大小估计不准会浪费或扩容频繁。
4. 多线程同步设计复杂。
5. 对象析构容易遗漏。
6. Debug 难度高于直接使用标准库。

---

## 5. 固定大小块内存池

固定大小块内存池是最经典、最容易实现的内存池。

它适用于：

```text
所有分配请求大小相同
或可以按固定大小类别管理
```

### 5.1 基本结构

假设每个块大小为 64 字节，池中有 1024 个块。

内存布局：

```text
[block0][block1][block2]...[block1023]
```

空闲块通过单链表串起来：

```text
free_list -> block17 -> block4 -> block99 -> ...
```

分配：

```text
从 free_list 弹出一个块
```

释放：

```text
把块压回 free_list
```

### 5.2 为什么空闲块可以存 next 指针

一个空闲块没有用户数据。  
所以可以把空闲块开头几个字节当作 `next` 指针。

```cpp
struct FreeNode {
    FreeNode* next;
};
```

当块被分配出去时，这块内存是用户对象。  
当块释放回来时，这块内存重新被解释为 `FreeNode`。

### 5.3 简化实现骨架

```cpp
#include <cstddef>
#include <cstdlib>
#include <new>

class FixedBlockPool {
public:
    FixedBlockPool(std::size_t block_size, std::size_t block_count)
        : block_size_(round_up(block_size, alignof(std::max_align_t))),
          block_count_(block_count) {
        buffer_ = static_cast<unsigned char*>(std::malloc(block_size_ * block_count_));
        if (!buffer_) {
            throw std::bad_alloc();
        }

        free_list_ = nullptr;
        for (std::size_t i = 0; i < block_count_; ++i) {
            auto* node = reinterpret_cast<FreeNode*>(buffer_ + i * block_size_);
            node->next = free_list_;
            free_list_ = node;
        }
    }

    ~FixedBlockPool() {
        std::free(buffer_);
    }

    void* allocate() {
        if (!free_list_) {
            throw std::bad_alloc();
        }
        FreeNode* node = free_list_;
        free_list_ = free_list_->next;
        return node;
    }

    void deallocate(void* p) {
        if (!p) {
            return;
        }
        auto* node = static_cast<FreeNode*>(p);
        node->next = free_list_;
        free_list_ = node;
    }

private:
    struct FreeNode {
        FreeNode* next;
    };

    static std::size_t round_up(std::size_t n, std::size_t align) {
        return (n + align - 1) / align * align;
    }

    std::size_t block_size_;
    std::size_t block_count_;
    unsigned char* buffer_;
    FreeNode* free_list_;
};
```

### 5.4 这个实现缺什么

上面只是教学骨架，真实工程还要补：

- 禁止拷贝。
- 检查释放指针是否属于池。
- 检查重复释放。
- 线程安全。
- 扩容策略。
- 统计信息。
- Debug 填充值。
- 对齐更严格的类型支持。
- 异常安全。

### 5.5 固定块内存池的复杂度

| 操作 | 复杂度 |
| --- | --- |
| 分配 | `O(1)` |
| 释放 | `O(1)` |
| 初始化 | `O(n)` |
| 内部碎片 | 取决于块大小和请求大小 |

固定块内存池的核心优势就是简单和稳定。

---

## 6. 可变大小内存池

可变大小内存池要处理不同大小请求。

经典方案：

1. 空闲链表。
2. 分离空闲链表。
3. Buddy 分配。
4. Slab/size class。
5. 区域分配器。

### 6.1 单一空闲链表

把所有空闲块放到一个链表里。

分配时搜索合适块：

- first-fit：找到第一个足够大的块。
- next-fit：从上次位置继续找。
- best-fit：找最接近请求大小的块。

优点：

- 实现简单。

缺点：

- 搜索可能慢。
- 容易产生碎片。

### 6.2 分离空闲链表

按大小类别维护多个空闲链表。

例如：

```text
8 字节链表
16 字节链表
32 字节链表
64 字节链表
128 字节链表
...
```

请求 40 字节时，可以去 64 字节类别找。

优点：

- 查找快。
- 小对象分配效率高。

缺点：

- 类别设计影响碎片。
- 大对象需要单独策略。

### 6.3 Buddy 分配

Buddy 系统把内存按 2 的幂大小管理。

例如一块 1024 字节内存：

```text
1024
-> 512 + 512
-> 256 + 256 + 512
```

分配时找到足够大的 2 的幂块，必要时不断二分。  
释放时，如果相邻 buddy 也空闲，就合并回更大块。

优点：

- 合并高效。
- 管理规则清晰。
- 适合页级或大块内存管理。

缺点：

- 内部碎片可能明显。
- 请求 513 字节可能要分配 1024 字节。

### 6.4 Slab / size class 思路

Slab 思路常用于内核和高性能分配器：

> 每种对象或大小类别有自己的缓存，从页面中切小块。

例如：

```text
32B class
64B class
128B class
...
```

每个 size class 管理多个 span/page。  
小对象从对应 class 的空闲列表分配。

现代通用分配器，如 jemalloc、tcmalloc、mimalloc，都大量使用 size class、线程缓存、中心缓存等思想。

### 6.5 区域分配器 Arena

Arena 分配器又叫区域分配器。

特点：

- 分配很快，通常只是指针递增。
- 单个对象不单独释放。
- 整个 arena 一次性释放。

适用：

- 编译器 AST。
- 请求级临时对象。
- 游戏一帧内临时内存。
- 批处理任务。

基本模型：

```text
[ used .... ][ free .... ]
            ^
          bump pointer
```

分配：

```text
ptr = current
current += aligned_size
```

释放：

```text
reset entire arena
```

优点：

- 极快。
- 无单对象释放碎片。

缺点：

- 不适合生命周期交错的对象。

---

## 7. 对象池的经典方案

对象池和内存池相似，但关注点不同。

内存池管理原始字节。  
对象池管理对象生命周期。

对象池要考虑：

- 构造。
- 析构。
- 对象重置。
- 复用。
- 所有权。
- 线程安全。

### 7.1 预构造对象池

初始化时构造所有对象。

分配时取一个空闲对象。  
释放时重置状态并放回池。

适用：

- 构造成本高。
- 对象数量上限明确。
- 对象可安全重置。

缺点：

- 启动成本高。
- 未使用对象也占资源。
- 对象状态重置必须非常可靠。

### 7.2 延迟构造对象池

池只预留原始内存，不提前构造对象。

获取对象时：

```cpp
T* obj = new (memory) T(args...);
```

归还对象时：

```cpp
obj->~T();
```

然后把内存块放回空闲链表。

优点：

- 不浪费构造成本。
- 对象生命周期清晰。

缺点：

- 必须正确调用析构。
- 异常安全要处理好。

### 7.3 简化对象池骨架

```cpp
#include <cstddef>
#include <memory>
#include <new>
#include <utility>

template <class T>
class ObjectPool {
public:
    explicit ObjectPool(std::size_t capacity)
        : storage_(static_cast<unsigned char*>(::operator new(sizeof(T) * capacity))),
          capacity_(capacity) {
        free_list_ = nullptr;
        for (std::size_t i = 0; i < capacity_; ++i) {
            auto* node = reinterpret_cast<Node*>(storage_ + i * sizeof(T));
            node->next = free_list_;
            free_list_ = node;
        }
    }

    ~ObjectPool() {
        ::operator delete(storage_);
    }

    template <class... Args>
    T* create(Args&&... args) {
        if (!free_list_) {
            throw std::bad_alloc();
        }
        Node* node = free_list_;
        free_list_ = free_list_->next;
        return new (node) T(std::forward<Args>(args)...);
    }

    void destroy(T* obj) {
        if (!obj) {
            return;
        }
        obj->~T();
        auto* node = reinterpret_cast<Node*>(obj);
        node->next = free_list_;
        free_list_ = node;
    }

private:
    struct Node {
        Node* next;
    };

    unsigned char* storage_;
    std::size_t capacity_;
    Node* free_list_;
};
```

### 7.4 这个对象池的关键问题

上面代码适合教学，不适合直接生产使用。

需要注意：

- `sizeof(T)` 可能小于指针大小，空闲节点放不下 `Node*`。
- `T` 的对齐可能大于默认对齐。
- 析构时无法知道哪些对象仍然存活。
- 重复释放会破坏 free list。
- 非线程安全。
- 不支持扩容。

更稳妥的块大小应是：

```text
max(sizeof(T), sizeof(Node))
```

并按：

```text
alignof(T)
```

或更严格对齐。

### 7.5 RAII 归还对象

对象池最好不要让用户手动调用 `destroy`。

可以使用自定义 deleter：

```cpp
template <class T>
struct PoolDeleter {
    ObjectPool<T>* pool;

    void operator()(T* p) const {
        pool->destroy(p);
    }
};
```

返回：

```cpp
std::unique_ptr<T, PoolDeleter<T>>
```

这样对象离开作用域时自动归还池。

### 7.6 对象重置的危险

预构造对象池常用 `reset()` 清理状态。

风险是：

- 某个字段忘记重置。
- 对象持有外部资源。
- 状态被上一次使用污染。
- 异常路径没有归还。

因此工程上常倾向：

> 获取时构造，归还时析构，只复用原始内存，不复用对象状态。

---

## 8. C++ allocator 与 PMR

C++ 标准库容器可以通过 allocator 定制内存分配。

### 8.1 allocator 的意义

例如：

```cpp
std::vector<T, MyAllocator<T>> v;
```

容器内部元素存储可以由 `MyAllocator` 提供。

适用：

- 容器大量小对象。
- 需要 arena 分配。
- 需要统计容器内存。
- 需要把容器内存放到指定区域。

### 8.2 std::pmr

C++17 引入 polymorphic memory resource。

核心类型：

```cpp
std::pmr::memory_resource
std::pmr::polymorphic_allocator
std::pmr::vector
std::pmr::string
```

常用资源：

- `std::pmr::monotonic_buffer_resource`
- `std::pmr::unsynchronized_pool_resource`
- `std::pmr::synchronized_pool_resource`

### 8.3 monotonic_buffer_resource

类似 arena。

特点：

- 分配快。
- 单个 deallocate 通常不回收。
- 整个 resource 释放时统一回收。

适合请求级临时对象。

### 8.4 pool_resource

`unsynchronized_pool_resource` 适合单线程。  
`synchronized_pool_resource` 提供线程同步。

它们按大小类别池化小对象分配。

工程建议：

> 如果标准 PMR 能满足需求，优先用 PMR，而不是手写复杂 allocator。

---

## 9. 多线程内存分配的挑战

多线程下，内存分配器会遇到：

1. 锁竞争。
2. 伪共享。
3. 跨线程释放。
4. NUMA 影响。
5. 缓存局部性下降。
6. 对象所有权不清。

### 9.1 全局锁的问题

最简单方案：

```text
所有 malloc/free 都加一把全局锁
```

优点：

- 实现简单。
- 正确性容易保证。

缺点：

- 多线程频繁分配时竞争严重。
- 核数越多越容易成为瓶颈。

### 9.2 伪共享

伪共享指：

> 多个线程修改不同变量，但这些变量位于同一个 cache line，导致缓存行来回失效。

例如两个线程分别频繁修改相邻对象，如果它们落在同一个 64 字节 cache line，性能可能很差。

对象池设计中要注意：

- 热字段隔离。
- 对齐到 cache line。
- 避免不同线程频繁写同一缓存行。

### 9.3 跨线程释放

线程 A 分配对象，线程 B 释放对象。

如果每个线程有自己的本地缓存，就会出现问题：

```text
对象应该回到 A 的缓存，还是 B 的缓存？
```

常见处理：

- 释放到当前线程缓存。
- 释放到对象所属 span 的中心结构。
- 放入远程释放队列，之后由 owner 线程回收。

---

## 10. 多线程分配器经典架构

现代高性能分配器常采用分层结构。

典型架构：

```text
Thread Cache -> Central Cache -> Page Heap / OS
```

### 10.1 Thread Cache

每个线程有自己的小对象空闲链表。

分配小对象：

```text
先从当前线程本地 free list 取
```

释放小对象：

```text
优先放回当前线程本地 free list
```

优点：

- 大多数分配释放无需锁。
- 缓存局部性好。

缺点：

- 每个线程都缓存内存，可能增加内存占用。
- 跨线程释放需要额外处理。

### 10.2 Central Cache

Thread Cache 不够用时，从 Central Cache 批量取一组对象。

Thread Cache 太多时，也可以批量还给 Central Cache。

好处：

> 避免每次小对象分配都访问全局结构，而是批量交互。

### 10.3 Page Heap

Central Cache 没有可用内存时，向更底层的 Page Heap 申请页级大块。

Page Heap 再向操作系统申请或归还内存。

常见系统接口包括：

- `mmap`
- `munmap`
- `brk/sbrk`，现代分配器较少直接依赖
- Windows 上的 `VirtualAlloc`

### 10.4 Size Class

小对象按大小分类。

例如：

```text
8, 16, 24, 32, 48, 64, 80, 96, 128, ...
```

用户请求 37 字节，归入 48 字节 class。

权衡：

- size class 越密，内部碎片越少，但管理复杂。
- size class 越稀，管理简单，但浪费更多。

### 10.5 Span / Run

分配器通常把若干连续页作为一个 span。

一个 span 可能服务某个 size class。

例如：

```text
一个 8KB span 被切成多个 64B 对象
```

span 需要记录：

- size class。
- 空闲对象列表。
- 已分配数量。
- 所属 central cache。

### 10.6 常见分配路径

小对象分配：

```text
Thread Cache 有空闲块 -> 直接返回
Thread Cache 没有 -> 从 Central Cache 批量取
Central Cache 没有 -> 从 Page Heap 申请 span
Page Heap 没有 -> 向 OS 申请内存
```

小对象释放：

```text
放回 Thread Cache
Thread Cache 超过阈值 -> 批量还给 Central Cache
```

大对象分配：

```text
绕过 Thread Cache，直接走 Page Heap / OS
```

### 10.7 tcmalloc / jemalloc / mimalloc 的共同思想

不同实现细节很多，但共同思想包括：

- 小对象 size class。
- 线程本地缓存。
- 批量迁移减少锁竞争。
- 页级 span/run 管理。
- 大对象单独路径。
- 尽量减少碎片和元数据开销。

工程上：

> 除非有非常明确的业务需求，否则优先使用成熟分配器，而不是自己实现通用多线程 malloc。

---

## 11. 多线程对象管理方案

内存分配只是对象管理的一部分。  
多线程对象还要处理所有权和生命周期。

### 11.1 每线程对象池

每个线程维护自己的对象池。

优点：

- 无锁或少锁。
- 局部性好。
- 简单高效。

缺点：

- 对象跨线程归还复杂。
- 线程多时内存占用增加。
- 线程退出时要回收本地池。

适用：

- 对象只在线程内使用。
- 工作线程固定。
- 生命周期短且局部。

### 11.2 全局对象池加锁

所有线程共享一个对象池，用互斥锁保护。

优点：

- 简单。
- 对象可跨线程归还。

缺点：

- 高并发下锁竞争明显。

适用：

- 分配频率不高。
- 对象较重，锁开销不是瓶颈。
- 正确性优先。

### 11.3 分片对象池

把对象池分成多个 shard。

线程根据线程 ID、CPU ID 或哈希选择 shard。

```text
pool[hash(thread_id) % shard_count]
```

优点：

- 降低锁竞争。
- 比每线程池更容易控制内存总量。

缺点：

- 实现复杂度中等。
- shard 不均衡时仍可能竞争。

### 11.4 无锁 free list

用原子操作维护空闲链表。

基本思想：

```text
CAS 更新 head 指针
```

简化模型：

```cpp
struct Node {
    Node* next;
};

std::atomic<Node*> head;
```

分配时：

1. 读取 `head`。
2. 读取 `head->next`。
3. CAS 把 `head` 改成 `next`。
4. 成功则返回旧 head，失败则重试。

释放时：

1. 读取 `head`。
2. `node->next = head`。
3. CAS 把 `head` 改成 `node`。
4. 失败则重试。

### 11.5 ABA 问题

无锁栈会遇到 ABA 问题。

线程看到 head 是 A。  
期间其他线程把 A 弹出，又把 A 放回。  
当前线程再看 head 仍然是 A，以为没有变化，但链表结构可能已经变了。

解决思路：

- 带版本号的指针。
- hazard pointer。
- epoch based reclamation。
- 避免立即释放节点内存。
- 使用成熟无锁库。

工程建议：

> 无锁对象池不是初学项目，除非非常清楚内存回收模型，否则优先使用锁、分片锁或成熟库。

### 11.6 引用计数

C++ 中常见：

```cpp
std::shared_ptr<T>
```

它使用引用计数管理共享对象生命周期。

优点：

- 跨线程共享方便。
- 最后一个引用释放时自动析构。

缺点：

- 引用计数原子操作有开销。
- 循环引用需要 `weak_ptr`。
- 所有权可能变得不清晰。

### 11.7 生产者消费者对象流转

常见模式：

```text
生产者创建对象 -> 队列 -> 消费者处理 -> 归还对象池
```

需要明确：

- 对象归谁释放。
- 处理失败时是否归还。
- 队列中对象是否仍然有效。
- 对象是否可能被重复归还。

推荐做法：

> 用 RAII handle 表达所有权，让归还动作自动发生。

---

## 12. CSAPP malloc 实现方案

《深入理解计算机系统》讲动态内存分配器时，重点不是做一个工业级 malloc，而是让读者理解：

> 堆分配器如何用块、头部、边界标记、空闲链表、分割和合并管理动态内存。

经典教学实现通常包含：

- 堆初始化。
- 块格式。
- 对齐。
- 隐式空闲链表。
- 查找合适空闲块。
- 放置和分割。
- 释放。
- 合并相邻空闲块。
- 扩展堆。

### 12.1 块结构

一个堆块通常包含：

```text
[ header ][ payload ][ optional padding ][ footer ]
```

header 记录：

- 块大小。
- 是否已分配。

footer 常用于空闲块合并，也记录块大小和分配位。

因为块大小按对齐要求通常是 8 或 16 的倍数，低几位可以用来存状态位。

例如：

```text
size | allocated_bit
```

### 12.2 为什么需要 header

用户调用：

```c
free(p);
```

只传入 payload 指针。  
分配器必须知道这个块有多大。

所以可以通过：

```text
payload 指针向前偏移
```

找到 header，再读出块大小。

### 12.3 为什么需要 footer

释放一个块后，分配器希望和相邻空闲块合并。

要判断前一个物理相邻块是否空闲，需要知道前一个块的信息。

footer 可以让分配器从当前块向前找到前一块大小。

这叫边界标记。

### 12.4 序言块和结尾块

教学实现常在堆开头放一个已分配的序言块，在堆结尾放一个已分配的结尾 header。

目的：

- 简化边界情况。
- 合并时不需要频繁判断是否越过堆边界。

堆大致像：

```text
[padding][prologue header][prologue footer][normal blocks...][epilogue header]
```

### 12.5 隐式空闲链表

隐式空闲链表不是显式保存 `next` 指针。

它通过块大小在堆上顺序跳转：

```text
current block -> current + block_size -> next block
```

每个块都在这个线性序列里，包括已分配块和空闲块。

分配时从头扫描，找空闲且足够大的块。

优点：

- 实现简单。
- 适合教学。

缺点：

- 分配时可能扫描很多已分配块。
- 吞吐量不高。

### 12.6 查找策略

常见策略：

#### first-fit

从头开始找第一个足够大的空闲块。

优点：

- 快。

缺点：

- 堆前部容易产生小碎片。

#### next-fit

从上次查找结束位置继续找。

优点：

- 避免每次从头扫描。

缺点：

- 碎片表现不一定稳定。

#### best-fit

找最接近请求大小的块。

优点：

- 可能减少剩余碎片。

缺点：

- 需要扫描更多块。
- 小碎片仍可能增加。

### 12.7 放置与分割

找到一个空闲块后，如果它比请求大很多，可以分割。

例如：

```text
空闲块 128B，请求 40B
```

可以变成：

```text
[已分配 48B][空闲 80B]
```

为什么是 48B？  
因为要包含 header/footer 和对齐后的 payload。

分割条件：

> 剩余部分必须足够形成一个合法空闲块。

否则不分割，避免产生无法使用的小碎片。

### 12.8 释放与合并

释放时：

1. 把当前块标记为空闲。
2. 查看前后相邻物理块。
3. 如果相邻块空闲，就合并。

四种情况：

```text
前已分配，后已分配 -> 不合并
前已分配，后空闲   -> 和后合并
前空闲，后已分配   -> 和前合并
前空闲，后空闲     -> 三块合并
```

合并可以减少外部碎片。

### 12.9 立即合并与延迟合并

立即合并：

```text
free 时立刻合并相邻空闲块
```

优点：

- 外部碎片较少。
- 后续大块请求更容易满足。

缺点：

- 如果很快又分配类似大小，可能刚合并又切开。

延迟合并：

```text
先不合并，在需要时或扫描时合并
```

优点：

- 可能减少不必要操作。

缺点：

- 实现更复杂。
- 碎片可能暂时更多。

### 12.10 扩展堆

如果找不到合适空闲块，分配器需要向系统申请更多堆空间。

教学实现通常抽象为：

```text
extend_heap(size)
```

扩展后得到一个新的空闲块，再尝试和原堆最后一个空闲块合并。

### 12.11 隐式链表 allocator 的核心流程

分配流程：

```text
malloc(size):
    调整 size，满足对齐并包含元数据
    在隐式链表中查找合适空闲块
    如果找到，放置并必要时分割
    如果找不到，扩展堆
    返回 payload 指针
```

释放流程：

```text
free(ptr):
    找到块 header
    标记为空闲
    与相邻空闲块合并
```

### 12.12 显式空闲链表改进

隐式链表扫描所有块，包括已分配块。

显式空闲链表只把空闲块串起来：

```text
free block <-> free block <-> free block
```

空闲块 payload 里存：

```text
prev pointer
next pointer
```

优点：

- 查找只遍历空闲块。
- 分配性能通常更好。

缺点：

- 实现更复杂。
- 插入删除链表要小心。
- 最小块大小变大。

### 12.13 分离空闲链表进一步改进

把空闲块按大小分类：

```text
small list
medium list
large list
...
```

请求某个大小时，从对应或更大类别查找。

优点：

- 查找更快。
- 更接近工业分配器思想。

这是从 CSAPP 教学 allocator 走向高性能 allocator 的自然路径。

### 12.14 CSAPP allocator 的学习价值

它让你掌握几个核心概念：

- 堆不是黑盒。
- 每个块都需要元数据。
- 对齐会造成内部碎片。
- 空闲块组织方式决定查找效率。
- 分割和合并决定碎片表现。
- 简单实现和高性能实现之间有明确演进路径。

---

## 13. 碎片、对齐与安全问题

### 13.1 常见内存错误

| 错误 | 含义 |
| --- | --- |
| 内存泄漏 | 分配后没有释放 |
| 重复释放 | 同一指针释放两次 |
| use-after-free | 释放后继续使用 |
| 越界写 | 写出了分配块范围 |
| mismatched free | `new[]` 配 `delete`，或 `malloc` 配 `delete` |
| 未初始化读取 | 使用未初始化内存 |

### 13.2 Debug 技术

常见手段：

- AddressSanitizer。
- Valgrind。
- guard page。
- canary。
- 释放后填充特殊字节。
- 分配统计。
- 泄漏检测。

### 13.3 Debug 填充值

分配器 debug 模式可能：

- 新分配内存填 `0xCD`。
- 释放内存填 `0xDD`。
- 边界放 canary。

这样更容易发现未初始化使用、释放后使用和越界写。

### 13.4 对齐 API

C/C++ 中可能遇到：

```c
aligned_alloc
posix_memalign
```

C++ 中：

```cpp
::operator new(size, std::align_val_t{alignment})
```

自定义内存池如果要支持 SIMD、cache line 或特殊硬件，需要明确处理对齐。

---

## 14. 工程实践选择指南

### 14.1 什么时候用默认分配器

优先用默认分配器，如果：

- 分配不频繁。
- 性能瓶颈不在内存分配。
- 对象大小复杂。
- 工程稳定性优先。

默认策略：

```text
先写正确，再用 profiling 证明分配是瓶颈。
```

### 14.2 什么时候用对象池

适合：

- 大量同类型对象。
- 高频创建销毁。
- 对象生命周期短。
- 构造或分配成本明显。
- 能清楚定义归还时机。

例如：

- 网络连接对象。
- 消息对象。
- 游戏实体组件。
- AST 节点。

### 14.3 什么时候用 arena

适合：

- 一批对象生命周期相同。
- 不需要单独释放。
- 结束时整体清理。

例如：

- 一次请求内的临时数据。
- 编译一个文件产生的节点。
- 一帧游戏逻辑临时内存。

### 14.4 什么时候换通用分配器

如果全局 `malloc/free` 是瓶颈，可以考虑：

- jemalloc。
- tcmalloc。
- mimalloc。

但要基于测试：

- 吞吐量。
- P99 延迟。
- 内存峰值。
- 碎片率。
- 多线程扩展性。

### 14.5 不建议手写通用 malloc 的场景

不建议自己写通用 malloc，如果：

- 只是为了“可能更快”。
- 没有 profiling 数据。
- 没有多线程和碎片测试。
- 没有 debug 工具。
- 没有长期维护成本预算。

可以手写：

- 固定块池。
- 特定对象池。
- request arena。
- 测试/教学 allocator。

---

## 15. 学习路线与实现练习

### 第一阶段：理解接口和生命周期

掌握：

- `malloc/free`
- `new/delete`
- placement new
- 构造/析构
- RAII
- 智能指针

练习：

```text
用 placement new 在一块 malloc 内存上构造并析构对象。
```

### 第二阶段：实现固定块内存池

掌握：

- free list。
- 对齐。
- O(1) 分配释放。
- 指针范围检查。

练习：

```text
实现一个固定大小块池，并加入重复释放检测。
```

### 第三阶段：实现对象池

掌握：

- 原始内存和对象生命周期分离。
- 构造失败处理。
- RAII 自动归还。

练习：

```text
实现 ObjectPool<T>，返回 unique_ptr<T, Deleter>。
```

### 第四阶段：实现 CSAPP 风格 malloc

掌握：

- header/footer。
- 隐式空闲链表。
- first-fit。
- splitting。
- coalescing。
- extend heap。

练习：

```text
写一个教学 allocator，支持 mm_malloc/mm_free/mm_realloc。
```

### 第五阶段：多线程优化

掌握：

- mutex 版本。
- 每线程缓存。
- 分片池。
- 跨线程释放。
- ABA 问题。

练习：

```text
把固定块池改成分片对象池，并用 benchmark 对比全局锁版本。
```

---

## 一句话总结

C/C++ 内存管理的核心不是“申请和释放”这么简单，而是：

```text
如何在正确生命周期、低碎片、低竞争、高局部性之间做权衡。
```

固定块池追求简单快速。  
对象池追求生命周期和复用。  
多线程分配器追求减少锁竞争和提升局部性。  
CSAPP 风格 allocator 帮你理解堆分配器的基本骨架：块、元数据、空闲链表、分割、合并和扩展堆。
