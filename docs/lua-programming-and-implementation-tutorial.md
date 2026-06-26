# Lua 编程语言教程：从零上手到熟练掌握与实现原理

> 目标读者：第一次系统学习 Lua 的初学者；也适合有 C/C++、Python、JavaScript、Go、Rust 等经验，但希望真正理解 Lua 语言模型、嵌入式脚本能力和底层实现原理的开发者。
> 学习目标：先能独立编写 Lua 脚本、配置文件、模块和小工具；再理解表、函数、闭包、元表、协程、错误处理和 C API；最后能从解释器、字节码、虚拟机、表实现、闭包、协程和垃圾回收角度理解 Lua 为什么轻量、灵活且适合嵌入。
> 说明：本文以 Lua 5.4 语义、官方参考手册和 PUC-Rio 官方 Lua 5.4.8 实现源码为主要依据做中文教程化整理。截至 2026 年，Lua 5.5 已发布；本文选择 Lua 5.4 作为基线，是因为它仍是大量工程、教材和嵌入式场景中常见的稳定版本。不同 Lua 版本和 LuaJIT 在语法、API、字节码和实现细节上存在差异，工程使用时应以对应版本官方文档和源码为准。

---

## 目录

0. [如何使用本文：从新手到熟练的学习路线](#0-如何使用本文从新手到熟练的学习路线)
1. [Lua 到底解决什么问题](#1-lua-到底解决什么问题)
2. [一句话理解 Lua](#2-一句话理解-lua)
3. [Lua 的设计哲学](#3-lua-的设计哲学)
4. [环境、工具链与第一个程序](#4-环境工具链与第一个程序)
5. [语法基础：chunk、语句、注释与作用域](#5-语法基础chunk语句注释与作用域)
6. [变量、nil、布尔值、数字与字符串](#6-变量nil布尔值数字与字符串)
7. [控制流：if、while、repeat、for](#7-控制流ifwhilerepeatfor)
8. [函数：一等公民、闭包、可变参数与多返回值](#8-函数一等公民闭包可变参数与多返回值)
9. [表 table：Lua 最核心的数据结构](#9-表-tablelua-最核心的数据结构)
10. [元表与元方法：让表拥有可定制行为](#10-元表与元方法让表拥有可定制行为)
11. [面向对象写法：原型、冒号语法与组合](#11-面向对象写法原型冒号语法与组合)
12. [模块系统：require、package 与工程组织](#12-模块系统requirepackage-与工程组织)
13. [错误处理：error、assert、pcall 与 xpcall](#13-错误处理errorassertpcall-与-xpcall)
14. [协程 coroutine：可暂停和恢复的控制流](#14-协程-coroutine可暂停和恢复的控制流)
15. [标准库学习路线](#15-标准库学习路线)
16. [C API：Lua 作为可嵌入语言的关键](#16-c-apilua-作为可嵌入语言的关键)
17. [实现总览：从源码到运行结果发生了什么](#17-实现总览从源码到运行结果发生了什么)
18. [值系统：TValue、类型标签与动态类型](#18-值系统tvalue类型标签与动态类型)
19. [表的实现原理：数组部分、哈希部分与重哈希](#19-表的实现原理数组部分哈希部分与重哈希)
20. [函数、闭包与 upvalue 的实现原理](#20-函数闭包与-upvalue-的实现原理)
21. [协程的实现原理：lua_State、栈与调度边界](#21-协程的实现原理luastate栈与调度边界)
22. [垃圾回收：可达性、屏障、增量与分代](#22-垃圾回收可达性屏障增量与分代)
23. [元表分派、环境与全局变量的底层理解](#23-元表分派环境与全局变量的底层理解)
24. [性能模型与调优方法](#24-性能模型与调优方法)
25. [常见误区](#25-常见误区)
26. [项目练习路线](#26-项目练习路线)
27. [官方资料](#27-官方资料)

---

## 0. 如何使用本文：从新手到熟练的学习路线

学习 Lua 不建议从“背语法表”开始。

更有效的路线是：

```text
能运行脚本 -> 能写函数和表 -> 能组织模块 -> 能用元表表达抽象 -> 能嵌入 C/C++ -> 能理解解释器实现
```

Lua 的语法很小，但它的表达力来自几个核心概念的组合：

```text
表 table
函数 function
闭包 closure
元表 metatable
协程 coroutine
C API
```

很多初学者觉得 Lua “太自由”，原因不是 Lua 难，而是它没有把对象、类、数组、字典、模块、迭代器这些概念做成很多互不相干的语法。Lua 更像给你几块小而强的积木，让你自己组合。

### 0.1 五遍学习法

| 阶段 | 目标 | 应读章节 | 能力标准 |
| --- | --- | --- | --- |
| 第一遍：能写脚本 | 运行 Lua、掌握变量、控制流、函数 | 1、2、3、4、5、6、7、8 | 能写命令行脚本和简单配置 |
| 第二遍：能组织数据 | 掌握 table、迭代、数组、字典、记录 | 9、15 | 能用 table 表达嵌套配置和业务对象 |
| 第三遍：能做抽象 | 掌握元表、元方法、对象写法、模块 | 10、11、12、13 | 能写可复用模块和对象风格接口 |
| 第四遍：能控制流程 | 掌握协程、错误边界、资源关闭 | 13、14、15 | 能写生成器、状态机和可恢复流程 |
| 第五遍：理解实现 | 掌握 C API、虚拟机、值系统、GC | 16 到 24 | 能解释 Lua 为什么小、快、易嵌入 |

### 0.2 新手最应该先掌握什么

第一阶段优先掌握：

```text
lua script.lua
local / nil / true / false
number / string / table / function
if / while / repeat / for
function / return / ...
table 构造器 {}
pairs / ipairs
require / return module_table
pcall / error
```

这些足够写很多实用脚本。

第二阶段再重点掌握：

```text
metatable / __index / __newindex
冒号语法 object:method(x)
coroutine.create / coroutine.resume / coroutine.yield
C API 栈模型
垃圾回收和性能习惯
```

### 0.3 每学一个知识点都问四个问题

本文建议每学一个 Lua 概念都问：

```text
1. 我在真实代码里什么时候用它？
2. 它解决的本质问题是什么？
3. Lua 为什么用这么少的机制表达它？
4. 如果我要自己实现解释器，这个机制大概要如何落地？
```

例如学习 `table` 时，不要只记“表既能当数组也能当字典”。

应该继续追问：

```text
真实代码里：配置、对象、数组、map、模块都可以用 table 表达。
本质问题：程序需要把名字和值组织在一起。
Lua 设计上：用一个统一结构减少语言核心复杂度。
实现上：table 内部通常同时有数组部分和哈希部分，以兼顾整数下标和键值映射。
```

这类追问会把语法知识变成可迁移的编程能力。

### 0.4 熟练 Lua 的判断标准

一个人是否熟练 Lua，不只看会不会写 `if` 和 `for`。

更好的判断标准是：

```text
能否默认使用 local，避免意外污染全局环境；
能否选择合适的 table 形态表达数据；
能否解释 #t 在有空洞数组上的限制；
能否用闭包保存私有状态；
能否用 __index 设计清楚的原型链；
能否区分点调用和冒号调用；
能否用 pcall 划清错误边界；
能否用协程写生成器和协作式流程；
能否理解 Lua 与宿主 C/C++ 之间如何交换数据；
能否根据虚拟机、GC、table 实现理解性能问题。
```

## 1. Lua 到底解决什么问题

Lua 是一门轻量、可嵌入、可扩展的脚本语言。

它最核心的定位不是“取代 C/C++ 写所有系统”，而是：

```text
让一个复杂宿主程序拥有灵活可修改的脚本层。
```

宿主程序可以是游戏引擎、网关、数据库、编辑器、嵌入式设备、插件系统，也可以是普通应用。

### 1.1 Lua 适合什么场景

| 场景 | Lua 的价值 |
| --- | --- |
| 游戏脚本 | 策划和玩法逻辑可以快速迭代，不必每次重编译引擎 |
| 配置语言 | 配置可以包含表达式、函数和条件逻辑，比静态 JSON 更灵活 |
| 插件系统 | 主程序提供 API，插件用 Lua 编写，扩展成本低 |
| 嵌入式设备 | 语言核心小，便于裁剪和嵌入 |
| Nginx/OpenResty 脚本 | 在高性能服务中写请求处理逻辑 |
| Redis 脚本 | 把多个操作组合成服务端原子逻辑 |
| 文本处理和小工具 | 启动快、语法短、表结构灵活 |

Lua 经常出现在“底层系统 + 高层脚本”的架构里：

```text
C/C++ 负责性能、内存、线程、I/O、图形、网络等重工作；
Lua 负责规则、配置、流程、插件、热更新和业务策略。
```

这也是理解 Lua 的第一把钥匙：Lua 不是为了把所有能力都放进语言核心，而是为了和宿主系统配合。

### 1.2 Lua 不适合什么场景

Lua 不是所有问题的最佳选择。

典型不适合场景包括：

```text
需要大型静态类型系统约束的业务平台；
需要庞大官方标准库的一站式后端应用；
需要强 IDE 静态分析保障的大型多人代码库；
需要极致数值计算生态的机器学习任务；
```

当然，可以通过类型标注工具、代码规范、模块化和宿主系统补足一部分能力。但初学者应该先理解 Lua 的本来强项：轻量、嵌入、可扩展。

## 2. 一句话理解 Lua

Lua 可以理解为：

```text
Lua = 小语言核心 + 强 table + 一等函数 + 元表机制 + 协程 + C API
```

其中最重要的是：

```text
table 负责组织数据；
function 负责组织行为；
metatable 负责定制行为；
coroutine 负责定制控制流；
C API 负责连接宿主世界。
```

如果把 Lua 想成一栋房子：

```text
table 是砖；
function 是动作；
closure 是带状态的动作；
metatable 是规则；
coroutine 是可暂停的执行路径；
C API 是房子和外部城市之间的门。
```

初学者掌握 Lua 的关键，是不要把它当成“小号 Python”或“弱化 JavaScript”。Lua 的强大来自极少数机制的组合，而不是大量专门语法。

## 3. Lua 的设计哲学

Lua 的设计经常体现三个取舍：

```text
小而完整；
简单机制优先；
嵌入和扩展优先。
```

### 3.1 小而完整

Lua 语言核心很小。

它没有内建类系统、继承关键字、异常类层级、复杂模块语法、大量集合类型，也没有把所有工具放进标准库。

这不是缺陷，而是刻意设计：

```text
语言核心越小，越容易嵌入；
实现越小，越容易移植；
规则越少，宿主程序越容易控制脚本环境；
机制越通用，用户越能按场景组合。
```

例如 Lua 没有单独的数组、字典、对象、模块四套结构，而是用 `table` 覆盖它们。

代价是：你需要学习 table 的边界和习惯。

收益是：你学会一个核心机制，就能解决很多问题。

### 3.2 机制优先，而不是语法优先

Lua 的很多“高级功能”不是新语法，而是机制组合。

例如对象方法：

```lua
local player = {name = "Ada", hp = 100}

function player:damage(n)
  self.hp = self.hp - n
end

player:damage(30)
print(player.hp) -- 70
```

这里看起来像面向对象，但底层只是：

```text
table 存数据；
function 存行为；
冒号语法自动传入 self；
```

再加上元表的 `__index`，就可以实现原型继承。

Lua 的学习重点不是“这个关键字怎么背”，而是“少数机制如何组合出更多模型”。

### 3.3 嵌入优先

Lua 从设计上就非常重视 C API。

很多语言是先作为独立程序运行，再后来支持嵌入。Lua 则从一开始就把“宿主程序可控制脚本”放在重要位置。

这带来几个结果：

```text
Lua 状态可以由 C 创建和销毁；
C 可以加载脚本、调用 Lua 函数、读取 Lua 表；
Lua 可以调用 C 注册的函数；
宿主程序可以决定暴露哪些 API；
宿主程序可以限制库、内存和执行边界。
```

这就是 Lua 在游戏、插件、网关和嵌入式场景里长盛不衰的重要原因。

## 4. 环境、工具链与第一个程序

Lua 的工具链很轻。

通常你只需要：

```text
lua      运行解释器和脚本
luac     编译并查看字节码
luarocks 管理第三方包，可选
```

### 4.1 第一个程序

创建 `hello.lua`：

```lua
print("hello, Lua")
```

运行：

```bash
lua hello.lua
```

输出：

```text
hello, Lua
```

从实现角度看，这一步大概发生了：

```text
读取源码文本
词法分析：识别 print、字符串、括号等 token
语法分析：构建函数调用语句
生成 Lua 虚拟机指令
解释执行指令
从全局环境中找到 print 函数
调用 print，向标准输出写字符串
```

初学者先不用掌握所有细节，但要建立一个大图景：Lua 不是逐字符“猜着执行”，而是先把源码变成内部可执行表示，再由虚拟机运行。

### 4.2 交互式 REPL

直接运行：

```bash
lua
```

进入交互式环境后可以输入：

```lua
> 1 + 2
3
> print("Lua")
Lua
```

REPL 适合快速试验语法。

但真正学习时，建议尽早写 `.lua` 文件，因为模块、作用域和错误定位都需要文件组织。

### 4.3 用 luac 观察编译结果

`luac` 可以把 Lua 源码编译成字节码，也可以列出指令：

```bash
luac -l hello.lua
```

你不需要一开始读懂每条指令，但要知道：

```text
Lua 解释器执行的是虚拟机指令；
同一段源码会被编译成函数原型和指令序列；
理解字节码有助于理解局部变量、常量、函数调用和闭包成本。
```

## 5. 语法基础：chunk、语句、注释与作用域

Lua 程序由 chunk 组成。

一个文件、一个字符串片段、交互式输入的一段代码，都可以看成 chunk。

```lua
local x = 10
local y = 20
print(x + y)
```

这整个文件就是一个 chunk。

从实现角度看，chunk 通常会被编译成一个匿名函数，再执行这个函数。

这能解释一个重要事实：

```text
Lua 文件顶层也可以有 local 变量；
这些 local 变量并不是全局变量，而是这个 chunk 函数的局部变量。
```

### 5.1 注释

单行注释：

```lua
-- this is a comment
```

多行注释：

```lua
--[[
multi-line
comment
]]
```

注释不会进入语法树，也不会生成虚拟机指令。

### 5.2 语句之间不强制分号

Lua 通常不写分号：

```lua
local x = 1
local y = 2
print(x + y)
```

可以写分号，但不常见：

```lua
local x = 1; local y = 2; print(x + y)
```

工程代码里保持一行一个清晰语句即可。

### 5.3 local 是第一习惯

Lua 里赋值给未声明名字，默认会访问全局环境。

```lua
x = 10       -- 全局变量，除非环境被特别设置
local y = 20 -- 局部变量
```

新手应该建立习惯：

```text
能用 local 就用 local。
```

原因有三点：

```text
局部变量查找更直接；
不会意外污染全局环境；
模块内部状态更容易控制。
```

很多 Lua bug 来自拼写错误：

```lua
local count = 10
counnt = count + 1 -- 拼错后创建或修改了全局变量
```

在严肃项目里，通常会配合静态检查工具或受限环境来发现这种问题。

### 5.4 块作用域

`do ... end` 可以创建块：

```lua
do
  local x = 10
  print(x)
end

print(x) -- nil，外面看不到块内 local
```

`if`、`for`、`while` 等结构也会引入块。

作用域理解的关键是：

```text
local 的可见范围从声明之后开始，到所在块结束。
```

## 6. 变量、nil、布尔值、数字与字符串

Lua 是动态类型语言。

变量本身没有固定类型，值有类型。

```lua
local x = 10
x = "hello"
x = {name = "Lua"}
```

这表示变量 `x` 可以先引用数字，再引用字符串，再引用表。

实现上，Lua 值会带类型标签。虚拟机操作值时，会根据标签判断它是数字、字符串、表、函数还是其他类型。

### 6.1 nil：没有值

`nil` 表示没有有效值。

```lua
local x
print(x) -- nil
```

表字段赋值为 `nil`，等价于删除这个键：

```lua
local t = {name = "Ada"}
t.name = nil
print(t.name) -- nil
```

这带来一个重要习惯：

```text
如果你需要在表里明确保存“空但存在”，不要直接用 nil，可以使用 false、特殊哨兵值或包装结构。
```

### 6.2 false 和 nil 才是假

Lua 条件判断里只有 `false` 和 `nil` 为假。

其他值都是真，包括：

```text
0
""
{}
```

示例：

```lua
if 0 then
  print("0 is true in Lua")
end

if "" then
  print("empty string is true in Lua")
end
```

这和 C/C++、JavaScript、Python 都不同。

所以 Lua 里不要写：

```lua
if count then
  -- count 为 0 时仍然进入
end
```

如果你要判断正数，应明确写：

```lua
if count > 0 then
  -- ...
end
```

### 6.3 数字：整数与浮点数

Lua 5.4 的 `number` 可以包含整数子类型和浮点子类型，具体表示也可以通过编译配置调整。

常见构建中：

```text
整数通常是 64 位整数；
浮点数通常是双精度浮点数。
```

示例：

```lua
local a = 10
local b = 3.14
local c = a + b
print(c)
```

学习数字时要区分两类问题：

```text
整数问题：计数、下标、位运算；
浮点问题：近似表示、舍入误差、科学计算。
```

浮点数不要直接做精确相等判断：

```lua
local x = 0.1 + 0.2
print(x == 0.3) -- 可能不是你期待的结果
```

更稳妥：

```lua
local function nearly_equal(a, b, eps)
  eps = eps or 1e-9
  return math.abs(a - b) < eps
end
```

### 6.4 字符串：不可变值

Lua 字符串是不可变的。

```lua
local s = "hello"
local t = s .. " world"
print(s) -- hello
print(t) -- hello world
```

`..` 会产生新字符串。

如果在循环中大量拼接：

```lua
local s = ""
for i = 1, 10000 do
  s = s .. i
end
```

可能产生很多临时字符串。

更好的写法是用表收集，再一次拼接：

```lua
local parts = {}
for i = 1, 10000 do
  parts[#parts + 1] = tostring(i)
end
local s = table.concat(parts)
```

实现上，Lua 会管理字符串对象，并对短字符串做内部化处理。理解这一点有助于理解字符串比较为什么可以很快，也能理解过度创建字符串为什么仍然会增加 GC 压力。

## 7. 控制流：if、while、repeat、for

控制流的本质是：

```text
根据条件选择执行路径；
根据条件重复执行代码；
按序列或迭代器遍历数据。
```

### 7.1 if

```lua
local score = 86

if score >= 90 then
  print("A")
elseif score >= 80 then
  print("B")
else
  print("C")
end
```

Lua 使用 `elseif`，不是 `else if`。

判断条件不需要括号，但条件表达式后必须有 `then`。

### 7.2 while

```lua
local i = 1
while i <= 5 do
  print(i)
  i = i + 1
end
```

`while` 先判断，再执行。

适合“不知道循环次数，但知道继续条件”的情况。

### 7.3 repeat until

```lua
local line
repeat
  line = io.read()
until line == "" or line == nil
```

`repeat ... until` 先执行，再判断。

这意味着循环体至少执行一次。

注意 `until` 后面的条件为真时停止循环。

### 7.4 数值 for

```lua
for i = 1, 5 do
  print(i)
end
```

带步长：

```lua
for i = 10, 1, -2 do
  print(i)
end
```

数值 for 的三个表达式是：

```text
初值、终值、步长
```

循环变量是局部变量。

不要在循环体内依赖修改循环变量来控制循环，这会让代码难读，也容易和语言规则冲突。

### 7.5 泛型 for

遍历数组式 table：

```lua
local names = {"Ada", "Bob", "Cyd"}

for i, name in ipairs(names) do
  print(i, name)
end
```

遍历键值：

```lua
local user = {name = "Ada", age = 20}

for k, v in pairs(user) do
  print(k, v)
end
```

`ipairs` 按整数下标从 1 开始遍历，遇到第一个 `nil` 停止。

`pairs` 遍历所有键值，但顺序不应依赖。

理解这一点很重要：

```text
table 是映射结构；
pairs 的顺序不是插入顺序；
如果业务需要稳定顺序，应显式保存顺序数组并排序。
```

## 8. 函数：一等公民、闭包、可变参数与多返回值

Lua 的函数是一等值。

这表示函数可以：

```text
赋给变量；
作为参数传入；
作为返回值返回；
存在 table 里；
捕获外层局部变量形成闭包。
```

### 8.1 定义函数

常规写法：

```lua
local function add(a, b)
  return a + b
end
```

等价理解：

```lua
local add
add = function(a, b)
  return a + b
end
```

函数名只是变量，函数本身是值。

这就是为什么函数可以放进表：

```lua
local math_ops = {}

function math_ops.add(a, b)
  return a + b
end

print(math_ops.add(1, 2))
```

### 8.2 多返回值

Lua 函数可以返回多个值：

```lua
local function divmod(a, b)
  return a // b, a % b
end

local q, r = divmod(17, 5)
print(q, r) -- 3 2
```

多返回值很适合表达：

```text
结果 + 错误；
商 + 余数；
解析结果 + 剩余输入；
状态 + 附加信息。
```

例如：

```lua
local function find_user(id)
  if id == 1 then
    return {id = 1, name = "Ada"}, nil
  end
  return nil, "user not found"
end

local user, err = find_user(2)
if not user then
  print(err)
end
```

### 8.3 多返回值的截断和展开

多返回值在不同上下文里会被调整。

```lua
local function values()
  return 1, 2, 3
end

local a = values()
print(a) -- 1

local x, y, z = values()
print(x, y, z) -- 1 2 3
```

如果函数调用在参数列表最后，会展开多个值：

```lua
print("values:", values()) -- values: 1 2 3
```

如果不在最后，通常只保留第一个：

```lua
print(values(), "end") -- 1 end
```

这个规则是 Lua 新手常见坑。

### 8.4 可变参数

`...` 表示可变参数：

```lua
local function sum(...)
  local total = 0
  for _, v in ipairs({...}) do
    total = total + v
  end
  return total
end

print(sum(1, 2, 3, 4))
```

但 `{...}` 会把参数放入表，遇到中间的 `nil` 时需要小心。

更稳妥地保留参数数量：

```lua
local function pack(...)
  return {n = select("#", ...), ...}
end

local args = pack(1, nil, 3)
print(args.n) -- 3
```

`select` 可以读取可变参数：

```lua
local function second(...)
  return select(2, ...)
end
```

### 8.5 闭包：函数携带环境

闭包是 Lua 的关键能力。

示例：

```lua
local function new_counter()
  local n = 0

  return function()
    n = n + 1
    return n
  end
end

local c1 = new_counter()
print(c1()) -- 1
print(c1()) -- 2

local c2 = new_counter()
print(c2()) -- 1
```

理解方式：

```text
new_counter 调用结束后，局部变量 n 没有消失；
返回的内部函数仍然引用它；
这个被函数捕获并延长生命周期的变量称为 upvalue。
```

闭包常用于：

```text
保存私有状态；
构造迭代器；
实现回调；
做函数工厂；
封装缓存和限流器。
```

从实现角度看，闭包需要让某些局部变量从普通栈变量变成可被函数长期引用的对象。第 20 章会展开。

## 9. 表 table：Lua 最核心的数据结构

如果只能学好一个 Lua 概念，那就是 table。

Lua 里的 table 可以当：

```text
数组；
字典；
结构体；
对象；
模块；
集合；
命名空间；
原型链节点。
```

### 9.1 创建表

空表：

```lua
local t = {}
```

数组式：

```lua
local names = {"Ada", "Bob", "Cyd"}
print(names[1]) -- Ada
```

记录式：

```lua
local user = {
  name = "Ada",
  age = 20,
}

print(user.name)
print(user["name"])
```

混合式：

```lua
local config = {
  "read",
  "write",
  host = "127.0.0.1",
  port = 8080,
}
```

Lua 数组习惯从 1 开始。

这不是技术上不能用 0，而是 Lua 标准库和习惯都围绕 1 起始序列设计。

### 9.2 table 是引用语义

```lua
local a = {x = 1}
local b = a
b.x = 2
print(a.x) -- 2
```

变量 `a` 和 `b` 引用同一个表。

理解方式：

```text
变量不是盒子本身；
变量保存的是对 table 对象的引用；
赋值复制引用，不复制整张表。
```

如果需要复制，必须显式写：

```lua
local function shallow_copy(t)
  local r = {}
  for k, v in pairs(t) do
    r[k] = v
  end
  return r
end
```

浅拷贝只复制第一层。嵌套 table 仍然共享。

### 9.3 数组、序列和 # 运算符

没有空洞的连续正整数键部分，通常称为序列：

```lua
local t = {"a", "b", "c"}
print(#t) -- 3
```

有空洞时，不要依赖 `#` 的直觉结果：

```lua
local t = {"a", nil, "c"}
print(#t) -- 不应把结果当成稳定业务语义
```

原因是：

```text
Lua table 本质是键值映射；
nil 表示不存在；
有空洞时，“长度”不再是唯一明确概念。
```

工程建议：

```text
数组中间不要存 nil；
需要表示空值时使用 false 或专门哨兵；
需要准确元素数量时单独维护 n 字段；
```

### 9.4 table.insert 和 table.remove

```lua
local t = {}
table.insert(t, "a")
table.insert(t, "b")
table.insert(t, 1, "start")

print(table.concat(t, ","))
```

`table.remove(t, i)` 会移除并移动后续元素。

如果在数组头部频繁插入或删除，会产生大量搬移。

更好的结构可能是：

```text
双端队列；
两个索引维护 head/tail；
或宿主语言提供的数据结构。
```

### 9.5 字典与集合

字典：

```lua
local scores = {
  Ada = 95,
  Bob = 88,
}

print(scores["Ada"])
```

集合：

```lua
local seen = {}
seen["task-1"] = true

if seen["task-1"] then
  print("already seen")
end
```

删除键：

```lua
seen["task-1"] = nil
```

### 9.6 table 作为模块

```lua
local M = {}

function M.add(a, b)
  return a + b
end

return M
```

调用者：

```lua
local mathx = require("mathx")
print(mathx.add(1, 2))
```

这个模式的本质：

```text
模块就是返回一个 table；
table 中的字段就是模块暴露的函数和常量；
没有放进返回表的 local 内容就是模块私有实现。
```

### 9.7 table 的思维模型

看到 table 时，先问：

```text
它是数组吗？键是否连续整数？
它是字典吗？键和值分别是什么类型？
它是记录吗？字段集合是否固定？
它是对象吗？是否带方法和元表？
它是模块吗？是否只暴露公共 API？
它是否需要稳定顺序？
它是否允许 nil？
```

这些问题比“table 语法怎么写”更重要。

## 10. 元表与元方法：让表拥有可定制行为

元表是 Lua 最有代表性的机制之一。

简单说：

```text
普通 table 保存数据；
元表保存这个 table 在特定操作下的行为规则。
```

元表本身也是 table。

### 10.1 setmetatable 和 getmetatable

```lua
local t = {}
local mt = {}

setmetatable(t, mt)
print(getmetatable(t) == mt) -- true
```

元表不会自动改变所有行为，只有当某些操作触发对应元方法时才起作用。

### 10.2 __index：字段找不到时怎么办

`__index` 是最常用的元方法。

```lua
local defaults = {
  timeout = 5,
  retries = 3,
}

local config = {
  timeout = 10,
}

setmetatable(config, {__index = defaults})

print(config.timeout) -- 10，自己有
print(config.retries) -- 3，自己没有，去 defaults 找
```

思维过程：

```text
访问 config.retries；
config 表中没有 retries；
Lua 查 config 的元表；
发现 __index 指向 defaults；
于是去 defaults.retries 找；
```

这就是 Lua 原型式继承的基础。

`__index` 也可以是函数：

```lua
local t = {}

setmetatable(t, {
  __index = function(_, key)
    return "missing:" .. tostring(key)
  end,
})

print(t.name) -- missing:name
```

函数形式适合做懒加载、计算字段、代理对象。

### 10.3 __newindex：写入不存在字段时怎么办

```lua
local data = {}
local proxy = {}

setmetatable(proxy, {
  __newindex = function(_, key, value)
    print("write", key, value)
    data[key] = value
  end,
  __index = data,
})

proxy.name = "Ada"
print(proxy.name)
```

`__newindex` 只在普通写入没有命中原表已有键时触发。

如果表里已经有这个键，直接写入，不触发元方法。

这常用于：

```text
只读表；
数据校验；
代理；
观察者；
配置写入日志。
```

### 10.4 运算符元方法

可以让表支持加法、相等、字符串化等行为。

```lua
local Vec = {}
Vec.__index = Vec

function Vec.new(x, y)
  return setmetatable({x = x, y = y}, Vec)
end

function Vec.__add(a, b)
  return Vec.new(a.x + b.x, a.y + b.y)
end

function Vec.__tostring(v)
  return "(" .. v.x .. ", " .. v.y .. ")"
end

local a = Vec.new(1, 2)
local b = Vec.new(3, 4)
print(a + b) -- (4, 6)
```

常见元方法包括：

| 元方法 | 触发场景 |
| --- | --- |
| `__index` | 读取不存在字段 |
| `__newindex` | 写入不存在字段 |
| `__add`、`__sub`、`__mul` | 算术运算 |
| `__eq`、`__lt`、`__le` | 比较 |
| `__len` | `#` 运算 |
| `__call` | 把值当函数调用 |
| `__tostring` | 字符串转换 |
| `__pairs` | `pairs` 遍历 |
| `__close` | to-be-closed 变量关闭 |

### 10.5 元表不是魔法

初学者容易把元表当“神秘黑盒”。

更准确的理解是：

```text
Lua 执行某个操作；
普通路径无法完成，或语义规定要检查元方法；
虚拟机读取对象元表；
查找固定名称的元方法；
调用这个函数或按这个表继续查找；
```

元表的强大来自“操作可拦截”，但它也会让代码行为不直观。

工程建议：

```text
公共模块可以使用元表封装复杂性；
业务代码不要滥用元表制造隐式行为；
__index 原型链应尽量简单；
调试困难时先检查 getmetatable 和 rawget。
```

## 11. 面向对象写法：原型、冒号语法与组合

Lua 没有内建 class 关键字，但可以用 table、函数和元表实现面向对象风格。

### 11.1 冒号语法

```lua
local player = {name = "Ada", hp = 100}

function player.damage(self, n)
  self.hp = self.hp - n
end

player.damage(player, 10)
```

上面可以写成：

```lua
function player:damage(n)
  self.hp = self.hp - n
end

player:damage(10)
```

冒号只是语法糖：

```text
obj:method(x) 等价于 obj.method(obj, x)
function obj:method(x) 等价于 function obj.method(self, x)
```

很多 Lua 对象 bug 来自点和冒号混用。

如果方法内部用 `self`，调用时通常应该用冒号。

### 11.2 原型对象

```lua
local Player = {}
Player.__index = Player

function Player.new(name)
  local self = {
    name = name,
    hp = 100,
  }
  return setmetatable(self, Player)
end

function Player:damage(n)
  self.hp = math.max(0, self.hp - n)
end

function Player:is_alive()
  return self.hp > 0
end

local p = Player.new("Ada")
p:damage(20)
print(p.hp, p:is_alive())
```

这段代码的本质：

```text
每个实例是一个普通 table；
Player 表保存共享方法；
实例找不到 damage 时，通过 __index 到 Player 里找；
冒号调用把实例作为 self 传入；
```

这不是传统类系统，而是原型模型。

优点：

```text
机制简单；
实例字段和方法查找规则透明；
可以灵活组合和替换原型；
```

代价：

```text
缺少编译期类型检查；
继承层次过深时调试困难；
命名和约定很重要。
```

### 11.3 私有状态：闭包方案

如果想隐藏内部状态，可以用闭包：

```lua
local function new_account(initial)
  local balance = initial or 0

  return {
    deposit = function(n)
      balance = balance + n
    end,
    withdraw = function(n)
      if n > balance then
        return false, "insufficient funds"
      end
      balance = balance - n
      return true
    end,
    balance = function()
      return balance
    end,
  }
end

local acc = new_account(100)
acc.deposit(50)
print(acc.balance())
```

这里 `balance` 不是返回表的字段，外部不能直接访问。

代价是每个对象都会创建一组函数闭包，实例很多时可能增加内存开销。

### 11.4 组合优先

Lua 里不建议一开始就设计复杂继承。

更稳妥的是组合：

```lua
local function new_health(hp)
  return {
    hp = hp,
    damage = function(self, n)
      self.hp = math.max(0, self.hp - n)
    end,
  }
end

local enemy = {
  name = "slime",
  health = new_health(30),
}

enemy.health:damage(10)
```

如果继承关系超过两层，先问：

```text
是否可以拆成组件？
是否可以用委托？
是否可以用函数组合？
是否真的需要共享状态和行为？
```

## 12. 模块系统：require、package 与工程组织

Lua 模块通常就是一个返回 table 的文件。

### 12.1 最常见模块写法

`mathx.lua`：

```lua
local M = {}

local function check_number(x)
  assert(type(x) == "number", "number expected")
end

function M.square(x)
  check_number(x)
  return x * x
end

function M.clamp(x, low, high)
  check_number(x)
  if x < low then return low end
  if x > high then return high end
  return x
end

return M
```

使用：

```lua
local mathx = require("mathx")
print(mathx.square(9))
```

设计原则：

```text
local 函数是私有实现；
M 表上的字段是公开 API；
文件最后 return M；
调用方用 local 接收 require 结果；
```

### 12.2 require 做了什么

概念上，`require("mathx")` 会：

```text
检查模块是否已经加载；
如果加载过，直接返回缓存结果；
根据 package.path 搜索 Lua 文件；
根据 package.cpath 搜索 C 模块；
加载并执行模块 chunk；
保存模块返回值；
返回该模块；
```

因此模块初始化只应做必要工作。

如果模块加载时产生大量副作用，会让测试和调试变复杂。

### 12.3 package.path

`package.path` 决定 Lua 文件搜索路径。

可以查看：

```lua
print(package.path)
```

路径中 `?` 会被模块名替换。

例如 `require("a.b")` 可能尝试：

```text
a/b.lua
a/b/init.lua
```

具体取决于当前 `package.path`。

### 12.4 模块设计建议

好的 Lua 模块通常具备：

```text
清晰的返回表；
少量稳定的公开函数；
内部状态尽量 local；
初始化副作用少；
错误信息明确；
依赖通过 require 显式写出；
```

目录示例：

```text
project/
  main.lua
  app/
    config.lua
    router.lua
    service.lua
  lib/
    mathx.lua
```

`main.lua`：

```lua
package.path = "./?.lua;./?/init.lua;" .. package.path

local config = require("app.config")
local service = require("app.service")

service.start(config)
```

小项目可以简单组织，大项目应制定模块命名、依赖方向和错误处理规范。

## 13. 错误处理：error、assert、pcall 与 xpcall

Lua 的错误处理不是 Java/C++ 那种类层级异常模型。

更常见的模式是：

```text
普通函数用 nil, err 表达可预期失败；
严重错误用 error 抛出；
边界处用 pcall 捕获；
需要 traceback 时用 xpcall。
```

### 13.1 assert

```lua
local function divide(a, b)
  assert(b ~= 0, "division by zero")
  return a / b
end
```

`assert(cond, msg)` 在条件为假或 nil 时抛出错误。

它适合检查程序员错误或不应继续运行的前置条件。

不要用 `assert` 处理所有业务失败。

例如用户输入错误通常应该返回错误信息，而不是让整个程序崩掉。

### 13.2 error

```lua
local function parse_port(s)
  local n = tonumber(s)
  if not n then
    error("invalid port: " .. tostring(s))
  end
  return n
end
```

`error` 会中断当前执行路径，沿调用栈向外传播，直到被保护调用捕获或终止程序。

### 13.3 pcall

```lua
local ok, result = pcall(function()
  return parse_port("abc")
end)

if not ok then
  print("failed:", result)
else
  print(result)
end
```

`pcall` 返回：

```text
true, 函数返回值...
false, 错误对象
```

适合在插件边界、用户脚本边界、任务调度边界捕获错误。

### 13.4 xpcall

`xpcall` 可以指定错误处理函数，常用于附加 traceback：

```lua
local function handler(err)
  return debug.traceback("error: " .. tostring(err), 2)
end

local ok, result = xpcall(function()
  error("boom")
end, handler)

if not ok then
  print(result)
end
```

### 13.5 什么时候返回 nil, err

可预期失败更适合返回：

```lua
local function read_config(path)
  local f, err = io.open(path, "r")
  if not f then
    return nil, err
  end

  local content = f:read("*a")
  f:close()
  return content
end
```

调用方：

```lua
local content, err = read_config("app.conf")
if not content then
  print("read failed:", err)
end
```

判断原则：

| 情况 | 建议 |
| --- | --- |
| 文件不存在、输入非法、网络失败 | 返回 `nil, err` |
| 程序内部不变量被破坏 | `error` 或 `assert` |
| 插件、脚本、任务边界 | `pcall` 或 `xpcall` |
| 需要资源释放 | 使用显式关闭或 Lua 5.4 的 to-be-closed 变量 |

## 14. 协程 coroutine：可暂停和恢复的控制流

协程是 Lua 非常重要的控制流机制。

它不是操作系统线程。

更准确地说：

```text
协程是一段可以主动暂停、以后再从暂停点继续执行的函数调用栈。
```

### 14.1 最小示例

```lua
local co = coroutine.create(function()
  print("A")
  coroutine.yield()
  print("B")
end)

print(coroutine.status(co)) -- suspended
coroutine.resume(co)        -- A
print(coroutine.status(co)) -- suspended
coroutine.resume(co)        -- B
print(coroutine.status(co)) -- dead
```

执行过程：

```text
create 创建协程，但不执行；
第一次 resume 从函数开头执行到 yield；
yield 把控制权交回调用者；
第二次 resume 从 yield 后面继续执行；
函数返回后协程 dead；
```

### 14.2 resume 和 yield 传值

```lua
local co = coroutine.create(function(a, b)
  print("start", a, b)
  local x = coroutine.yield("paused")
  print("resumed with", x)
  return "done"
end)

print(coroutine.resume(co, 1, 2))      -- true start... paused
print(coroutine.resume(co, "hello"))   -- true resumed... done
```

理解规则：

```text
第一次 resume 的参数传给协程主函数；
yield 的参数返回给 resume 调用者；
下一次 resume 的参数成为 yield 表达式的返回值；
协程函数 return 的值返回给最后一次 resume；
```

### 14.3 用协程写生成器

```lua
local function range(n)
  return coroutine.wrap(function()
    for i = 1, n do
      coroutine.yield(i)
    end
  end)
end

for value in range(3) do
  print(value)
end
```

`coroutine.wrap` 返回一个函数，每次调用会 resume 协程。

生成器的本质：

```text
生产者不需要一次性生成所有数据；
每次 yield 一个值；
消费者按需恢复生产者；
```

### 14.4 协程不是并行

协程不会自动在多核 CPU 上并行。

它是协作式调度：

```text
只有当前协程主动 yield，其他流程才有机会继续；
如果协程里死循环不 yield，整个调度会被卡住；
```

真正并行通常需要宿主程序、线程库、事件循环或外部运行时配合。

### 14.5 协程适合什么

| 场景 | 原因 |
| --- | --- |
| 生成器 | 保留局部状态，按需产出 |
| 状态机 | 每个阶段可以写成顺序代码 |
| 游戏脚本 | 等待几帧、等待事件、等待动画 |
| 异步流程封装 | 把回调风格改成顺序风格 |
| 协作式任务 | 调度器控制 resume/yield |

协程的核心价值，是把复杂控制流写成更接近人脑顺序理解的代码。

## 15. 标准库学习路线

Lua 标准库不庞大，但常用部分必须熟。

### 15.1 基础函数

常用基础函数：

```text
print
type
tostring
tonumber
assert
error
pcall
xpcall
next
pairs
ipairs
select
rawget
rawset
rawequal
setmetatable
getmetatable
```

学习重点不是背名字，而是理解它们操作的层级：

```text
type/tostring/tonumber 处理值；
pairs/ipairs/next 处理遍历；
rawget/rawset 绕过元表；
pcall/xpcall 处理错误边界；
setmetatable/getmetatable 处理行为规则；
```

### 15.2 string 库

```lua
local s = "hello.lua"

print(string.upper(s))
print(string.sub(s, 1, 5))
print(string.find(s, "%.lua$"))
print((s:gsub("%.lua$", ".txt")))
```

Lua 的模式匹配不是完整正则表达式。

它是更小的 pattern 语言。

优点：

```text
实现轻量；
够用且快速；
适合嵌入式场景；
```

缺点：

```text
复杂正则特性不完整；
从其他语言迁移时容易误以为完全相同；
```

### 15.3 table 库

常用：

```text
table.insert
table.remove
table.concat
table.sort
table.move
table.pack
table.unpack
```

排序示例：

```lua
local users = {
  {name = "Ada", score = 95},
  {name = "Bob", score = 88},
  {name = "Cyd", score = 91},
}

table.sort(users, function(a, b)
  return a.score > b.score
end)
```

注意比较函数必须形成稳定的严格弱序关系。不要写随机比较、状态变化比较或前后矛盾的比较。

### 15.4 math 库

```lua
print(math.floor(3.8))
print(math.ceil(3.2))
print(math.max(1, 9, 3))
print(math.random())
```

数值程序要关注：

```text
整数和浮点的边界；
随机数种子；
舍入误差；
平台差异；
```

### 15.5 io 和 os

读取文件：

```lua
local f, err = io.open("input.txt", "r")
if not f then
  error(err)
end

local content = f:read("*a")
f:close()
print(content)
```

写文件：

```lua
local f = assert(io.open("output.txt", "w"))
f:write("hello\n")
f:close()
```

工程建议：

```text
打开文件后尽快明确关闭；
错误信息要带路径；
不要把用户输入直接拼进危险 shell 命令；
嵌入场景下宿主程序可能禁用 io/os 库；
```

### 15.6 debug 库

`debug` 库可以检查栈、upvalue、traceback 等。

它适合调试工具和框架，不适合普通业务逻辑滥用。

原因：

```text
debug 库可能破坏封装；
可能影响性能；
宿主环境可能禁用；
不同版本细节可能变化；
```

## 16. C API：Lua 作为可嵌入语言的关键

Lua 的 C API 是它区别于很多脚本语言的核心优势。

理解 C API 时，先抓住一个模型：

```text
C 和 Lua 通过一组虚拟栈交换数据。
```

C 代码不直接拿 Lua 对象内部指针随便操作，而是通过 API 在栈上压入、读取、调用和弹出值。

### 16.1 最小嵌入示例

```c
#include <stdio.h>

#include <lua.h>
#include <lauxlib.h>
#include <lualib.h>

int main(void) {
  lua_State *L = luaL_newstate();
  luaL_openlibs(L);

  if (luaL_dofile(L, "script.lua") != LUA_OK) {
    const char *msg = lua_tostring(L, -1);
    fprintf(stderr, "lua error: %s\n", msg);
    lua_pop(L, 1);
  }

  lua_close(L);
  return 0;
}
```

这个程序做了：

```text
创建 Lua 状态；
打开标准库；
执行脚本文件；
如果出错，从栈顶取错误信息；
关闭 Lua 状态并释放资源；
```

### 16.2 栈索引

Lua C API 的栈可以用正数或负数索引：

```text
1 表示栈底第一个值；
2 表示第二个值；
-1 表示栈顶；
-2 表示栈顶下面一个；
```

示例：

```c
lua_pushnumber(L, 10);
lua_pushnumber(L, 20);

double b = lua_tonumber(L, -1);
double a = lua_tonumber(L, -2);
```

学习 C API 时必须养成习惯：

```text
每个函数调用前后，栈上有什么值？
错误路径是否清理了栈？
返回给 Lua 的值压了几个？
```

### 16.3 注册 C 函数给 Lua 调用

```c
static int l_add(lua_State *L) {
  double a = luaL_checknumber(L, 1);
  double b = luaL_checknumber(L, 2);
  lua_pushnumber(L, a + b);
  return 1;
}

int luaopen_mymath(lua_State *L) {
  lua_newtable(L);
  lua_pushcfunction(L, l_add);
  lua_setfield(L, -2, "add");
  return 1;
}
```

Lua 使用：

```lua
local mymath = require("mymath")
print(mymath.add(1, 2))
```

C 函数返回值是一个整数，表示压回 Lua 的返回值数量。

### 16.4 从 C 调 Lua 函数

Lua：

```lua
function on_event(name, value)
  print("event", name, value)
end
```

C 侧概念流程：

```c
lua_getglobal(L, "on_event");
lua_pushstring(L, "score");
lua_pushinteger(L, 100);

if (lua_pcall(L, 2, 0, 0) != LUA_OK) {
  const char *msg = lua_tostring(L, -1);
  fprintf(stderr, "callback failed: %s\n", msg);
  lua_pop(L, 1);
}
```

含义：

```text
把 Lua 函数压栈；
压入两个参数；
保护调用；
期望 0 个返回值；
错误时错误对象在栈顶；
```

### 16.5 C API 的设计原则

Lua C API 的好处：

```text
ABI 边界清晰；
宿主可控制状态生命周期；
Lua 值通过栈传递，避免暴露内部结构；
C 函数天然可被 Lua 调用；
Lua 函数也可被 C 调用；
```

代价：

```text
栈平衡需要程序员负责；
错误处理必须谨慎；
C 侧生命周期和 Lua GC 要协同；
跨语言调试更复杂；
```

熟练使用 C API 的关键不是背函数，而是画清楚调用前后的栈。

## 17. 实现总览：从源码到运行结果发生了什么

PUC-Rio Lua 官方实现是理解 Lua 的最好材料之一。

它整体结构非常紧凑，适合学习解释器。

### 17.1 运行流水线

概念上，执行一个 Lua 文件会经历：

```text
源码文本
  -> 词法分析 lexer
  -> 语法分析 parser
  -> 代码生成 code generator
  -> 函数原型 Proto
  -> 虚拟机指令 bytecode
  -> 解释器循环 VM
  -> 调用标准库或 C 函数
```

对应源码文件大致包括：

| 模块 | 作用 |
| --- | --- |
| `llex.c` | 词法分析 |
| `lparser.c` | 语法分析 |
| `lcode.c` | 生成虚拟机指令 |
| `lopcodes.h` | 指令格式和 opcode |
| `lvm.c` | 虚拟机执行 |
| `ldo.c` | 调用、保护调用、栈增长等执行控制 |
| `lstate.c` / `lstate.h` | Lua 状态、线程、调用栈 |
| `lobject.h` | 值表示、对象结构 |
| `ltable.c` | table 实现 |
| `lstring.c` | 字符串管理 |
| `lfunc.c` | 函数和闭包 |
| `lgc.c` | 垃圾回收 |
| `lapi.c` | C API 实现 |

不用一开始逐行读源码。

更好的阅读顺序是：

```text
lobject.h 看值和对象；
ltable.c 看 table；
lvm.c 看虚拟机如何执行指令；
lapi.c 看 C API 如何包装内部操作；
lgc.c 看对象生命周期；
```

### 17.2 虚拟机为什么存在

Lua 不是把每条源码语句直接解释成 C 分支。

它会先编译成更紧凑的虚拟机指令。

好处：

```text
语法解析只做一次；
执行阶段面对统一指令格式；
函数、闭包、常量、局部变量都能组织进 Proto；
可以用 luac 保存或查看字节码；
虚拟机实现可移植；
```

理解虚拟机后，很多语言现象会更清楚：

```text
local 变量为什么通常比全局变量快；
闭包为什么需要 upvalue；
函数调用为什么有栈帧；
table 字段访问为什么可能触发元方法；
```

### 17.3 Lua 实现的核心思路

可以把 Lua 实现看成几层：

```text
值层：TValue 表示各种动态类型值；
对象层：字符串、table、函数、userdata、thread 等 GC 对象；
执行层：栈、调用帧、虚拟机指令；
语义层：元表、闭包、协程、错误处理；
接口层：C API 把内部能力安全地暴露给宿主；
内存层：垃圾回收管理对象生命周期；
```

初学者理解实现时，不要被 C 宏和细节吓住。

先问：

```text
这个结构在表达哪条语言规则？
这个字段是为了执行、查找、GC 还是调试？
这个函数是在普通路径、错误路径还是元方法路径上？
```

## 18. 值系统：TValue、类型标签与动态类型

Lua 是动态类型语言，运行时值必须携带类型信息。

例如：

```lua
local x = 1
x = "hello"
x = {}
```

同一个变量可以保存不同类型的值，因此解释器不能只看变量名判断类型。

### 18.1 值与对象的区别

概念上可以分两类：

```text
直接值：nil、boolean、number 等；
引用到 GC 对象的值：string、table、function、userdata、thread 等；
```

实际实现会使用统一的值结构保存：

```text
类型标签；
具体数据；
```

这样虚拟机执行加法、索引、调用等操作时，可以先检查类型标签，再进入对应路径。

### 18.2 为什么需要类型标签

考虑：

```lua
local a = 1
local b = "2"
print(a + b)
```

虚拟机执行 `a + b` 时必须知道：

```text
a 是整数还是浮点数？
b 是字符串吗？
是否允许转换？
是否存在 __add 元方法？
失败时如何报错？
```

类型标签就是运行时决策的入口。

动态语言的灵活性，背后通常都需要运行时类型描述来支撑。

### 18.3 GC 对象

字符串、table、闭包、userdata、thread 等对象需要由垃圾回收管理。

它们通常会带有：

```text
对象类型；
GC 标记；
链表指针；
对象自身数据；
```

GC 标记用于判断对象是否可达、是否已访问、是否需要清理。

这也是为什么实现里经常会看到“值结构”和“对象头”分离：一个表达“某个位置上是什么值”，一个表达“堆上对象如何被管理”。

## 19. 表的实现原理：数组部分、哈希部分与重哈希

Lua table 的设计很精巧。

语言层面上，table 是统一的键值映射。

实现层面上，为了兼顾数组和字典，它通常同时包含：

```text
数组部分 array part；
哈希部分 hash part；
```

### 19.1 为什么要分数组和哈希

考虑：

```lua
local t = {10, 20, 30}
print(t[1], t[2], t[3])
```

如果每个整数下标都走通用哈希表，会浪费空间和时间。

数组部分可以让连续整数键访问更快、更紧凑。

再考虑：

```lua
local user = {name = "Ada", age = 20}
```

字符串键不适合放数组部分，需要哈希部分。

因此 Lua table 的实现策略是：

```text
适合数组的正整数键尽量放数组部分；
其他键放哈希部分；
必要时重新计算布局并搬迁元素；
```

### 19.2 数组部分

数组部分适合键为 `1..n` 的正整数。

```lua
local t = {}
for i = 1, 1000 do
  t[i] = i
end
```

这类 table 通常会得到比较高效的存储。

工程建议：

```text
数组从 1 开始；
尽量保持连续；
避免中间大量 nil；
尾部追加用 t[#t + 1] = value 或 table.insert；
```

### 19.3 哈希部分

哈希部分处理非数组键：

```lua
local t = {}
t.name = "Ada"
t[true] = "yes"
t[{}] = "object key"
```

Lua table 的键可以是很多类型，但 `nil` 不能作为键。

浮点 NaN 也不能作为 table 键，因为它不等于自身，会破坏查找语义。

### 19.4 重哈希

当插入新键导致容量或布局不合适时，table 需要重新组织。

重哈希大概会做：

```text
统计整数键分布；
决定数组部分大小；
分配新的数组和哈希区域；
把旧元素迁移到新位置；
```

这解释了一个性能现象：

```text
单次 table 插入通常很快；
但某些插入会触发扩容或重哈希，成本明显更高；
```

因此在性能敏感路径里，应避免无意义地频繁创建和扩张 table。

如果知道大致大小，可以通过宿主侧、库函数或复用策略减少分配压力。

### 19.5 rawget 和元表路径

普通字段访问可能触发元表：

```lua
local v = t.key
```

`rawget` 绕过元表：

```lua
local v = rawget(t, "key")
```

理解区别：

```text
普通访问表达语言语义；
rawget/rawset 表达直接 table 存取；
```

实现和调试元表逻辑时，经常需要 rawget/rawset 避免递归触发 `__index` 或 `__newindex`。

## 20. 函数、闭包与 upvalue 的实现原理

Lua 函数看起来简单：

```lua
local function add(a, b)
  return a + b
end
```

实现上至少涉及：

```text
函数原型 Proto；
闭包 Closure；
上值 Upvalue；
调用栈 CallInfo；
```

### 20.1 Proto：函数的静态描述

源码中的函数会被编译成函数原型。

Proto 可以理解为：

```text
指令列表；
常量表；
嵌套函数原型；
局部变量调试信息；
upvalue 描述；
参数数量和可变参数信息；
```

同一个函数原型可以创建多个闭包。

例如：

```lua
local function make_adder(n)
  return function(x)
    return x + n
  end
end

local add1 = make_adder(1)
local add10 = make_adder(10)
```

内部匿名函数的代码形状相同，但捕获的 `n` 不同。

因此需要区分：

```text
Proto：代码模板；
Closure：带运行时环境的函数值；
```

### 20.2 upvalue：被捕获的局部变量

```lua
local function outer()
  local n = 0
  return function()
    n = n + 1
    return n
  end
end
```

`n` 本来是 `outer` 栈帧里的局部变量。

但内部函数返回后仍要访问 `n`。

实现必须解决：

```text
outer 返回后栈帧会消失；
内部闭包仍然需要 n；
多个闭包可能共享同一个 n；
```

典型思路是：

```text
当外层函数还在运行时，upvalue 可以指向栈上的局部变量；
当外层函数即将返回时，把仍被引用的变量关闭到堆上；
闭包之后访问堆上的 upvalue；
```

这就是“open upvalue”和“closed upvalue”的核心思想。

### 20.3 多个闭包共享 upvalue

```lua
local function pair()
  local n = 0

  local function inc()
    n = n + 1
    return n
  end

  local function get()
    return n
  end

  return inc, get
end

local inc, get = pair()
inc()
inc()
print(get()) -- 2
```

`inc` 和 `get` 共享同一个 `n`。

这说明 upvalue 不是简单地把值复制进每个函数。

如果复制，`inc` 修改后 `get` 看不到。

正确模型是：

```text
多个闭包引用同一个 upvalue 容器；
容器里保存当前值；
```

### 20.4 闭包的工程含义

闭包强大，但也要注意：

```text
闭包会延长被捕获变量的生命周期；
捕获大 table 可能让内存迟迟不能释放；
循环中创建闭包要确认捕获的是不是你想要的变量；
```

例如：

```lua
local callbacks = {}

for i = 1, 3 do
  callbacks[i] = function()
    return i
  end
end
```

理解闭包捕获规则后，才能判断这里每个函数返回什么，以及不同 Lua 版本或写法是否符合预期。

## 21. 协程的实现原理：lua_State、栈与调度边界

协程在 Lua 中对应独立的执行状态。

在 C API 里，你经常会看到 `lua_State *L`。

初学者容易以为一个 `lua_State` 就只表示整个虚拟机。

更准确地说：

```text
lua_State 表示一个 Lua 线程，也就是一条可执行控制流；
多个 Lua 线程可以共享同一个全局状态；
协程就是 Lua 线程对象；
```

### 21.1 协程为什么能暂停

普通函数调用栈执行到一半时，通常不能随便保存再恢复。

Lua 协程能做到，是因为解释器维护了：

```text
当前执行栈；
调用帧信息；
程序计数位置；
协程状态；
yield/resume 边界；
```

当 `yield` 发生时：

```text
当前协程保存栈和调用状态；
控制权返回 resume 的调用者；
协程状态变为 suspended；
```

下一次 `resume`：

```text
把参数传回 yield 点；
从之前保存的位置继续执行；
```

### 21.2 协程和 C 调用边界

Lua 协程与 C 函数交互时要关注 yield 边界。

不是所有 C 调用点都可以无条件 yield。

原因是：

```text
C 调用栈由宿主语言管理；
Lua 虚拟机可以保存自己的调用帧；
跨 C 边界暂停需要额外约定；
```

所以写可 yield 的 C API 或在嵌入场景里使用协程时，需要认真查对应版本手册。

### 21.3 协程调度器的基本形状

一个极简调度器可以是：

```lua
local tasks = {}

local function spawn(fn)
  tasks[#tasks + 1] = coroutine.create(fn)
end

local function run()
  while #tasks > 0 do
    local co = table.remove(tasks, 1)
    local ok, err = coroutine.resume(co)

    if not ok then
      print("task failed:", err)
    elseif coroutine.status(co) ~= "dead" then
      tasks[#tasks + 1] = co
    end
  end
end

spawn(function()
  print("A1")
  coroutine.yield()
  print("A2")
end)

spawn(function()
  print("B1")
  coroutine.yield()
  print("B2")
end)

run()
```

这不是高性能调度器，但能说明协程本质：

```text
调度器决定什么时候 resume；
任务自己决定什么时候 yield；
没有抢占；
错误要在调度边界捕获；
```

这个示例为了清晰使用 `table.remove(tasks, 1)` 取队首。真实高频调度器不应频繁从数组头部删除元素，更常见的做法是维护 `head` 和 `tail` 索引，避免每次移动后续元素。

## 22. 垃圾回收：可达性、屏障、增量与分代

Lua 使用垃圾回收管理对象生命周期。

这对初学者意味着：

```text
大多数 table、function、string、userdata 不需要手动 free；
当对象不可达后，GC 最终会回收它；
但“最终”不是立刻；
持有引用就会阻止回收；
```

### 22.1 可达性

对象是否可回收，核心看它是否可达。

可达对象通常从根集合出发能找到：

```text
栈上的值；
全局环境；
注册表；
闭包 upvalue；
正在运行的协程；
C API 持有的引用；
```

如果一个 table 没有任何路径能访问到，它就可以被回收。

```lua
local t = {data = string.rep("x", 1000000)}
t = nil
-- 之后某次 GC 可以回收原 table 和其中数据
```

### 22.2 标记和清扫

概念上的标记清扫：

```text
从根对象开始；
标记所有能到达的对象；
未标记对象就是垃圾；
清扫并释放垃圾；
```

这很容易理解，但直接完整执行可能造成停顿。

所以 Lua 5.4 支持增量 GC 和分代 GC，用不同策略平衡吞吐和停顿。

### 22.3 增量 GC

增量 GC 的思想：

```text
不要一次做完全部 GC 工作；
把标记和清扫拆成很多小步；
程序运行一段，GC 做一点；
减少单次长暂停；
```

代价是实现需要维护写屏障。

写屏障解决的问题是：

```text
GC 标记过程中，程序仍在修改对象引用；
如果黑色对象新指向白色对象，GC 可能漏标；
屏障负责维持标记不变量；
```

初学者可以先不背颜色细节，但要理解：

```text
增量 GC 的难点在于程序和 GC 交错运行；
对象引用变化必须被记录或修正；
```

### 22.4 分代 GC

分代 GC 基于经验规律：

```text
很多对象很快死亡；
活得久的对象往往继续活得久；
```

因此可以更频繁地检查年轻对象，较少扫描老对象。

这适合很多脚本程序，因为临时 table、临时字符串、临时闭包可能大量产生。

代价是仍然需要处理老对象指向新对象的引用，仍然需要屏障和记忆集合等机制。

### 22.5 Lua 代码里的 GC 友好习惯

```text
避免在热循环里创建大量临时 table；
字符串大量拼接时用 table.concat；
及时移除不再需要的 table 引用；
缓存可复用函数和常量；
不要让闭包意外捕获大对象；
长生命周期全局表要定期清理无用键；
```

如果嵌入 C/C++，还要注意：

```text
C 侧引用 Lua 对象应使用注册表或合适引用机制；
userdata 的资源释放要设计 __gc 或显式 close；
不要保存可能被 GC 移动或释放的内部指针假设；
```

## 23. 元表分派、环境与全局变量的底层理解

很多 Lua 行为都可以从“普通路径 + 特殊分派”理解。

### 23.1 字段读取的分派

表达式：

```lua
local v = obj.name
```

概念流程：

```text
如果 obj 是 table，先在 table 中直接查 name；
如果找到非 nil 值，返回；
如果没找到，检查元表；
如果元表有 __index：
  若 __index 是 table，继续查这个 table；
  若 __index 是 function，调用它；
否则返回 nil；
```

这解释了为什么 `rawget(obj, "name")` 能绕过 `__index`。

### 23.2 字段写入的分派

```lua
obj.name = "Ada"
```

概念流程：

```text
如果 obj 是 table 且已有 name 键，直接写；
如果没有该键，检查 __newindex；
若 __newindex 是 table，写到那个 table；
若 __newindex 是 function，调用它；
否则写入 obj；
```

这解释了代理表和只读表如何实现。

### 23.3 全局变量不是特殊魔法

Lua 5.4 中，全局变量访问可以理解为访问 `_ENV` 表字段。

```lua
print(x)
```

可以理解为：

```lua
_ENV.print(_ENV.x)
```

这就是为什么：

```text
全局变量本质上是环境表里的字段；
修改环境可以限制脚本能力；
local 变量不会进入全局环境；
```

嵌入场景里，宿主程序常通过控制环境表来限制脚本可见能力。

例如只暴露安全 API，不暴露 `io`、`os.execute` 等危险函数。

### 23.4 只读配置表示例

```lua
local function readonly(t)
  return setmetatable({}, {
    __index = t,
    __newindex = function(_, key)
      error("readonly field: " .. tostring(key), 2)
    end,
    __pairs = function()
      return pairs(t)
    end,
  })
end

local config = readonly({
  host = "127.0.0.1",
  port = 8080,
})

print(config.host)
config.port = 9000 -- error
```

注意：这只是浅层只读。

如果字段值本身是 table，内部 table 仍可能被修改，需要递归包装或设计不可变数据约定。

## 24. 性能模型与调优方法

Lua 性能优化不应从猜测开始。

先建立性能模型，再用测量验证。

### 24.1 先理解成本来源

常见成本：

```text
table 创建和重哈希；
字段查找，尤其是全局查找和元表链；
函数调用；
闭包创建；
字符串拼接；
临时对象导致 GC 压力；
C/Lua 边界频繁切换；
协程频繁 resume/yield；
```

不是说这些都不能用，而是在热路径里要知道它们有成本。

### 24.2 local 缓存

全局函数查找需要从环境表取字段。

在热循环中可以局部缓存：

```lua
local sin = math.sin

local total = 0
for i = 1, 1000000 do
  total = total + sin(i)
end
```

这能减少重复表查找，也让依赖更明确。

但不要为了微优化把所有代码都写得难读。

原则是：

```text
先写清楚；
测出热点；
再优化热点；
保留可读性；
```

### 24.3 避免热循环临时对象

不佳：

```lua
for i = 1, n do
  local point = {x = xs[i], y = ys[i]}
  process(point)
end
```

如果这是高频路径，可能产生大量临时 table。

可考虑：

```lua
for i = 1, n do
  process_xy(xs[i], ys[i])
end
```

或复用对象：

```lua
local point = {}
for i = 1, n do
  point.x = xs[i]
  point.y = ys[i]
  process(point)
end
```

复用对象要确认 `process` 不会保存引用，否则后续修改会影响已保存数据。

### 24.4 字符串拼接

少量拼接可以直接用 `..`。

大量循环拼接用：

```lua
local parts = {}
for i = 1, n do
  parts[i] = tostring(i)
end
local s = table.concat(parts, ",")
```

### 24.5 C/Lua 边界

跨 C/Lua 边界调用非常强大，但频繁细粒度调用可能有成本。

例如：

```text
Lua 每处理一个元素就调用一次 C；
C 每处理一个字段就回调一次 Lua；
```

可能不如批量传递：

```text
一次传入整个数组；
C 侧批处理；
一次返回结果；
```

性能优化时要关注“边界次数”，不只关注单次函数快慢。

### 24.6 Lua 与 LuaJIT

LuaJIT 是另一个非常重要的实现，具有 JIT 编译和 FFI 等能力。

但它和 PUC-Rio Lua 在版本语义、标准支持、实现结构和性能模型上不同。

本文主线以官方 Lua 5.4 为准。

学习建议：

```text
先掌握标准 Lua 语言模型；
再根据项目需要学习 LuaJIT；
不要把 LuaJIT 的性能经验无条件套到 Lua 5.4；
```

## 25. 常见误区

### 25.1 忘记 local

```lua
count = 0
```

这通常创建或修改全局变量。

更好：

```lua
local count = 0
```

除非你明确要写全局环境。

### 25.2 以为数组从 0 开始

```lua
local t = {"a", "b"}
print(t[0]) -- nil
print(t[1]) -- a
```

Lua 习惯 1 起始。

### 25.3 以为 0 和空字符串是假

```lua
if 0 then
  print("true")
end

if "" then
  print("true")
end
```

它们都是真。

### 25.4 依赖 pairs 顺序

```lua
for k, v in pairs(t) do
  print(k, v)
end
```

不要把输出顺序当业务语义。

需要顺序时：

```lua
local keys = {}
for k in pairs(t) do
  keys[#keys + 1] = k
end
table.sort(keys)

for _, k in ipairs(keys) do
  print(k, t[k])
end
```

### 25.5 混用点和冒号

```lua
obj.method(obj, x)
obj:method(x)
```

这两者等价。

但下面通常是错的：

```lua
obj.method(x) -- self 没传
```

### 25.6 滥用元表

元表能让代码优雅，也能让代码难以追踪。

判断标准：

```text
如果元表隐藏的是稳定抽象，可以用；
如果元表只是为了炫技，让普通字段访问产生意外副作用，应避免；
```

### 25.7 闭包意外持有大对象

```lua
local function build()
  local big = load_big_data()
  return function(id)
    return big[id]
  end
end
```

只要返回的函数还活着，`big` 就不会被回收。

这可能是设计需要，也可能是内存泄漏来源。

## 26. 项目练习路线

学习 Lua 最好用小项目串起来。

### 26.1 练习一：配置加载器

目标：

```text
读取 Lua 配置文件；
校验字段；
提供默认值；
返回只读配置；
```

你会练到：

```text
table；
模块；
assert/error；
元表 __index/__newindex；
环境控制；
```

示例配置：

```lua
return {
  host = "127.0.0.1",
  port = 8080,
  workers = 4,
}
```

### 26.2 练习二：事件分发器

目标：

```text
注册事件回调；
触发事件；
移除回调；
错误隔离；
```

你会练到：

```text
函数作为值；
table 作为列表和字典；
pcall 隔离回调错误；
模块 API 设计；
```

### 26.3 练习三：原型对象系统

目标：

```text
实现 class-like 工具；
支持 new；
支持方法；
支持简单继承；
```

你会练到：

```text
__index；
冒号语法；
对象初始化；
继承链调试；
```

不要一开始追求完整 class 框架。先实现一个最小清晰版本。

### 26.4 练习四：协程任务调度器

目标：

```text
spawn 任务；
任务 yield；
调度器轮询 resume；
捕获任务错误；
支持 sleep 模拟；
```

你会练到：

```text
coroutine.create/resume/yield/status；
队列；
错误边界；
状态机；
```

### 26.5 练习五：C 扩展模块

目标：

```text
用 C 写一个 Lua 模块；
注册 add、sub 等函数；
从 Lua require 调用；
处理参数错误；
```

你会练到：

```text
lua_State；
C API 栈；
luaL_check*；
lua_push*；
luaopen_*；
```

这是从“会写 Lua”走向“理解 Lua 生态”的关键练习。

### 26.6 练习六：读源码小任务

不要试图第一天读完整个解释器。

建议小任务：

```text
找到 TValue 的定义，解释类型标签和数据如何放在一起；
找到 table 创建和查找相关函数，画出数组部分和哈希部分；
用 luac -l 观察一个函数的字节码；
找到 __index 相关分派路径，解释 rawget 为什么绕过元表；
找到 coroutine.resume 的库函数入口，追踪到执行控制层；
找到 GC 标记对象的大致流程；
```

每个任务只追一个问题。

读解释器源码的目标不是背实现细节，而是训练这种能力：

```text
从语言现象出发；
找到运行时结构；
找到关键执行路径；
反过来解释性能和边界；
```

## 27. 官方资料

- Lua 官方网站：https://www.lua.org/
- Lua 5.4 Reference Manual：https://www.lua.org/manual/5.4/
- Lua 5.4 官方源码浏览：https://www.lua.org/source/5.4/
- Lua 版本历史：https://www.lua.org/versions.html
- Lua 官方源码下载：https://www.lua.org/ftp/
- Programming in Lua：https://www.lua.org/pil/

建议阅读顺序：

```text
先确认项目使用的是 Lua 5.4、Lua 5.5、LuaJIT 还是宿主程序定制版本；
再读对应 Reference Manual 的基础语法、类型、函数、table、metatable、coroutine；
边写小项目边查标准库；
再用 luac -l 观察字节码；
最后按问题阅读官方源码；
```

真正掌握 Lua 的标志，不是记住所有 API，而是能解释：

```text
为什么 table 能表达这么多结构；
为什么闭包能保存状态；
为什么元表能改变操作行为；
为什么协程能暂停和恢复；
为什么 C API 用栈交换数据；
为什么 GC 能自动管理对象；
为什么 Lua 适合嵌入到更大的系统里。
```
