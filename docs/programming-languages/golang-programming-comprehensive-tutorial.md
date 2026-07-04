# Go 编程语言教程：从零上手到熟练掌握

> 目标读者：第一次系统学习 Go 的初学者；也适合有 C/C++、Java、Python、JavaScript、Rust 等经验，但希望真正理解 Go 语言设计原则、并发模型和工程实践的开发者。
> 学习目标：先能用 Go 独立写命令行工具、HTTP 服务和测试；再理解 Go 的类型系统、接口、错误处理、goroutine、channel、context、内存模型和模块系统；最后能写出清晰、可维护、可观测、适合生产环境的 Go 程序。
> 说明：Go 是官方名称，Golang 是社区常见搜索词。本文依据 Go 官方文档、A Tour of Go、Effective Go、Go Modules Reference、Go Memory Model、标准库文档和官方教程做中文教程化整理。Go 版本、工具链和标准库会演进，实际项目以当前官方文档为准。

---

## 目录

0. [如何使用本文：从新手到熟练的学习路线](#0-如何使用本文从新手到熟练的学习路线)
1. [Go 到底解决什么问题](#1-go-到底解决什么问题)
2. [一句话理解 Go](#2-一句话理解-go)
3. [Go 的设计哲学](#3-go-的设计哲学)
4. [安装、工具链与第一个项目](#4-安装工具链与第一个项目)
5. [包、模块与工程结构](#5-包模块与工程结构)
6. [变量、常量与基本类型](#6-变量常量与基本类型)
7. [控制流：if、for、switch、defer](#7-控制流ifforswitchdefer)
8. [函数：参数、返回值、闭包与 defer](#8-函数参数返回值闭包与-defer)
9. [数组、切片与 map](#9-数组切片与-map)
10. [结构体与方法](#10-结构体与方法)
11. [接口：Go 抽象能力的核心](#11-接口go-抽象能力的核心)
12. [错误处理：显式面对失败](#12-错误处理显式面对失败)
13. [panic、recover 与异常边界](#13-panicrecover-与异常边界)
14. [指针、值语义与内存分配](#14-指针值语义与内存分配)
15. [goroutine：轻量并发执行单元](#15-goroutine轻量并发执行单元)
16. [channel：用通信组织并发](#16-channel用通信组织并发)
17. [select、超时、取消与 context](#17-select超时取消与-context)
18. [sync、atomic 与共享内存并发](#18-syncatomic-与共享内存并发)
19. [Go 内存模型与数据竞争](#19-go-内存模型与数据竞争)
20. [泛型：类型参数与约束](#20-泛型类型参数与约束)
21. [标准库学习路线](#21-标准库学习路线)
22. [文件、JSON、HTTP 与数据库常见任务](#22-文件jsonhttp-与数据库常见任务)
23. [测试、基准测试、模糊测试与 race 检测](#23-测试基准测试模糊测试与-race-检测)
24. [日志、配置、可观测性与服务工程](#24-日志配置可观测性与服务工程)
25. [性能模型、逃逸分析、GC 与 pprof](#25-性能模型逃逸分析gc-与-pprof)
26. [常见误区](#26-常见误区)
27. [项目练习路线](#27-项目练习路线)
28. [官方资料](#28-官方资料)

---

## 0. 如何使用本文：从新手到熟练的学习路线

学习 Go 不建议从“背完所有语法”开始。

更有效的路线是：

```text
能运行 -> 能写小工具 -> 能组织包和模块 -> 能写 HTTP 服务 -> 能写并发程序 -> 能测试和优化
```

Go 的语法相对少，但它对工程习惯要求很高：

```text
错误要显式处理；
接口要小而清晰；
并发要能取消、能超时、能收敛；
测试、格式化和模块管理要成为日常动作。
```

建议分五遍读：

| 阶段 | 目标 | 应读章节 | 能力标准 |
| --- | --- | --- | --- |
| 第一遍：能写起来 | 工具链、基础语法、函数、切片、map | 1、2、3、4、6、7、8、9 | 能写命令行文本处理工具 |
| 第二遍：能组织代码 | 包、模块、结构体、方法、接口、错误处理 | 5、10、11、12、13 | 能把程序拆成清晰包并写测试 |
| 第三遍：能做服务 | JSON、HTTP、context、日志、配置 | 17、21、22、24 | 能写一个可关闭的 HTTP API 服务 |
| 第四遍：能并发 | goroutine、channel、sync、内存模型 | 15、16、17、18、19 | 能写不会泄漏 goroutine 的并发任务 |
| 第五遍：能工程化 | 泛型、测试、性能、pprof、GC | 20、23、25、26、27 | 能分析慢请求、内存分配和数据竞争 |

### 0.1 新手最应该先掌握什么

第一阶段优先掌握：

```text
go mod init / go run / go test / go fmt
package / import
var / const / :=
if / for / switch / defer
func
slice / map / struct
error / errors.Is / fmt.Errorf
interface
goroutine / channel / context
```

这些足够写很多实用程序。

### 0.2 每个知识点都问三个问题

本文建议每学一个概念都问：

```text
1. 我在真实代码里什么时候会用它？
2. Go 为什么这样设计？
3. 如果用其他语言写，同类问题会怎么处理？
```

例如学习接口时：

```text
真实代码里：函数只依赖行为，不依赖具体实现。
Go 设计上：接口由使用方定义，小接口更容易组合。
其他语言对比：Java 常显式 implements，Go 是隐式满足。
```

### 0.3 熟练 Go 的判断标准

一个人是否熟练 Go，不只看会不会写语法。

更好的判断标准：

```text
能否把包边界设计清楚；
能否让接口小而稳定；
能否用 error 表达可恢复失败；
能否正确使用 context 取消请求；
能否避免 goroutine 泄漏；
能否写表格驱动测试；
能否用 race detector 发现数据竞争；
能否用 pprof 定位 CPU 和内存问题；
能否写出简单、可读、可维护的代码。
```

## 1. Go 到底解决什么问题

Go 是一门面向工程效率的编程语言。

它诞生时要解决的核心问题包括：

```text
大型代码库编译慢
依赖管理复杂
并发编程困难
服务端程序需要高并发 I/O
语言特性过多导致团队代码风格分裂
部署需要简单、可靠、可复制
```

Go 的回答是：

```text
语法少一点；
工具链统一一点；
并发模型内建一点；
错误处理显式一点；
接口抽象轻量一点；
编译和部署简单一点。
```

Go 不是为了成为语法最华丽的语言。

它更像一种工程取舍：

```text
牺牲部分表达花样，换取团队协作、构建速度、部署简洁和长期维护。
```

典型应用场景：

| 场景 | Go 的优势 |
| --- | --- |
| 后端 HTTP/RPC 服务 | 标准库强，部署简单，并发模型直接 |
| 云原生基础设施 | 编译成单文件，适合容器和运维工具 |
| 命令行工具 | 启动快，交叉编译方便 |
| 网络代理和网关 | goroutine 适合大量连接 |
| 数据处理小工具 | 标准库够用，代码简洁 |
| DevOps 和平台工程 | 工具链简单，易分发 |

Kubernetes、Docker、Prometheus、etcd、Terraform 等项目都大量使用 Go。

## 2. 一句话理解 Go

Go 可以理解为：

```text
Go = 简洁语法 + 统一工具链 + 显式错误处理 + 内建并发 + 小接口组合
```

再具体一点：

```text
Go 不追求把所有抽象能力放进语言，而是用少量机制覆盖大量工程场景。
```

它的核心关键词：

| 关键词 | 含义 |
| --- | --- |
| 简单 | 语法和语言特性较少，读代码成本低 |
| 显式 | 错误、依赖、控制流尽量直接写出来 |
| 组合 | 通过 struct、interface、package 组合能力 |
| 并发 | goroutine 和 channel 是语言级并发工具 |
| 工程 | gofmt、go test、go mod、go vet 等工具链统一 |

Go 程序常见长相：

```go
func DoSomething(ctx context.Context, input Input) (Output, error) {
    if err := validate(input); err != nil {
        return Output{}, err
    }

    result, err := callDependency(ctx, input)
    if err != nil {
        return Output{}, fmt.Errorf("call dependency: %w", err)
    }

    return result, nil
}
```

这段代码体现了 Go 的几个习惯：

```text
context 放在第一个参数。
返回值显式带 error。
错误立刻检查。
用 %w 包装错误。
正常路径尽量保持直线。
```

## 3. Go 的设计哲学

### 3.1 少即是多

Go 的语法比 C++、Rust、Scala 简单得多。

这不是能力不足，而是有意选择。

语言特性越多，团队内部就越容易出现：

```text
每个人用不同范式写代码；
代码审查关注语法技巧而不是业务逻辑；
新人读代码成本高；
工具链和静态分析更复杂。
```

Go 选择限制表达方式，让多数代码长得相似。

这种一致性对大型团队非常重要。

### 3.2 显式优先

Go 不使用异常作为普通错误处理机制。

它鼓励：

```go
value, err := doSomething()
if err != nil {
    return err
}
```

很多初学者觉得这啰嗦。

但它的好处是：

```text
失败路径清楚；
资源释放点清楚；
调用方能看到哪些操作可能失败；
代码审查能直接检查错误是否被处理。
```

Go 的哲学不是“少写几行”，而是“让控制流看得见”。

### 3.3 组合优先于继承

Go 没有类继承。

它通过：

```text
struct 组合数据
method 添加行为
interface 抽象能力
embedding 复用字段和方法
```

来实现可复用设计。

这减少了复杂继承树。

一个常见 Go 风格：

```go
type Reader interface {
    Read(p []byte) (n int, err error)
}
```

这个接口很小，却能连接文件、网络连接、压缩流、内存 buffer 等大量对象。

### 3.4 并发是组织程序的方式

Go 的名言常被概括为：

```text
不要通过共享内存来通信；
要通过通信来共享内存。
```

这不是说不能用锁。

它强调的是：

```text
如果数据所有权能通过 channel 明确流动，程序通常更容易推理；
如果多个 goroutine 共享同一块内存，就必须用 sync 或 atomic 保证同步。
```

## 4. 安装、工具链与第一个项目

Go 官方工具链很统一。

常用命令：

| 命令 | 作用 |
| --- | --- |
| `go version` | 查看 Go 版本 |
| `go env` | 查看 Go 环境 |
| `go mod init` | 初始化模块 |
| `go run` | 编译并运行 |
| `go build` | 编译生成二进制 |
| `go test` | 运行测试 |
| `go fmt` | 格式化代码 |
| `go vet` | 静态检查常见问题 |
| `go doc` | 查看文档 |
| `go get` | 添加或更新依赖 |
| `go mod tidy` | 清理模块依赖 |

### 4.1 第一个项目

创建目录：

```bash
mkdir hello-go
cd hello-go
go mod init example.com/hello-go
```

创建 `main.go`：

```go
package main

import "fmt"

func main() {
    fmt.Println("hello, go")
}
```

运行：

```bash
go run .
```

构建：

```bash
go build .
```

这会生成可执行文件。

### 4.2 这段程序背后的原则

```go
package main
```

表示这是一个可执行程序的入口包。

```go
import "fmt"
```

引入标准库格式化输出包。

```go
func main()
```

是程序入口函数。

Go 程序最基本结构是：

```text
模块 module
  -> 包 package
    -> 文件 .go
      -> 函数、类型、变量
```

### 4.3 命令行参数示例

```go
package main

import (
    "fmt"
    "os"
)

func main() {
    if len(os.Args) < 2 {
        fmt.Fprintln(os.Stderr, "usage: hello-go <name>")
        os.Exit(1)
    }

    fmt.Printf("hello, %s\n", os.Args[1])
}
```

这段代码涉及：

| 代码 | 意义 |
| --- | --- |
| `os.Args` | 命令行参数 |
| `len` | 获取切片长度 |
| `os.Stderr` | 标准错误输出 |
| `os.Exit(1)` | 非零退出码 |
| `fmt.Printf` | 格式化输出 |

新手要理解：

```text
os.Args[0] 是程序名；
第一个用户参数是 os.Args[1]；
访问切片前要检查长度，否则会 panic。
```

### 4.4 日常开发流程

建议固定成：

```text
写代码
go fmt ./...
go test ./...
go vet ./...
go run .
```

如果是长期项目：

```text
提交前至少 go test ./...
```

Go 的工具链强调统一风格：

```text
不要争论缩进和格式，交给 gofmt。
```

## 5. 包、模块与工程结构

### 5.1 module 是依赖边界

`go.mod` 描述模块：

```go
module example.com/myapp

go 1.xx
```

模块通常对应一个仓库。

它定义：

```text
模块路径
Go 语言版本，实际项目中写当前使用的 Go 版本，例如 go 1.25
依赖模块及版本
```

### 5.2 package 是代码组织边界

一个目录通常对应一个 package。

例如：

```text
myapp/
  go.mod
  main.go
  internal/
    user/
      user.go
      service.go
```

`internal/user` 可以是包：

```go
package user
```

Go 的包名通常短小、语义明确。

不要写：

```text
common
utils
helpers
```

这类包名容易变成垃圾桶。

更好的包名：

```text
user
order
config
httpserver
storage
```

### 5.3 可见性：首字母大小写

Go 用首字母大小写控制导出：

```go
type User struct {
    ID   int64
    name string
}

func NewUser(id int64, name string) User {
    return User{ID: id, name: name}
}
```

`User`、`ID`、`NewUser` 可被其他包访问。

`name` 不可被其他包直接访问。

原则：

```text
默认隐藏实现细节；
只导出调用者真正需要的名字。
```

### 5.4 推荐项目结构

小项目可以简单：

```text
mytool/
  go.mod
  main.go
  main_test.go
```

稍大项目：

```text
myservice/
  go.mod
  cmd/
    myservice/
      main.go
  internal/
    config/
    user/
    storage/
    httpapi/
  pkg/
    client/
```

常见约定：

| 目录 | 用途 |
| --- | --- |
| `cmd/<name>` | 可执行程序入口 |
| `internal` | 只允许当前模块内部使用的包 |
| `pkg` | 可以被外部使用的库包，谨慎使用 |
| `testdata` | 测试数据目录 |

不要一开始过度设计目录。

Go 项目结构原则：

```text
先让代码清楚；
包边界从真实依赖中长出来；
不要为了“看起来架构复杂”而拆包。
```

## 6. 变量、常量与基本类型

### 6.1 变量声明

Go 有几种声明方式：

```go
var a int
var b int = 10
var c = 20
d := 30
```

函数内部最常用：

```go
d := 30
```

包级变量不能使用 `:=`。

### 6.2 零值原则

Go 的每种类型都有零值：

| 类型 | 零值 |
| --- | --- |
| `int` | `0` |
| `float64` | `0` |
| `bool` | `false` |
| `string` | `""` |
| 指针 | `nil` |
| slice | `nil` |
| map | `nil` |
| channel | `nil` |
| struct | 每个字段都是零值 |

Go 很重视“零值可用”。

例如：

```go
var b strings.Builder
b.WriteString("hello")
fmt.Println(b.String())
```

不需要显式构造。

设计自己的类型时，也应思考：

```text
这个类型的零值是否可用？
如果不可用，是否需要 NewXxx 构造函数？
```

### 6.3 常量

```go
const Pi = 3.14159
const MaxRetries = 3
```

Go 常量可以是无类型常量：

```go
const n = 10
var x int64 = n
var y float64 = n
```

这让常量使用更灵活。

### 6.4 基本类型选择

| 需求 | 推荐类型 |
| --- | --- |
| 普通整数 | `int` |
| 明确位宽 | `int32`、`int64`、`uint64` |
| 字节 | `byte`，即 `uint8` |
| Unicode 字符 | `rune`，即 `int32` |
| 文本 | `string` |
| 布尔 | `bool` |
| 金额 | 不建议用浮点，考虑整数分或 decimal 库 |

Go 不会自动隐式转换不同数值类型：

```go
var a int = 10
var b int64 = int64(a)
```

这减少了隐式转换导致的错误。

## 7. 控制流：if、for、switch、defer

### 7.1 if

```go
if x > 0 {
    fmt.Println("positive")
} else {
    fmt.Println("non-positive")
}
```

Go 的 `if` 可以带初始化语句：

```go
if value, ok := m["key"]; ok {
    fmt.Println(value)
}
```

`value` 和 `ok` 的作用域只在 `if` 和 `else` 中。

这能减少临时变量污染外层作用域。

### 7.2 for 是唯一循环关键字

Go 没有 `while`。

三种写法：

```go
for i := 0; i < 10; i++ {
    fmt.Println(i)
}

for n > 0 {
    n--
}

for {
    break
}
```

遍历：

```go
for i, v := range values {
    fmt.Println(i, v)
}
```

注意 range 变量是每次迭代的变量值。

如果在闭包里使用循环变量，要理解当前 Go 版本语义和项目版本，历史上这类问题很常见。写并发循环时，仍建议显式传参让意图清晰：

```go
for _, v := range values {
    v := v
    go func() {
        fmt.Println(v)
    }()
}
```

### 7.3 switch

```go
switch os := runtime.GOOS; os {
case "darwin":
    fmt.Println("macOS")
case "linux":
    fmt.Println("Linux")
default:
    fmt.Println(os)
}
```

Go 的 `switch` 默认不会自动 fallthrough。

这避免了 C/C++ 中忘记 `break` 的常见错误。

### 7.4 defer

`defer` 会在当前函数返回前执行。

常用于释放资源：

```go
f, err := os.Open("data.txt")
if err != nil {
    return err
}
defer f.Close()
```

原则：

```text
资源获取成功后，立即 defer 释放。
```

多个 defer 按后进先出执行。

```go
defer fmt.Println("first")
defer fmt.Println("second")
```

输出顺序：

```text
second
first
```

## 8. 函数：参数、返回值、闭包与 defer

### 8.1 函数定义

```go
func add(a int, b int) int {
    return a + b
}
```

相同类型可以合并：

```go
func add(a, b int) int {
    return a + b
}
```

多返回值：

```go
func div(a, b int) (int, error) {
    if b == 0 {
        return 0, errors.New("division by zero")
    }
    return a / b, nil
}
```

这就是 Go 错误处理的基础。

### 8.2 参数传递：值传递

Go 参数传递是值传递。

```go
func inc(x int) {
    x++
}
```

不会修改外部变量。

如果要修改：

```go
func inc(x *int) {
    *x++
}
```

但实际项目里，不要因为想省复制就到处用指针。

选择指针接收或参数通常考虑：

```text
是否需要修改原对象；
对象是否很大；
是否要表达共享身份；
方法集合是否需要一致；
是否要避免复制锁等不可复制字段。
```

### 8.3 闭包

```go
func makeCounter() func() int {
    count := 0
    return func() int {
        count++
        return count
    }
}
```

闭包捕获外部变量。

```go
counter := makeCounter()
fmt.Println(counter()) // 1
fmt.Println(counter()) // 2
```

原理：

```text
函数值不仅包含代码，还包含它捕获的环境。
```

捕获变量可能导致变量逃逸到堆上，由编译器决定。

### 8.4 函数设计原则

好的 Go 函数通常：

```text
参数少；
返回值清楚；
错误显式；
不隐藏复杂副作用；
函数名说明行为；
context.Context 放第一个参数。
```

示例：

```go
func FetchUser(ctx context.Context, id int64) (User, error) {
    if id <= 0 {
        return User{}, fmt.Errorf("invalid user id: %d", id)
    }

    return callUserService(ctx, id)
}
```

## 9. 数组、切片与 map

### 9.1 数组

数组长度是类型的一部分：

```go
var a [3]int
var b [4]int
```

`[3]int` 和 `[4]int` 是不同类型。

数组在 Go 中不如 slice 常用。

### 9.2 切片 slice

slice 可以理解为：

```text
指向底层数组的指针 + 长度 len + 容量 cap
```

示例：

```go
s := []int{1, 2, 3}
fmt.Println(len(s), cap(s))
```

追加：

```go
s = append(s, 4)
```

注意：

```text
append 可能复用原底层数组，也可能分配新数组。
所以 append 后要使用返回的新 slice。
```

错误写法：

```go
append(s, 4) // 编译错误，结果未使用
```

正确写法：

```go
s = append(s, 4)
```

### 9.3 slice 共享底层数组

```go
a := []int{1, 2, 3, 4}
b := a[1:3]
b[0] = 20
fmt.Println(a) // [1 20 3 4]
```

因为 `b` 和 `a` 共享同一个底层数组。

这很重要：

```text
slice 不是数组本身；
slice 是对数组片段的描述。
```

如果需要独立副本：

```go
c := append([]int(nil), b...)
```

或：

```go
c := make([]int, len(b))
copy(c, b)
```

### 9.4 map

```go
m := map[string]int{
    "alice": 10,
    "bob":   20,
}
```

读取：

```go
score, ok := m["alice"]
if ok {
    fmt.Println(score)
}
```

`ok` 表示 key 是否存在。

如果直接读不存在的 key，会得到 value 类型的零值：

```go
fmt.Println(m["missing"]) // 0
```

所以需要区分：

```text
key 不存在；
key 存在但值刚好是零值。
```

### 9.5 nil slice 和 nil map

nil slice 可以 append：

```go
var s []int
s = append(s, 1)
```

nil map 不能写入：

```go
var m map[string]int
// m["a"] = 1 // panic
```

需要 make：

```go
m := make(map[string]int)
m["a"] = 1
```

## 10. 结构体与方法

### 10.1 struct

```go
type User struct {
    ID   int64
    Name string
}
```

创建：

```go
u := User{ID: 1, Name: "Alice"}
```

字段访问：

```go
fmt.Println(u.Name)
```

### 10.2 方法

```go
func (u User) DisplayName() string {
    return u.Name
}
```

接收者可以是值，也可以是指针：

```go
func (u *User) Rename(name string) {
    u.Name = name
}
```

选择原则：

| 接收者 | 适用 |
| --- | --- |
| 值接收者 | 不修改对象，对象小，复制安全 |
| 指针接收者 | 需要修改对象，对象较大，包含锁或不可复制字段 |

同一个类型的方法最好保持接收者风格一致，避免方法集合造成困惑。

### 10.3 构造函数习惯

Go 没有构造函数关键字。

常用 `NewXxx`：

```go
func NewUser(id int64, name string) User {
    return User{ID: id, Name: name}
}
```

如果返回指针：

```go
func NewUser(id int64, name string) *User {
    return &User{ID: id, Name: name}
}
```

返回值还是指针，取决于语义，不是固定规则。

### 10.4 组合和 embedding

```go
type Logger struct{}

func (Logger) Log(msg string) {
    fmt.Println(msg)
}

type Server struct {
    Logger
}
```

`Server` 可以直接调用：

```go
s := Server{}
s.Log("started")
```

这不是继承。

更准确地说：

```text
Server 嵌入了 Logger，Logger 的方法被提升到 Server 上。
```

要避免把 embedding 当成继承树使用。

Go 更推荐简单组合。

## 11. 接口：Go 抽象能力的核心

### 11.1 接口是什么

接口是一组方法的集合：

```go
type Reader interface {
    Read(p []byte) (n int, err error)
}
```

任何类型只要实现了这些方法，就自动满足接口。

不需要显式写：

```text
implements Reader
```

这叫结构化类型或隐式接口满足。

### 11.2 小接口原则

Go 标准库中很多接口很小：

```go
type Reader interface {
    Read(p []byte) (n int, err error)
}

type Writer interface {
    Write(p []byte) (n int, err error)
}
```

小接口的优势：

```text
容易实现；
容易测试；
容易组合；
调用者只依赖自己需要的能力。
```

不要一开始设计巨大的接口：

```go
type UserService interface {
    Create(...)
    Update(...)
    Delete(...)
    Find(...)
    List(...)
    Export(...)
    Import(...)
}
```

更好的方式是让使用方定义需要的最小接口：

```go
type UserFinder interface {
    FindUser(ctx context.Context, id int64) (User, error)
}
```

### 11.3 接口放在哪里

Go 常见原则：

```text
接口通常由使用方定义，而不是由实现方定义。
```

原因是使用方最清楚自己需要哪些方法。

例如：

```go
func PrintReport(w io.Writer, report Report) error {
    _, err := fmt.Fprintf(w, "report: %s\n", report.Title)
    return err
}
```

这里函数只需要 `Write` 能力，所以依赖 `io.Writer`。

调用方可以传入：

```text
文件
网络连接
bytes.Buffer
HTTP response writer
测试 mock
```

### 11.4 空接口 any

`any` 是 `interface{}` 的别名。

```go
func Print(v any) {
    fmt.Println(v)
}
```

它可以接收任意类型。

但不要滥用。

过多使用 `any` 会丢失类型信息，让错误推迟到运行期。

如果能用具体类型或泛型，就不要用 `any`。

## 12. 错误处理：显式面对失败

### 12.1 error 是普通值

Go 的错误通常作为最后一个返回值：

```go
data, err := os.ReadFile("config.json")
if err != nil {
    return err
}
```

`error` 是接口：

```go
type error interface {
    Error() string
}
```

错误是普通值，可以传递、包装、比较、记录。

### 12.2 为什么 Go 不用异常作为普通错误流

Go 鼓励显式错误处理：

```go
if err != nil {
    return err
}
```

这样做的原则：

```text
失败路径可见；
调用者无法假装没有失败；
函数签名直接说明可能失败；
资源清理和错误返回顺序清楚。
```

这会让代码多几行，但在服务端工程里很有价值。

### 12.3 包装错误

```go
if err != nil {
    return fmt.Errorf("read config: %w", err)
}
```

`%w` 会保留原始错误，便于后续用 `errors.Is` 或 `errors.As` 判断。

```go
if errors.Is(err, os.ErrNotExist) {
    fmt.Println("file not found")
}
```

### 12.4 自定义错误

简单错误：

```go
var ErrNotFound = errors.New("not found")
```

带上下文的错误类型：

```go
type ParseError struct {
    Line int
    Msg  string
}

func (e ParseError) Error() string {
    return fmt.Sprintf("line %d: %s", e.Line, e.Msg)
}
```

判断：

```go
var parseErr ParseError
if errors.As(err, &parseErr) {
    fmt.Println(parseErr.Line)
}
```

### 12.5 错误处理实践

好的错误处理：

```text
在低层保留具体原因；
在边界补充上下文；
在最外层记录或返回给用户；
不要重复记录同一个错误；
不要吞掉错误。
```

不推荐：

```go
_, _ = os.ReadFile("config.json")
```

除非你能解释为什么忽略错误。

## 13. panic、recover 与异常边界

`panic` 表示程序遇到无法正常继续的情况。

```go
panic("impossible state")
```

普通可恢复错误不应该用 panic。

适合 panic 的场景：

```text
程序启动时配置严重错误；
内部不变量被破坏；
测试中快速失败；
确实无法恢复的 bug。
```

`recover` 只能在 deferred 函数中捕获 panic。

```go
func safeCall(fn func()) {
    defer func() {
        if r := recover(); r != nil {
            fmt.Println("recovered:", r)
        }
    }()
    fn()
}
```

服务端常见做法是在请求边界 recover，避免一个请求导致整个进程退出。

但不要把 recover 当成普通控制流。

## 14. 指针、值语义与内存分配

### 14.1 指针

```go
x := 10
p := &x
*p = 20
fmt.Println(x)
```

Go 有指针，但没有 C/C++ 那样的指针算术。

这降低了很多内存错误风险。

### 14.2 值语义

赋值会复制值：

```go
a := User{Name: "Alice"}
b := a
b.Name = "Bob"
fmt.Println(a.Name) // Alice
```

但 slice、map、channel 内部包含引用性质的数据结构。

复制 slice 会复制 slice header，而不是复制底层数组。

这解释了为什么修改子切片会影响原切片。

### 14.3 栈、堆与逃逸分析

Go 编译器会决定变量放在栈上还是堆上。

如果一个局部变量在函数返回后仍需被使用，它可能逃逸到堆上。

示例：

```go
func NewCounter() *int {
    x := 0
    return &x
}
```

在 C/C++ 中返回局部变量地址是严重错误。

在 Go 中，编译器会让 `x` 逃逸到堆上，从而保持有效。

这不是免费午餐：

```text
堆分配会增加 GC 压力；
但它保证内存安全。
```

可以用：

```bash
go build -gcflags="-m" .
```

查看逃逸分析信息。

## 15. goroutine：轻量并发执行单元

goroutine 是 Go 的并发执行单元。

```go
go func() {
    fmt.Println("running")
}()
```

它比操作系统线程更轻量，由 Go runtime 调度到系统线程上执行。

### 15.1 goroutine 不是免费资源

goroutine 很轻，但不是无限免费。

常见问题：

```text
启动后没人等待；
阻塞在 channel 发送或接收；
阻塞在没有超时的网络调用；
请求结束后后台 goroutine 还在跑；
无限创建 goroutine 导致内存上涨。
```

这叫 goroutine 泄漏。

### 15.2 等待 goroutine 完成

```go
var wg sync.WaitGroup

for i := 0; i < 3; i++ {
    i := i
    wg.Add(1)
    go func() {
        defer wg.Done()
        fmt.Println(i)
    }()
}

wg.Wait()
```

原则：

```text
启动 goroutine 时，就要知道它何时退出、由谁等待、如何取消。
```

## 16. channel：用通信组织并发

channel 用于 goroutine 之间传递值。

```go
ch := make(chan int)

go func() {
    ch <- 42
}()

v := <-ch
fmt.Println(v)
```

### 16.1 无缓冲 channel

无缓冲 channel 发送和接收必须同时准备好。

```go
ch := make(chan int)
```

它像一次同步交接：

```text
发送方把值交给接收方；
双方在交接点同步。
```

### 16.2 有缓冲 channel

```go
ch := make(chan int, 10)
```

缓冲区未满时发送不阻塞。

缓冲区为空时接收阻塞。

缓冲 channel 适合做有限队列，但不要用无限缓冲掩盖消费慢的问题。

### 16.3 关闭 channel

发送方关闭 channel：

```go
close(ch)
```

接收方可以用 range：

```go
for v := range ch {
    fmt.Println(v)
}
```

原则：

```text
由发送方关闭 channel；
不要由接收方关闭；
不要重复关闭。
```

关闭 channel 表示：

```text
不会再发送新值。
```

不是用来强行终止接收方的万能信号。

## 17. select、超时、取消与 context

### 17.1 select

`select` 等待多个 channel 操作：

```go
select {
case v := <-ch:
    fmt.Println(v)
case <-time.After(time.Second):
    fmt.Println("timeout")
}
```

常用于：

```text
超时
取消
多路 channel 选择
非阻塞尝试
```

### 17.2 context

`context.Context` 用来传递：

```text
取消信号
截止时间
请求范围值
```

常见函数签名：

```go
func FetchUser(ctx context.Context, id int64) (User, error) {
    req, err := http.NewRequestWithContext(ctx, http.MethodGet, url, nil)
    if err != nil {
        return User{}, err
    }
    _ = req
    return User{}, nil
}
```

原则：

```text
context 通常作为第一个参数；
不要把 context 存进结构体长期保存；
请求结束时，相关 goroutine 应该能收到取消信号并退出。
```

### 17.3 worker pool 示例

```go
func worker(ctx context.Context, jobs <-chan int, results chan<- int) {
    for {
        select {
        case <-ctx.Done():
            return
        case job, ok := <-jobs:
            if !ok {
                return
            }
            results <- job * job
        }
    }
}
```

这里：

```text
jobs <-chan int 表示只接收；
results chan<- int 表示只发送；
ctx.Done() 负责取消；
jobs 关闭后 worker 退出。
```

这就是可收敛并发的基本形态。

## 18. sync、atomic 与共享内存并发

虽然 Go 鼓励通信，但共享内存仍然常用。

### 18.1 Mutex

```go
type Counter struct {
    mu sync.Mutex
    n  int
}

func (c *Counter) Inc() {
    c.mu.Lock()
    defer c.mu.Unlock()
    c.n++
}
```

原则：

```text
谁保护数据，锁就和数据放在一起。
```

不要把锁和被保护的数据分散到不同地方。

### 18.2 RWMutex

`sync.RWMutex` 允许多个读者或一个写者。

适合读多写少。

不要盲目使用 RWMutex。

如果临界区很短，普通 Mutex 可能更简单、更快。

### 18.3 Once

```go
var once sync.Once

once.Do(func() {
    initConfig()
})
```

保证初始化函数只执行一次。

### 18.4 atomic

原子操作适合简单计数或状态标志。

```go
var n atomic.Int64
n.Add(1)
fmt.Println(n.Load())
```

如果状态不止一个字段，通常用 Mutex 更清楚。

## 19. Go 内存模型与数据竞争

数据竞争指：

```text
两个 goroutine 同时访问同一变量；
至少一个是写；
没有同步。
```

数据竞争会让程序行为不可预测。

示例：

```go
var n int

go func() {
    n++
}()

n++
```

这是数据竞争。

检测：

```bash
go test -race ./...
```

### 19.1 happens-before

Go 内存模型讨论的是：

```text
一个 goroutine 的写入，什么时候保证能被另一个 goroutine 看见？
```

同步手段包括：

```text
channel 发送和接收
Mutex Lock/Unlock
atomic 操作
goroutine 创建的一些顺序关系
```

新手不用一开始背完整内存模型，但必须记住：

```text
没有同步，就不要假设另一个 goroutine 能正确看到你的写入。
```

## 20. 泛型：类型参数与约束

Go 1.18 引入泛型。

泛型用于写与类型无关的代码。

例子：

```go
func First[T any](items []T) (T, bool) {
    var zero T
    if len(items) == 0 {
        return zero, false
    }
    return items[0], true
}
```

调用：

```go
n, ok := First([]int{1, 2, 3})
s, ok := First([]string{"a", "b"})
```

### 20.1 约束

```go
type Number interface {
    ~int | ~int64 | ~float64
}

func Sum[T Number](items []T) T {
    var total T
    for _, item := range items {
        total += item
    }
    return total
}
```

`~int` 表示底层类型是 `int` 的自定义类型也满足约束。

### 20.2 什么时候用泛型

适合：

```text
容器
算法
类型安全的复用逻辑
减少重复但逻辑完全相同的代码
```

不适合：

```text
为了炫技；
业务逻辑差异明显；
接口已经更自然；
让代码变得难读。
```

Go 的泛型哲学仍然是简单。

## 21. 标准库学习路线

Go 标准库很强。

建议按任务学习：

| 任务 | 包 |
| --- | --- |
| 格式化输入输出 | `fmt` |
| 字符串处理 | `strings`、`strconv` |
| 文件和目录 | `os`、`io`、`bufio`、`path/filepath` |
| 时间 | `time` |
| JSON | `encoding/json` |
| HTTP | `net/http` |
| URL | `net/url` |
| 日志 | `log`、`log/slog` |
| 并发同步 | `sync`、`sync/atomic` |
| 上下文 | `context` |
| 测试 | `testing` |
| 性能分析 | `runtime/pprof`、`net/http/pprof` |

学习标准库的方法：

```text
先看包文档；
再读 Example；
最后写一个小程序验证。
```

## 22. 文件、JSON、HTTP 与数据库常见任务

### 22.1 读取文件

```go
data, err := os.ReadFile("config.json")
if err != nil {
    return fmt.Errorf("read config: %w", err)
}
fmt.Println(string(data))
```

大文件逐行读：

```go
f, err := os.Open("access.log")
if err != nil {
    return err
}
defer f.Close()

scanner := bufio.NewScanner(f)
for scanner.Scan() {
    line := scanner.Text()
    fmt.Println(line)
}
if err := scanner.Err(); err != nil {
    return err
}
```

### 22.2 JSON

```go
type User struct {
    ID   int64  `json:"id"`
    Name string `json:"name"`
}

var u User
if err := json.Unmarshal(data, &u); err != nil {
    return err
}
```

输出：

```go
out, err := json.MarshalIndent(u, "", "  ")
if err != nil {
    return err
}
fmt.Println(string(out))
```

### 22.3 HTTP 服务

```go
func main() {
    mux := http.NewServeMux()
    mux.HandleFunc("/healthz", func(w http.ResponseWriter, r *http.Request) {
        w.WriteHeader(http.StatusOK)
        _, _ = w.Write([]byte("ok"))
    })

    server := &http.Server{
        Addr:    ":8080",
        Handler: mux,
    }

    if err := server.ListenAndServe(); err != nil && err != http.ErrServerClosed {
        log.Fatal(err)
    }
}
```

真实服务还要处理：

```text
超时配置
优雅关闭
请求日志
错误响应
context 取消
依赖注入
测试
```

### 22.4 HTTP 客户端

```go
req, err := http.NewRequestWithContext(ctx, http.MethodGet, "https://example.com", nil)
if err != nil {
    return err
}

resp, err := http.DefaultClient.Do(req)
if err != nil {
    return err
}
defer resp.Body.Close()
```

原则：

```text
请求要带 context；
响应 body 要关闭；
生产环境配置超时，不要永远等待。
```

## 23. 测试、基准测试、模糊测试与 race 检测

### 23.1 单元测试

文件命名：

```text
xxx_test.go
```

测试函数：

```go
func TestAdd(t *testing.T) {
    got := Add(1, 2)
    if got != 3 {
        t.Fatalf("got %d, want %d", got, 3)
    }
}
```

运行：

```bash
go test ./...
```

### 23.2 表格驱动测试

Go 常用表格驱动测试：

```go
func TestAdd(t *testing.T) {
    tests := []struct {
        name string
        a    int
        b    int
        want int
    }{
        {name: "positive", a: 1, b: 2, want: 3},
        {name: "zero", a: 0, b: 0, want: 0},
    }

    for _, tt := range tests {
        t.Run(tt.name, func(t *testing.T) {
            got := Add(tt.a, tt.b)
            if got != tt.want {
                t.Fatalf("got %d, want %d", got, tt.want)
            }
        })
    }
}
```

优势：

```text
新增用例简单；
边界条件清楚；
失败时能看到子测试名称。
```

### 23.3 基准测试

```go
func BenchmarkAdd(b *testing.B) {
    for i := 0; i < b.N; i++ {
        _ = Add(1, 2)
    }
}
```

运行：

```bash
go test -bench=. ./...
```

### 23.4 模糊测试

模糊测试用于自动生成输入，发现边界问题。

```go
func FuzzParse(f *testing.F) {
    f.Add("123")
    f.Fuzz(func(t *testing.T, input string) {
        _, _ = strconv.Atoi(input)
    })
}
```

### 23.5 race 检测

```bash
go test -race ./...
```

并发代码建议经常跑 race 检测。

它会降低运行速度，但能发现很多真实问题。

## 24. 日志、配置、可观测性与服务工程

### 24.1 日志

Go 标准库提供 `log/slog` 结构化日志。

```go
logger := slog.Default()
logger.Info("user created", "user_id", 123)
```

原则：

```text
日志要包含上下文；
不要只写 failed；
不要记录敏感信息；
错误只在边界处记录一次。
```

### 24.2 配置

配置来源常见：

```text
命令行参数
环境变量
配置文件
服务发现或配置中心
```

小工具可以用 `flag`：

```go
port := flag.Int("port", 8080, "listen port")
flag.Parse()
fmt.Println(*port)
```

服务程序通常需要统一配置结构体。

### 24.3 优雅关闭

服务要处理退出信号：

```go
ctx, stop := signal.NotifyContext(context.Background(), os.Interrupt, syscall.SIGTERM)
defer stop()

<-ctx.Done()
```

HTTP 服务关闭：

```go
shutdownCtx, cancel := context.WithTimeout(context.Background(), 5*time.Second)
defer cancel()
_ = server.Shutdown(shutdownCtx)
```

原则：

```text
停止接收新请求；
给正在处理的请求一点时间；
超时后退出。
```

## 25. 性能模型、逃逸分析、GC 与 pprof

### 25.1 Go 性能从哪里来

Go 性能特点：

```text
编译成本低；
运行性能接近系统语言；
goroutine 调度适合高并发 I/O；
GC 降低手动内存管理成本；
标准库网络和 HTTP 性能良好。
```

但 GC 也意味着：

```text
大量堆分配会增加 GC 压力；
延迟敏感程序要关注分配和对象生命周期。
```

### 25.2 常见性能陷阱

| 问题 | 原因 | 方向 |
| --- | --- | --- |
| 分配过多 | 字符串拼接、临时 slice、interface 装箱 | 复用 buffer，减少临时对象 |
| goroutine 泄漏 | 没有取消、channel 阻塞 | context、关闭 channel、WaitGroup |
| 锁竞争 | 临界区过大 | 缩小锁范围、分片、无锁计数 |
| map 热点 | 单锁保护大 map | 分片或 sync.Map，按场景选择 |
| JSON 慢 | 反射和分配 | 先测量，再考虑替代库 |

### 25.3 pprof

HTTP 服务中启用 pprof：

```go
import _ "net/http/pprof"
```

这个空导入会把 pprof handler 注册到 `http.DefaultServeMux`。

如果你的服务直接使用默认 mux，启动 HTTP 调试端口后即可访问 `/debug/pprof/`。如果项目使用自定义 mux，需要显式挂载 pprof handler，或者单独启动一个使用默认 mux 的调试端口。

常用分析：

```bash
go tool pprof http://localhost:6060/debug/pprof/profile
go tool pprof http://localhost:6060/debug/pprof/heap
```

调优顺序：

```text
先测量；
找到热点；
理解原因；
改最小代码；
再测量验证。
```

不要凭感觉优化。

## 26. 常见误区

### 26.1 误区：Go 很简单，所以不需要学原理

Go 语法简单，不代表工程简单。

你仍然要理解：

```text
slice 底层数组共享；
interface 动态类型；
goroutine 生命周期；
context 取消；
数据竞争；
逃逸分析；
GC 压力；
错误包装。
```

### 26.2 误区：所有错误都直接 return err

直接返回有时会丢失上下文。

更好：

```go
return fmt.Errorf("load config %s: %w", path, err)
```

这样排查问题时知道错误发生在哪一步。

### 26.3 误区：channel 能替代所有锁

channel 适合表达所有权转移、任务队列和事件通知。

锁适合保护共享状态。

如果只是保护一个 map，Mutex 可能比 channel 更简单。

### 26.4 误区：启动 goroutine 后就不用管

每个 goroutine 都要有退出路径。

启动时问：

```text
它什么时候结束？
谁通知它结束？
如果下游没人接收，它会不会阻塞？
请求取消后它会不会泄漏？
```

### 26.5 误区：interface 越抽象越好

Go 接口应该小。

不要为了“架构感”提前设计巨大接口。

先写具体代码，再从使用方抽出真正需要的接口。

### 26.6 误区：nil 都一样

Go 中 nil slice、nil map、nil channel 行为不同。

```text
nil slice 可以 append；
nil map 写入会 panic；
nil channel 发送和接收会永久阻塞。
```

必须区分。

## 27. 项目练习路线

### 27.1 第一阶段：命令行文本工具

项目：词频统计器。

要求：

```text
读取文件路径参数；
逐行读取文本；
统计单词频率；
输出 Top N；
文件错误返回清晰信息；
核心逻辑有测试。
```

训练：

```text
os、bufio、strings、map、slice、sort、error、testing
```

验收标准：

```text
go test ./... 通过；
没有忽略错误；
函数参数清晰；
输出格式稳定。
```

### 27.2 第二阶段：HTTP JSON API

项目：任务管理 API。

要求：

```text
POST /tasks 创建任务；
GET /tasks 列出任务；
PATCH /tasks/{id} 完成任务；
DELETE /tasks/{id} 删除任务；
使用 JSON；
内存存储即可；
有单元测试和 HTTP handler 测试。
```

训练：

```text
net/http、encoding/json、struct tag、context、sync.Mutex、测试
```

### 27.3 第三阶段：并发爬取器

项目：URL 状态检查器。

要求：

```text
读取 URL 列表；
限制最大并发数；
每个请求有超时；
输出状态码、耗时和错误；
支持 Ctrl+C 取消；
无 goroutine 泄漏。
```

训练：

```text
goroutine、channel、context、WaitGroup、HTTP client、select
```

### 27.4 第四阶段：可观测服务

项目：带日志和 pprof 的 HTTP 服务。

要求：

```text
结构化日志；
健康检查；
优雅关闭；
pprof；
配置端口；
go test -race 通过。
```

训练：

```text
slog、signal、context、pprof、race detector、服务工程
```

### 27.5 第五阶段：泛型库

项目：泛型 LRU 或 Set。

要求：

```text
使用类型参数；
提供清晰 API；
有表格驱动测试；
有 benchmark；
文档示例可运行。
```

训练：

```text
generics、constraints、container/list、testing、benchmark、API 设计
```

### 27.6 熟练度复盘清单

每完成一个项目，检查：

```text
包名是否清楚？
错误是否有上下文？
是否有 goroutine 退出路径？
是否需要 context？
是否有数据竞争？
接口是否过大？
测试是否覆盖错误路径？
是否运行 go fmt、go test、go vet、go test -race？
```

能稳定做到这些，才算从 Go 语法入门走向 Go 工程熟练。

## 28. 官方资料

- [Go Documentation](https://go.dev/doc/)
- [A Tour of Go](https://go.dev/tour/)
- [Effective Go](https://go.dev/doc/effective_go)
- [Go by Example](https://gobyexample.com/)
- [Go Language Specification](https://go.dev/ref/spec)
- [Go Modules Reference](https://go.dev/ref/mod)
- [Go Memory Model](https://go.dev/ref/mem)
- [Go Standard Library](https://pkg.go.dev/std)
- [Go Blog](https://go.dev/blog/)
- [Go Testing Package](https://pkg.go.dev/testing)
- [Go pprof Package](https://pkg.go.dev/net/http/pprof)
