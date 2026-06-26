# Rust 编程语言教程：核心原理与 C/C++ 对比

> 目标读者：学过 C 或 C++，希望系统学习 Rust 的初学者；也适合已经会写一点 Rust，但想理解所有权、借用、生命周期、trait、并发和 unsafe 背后原则的开发者。
> 学习目标：不靠背语法，而是掌握 Rust 的设计模型，能解释“为什么编译器这样限制我”，并能把 C/C++ 的经验迁移成 Rust 的安全系统编程能力。
> 说明：本文依据 The Rust Programming Language、Rust Reference、Rustonomicon、Cargo Book、Rust API 文档和 C++ Core Guidelines 做原理化整理。语言和标准库会演进，具体 API 和 edition 细节以官方当前文档为准。

---

## 目录

1. [Rust 到底解决什么问题](#1-rust-到底解决什么问题)
2. [一句话理解 Rust](#2-一句话理解-rust)
3. [从 C/C++ 迁移到 Rust 的思维变化](#3-从-cc-迁移到-rust-的思维变化)
4. [环境、工具链与第一个项目](#4-环境工具链与第一个项目)
5. [变量、可变性与作用域](#5-变量可变性与作用域)
6. [类型系统：让错误尽量在编译期暴露](#6-类型系统让错误尽量在编译期暴露)
7. [所有权：Rust 最核心的资源管理模型](#7-所有权rust-最核心的资源管理模型)
8. [Move、Copy、Clone 与 Drop](#8-movecopyclone-与-drop)
9. [借用：用引用访问，但不拥有](#9-借用用引用访问但不拥有)
10. [生命周期：引用有效性的静态证明](#10-生命周期引用有效性的静态证明)
11. [切片、字符串与集合](#11-切片字符串与集合)
12. [结构体、枚举与模式匹配](#12-结构体枚举与模式匹配)
13. [Option 和 Result：把空值与错误变成类型](#13-option-和-result把空值与错误变成类型)
14. [泛型、trait 与零成本抽象](#14-泛型trait-与零成本抽象)
15. [trait object 与动态分发](#15-trait-object-与动态分发)
16. [模块、crate 与可见性](#16-模块crate-与可见性)
17. [迭代器、闭包与函数式风格](#17-迭代器闭包与函数式风格)
18. [智能指针与内部可变性](#18-智能指针与内部可变性)
19. [并发：把数据竞争变成编译错误](#19-并发把数据竞争变成编译错误)
20. [异步编程：Future、async 与 await](#20-异步编程futureasync-与-await)
21. [unsafe Rust：把不变量封装在安全接口背后](#21-unsafe-rust把不变量封装在安全接口背后)
22. [FFI：与 C/C++ 互操作](#22-ffi与-cc-互操作)
23. [宏、过程宏与元编程](#23-宏过程宏与元编程)
24. [测试、文档与工程实践](#24-测试文档与工程实践)
25. [性能模型与调优方法](#25-性能模型与调优方法)
26. [从 C/C++ 到 Rust 的常见误区](#26-从-cc-到-rust-的常见误区)
27. [学习路线与练习](#27-学习路线与练习)
28. [官方资料](#28-官方资料)

---

## 1. Rust 到底解决什么问题

Rust 是一门系统编程语言。

它希望在接近 C/C++ 性能和控制力的同时，尽量避免系统编程中最常见、最昂贵的一类错误：

```text
悬垂指针
重复释放
释放后使用
空指针解引用
数据竞争
迭代器失效
错误的资源生命周期
异常路径下资源泄漏
```

C/C++ 给程序员很大自由：

```text
你可以拿到地址。
你可以手动释放。
你可以用指针别名修改同一块内存。
你可以跨线程共享对象。
你可以把任意字节解释成某种类型。
```

这种自由带来高性能和底层控制力，也带来风险。

Rust 的设计不是“不要底层控制”，而是：

```text
把绝大多数资源生命周期、别名修改和并发共享规则放进类型系统；
让编译器在程序运行前证明这些规则没有被破坏；
实在无法静态证明的底层操作，放进 unsafe 边界中显式承担责任。
```

所以 Rust 的核心价值可以概括为：

| 问题 | C/C++ 常见方式 | Rust 的方式 |
| --- | --- | --- |
| 谁负责释放内存 | C 手动 `free`，C++ RAII | 所有权 + 自动 Drop |
| 谁能修改数据 | 指针、引用、`const` 约定 | `&T` 共享借用，`&mut T` 独占借用 |
| 引用是否悬垂 | 靠程序员和工具发现 | 生命周期检查 |
| 空值如何表达 | `NULL`、`nullptr`、约定 | `Option<T>` |
| 错误如何表达 | 返回码、`errno`、异常 | `Result<T, E>` |
| 并发共享是否安全 | 锁、约定、静态分析工具 | `Send`、`Sync`、所有权和锁类型 |
| 泛型抽象是否有运行成本 | C 宏、C++ 模板 | 泛型单态化，trait 约束 |
| 不安全底层操作 | 默认就能做 | 必须写在 `unsafe` 中 |

Rust 不承诺“不会有 bug”。

它主要把一大类内存安全和数据竞争问题从运行期提前到编译期。

## 2. 一句话理解 Rust

Rust 可以理解为：

```text
Rust = C/C++ 级别的控制力 + 编译期资源安全证明 + 明确的 unsafe 边界
```

再具体一点：

```text
Rust 通过所有权、借用、生命周期、trait 和类型系统，把许多 C/C++ 中靠经验和代码审查维护的规则，变成编译器可以检查的规则。
```

这也是 Rust 初学者最容易感到“不适应”的地方。

在 C/C++ 中，你经常先写：

```text
我知道这样没问题。
```

在 Rust 中，编译器会问：

```text
你能用类型、作用域和生命周期证明它没问题吗？
```

Rust 编译器不是在阻止你写程序，而是在要求你把资源关系说清楚。

## 3. 从 C/C++ 迁移到 Rust 的思维变化

### 3.1 从“我记得释放”到“谁拥有资源”

C 中常见问题：

```c
char* p = malloc(100);
if (!p) return -1;

if (some_error()) {
    return -2;  // 忘记 free(p)
}

free(p);
```

C++ 用 RAII 改善：

```cpp
std::vector<int> v;
std::unique_ptr<Foo> p = std::make_unique<Foo>();
```

Rust 更进一步，把 RAII 和所有权规则放进语言核心：

```rust
fn example() {
    let v = vec![1, 2, 3];
    // 离开作用域时 v 自动 drop
}
```

Rust 思考问题的第一问是：

```text
这个值的 owner 是谁？
它什么时候被 move？
谁只是临时借用？
借用持续到哪里结束？
```

### 3.2 从“指针能不能用”到“引用能否被证明有效”

C/C++ 指针可以指向：

- 栈对象。
- 堆对象。
- 全局对象。
- 数组中间。
- 已经释放的内存。
- 未初始化内存。
- 不满足对齐要求的地址。

语言本身很少能帮你完整证明这些指针仍然有效。

Rust 的普通引用 `&T` 和 `&mut T` 必须满足：

```text
引用指向的值仍然活着。
共享引用期间不能有可变引用修改同一值。
可变引用期间不能有其他活跃引用访问同一值。
```

如果无法证明，就不能写成普通安全 Rust。

### 3.3 从“注释约定”到“类型表达不变量”

C/C++ 常见写法：

```cpp
// 如果返回 nullptr 表示没找到
User* find_user(int id);
```

Rust 倾向写成：

```rust
fn find_user(id: u64) -> Option<User> {
    todo!()
}
```

这样调用方必须处理：

```rust
match find_user(1) {
    Some(user) => println!("{}", user.name),
    None => println!("not found"),
}
```

原则是：

```text
能放进类型系统的业务规则，不要只放进注释。
```

### 3.4 从“默认可变”到“默认不可变”

C/C++ 中变量默认可变。

Rust 中变量绑定默认不可变：

```rust
let x = 10;
// x = 20; // 编译错误

let mut y = 10;
y = 20;
```

这不是语法洁癖，而是帮助读代码的人快速判断：

```text
这个名字后面会不会被重新赋值？
这块数据是否需要可变访问？
```

默认不可变能降低推理成本。

## 4. 环境、工具链与第一个项目

Rust 常用工具链包括：

| 工具 | 作用 |
| --- | --- |
| `rustup` | 安装和管理 Rust 工具链 |
| `rustc` | Rust 编译器 |
| `cargo` | 包管理、构建、测试、运行、发布工具 |
| `rustfmt` | 代码格式化 |
| `clippy` | 静态检查和风格建议 |
| `rustdoc` | 文档生成 |

创建项目：

```bash
cargo new hello_rust
cd hello_rust
cargo run
```

生成结构：

```text
hello_rust/
  Cargo.toml
  src/
    main.rs
```

`Cargo.toml` 类似 C/C++ 项目中的构建配置，但它同时承担包元数据和依赖声明：

```toml
[package]
name = "hello_rust"
version = "0.1.0"
edition = "2024"

[dependencies]
```

`src/main.rs`：

```rust
fn main() {
    println!("hello, rust");
}
```

### 4.1 与 C/C++ 构建体系对比

| 维度 | C/C++ | Rust |
| --- | --- | --- |
| 编译器 | gcc、clang、MSVC 等 | `rustc` |
| 构建系统 | Make、CMake、Bazel、Meson 等 | Cargo 是默认统一入口 |
| 包管理 | vcpkg、Conan、系统包、源码集成 | Cargo + crates.io |
| 格式化 | clang-format 等 | rustfmt 是官方常用工具 |
| 静态检查 | clang-tidy、cppcheck 等 | clippy |
| 单元测试 | gtest、Catch2、doctest 等 | 语言和 Cargo 内建支持 |

Rust 初学者要尽早接受：

```text
Cargo 不是附属工具，而是 Rust 工程模型的一部分。
```

## 5. 变量、可变性与作用域

### 5.1 `let` 默认不可变

```rust
let name = String::from("Alice");
// name.push_str(" Smith"); // 编译错误
```

如果需要修改：

```rust
let mut name = String::from("Alice");
name.push_str(" Smith");
```

这里 `mut` 修饰的是变量绑定允许可变访问。

### 5.2 shadowing 不是普通赋值

Rust 允许 shadowing：

```rust
let x = "42";
let x: i32 = x.parse().unwrap();
let x = x + 1;
```

这不是修改同一个变量，而是创建新的绑定遮蔽旧绑定。

与 C/C++ 对比：

```cpp
int x = 1;
{
    double x = 2.0; // 内层作用域遮蔽
}
```

Rust 的 shadowing 更常用于类型转换流水线：

```rust
let input = "100";
let input: u32 = input.parse().unwrap();
```

### 5.3 作用域决定 Drop 时机

```rust
fn main() {
    {
        let s = String::from("hello");
        println!("{s}");
    } // s 在这里 drop
}
```

这与 C++ RAII 很接近：

```cpp
{
    std::string s = "hello";
} // s 析构
```

与 C 的区别是，C 没有统一的析构机制：

```c
{
    char* s = malloc(100);
} // 不会自动 free
```

### 5.4 原则对比

| 原则 | Rust | C | C++ |
| --- | --- | --- | --- |
| 默认可变性 | 默认不可变，显式 `mut` | 默认可变 | 默认可变，可用 `const` |
| 资源释放 | 所有权离开作用域自动 Drop | 手动释放 | RAII 析构 |
| 变量遮蔽 | 常用，支持类型变化 | 支持但较少作为转换风格 | 支持但通常谨慎使用 |
| 编译期推理 | 可变性参与借用检查 | 主要靠约定 | `const`、RAII、类型系统，但不限制别名可变 |

## 6. 类型系统：让错误尽量在编译期暴露

Rust 是静态强类型语言。

但它大量使用类型推断，所以代码不会到处写类型：

```rust
let x = 42;       // 推断为 i32
let y = 3.14;     // 推断为 f64
let v = vec![1];  // 推断为 Vec<i32>
```

### 6.1 标量类型

常见标量类型：

| 类型 | 示例 | 说明 |
| --- | --- | --- |
| 整数 | `i32`、`u64`、`usize` | 有符号、无符号、平台大小 |
| 浮点 | `f32`、`f64` | IEEE 浮点 |
| 布尔 | `bool` | `true` / `false` |
| 字符 | `char` | Unicode scalar value，不是单字节 char |

与 C/C++ 的重要差异：

```text
Rust 的整数大小更显式，i32、u64 这类名字是常规写法。
C/C++ 中 int、long 的大小与平台和 ABI 相关。
```

### 6.2 整数溢出

Rust 不鼓励依赖普通整数运算的溢出行为。

需要明确语义时使用：

```rust
let x = u8::MAX;

let a = x.wrapping_add(1);    // 回绕
let b = x.checked_add(1);     // Option<u8>
let c = x.saturating_add(1);  // 饱和到最大值
let d = x.overflowing_add(1); // (结果, 是否溢出)
```

与 C/C++ 对比：

| 场景 | Rust | C/C++ |
| --- | --- | --- |
| 无符号溢出 | 可用 wrapping 方法明确表达 | 按模回绕 |
| 有符号溢出 | 不应依赖普通运算溢出 | C/C++ 中有符号溢出通常是未定义行为 |
| 想检查溢出 | `checked_*` 返回 `Option` | 需要手动检查或编译器内建函数 |
| 想饱和 | `saturating_*` | 标准整数无统一饱和运算 |

原则：

```text
Rust 倾向把“你想要哪种溢出语义”写在函数名里。
```

### 6.3 元组、数组和切片

```rust
let pair: (i32, &str) = (1, "one");
let arr: [i32; 3] = [1, 2, 3];
let slice: &[i32] = &arr[0..2];
```

C 数组容易退化成指针：

```c
void f(int* p);
```

调用者和被调用者必须额外传长度：

```c
void f(int* p, size_t len);
```

Rust 切片把指针和长度绑定在一起：

```rust
fn f(values: &[i32]) {
    println!("{}", values.len());
}
```

切片可以理解为：

```text
data pointer + length
```

这比裸指针更容易检查边界。

## 7. 所有权：Rust 最核心的资源管理模型

所有权是 Rust 的核心。

三条基本规则：

```text
1. Rust 中每个值都有一个 owner。
2. 同一时间只能有一个 owner。
3. owner 离开作用域时，值被 drop。
```

### 7.1 一个 String 的生命周期

```rust
fn main() {
    let s = String::from("hello");
    println!("{s}");
} // s 离开作用域，堆内存被释放
```

内存模型：

```text
栈上 s:
  ptr  -----> 堆上 "hello"
  len
  capacity
```

`String` 自身的三个字段在栈上，实际字符数据在堆上。

离开作用域时：

```text
Drop for String -> 释放堆内存
```

### 7.2 Move：所有权转移

```rust
let s1 = String::from("hello");
let s2 = s1;

// println!("{s1}"); // 编译错误
println!("{s2}");
```

这不是深拷贝。

更像：

```text
s1 的所有权转给 s2。
s1 从此不可再使用。
```

为什么？

如果允许 `s1` 和 `s2` 都继续有效，就会出现两个 owner 指向同一块堆内存：

```text
s1.ptr -> heap
s2.ptr -> heap
```

离开作用域时可能 double free。

C++ 也有 move：

```cpp
std::string s1 = "hello";
std::string s2 = std::move(s1);
```

但 C++ 中 moved-from 对象仍然存在，只是处于有效但未指定的状态。

Rust 中被 move 的变量通常直接不能再用。

### 7.3 函数参数也会 move

```rust
fn consume(s: String) {
    println!("{s}");
}

fn main() {
    let name = String::from("Alice");
    consume(name);
    // println!("{name}"); // 编译错误
}
```

传参就是一次所有权转移。

如果函数不需要拥有，只需要看一下，应使用借用：

```rust
fn print_name(s: &String) {
    println!("{s}");
}

fn main() {
    let name = String::from("Alice");
    print_name(&name);
    println!("{name}");
}
```

更 idiomatic 的签名是：

```rust
fn print_name(s: &str) {
    println!("{s}");
}
```

因为 `&str` 可以接收 `String` 的切片，也能接收字符串字面量。

### 7.4 与 C/C++ 对比

| 问题 | C | C++ | Rust |
| --- | --- | --- | --- |
| 谁释放堆内存 | 约定谁 `free` | RAII 对象析构 | owner 离开作用域 Drop |
| 复制对象 | 常见浅拷贝风险 | 拷贝构造、移动构造 | 默认 move，显式 `clone` |
| 释放后使用 | 运行期错误 | 仍可能发生 | 安全引用通常编译期阻止 |
| 重复释放 | 容易发生 | 智能指针可避免 | 所有权规则避免 |
| 函数是否接管资源 | 靠命名、注释、智能指针类型 | 靠类型和约定 | 参数类型直接表达 move 或 borrow |

## 8. Move、Copy、Clone 与 Drop

### 8.1 Copy：可以按位复制的轻量类型

某些类型实现 `Copy`：

```rust
let a = 10;
let b = a;
println!("{a}, {b}");
```

`i32` 是 `Copy`，赋值后 `a` 仍然能用。

常见 `Copy` 类型：

- 整数。
- 浮点数。
- `bool`。
- `char`。
- 只包含 `Copy` 字段的元组和结构体。
- 共享引用 `&T`。

不能 `Copy` 的典型类型：

- `String`。
- `Vec<T>`。
- `Box<T>`。
- 实现了 `Drop` 的类型。

原则：

```text
如果按位复制后两个值都能独立安全使用，它才适合 Copy。
```

### 8.2 Clone：显式深拷贝或逻辑复制

```rust
let s1 = String::from("hello");
let s2 = s1.clone();

println!("{s1}, {s2}");
```

`clone` 明确告诉读者：

```text
这里可能发生分配和数据复制。
```

C++ 中拷贝可能隐藏在赋值或传参里：

```cpp
std::string a = "hello";
std::string b = a; // 拷贝
```

Rust 倾向让昂贵复制更显式。

### 8.3 Drop：确定性析构

```rust
struct FileGuard {
    name: String,
}

impl Drop for FileGuard {
    fn drop(&mut self) {
        println!("close {}", self.name);
    }
}

fn main() {
    let guard = FileGuard {
        name: String::from("data.txt"),
    };
} // 自动调用 drop
```

这和 C++ 析构函数非常接近。

差异在于：

- Rust 没有异常展开中析构函数抛异常的问题。
- Rust 的 move 后变量不可用，减少析构重复所有权的风险。
- Rust 不允许手动直接调用 `drop` 方法；要提前释放可以调用标准库的 `std::mem::drop(value)`，本质是移动并让它离开作用域。

### 8.4 原则对比

| 概念 | Rust | C++ |
| --- | --- | --- |
| 默认赋值 | 对非 Copy 类型是 move | 通常是 copy，除非显式 move 或优化 |
| 深拷贝 | 显式 `clone` | 拷贝构造可能隐式发生 |
| 析构 | `Drop` trait | 析构函数 |
| move 后对象 | 原绑定通常不可用 | moved-from 对象仍有效但状态受类型约定 |
| 资源唯一所有权 | 语言默认规则支持 | `std::unique_ptr` 等类型表达 |

## 9. 借用：用引用访问，但不拥有

所有权解决“谁释放”。

借用解决“谁可以临时访问”。

Rust 有两类安全引用：

```text
&T      共享引用，只读借用
&mut T  可变引用，独占借用
```

### 9.1 共享借用

```rust
fn len(s: &String) -> usize {
    s.len()
}

fn main() {
    let s = String::from("hello");
    let n = len(&s);
    println!("{s}: {n}");
}
```

函数 `len` 不拥有 `String`。

它只是借用。

### 9.2 可变借用

```rust
fn append_world(s: &mut String) {
    s.push_str(" world");
}

fn main() {
    let mut s = String::from("hello");
    append_world(&mut s);
    println!("{s}");
}
```

### 9.3 借用规则

核心规则：

```text
同一时间，可以有多个 &T；
或者一个 &mut T；
但不能同时有 &T 和 &mut T 指向同一数据。
```

错误示例：

```rust
let mut s = String::from("hello");
let r1 = &s;
let r2 = &mut s; // 编译错误
println!("{r1}");
```

为什么？

如果 `r2` 修改甚至重新分配 `s` 的堆内存，`r1` 看到的内容可能失效。

Rust 用规则防止这种别名可变问题。

### 9.4 与 C/C++ 指针和引用对比

C++ 引用：

```cpp
void f(std::string& s);
void g(const std::string& s);
```

C 指针：

```c
void f(char* p);
void g(const char* p);
```

它们能表达“是否打算修改”，但通常不能完整表达：

```text
这个引用期间有没有其他别名也能修改同一对象？
这个指针是否悬垂？
这个指针是否跨线程安全？
```

Rust 的 `&mut T` 不是简单的“可修改引用”。

它更强：

```text
&mut T = 可修改 + 独占访问权
```

这就是 Rust 能在安全代码中阻止数据竞争的重要原因。

### 9.5 借用不是锁

借用规则是编译期检查。

它不会在运行期加锁，也不会产生隐藏引用计数。

```rust
let mut x = 1;
let r = &mut x;
*r += 1;
```

这通常就是普通指针级别的机器代码。

原则：

```text
Rust 用编译期规则换取运行期零额外成本。
```

## 10. 生命周期：引用有效性的静态证明

生命周期是 Rust 初学者最容易误解的概念。

它不是：

```text
对象在运行期活了多久的计时器。
```

它是：

```text
编译器用来证明引用不会比被引用对象活得更久的静态关系。
```

### 10.1 悬垂引用问题

C/C++ 中错误示例：

```cpp
const std::string& bad() {
    std::string s = "hello";
    return s; // 悬垂引用
}
```

Rust 会拒绝类似代码：

```rust
fn bad() -> &String {
    let s = String::from("hello");
    &s
}
```

原因：

```text
s 在函数结束时 drop；
返回的引用会指向已释放的值。
```

### 10.2 生命周期标注表达关系

```rust
fn longest<'a>(x: &'a str, y: &'a str) -> &'a str {
    if x.len() > y.len() {
        x
    } else {
        y
    }
}
```

`'a` 的意思不是“x 和 y 必须活得一样久”。

更准确地说：

```text
返回引用的有效范围不能超过 x 和 y 中较短的那个有效范围。
```

生命周期标注不是改变生命周期，而是描述生命周期关系。

### 10.3 生命周期省略

很多函数不需要显式写生命周期：

```rust
fn first_word(s: &str) -> &str {
    s.split_whitespace().next().unwrap_or("")
}
```

编译器能根据规则推断：

```text
输出引用来自输入引用 s。
```

如果多个输入引用可能成为输出，编译器可能要求你显式说明。

### 10.4 结构体中的引用

```rust
struct UserView<'a> {
    name: &'a str,
}
```

这表示：

```text
UserView 不能比它引用的 name 活得更久。
```

与 C++ 对比：

```cpp
struct UserView {
    std::string_view name;
};
```

`std::string_view` 很轻量，但可能悬垂。

Rust 的 `&str` 会携带生命周期关系，编译器会尽力阻止常见悬垂场景。

### 10.5 `'static`

`'static` 表示引用可以在整个程序生命周期内有效，常见于字符串字面量：

```rust
let s: &'static str = "hello";
```

不要把 `'static` 理解为“这个值一定在全局区”。

更实用的理解是：

```text
引用本身可以安全地活到程序结束。
```

### 10.6 原则对比

| 问题 | C/C++ | Rust |
| --- | --- | --- |
| 返回局部变量引用 | 可能编译警告，但语言层面容易写出 | 安全 Rust 编译拒绝 |
| 结构体保存外部引用 | 靠程序员保证外部对象活得更久 | 生命周期参数表达关系 |
| 字符串视图 | `char*`、`std::string_view` 可能悬垂 | `&str` 带生命周期检查 |
| 生命周期成本 | 运行期无检查 | 编译期检查，运行期无额外成本 |

## 11. 切片、字符串与集合

### 11.1 `String` 与 `&str`

Rust 中字符串常见两种形态：

| 类型 | 含义 |
| --- | --- |
| `String` | 拥有堆上 UTF-8 字符串 |
| `&str` | 借用一段 UTF-8 字符串切片 |

```rust
let owned: String = String::from("hello");
let borrowed: &str = &owned;
let literal: &str = "world";
```

函数参数尽量使用 `&str`：

```rust
fn greet(name: &str) {
    println!("hello, {name}");
}
```

这样既能接收：

```rust
greet("Alice");

let name = String::from("Bob");
greet(&name);
```

与 C/C++ 对比：

| 需求 | C | C++ | Rust |
| --- | --- | --- | --- |
| 拥有字符串 | `char*` + `malloc/free` | `std::string` | `String` |
| 借用字符串 | `const char*` + 约定长度或 NUL | `std::string_view` | `&str` |
| 编码约束 | 通常约定 | 取决于应用 | `String`/`&str` 保证 UTF-8 |
| 长度获取 | `strlen` O(n) for NUL string | `size()` | `len()` 字节长度 |

注意：

```rust
let s = "你好";
println!("{}", s.len()); // 字节数，不是字符数
```

Rust 的 `str` 按 UTF-8 字节存储。

### 11.2 `Vec<T>`

`Vec<T>` 类似 C++ `std::vector<T>`：

```rust
let mut v = Vec::new();
v.push(1);
v.push(2);
v.push(3);
```

内存模型：

```text
Vec<T>:
  ptr -> heap buffer
  len
  capacity
```

与 C 数组相比：

- 自动管理容量。
- 越界访问通过 `get` 返回 `Option`，或索引 panic。
- move 后旧变量不可用，避免双重释放。

### 11.3 `HashMap<K, V>`

```rust
use std::collections::HashMap;

let mut scores = HashMap::new();
scores.insert(String::from("Alice"), 10);
scores.insert(String::from("Bob"), 20);

if let Some(score) = scores.get("Alice") {
    println!("{score}");
}
```

`get` 返回 `Option<&V>`。

这迫使你处理 key 不存在的情况。

### 11.4 集合中的所有权

```rust
let name = String::from("Alice");
let mut names = Vec::new();
names.push(name);

// println!("{name}"); // name 已经 move 到 Vec 中
```

如果需要保留原值：

```rust
names.push(name.clone());
println!("{name}");
```

这与 C++ `std::vector<std::string>` 拷贝或移动很像，但 Rust 把 move 后不可用变成编译期规则。

## 12. 结构体、枚举与模式匹配

### 12.1 结构体

```rust
struct User {
    id: u64,
    name: String,
    active: bool,
}

let user = User {
    id: 1,
    name: String::from("Alice"),
    active: true,
};
```

实现方法：

```rust
impl User {
    fn new(id: u64, name: String) -> Self {
        Self {
            id,
            name,
            active: true,
        }
    }

    fn deactivate(&mut self) {
        self.active = false;
    }
}
```

与 C++ 对比：

```cpp
struct User {
    uint64_t id;
    std::string name;
    bool active;

    void deactivate() { active = false; }
};
```

Rust 把数据定义和方法实现分开放在 `struct` 与 `impl` 中。

### 12.2 枚举可以携带数据

C 枚举：

```c
enum Status {
    OK,
    NOT_FOUND,
    ERROR
};
```

Rust 枚举更像代数数据类型：

```rust
enum Message {
    Quit,
    Move { x: i32, y: i32 },
    Write(String),
    ChangeColor(u8, u8, u8),
}
```

每个 variant 可以携带不同数据。

这能表达很多 C/C++ 中需要手写 tagged union 的结构。

### 12.3 match 必须覆盖所有情况

```rust
fn handle(msg: Message) {
    match msg {
        Message::Quit => println!("quit"),
        Message::Move { x, y } => println!("move to {x}, {y}"),
        Message::Write(text) => println!("{text}"),
        Message::ChangeColor(r, g, b) => println!("{r},{g},{b}"),
    }
}
```

如果漏掉某个 variant，编译器会报错。

与 C/C++ 对比：

- C `switch` 漏 case 很常见。
- C++ `std::variant` + `std::visit` 能实现类似风格，但语法和生态不同。
- Rust 的 enum + match 是语言核心，使用频率非常高。

### 12.4 if let 和 while let

只关心一种情况：

```rust
if let Some(value) = maybe_value {
    println!("{value}");
}
```

循环处理：

```rust
while let Some(value) = stack.pop() {
    println!("{value}");
}
```

## 13. Option 和 Result：把空值与错误变成类型

### 13.1 `Option<T>` 替代空指针

定义：

```rust
enum Option<T> {
    Some(T),
    None,
}
```

使用：

```rust
fn find_user(id: u64) -> Option<String> {
    if id == 1 {
        Some(String::from("Alice"))
    } else {
        None
    }
}
```

调用：

```rust
match find_user(2) {
    Some(name) => println!("{name}"),
    None => println!("not found"),
}
```

与 C/C++ 对比：

| 需求 | C | C++ | Rust |
| --- | --- | --- | --- |
| 可能没有值 | `NULL` 指针、特殊返回值 | `nullptr`、`std::optional<T>` | `Option<T>` |
| 调用方是否必须处理 | 不一定 | 取决于写法 | 模式匹配或方法链强迫处理 |
| 空指针解引用 | 常见运行期错误 | 仍可能发生 | `Option<T>` 不能直接当 `T` 用 |

### 13.2 `Result<T, E>` 表达可恢复错误

定义近似为：

```rust
enum Result<T, E> {
    Ok(T),
    Err(E),
}
```

示例：

```rust
use std::fs;
use std::io;

fn read_config(path: &str) -> Result<String, io::Error> {
    fs::read_to_string(path)
}
```

调用：

```rust
match read_config("app.toml") {
    Ok(text) => println!("{text}"),
    Err(err) => eprintln!("read failed: {err}"),
}
```

### 13.3 `?` 运算符

```rust
use std::fs;
use std::io;

fn load_user_name(path: &str) -> Result<String, io::Error> {
    let text = fs::read_to_string(path)?;
    Ok(text.trim().to_string())
}
```

`?` 的含义：

```text
如果是 Ok(value)，取出 value 继续执行。
如果是 Err(error)，提前返回 Err(error)。
```

它类似 C 中连续检查返回码，但更紧凑：

```c
int rc = read_config(path, &buf);
if (rc != 0) return rc;
```

也不同于 C++ 异常：

- 错误类型在函数签名里显式出现。
- 调用方能从类型看到可恢复错误。
- 控制流仍然是显式的，只是语法简化。

### 13.4 panic 不是普通错误处理

Rust 有 `panic!`：

```rust
panic!("something impossible happened");
```

但常规可恢复错误应使用 `Result`。

经验：

| 场景 | 推荐 |
| --- | --- |
| 文件不存在、网络失败、解析失败 | `Result` |
| 调用者违反前置条件 | 视场景使用 `panic!` 或返回错误 |
| 程序内部不变量被破坏 | `panic!` |
| 示例代码快速失败 | 可以 `unwrap`，生产代码谨慎 |

### 13.5 `unwrap` 与 `expect`

```rust
let value = maybe.unwrap();
let value = maybe.expect("config must contain port");
```

它们失败时会 panic。

初学者不要用 `unwrap` 回避错误处理。

更好的学习方式是：

```rust
match maybe {
    Some(value) => value,
    None => return,
}
```

或用组合方法：

```rust
let port = config.get("port").unwrap_or(&"8080");
```

## 14. 泛型、trait 与零成本抽象

### 14.1 泛型函数

```rust
fn first<T>(items: &[T]) -> Option<&T> {
    items.first()
}
```

`T` 是类型参数。

与 C++ 模板类似，Rust 泛型通常通过单态化生成具体类型版本：

```text
first::<i32>
first::<String>
```

运行期通常没有泛型抽象成本。

### 14.2 trait 表达能力约束

```rust
trait Summary {
    fn summarize(&self) -> String;
}

struct Article {
    title: String,
}

impl Summary for Article {
    fn summarize(&self) -> String {
        self.title.clone()
    }
}
```

使用 trait bound：

```rust
fn print_summary<T: Summary>(item: &T) {
    println!("{}", item.summarize());
}
```

这里编译器知道：

```text
T 必须实现 Summary，所以可以调用 summarize。
```

### 14.3 与 C++ 模板和 Concepts 对比

C++ 模板传统上是鸭子类型风格：

```cpp
template <typename T>
void print_summary(const T& item) {
    std::cout << item.summarize();
}
```

如果 `T` 没有 `summarize`，错误通常出现在模板实例化时。

C++20 Concepts 改善了这一点：

```cpp
template <Summary T>
void print_summary(const T& item) {
    std::cout << item.summarize();
}
```

Rust 从一开始就要求泛型能力写成 trait bound：

```rust
fn print_summary<T: Summary>(item: &T) {}
```

区别：

| 维度 | C++ 模板 | Rust 泛型 + trait |
| --- | --- | --- |
| 能力约束 | 传统模板隐式，Concepts 显式 | trait bound 显式 |
| 错误位置 | 可能很深 | 通常围绕 trait bound |
| 编译模型 | 模板实例化 | 单态化 |
| 抽象成本 | 通常零成本 | 通常零成本 |
| 接口文档 | Concepts/约定 | trait 本身就是接口 |

### 14.4 trait 默认实现

```rust
trait Summary {
    fn summarize(&self) -> String {
        String::from("(no summary)")
    }
}
```

实现者可以使用默认实现，也可以覆盖。

这类似 C++ 抽象基类中提供默认虚函数实现，但 Rust trait 不等同于继承。

### 14.5 关联类型

```rust
trait Iterator {
    type Item;

    fn next(&mut self) -> Option<Self::Item>;
}
```

`Item` 是实现者决定的关联类型。

这比每次写 `Iterator<T>` 更适合表达：

```text
一个具体迭代器有一个自然的元素类型。
```

### 14.6 orphan rule

Rust 规定：

```text
你只能在当前 crate 中为本地类型实现外部 trait，
或者为外部类型实现本地 trait。
不能为外部类型实现外部 trait。
```

目的：

```text
避免不同 crate 为同一类型和同一 trait 提供冲突实现。
```

C++ 中有 ADL、模板特化、重载等机制，灵活但容易产生复杂解析规则。

Rust 用更严格规则换取全局一致性。

## 15. trait object 与动态分发

泛型通常是静态分发：

```rust
fn draw<T: Draw>(item: &T) {
    item.draw();
}
```

编译器为具体类型生成代码。

trait object 使用动态分发：

```rust
trait Draw {
    fn draw(&self);
}

fn draw_all(items: &[Box<dyn Draw>]) {
    for item in items {
        item.draw();
    }
}
```

`dyn Draw` 可以理解为：

```text
data pointer + vtable pointer
```

这和 C++ 虚函数表很相似。

### 15.1 什么时候用泛型，什么时候用 trait object

| 需求 | 推荐 |
| --- | --- |
| 编译期知道具体类型 | 泛型 `T: Trait` |
| 需要最高性能并允许代码膨胀 | 泛型 |
| 一个集合里放多种实现类型 | `Box<dyn Trait>` |
| 运行期插件式分发 | `dyn Trait` |
| API 希望隐藏具体类型但保持静态分发 | `impl Trait` |

### 15.2 与 C++ 虚函数对比

| 维度 | C++ virtual | Rust `dyn Trait` |
| --- | --- | --- |
| 动态分发 | 虚函数表 | vtable |
| 对象存储 | 通常基类指针/引用 | `&dyn Trait`、`Box<dyn Trait>` 等 |
| 继承 | 类继承体系 | trait 实现，不要求继承 |
| 默认选择 | OOP 中常用 | Rust 更常优先静态分发 |
| 成本 | 间接调用，可能影响内联 | 间接调用，可能影响内联 |

Rust 没有传统类继承。

它更偏组合：

```text
struct 保存数据；
trait 描述行为；
impl 把行为附加到类型上。
```

## 16. 模块、crate 与可见性

Rust 工程的层次：

```text
package  包，Cargo 管理
crate    编译单元，库或二进制
module   代码命名空间
item     函数、结构体、trait、常量等
```

### 16.1 module 示例

```rust
mod network {
    pub fn connect() {
        println!("connect");
    }

    fn private_helper() {}
}

fn main() {
    network::connect();
}
```

默认私有，使用 `pub` 暴露。

### 16.2 与 C/C++ 头文件对比

C/C++ 常见结构：

```text
foo.h    声明
foo.cpp  实现
```

Rust 没有传统头文件。

模块系统负责：

- 命名空间。
- 可见性。
- 文件组织。
- crate 边界。

Rust 的 API 通常直接从源码 item 和 `pub` 推导。

### 16.3 crate 边界与编译

crate 是 Rust 编译的重要边界。

一个库 crate 对外暴露公共 API。

下游 crate 只能依赖这些 `pub` item。

这类似 C++ 中库头文件暴露接口，但 Rust 的可见性更统一。

## 17. 迭代器、闭包与函数式风格

### 17.1 闭包

```rust
let add = |a: i32, b: i32| a + b;
println!("{}", add(1, 2));
```

闭包可以捕获环境：

```rust
let prefix = String::from("user:");
let make_key = |id: u64| format!("{prefix}{id}");
```

闭包捕获方式由使用方式推断：

- 不可变借用。
- 可变借用。
- move 捕获所有权。

### 17.2 Iterator 是惰性的

```rust
let v = vec![1, 2, 3];
let iter = v.iter().map(|x| x * 2);
```

此时并没有真正遍历。

需要消费：

```rust
let result: Vec<_> = iter.collect();
```

### 17.3 map/filter/fold

```rust
let nums = vec![1, 2, 3, 4, 5];

let sum: i32 = nums
    .iter()
    .filter(|x| **x % 2 == 1)
    .map(|x| x * x)
    .sum();

println!("{sum}");
```

这通常可以被优化成接近手写循环的代码。

Rust 标准库迭代器是零成本抽象的重要例子。

### 17.4 与 C++ ranges 对比

C++ 也有 STL 算法、lambda、ranges。

Rust 的差异：

- Iterator trait 是核心抽象之一。
- 所有权和借用会参与迭代器类型检查。
- 消费型迭代、借用型迭代、可变借用型迭代区分清楚。

```rust
for x in v.iter() {}       // &T
for x in v.iter_mut() {}   // &mut T
for x in v.into_iter() {}  // T，消费 v
```

这三个在 C++ 中也能表达，但 Rust 把所有权影响写得更明显。

## 18. 智能指针与内部可变性

Rust 的所有权默认是单 owner。

某些场景需要更复杂的所有权结构。

### 18.1 `Box<T>`：堆分配唯一所有权

```rust
let value = Box::new(42);
```

`Box<T>` 类似 C++ `std::unique_ptr<T>`：

```text
唯一 owner；
离开作用域释放堆对象；
可以 move，不能随意 copy。
```

常见用途：

- 大对象放堆上。
- 递归类型。
- trait object：`Box<dyn Trait>`。

### 18.2 `Rc<T>`：单线程引用计数

```rust
use std::rc::Rc;

let a = Rc::new(String::from("shared"));
let b = Rc::clone(&a);
let c = Rc::clone(&a);
```

`Rc<T>` 允许多个 owner 共享只读数据。

它不是线程安全的。

类似 C++ `std::shared_ptr<T>`，但 Rust 区分单线程 `Rc` 和多线程 `Arc`。

### 18.3 `Arc<T>`：多线程引用计数

```rust
use std::sync::Arc;

let shared = Arc::new(String::from("shared"));
let other = Arc::clone(&shared);
```

`Arc<T>` 使用原子引用计数，可跨线程共享。

代价是原子操作成本。

### 18.4 `RefCell<T>`：运行期借用检查

```rust
use std::cell::RefCell;

let value = RefCell::new(1);
*value.borrow_mut() += 1;
```

`RefCell<T>` 把借用规则从编译期推迟到运行期。

如果违反规则，会 panic：

```rust
let a = value.borrow();
let b = value.borrow_mut(); // 运行期 panic
```

适用场景：

- 编译器无法证明，但程序逻辑能保证正确。
- 单线程内部可变性。
- 测试 mock。

### 18.5 `Cell<T>`

`Cell<T>` 适合 `Copy` 类型的内部可变性：

```rust
use std::cell::Cell;

let counter = Cell::new(0);
counter.set(counter.get() + 1);
```

### 18.6 `Mutex<T>` 与 `RwLock<T>`

多线程内部可变性使用同步原语：

```rust
use std::sync::Mutex;

let value = Mutex::new(0);
{
    let mut guard = value.lock().unwrap();
    *guard += 1;
} // guard drop，自动解锁
```

这与 C++ `std::mutex` + `std::lock_guard` 很相似。

Rust 的 `Mutex<T>` 把被保护的数据放在锁里面：

```text
Mutex<T> = 锁 + 数据
```

C++ 常见模式是：

```cpp
std::mutex m;
Data data;
```

程序员要靠约定保证访问 `data` 前先锁 `m`。

Rust 的类型把这个关系表达得更强。

### 18.7 智能指针对比

| 需求 | Rust | C++ |
| --- | --- | --- |
| 唯一堆对象 | `Box<T>` | `std::unique_ptr<T>` |
| 单线程共享所有权 | `Rc<T>` | `std::shared_ptr<T>`，但线程安全计数 |
| 多线程共享所有权 | `Arc<T>` | `std::shared_ptr<T>` |
| 单线程内部可变 | `Cell<T>`、`RefCell<T>` | `mutable`、自定义约定 |
| 多线程可变共享 | `Mutex<T>`、`RwLock<T>` | `std::mutex`、`std::shared_mutex` + 数据 |

## 19. 并发：把数据竞争变成编译错误

Rust 安全并发的核心不是“自动并发”，而是：

```text
所有权、借用和类型系统共同阻止数据竞争。
```

数据竞争通常需要同时满足：

```text
两个或多个线程访问同一内存；
至少一个是写；
没有同步；
```

Rust 安全代码尽量让这种情况无法通过编译。

### 19.1 线程与 move

```rust
use std::thread;

fn main() {
    let data = vec![1, 2, 3];

    let handle = thread::spawn(move || {
        println!("{data:?}");
    });

    handle.join().unwrap();
}
```

`move` 把 `data` 的所有权转移到新线程。

这样主线程不能再使用 `data`，避免悬垂和并发访问冲突。

### 19.2 `Send` 与 `Sync`

两个关键 trait：

| trait | 含义 |
| --- | --- |
| `Send` | 类型的值可以安全转移到另一个线程 |
| `Sync` | `&T` 可以安全在线程间共享 |

简单理解：

```text
T: Send  表示所有权能跨线程移动。
T: Sync  表示多个线程能共享 &T。
```

大多数普通类型自动实现。

但 `Rc<T>` 不是 `Send`/`Sync`，因为引用计数不是原子的。

如果要跨线程共享，使用 `Arc<T>`。

### 19.3 共享可变状态

```rust
use std::sync::{Arc, Mutex};
use std::thread;

fn main() {
    let counter = Arc::new(Mutex::new(0));
    let mut handles = Vec::new();

    for _ in 0..10 {
        let counter = Arc::clone(&counter);
        handles.push(thread::spawn(move || {
            let mut value = counter.lock().unwrap();
            *value += 1;
        }));
    }

    for handle in handles {
        handle.join().unwrap();
    }

    println!("{}", *counter.lock().unwrap());
}
```

这里的类型含义非常清楚：

```text
Arc      多线程共享所有权
Mutex    同一时间只允许一个线程可变访问内部数据
```

### 19.4 与 C++ 并发对比

C++：

```cpp
std::mutex m;
int counter = 0;

void inc() {
    std::lock_guard<std::mutex> lock(m);
    ++counter;
}
```

Rust：

```rust
let counter = Arc::new(Mutex::new(0));
```

差异：

| 问题 | C++ | Rust |
| --- | --- | --- |
| 数据和锁关系 | 常靠约定放在一起 | `Mutex<T>` 类型绑定 |
| 跨线程所有权 | 靠程序员保证生命周期 | `move` + `Send` 检查 |
| 非线程安全引用计数 | `shared_ptr` 计数线程安全，但对象访问仍需同步 | `Rc` 不可跨线程，`Arc` 可跨线程 |
| 数据竞争 | 未定义行为 | 安全 Rust 通常编译期阻止 |

Rust 仍然可能死锁。

类型系统不能自动证明所有并发逻辑正确。

它主要防止数据竞争，不保证没有竞态条件、死锁或业务顺序错误。

## 20. 异步编程：Future、async 与 await

Rust 异步编程围绕 `Future`。

```rust
async fn fetch_user(id: u64) -> String {
    format!("user-{id}")
}
```

`async fn` 返回的是 Future，不是立即执行完成的值。

你需要 `.await`：

```rust
let name = fetch_user(1).await;
```

### 20.1 Future 是状态机

直观理解：

```text
async 函数被编译成一个状态机。
await 点是可能暂停和恢复的位置。
```

类似：

```text
state 0: 发起 I/O
state 1: 等待 I/O 完成
state 2: 继续处理结果
```

这种模型允许大量任务复用少量线程。

### 20.2 Rust async 与线程不同

线程：

```text
操作系统调度；
每个线程有栈；
阻塞会占住线程。
```

async task：

```text
由 executor 调度；
Future 状态保存在对象中；
await 时让出执行权。
```

如果在 async 函数中调用阻塞 I/O，可能阻塞 executor 线程。

### 20.3 与 C++ coroutine 对比

C++20 提供协程语言机制。

Rust async/await 也是语言级状态机转换，但生态设计不同：

| 维度 | C++ coroutine | Rust async |
| --- | --- | --- |
| 核心抽象 | coroutine frame、promise type、awaiter | `Future` trait |
| 执行器 | 标准层面较底层，依赖库设计 | 标准库定义 Future，实际 executor 由 Tokio、async-std 等提供 |
| 内存安全 | 仍需遵守 C++ 生命周期规则 | borrow checker 参与 async 状态机检查 |
| Pin 问题 | 由库和对象生命周期管理 | `Pin` 明确表达不可移动需求 |

### 20.4 Pin 的直观理解

某些 Future 内部可能自引用：

```text
Future struct:
  buffer
  pointer_to_buffer
```

如果这个 Future 被 move，内部指针可能失效。

`Pin` 用来表达：

```text
这个值放到某个位置后，不能再被安全移动。
```

初学者不需要一开始深入实现 Pin，但要知道它是 Rust async 和自引用结构中的重要底层概念。

## 21. unsafe Rust：把不变量封装在安全接口背后

`unsafe` 不是关闭所有检查。

它只允许你做少数安全 Rust 禁止的操作。

常见能力包括：

```text
解引用裸指针；
调用 unsafe 函数或方法；
访问或修改可变静态变量；
实现 unsafe trait；
访问 union 字段。
```

### 21.1 unsafe 块的含义

```rust
let mut x = 10;
let p = &mut x as *mut i32;

unsafe {
    *p += 1;
}
```

`unsafe` 的含义不是：

```text
这里一定危险。
```

而是：

```text
这里有编译器无法完全检查的安全前提，程序员负责证明这些前提成立。
```

### 21.2 unsafe 的正确设计方式

好的 Rust 库通常这样组织：

```text
少量 unsafe 内部实现
  |
  v
安全 public API
  |
  v
调用者无需写 unsafe
```

例如一个安全抽象需要保证：

- 指针非空。
- 指针对齐正确。
- 指向初始化内存。
- 没有违反别名规则。
- 生命周期足够长。
- 并发访问有同步。

把这些不变量封装在类型和 API 中。

### 21.3 与 C/C++ 对比

C/C++ 默认就允许大量底层操作：

```cpp
int* p = reinterpret_cast<int*>(addr);
*p = 42;
```

Rust 把这类操作集中在 `unsafe` 中。

优势：

```text
代码审查可以优先关注 unsafe 边界。
普通 safe Rust 调用方不需要重新证明底层指针不变量。
```

但要警惕：

```text
unsafe 块内部写错，可能让外部 safe API 也变得不安全。
```

所以 Rust 社区常说：

```text
unsafe does not mean incorrect;
safe interface over unsound unsafe is incorrect.
```

也就是说，`unsafe` 本身不是罪，错误的安全抽象才是问题。

## 22. FFI：与 C/C++ 互操作

Rust 可以和 C ABI 互操作。

### 22.1 调用 C 函数

```rust
unsafe extern "C" {
    fn abs(input: i32) -> i32;
}

fn main() {
    let value = unsafe { abs(-3) };
    println!("{value}");
}
```

为什么调用是 unsafe？

因为 Rust 编译器无法检查外部 C 函数是否满足：

- 参数有效。
- 指针不悬垂。
- 没有违反线程安全。
- 返回值符合约定。

### 22.2 导出 Rust 函数给 C

```rust
#[unsafe(no_mangle)]
pub extern "C" fn add(a: i32, b: i32) -> i32 {
    a + b
}
```

关键点：

- `extern "C"` 指定 ABI。
- `no_mangle` 保持符号名。
- 传递复杂结构时需要谨慎布局。

### 22.3 `repr(C)`

Rust 默认结构体布局不保证和 C 一致。

需要 C 兼容布局：

```rust
#[repr(C)]
pub struct Point {
    pub x: f64,
    pub y: f64,
}
```

### 22.4 字符串和所有权边界

C 字符串：

```text
char*，以 NUL 结尾，编码靠约定。
```

Rust 字符串：

```text
String / &str，UTF-8，长度已知，不要求 NUL 结尾。
```

FFI 中要使用：

```rust
use std::ffi::{CStr, CString};
```

原则：

```text
跨 FFI 边界时，必须明确谁分配、谁释放、字符串如何编码、指针能活多久。
```

这和 C/C++ 库设计中的 ABI 边界问题完全一样，只是 Rust 会强迫你把不安全部分显式标出来。

## 23. 宏、过程宏与元编程

Rust 有两类常见宏：

```text
声明宏 macro_rules!
过程宏 procedural macro
```

### 23.1 声明宏

```rust
macro_rules! say_hello {
    () => {
        println!("hello");
    };
}

say_hello!();
```

声明宏做模式匹配式代码生成。

### 23.2 常见过程宏

你经常看到：

```rust
#[derive(Debug, Clone)]
struct User {
    id: u64,
    name: String,
}
```

`derive` 宏根据结构体生成 trait 实现。

还有属性宏：

```rust
#[tokio::main]
async fn main() {}
```

### 23.3 与 C/C++ 宏和模板对比

| 维度 | C 宏 | C++ 模板 | Rust 宏 |
| --- | --- | --- | --- |
| 工作层次 | 文本替换 | 类型级泛型和编译期实例化 | token 级或 AST 级代码生成 |
| 类型检查 | 替换后检查 | 模板实例化时检查 | 展开后检查 |
| 常见用途 | 条件编译、代码生成 | 泛型、元编程 | 消除样板、derive、DSL |
| 风险 | 容易污染命名和产生副作用 | 错误信息复杂 | 宏过度使用会降低可读性 |

Rust 不鼓励用宏替代普通函数和泛型。

原则：

```text
能用函数就用函数；
能用泛型和 trait 就用泛型和 trait；
只有需要生成结构性代码时才用宏。
```

## 24. 测试、文档与工程实践

### 24.1 单元测试

```rust
fn add(a: i32, b: i32) -> i32 {
    a + b
}

#[cfg(test)]
mod tests {
    use super::*;

    #[test]
    fn adds_two_numbers() {
        assert_eq!(add(1, 2), 3);
    }
}
```

运行：

```bash
cargo test
```

### 24.2 文档测试

```rust
/// Adds two numbers.
///
/// ```
/// let result = my_crate::add(1, 2);
/// assert_eq!(result, 3);
/// ```
pub fn add(a: i32, b: i32) -> i32 {
    a + b
}
```

文档中的代码也能被测试。

这比很多 C/C++ 项目里“文档示例长期失效”的情况更可控。

### 24.3 clippy 与 rustfmt

```bash
cargo fmt
cargo clippy
```

建议：

- 提交前运行 `cargo fmt`。
- 对重要项目启用 clippy。
- CI 中运行测试和格式检查。

### 24.4 与 C/C++ 工程实践对比

| 需求 | C/C++ | Rust |
| --- | --- | --- |
| 格式化 | clang-format 等 | rustfmt |
| 静态检查 | clang-tidy、cppcheck | clippy |
| 单元测试 | 第三方常见 | Cargo 内建 |
| 文档测试 | 需要额外工具 | rustdoc 支持 |
| 包管理 | 多方案并存 | Cargo 是默认路径 |

Rust 工程的一大优势是：

```text
工具链默认统一，新项目起步成本低。
```

## 25. 性能模型与调优方法

Rust 的目标是零成本抽象。

但“零成本”不是“怎么写都快”。

它的含义更接近：

```text
你不用为未使用的抽象能力付费；
高级抽象通常能优化到接近手写底层代码。
```

### 25.1 常见性能来源

Rust 性能来自：

- 无 GC 停顿。
- 所有权让释放时机确定。
- 泛型单态化。
- 迭代器可内联优化。
- enum 和 match 能生成紧凑分支。
- 借用引用通常就是普通指针。

### 25.2 常见性能陷阱

| 陷阱 | 原因 | 方向 |
| --- | --- | --- |
| 过度 `clone` | 为绕过 borrow checker 做深拷贝 | 重新设计所有权或借用 |
| 过度 `Arc<Mutex<T>>` | 把所有共享都放锁里 | 拆分状态，用消息传递或更细粒度锁 |
| 大量小分配 | `String`、`Vec` 频繁增长 | 预分配容量、复用 buffer |
| 动态分发过多 | `dyn Trait` 间接调用 | 热路径使用泛型 |
| 错误的 key/string 处理 | 反复分配和转换 | 使用 `&str`、切片、Cow |
| async 中阻塞 | 阻塞 executor 线程 | 使用异步 I/O 或专用阻塞线程池 |

### 25.3 调优顺序

不要先猜。

建议顺序：

```text
1. 写清楚正确代码。
2. 用测试保护行为。
3. 用 benchmark 或 profiler 找热点。
4. 减少不必要分配和 clone。
5. 优化数据布局和算法。
6. 最后才考虑 unsafe。
```

这和优秀 C/C++ 工程实践一致：

```text
先测量，再优化；
先算法和数据结构，再微优化；
把不安全优化限制在小范围。
```

### 25.4 数据布局思维

Rust 也需要理解底层布局。

例如：

```rust
struct Particle {
    x: f32,
    y: f32,
    z: f32,
    vx: f32,
    vy: f32,
    vz: f32,
}
```

这类似 C/C++ struct。

如果要 SIMD 或缓存友好，可能改成 SoA：

```rust
struct Particles {
    x: Vec<f32>,
    y: Vec<f32>,
    z: Vec<f32>,
    vx: Vec<f32>,
    vy: Vec<f32>,
    vz: Vec<f32>,
}
```

Rust 不替你自动做所有性能设计。

它只是让你在做底层优化时更容易保持安全边界。

## 26. 从 C/C++ 到 Rust 的常见误区

### 26.1 误区：遇到 borrow checker 就 clone

错误倾向：

```rust
let b = a.clone();
```

不断 clone 可能掩盖所有权设计问题。

先问：

```text
这个函数真的需要拥有值吗？
能不能传 &T 或 &mut T？
数据结构是否应该调整 owner？
是否需要把计算拆成更短的作用域？
```

### 26.2 误区：到处使用 `Arc<Mutex<T>>`

`Arc<Mutex<T>>` 很有用，但它不是万能解法。

过度使用会带来：

- 锁竞争。
- 死锁风险。
- 代码复杂。
- 并发性能下降。

优先考虑：

- 单 owner。
- 消息传递。
- 拆分锁。
- 不可变共享。
- 原子类型。

### 26.3 误区：把 Rust trait 当 C++ 继承

Rust trait 不是类继承。

Rust 更推荐：

```text
数据组合 + trait 行为抽象
```

而不是：

```text
深层继承树
```

如果你想写：

```text
Base -> DerivedA -> DerivedB
```

先考虑是否可以用：

```text
struct 组合字段
trait 表达行为
enum 表达有限变体
```

### 26.4 误区：生命周期标注能延长对象生命

生命周期标注不能让值活得更久。

它只是告诉编译器引用之间的关系。

错误理解：

```text
加一个 'static 就能修复悬垂引用。
```

正确理解：

```text
如果数据本来不能活到那么久，标注更长生命周期只会被编译器拒绝。
```

### 26.5 误区：unsafe 能快速解决一切

`unsafe` 可以绕过部分静态检查，但你必须自己维护不变量。

如果只是为了绕过 borrow checker，通常说明还没有理解所有权关系。

合理使用 unsafe 的场景：

- FFI。
- 高性能数据结构内部。
- 操作系统或硬件接口。
- 自定义分配器。
- 编译器无法表达但可以严格证明的内存布局。

### 26.6 误区：Rust 不需要理解内存

Rust 让内存安全更容易，但不是让你不用理解内存。

你仍然要知道：

- 栈和堆。
- move 和 copy。
- 分配和释放。
- 缓存局部性。
- 对齐。
- ABI。
- 原子操作。
- 锁和调度。

Rust 是系统语言，不是脚本语言。

## 27. 学习路线与练习

### 27.1 第一阶段：写小程序

目标：

- 会用 Cargo。
- 会写函数、结构体、枚举。
- 会处理 `Option` 和 `Result`。
- 会用 `Vec`、`String`、`HashMap`。

练习：

1. 写一个命令行 todo 工具。
2. 写一个读取文件并统计词频的小程序。
3. 写一个解析简单 CSV 的程序。

### 27.2 第二阶段：掌握所有权和借用

目标：

- 能解释 move、borrow、lifetime。
- 知道什么时候用 `String`、`&str`。
- 知道什么时候传 `T`、`&T`、`&mut T`。

练习：

1. 实现一个简单栈 `Stack<T>`。
2. 写一个函数，返回字符串中第一个单词的 `&str`。
3. 把一个频繁 clone 的程序改成借用。

### 27.3 第三阶段：抽象能力

目标：

- 会写 trait。
- 会写泛型。
- 理解 `impl Trait` 和 `dyn Trait`。
- 会写迭代器风格代码。

练习：

1. 定义 `Storage` trait，实现内存版 KV store。
2. 用泛型写一个排序统计工具。
3. 用 `Box<dyn Trait>` 实现多种输出格式。

### 27.4 第四阶段：并发与异步

目标：

- 会用 `thread`、`Arc`、`Mutex`。
- 理解 `Send` 和 `Sync`。
- 理解 async/await 需要 executor。

练习：

1. 多线程统计大文件行数。
2. 用 channel 实现任务分发。
3. 用 async HTTP 客户端并发请求多个 URL。

### 27.5 第五阶段：unsafe 和 FFI

目标：

- 知道 unsafe 能做什么。
- 会写小型 C FFI。
- 理解 `repr(C)`、裸指针和 CString。
- 知道怎样把 unsafe 封装成 safe API。

练习：

1. 调用一个 C 函数。
2. 导出一个 Rust 动态库给 C 调用。
3. 写一个小型安全 wrapper，内部使用裸指针。

## 28. 官方资料

Rust：

- [The Rust Programming Language](https://doc.rust-lang.org/book/)
- [Understanding Ownership](https://doc.rust-lang.org/book/ch04-00-understanding-ownership.html)
- [References and Borrowing](https://doc.rust-lang.org/book/ch04-02-references-and-borrowing.html)
- [Generic Types, Traits, and Lifetimes](https://doc.rust-lang.org/book/ch10-00-generics.html)
- [Fearless Concurrency](https://doc.rust-lang.org/book/ch16-00-concurrency.html)
- [Unsafe Rust](https://doc.rust-lang.org/book/ch20-01-unsafe-rust.html)
- [The Rust Reference](https://doc.rust-lang.org/reference/)
- [The Rustonomicon](https://doc.rust-lang.org/nomicon/)
- [The Cargo Book](https://doc.rust-lang.org/cargo/)
- [Rust Standard Library](https://doc.rust-lang.org/std/)

C++ 对比参考：

- [C++ Core Guidelines](https://isocpp.github.io/CppCoreGuidelines/CppCoreGuidelines)
- [ISO C++: smart pointers and RAII FAQ](https://isocpp.org/wiki/faq/freestore-mgmt)
