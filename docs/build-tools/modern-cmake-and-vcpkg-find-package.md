# 现代 CMake 语法、规则与 vcpkg find_package 原理教程

> 目标读者：第一次系统学习 CMake 的 C/C++ 开发者；也适合已经会写简单 `CMakeLists.txt`，但经常被 include path、link library、`find_package`、vcpkg、toolchain、Debug/Release、安装导出等问题困住的工程师。
> 学习目标：先能写出目标导向的现代 CMake；再理解 `PUBLIC`、`PRIVATE`、`INTERFACE`、target、property、generator expression、install/export、package config 等核心规则；最后能解释 vcpkg 为什么能让 `find_package` 找到依赖，以及出现“找不到包、链接错库、Debug/Release 混用、triplet 不一致”时应该如何排查。
> 说明：本文依据 CMake 官方文档、CMake packages/buildsystem/manual、CMake Presets 文档，以及 Microsoft vcpkg 官方 CMake integration、manifest mode、triplets 文档做中文教程化整理。CMake 和 vcpkg 都在持续演进，具体命令参数和策略行为以项目锁定版本及官方文档为准。

---

## 目录

0. [如何使用本文：从能跑到能设计构建系统](#0-如何使用本文从能跑到能设计构建系统)
1. [CMake 到底解决什么问题](#1-cmake-到底解决什么问题)
2. [一句话理解现代 CMake](#2-一句话理解现代-cmake)
3. [现代 CMake 的核心原则](#3-现代-cmake-的核心原则)
4. [最小项目：从 configure 到 build](#4-最小项目从-configure-到-build)
5. [target：现代 CMake 的基本单位](#5-target现代-cmake-的基本单位)
6. [PUBLIC、PRIVATE、INTERFACE：使用需求传播模型](#6-publicprivateinterface使用需求传播模型)
7. [变量、缓存、属性与作用域](#7-变量缓存属性与作用域)
8. [头文件、源文件与目录组织](#8-头文件源文件与目录组织)
9. [编译特性、编译选项与宏定义](#9-编译特性编译选项与宏定义)
10. [链接规则：库、传递依赖与 imported target](#10-链接规则库传递依赖与-imported-target)
11. [生成器表达式：按配置、平台和上下文生成规则](#11-生成器表达式按配置平台和上下文生成规则)
12. [选项、配置类型与 CMake Presets](#12-选项配置类型与-cmake-presets)
13. [安装、导出与让自己的库可被 find_package 找到](#13-安装导出与让自己的库可被-findpackage-找到)
14. [find_package 的工作原理](#14-findpackage-的工作原理)
15. [vcpkg 解决什么问题](#15-vcpkg-解决什么问题)
16. [vcpkg 与 CMake 集成的核心原理](#16-vcpkg-与-cmake-集成的核心原理)
17. [vcpkg find_package 的真实流程](#17-vcpkg-findpackage-的真实流程)
18. [manifest、triplet、overlay 与版本基线](#18-manifesttripletoverlay-与版本基线)
19. [完整示例：用 vcpkg 查找 fmt、spdlog、zlib](#19-完整示例用-vcpkg-查找-fmtspdlogzlib)
20. [常见问题与排查方法](#20-常见问题与排查方法)
21. [现代 CMake 代码审查清单](#21-现代-cmake-代码审查清单)
22. [学习路线与练习项目](#22-学习路线与练习项目)
23. [官方资料](#23-官方资料)

---

## 0. 如何使用本文：从能跑到能设计构建系统

学习 CMake 最容易犯的错误，是从复制命令开始：

```cmake
include_directories(...)
link_directories(...)
add_definitions(...)
```

这些写法在旧项目里很常见，也可能还能工作，但它们会把构建规则扩散到目录全局，让大型项目变得难以维护。

现代 CMake 的学习路线应该是：

```text
先理解 target；
再理解 target 的属性；
再理解属性如何通过 PUBLIC/PRIVATE/INTERFACE 传播；
再理解 find_package 返回的是 imported target；
最后理解 vcpkg 如何把依赖安装位置接入 CMake 的包搜索机制。
```

### 0.1 五遍学习法

| 阶段 | 目标 | 应读章节 | 能力标准 |
| --- | --- | --- | --- |
| 第一遍：能构建 | 会写最小 `CMakeLists.txt`，会 configure/build | 1、2、3、4 | 能独立构建一个可执行程序 |
| 第二遍：能组织目标 | 掌握 target、include、compile、link | 5、6、8、9、10 | 能把库和可执行文件拆清楚 |
| 第三遍：能管理配置 | 掌握变量、缓存、Presets、Debug/Release | 7、11、12 | 能让同一项目跨平台稳定配置 |
| 第四遍：能发布库 | 掌握 install/export/package config | 13、14 | 能让自己的库被其他项目 `find_package` |
| 第五遍：能管理依赖 | 掌握 vcpkg toolchain、manifest、triplet | 15 到 20 | 能解释并排查 vcpkg 包发现问题 |

### 0.2 每学一个 CMake 命令都问三个问题

```text
1. 这个命令是在配置阶段、生成阶段还是构建阶段发挥作用？
2. 它修改的是变量、目录属性、目标属性，还是生成器规则？
3. 它的影响范围是当前 target、依赖使用者，还是整个目录树？
```

例如：

```cmake
target_include_directories(mylib PUBLIC include)
```

不要只记住“添加头文件目录”。

应该理解：

```text
它修改 mylib 的 include 相关属性；
PUBLIC 表示 mylib 自己编译需要 include，链接 mylib 的使用者也需要 include；
如果 myapp 链接 mylib，myapp 会自动继承 mylib 的接口 include 目录。
```

这就是现代 CMake 的核心：依赖关系携带构建信息。

### 0.3 熟练 CMake 的判断标准

一个人是否熟练 CMake，不是看能不能把项目编译过。

更好的判断标准是：

```text
能否把 include path、compile definition、compile option 绑定到正确 target；
能否解释 PUBLIC/PRIVATE/INTERFACE 的传播边界；
能否避免全局 include_directories、link_directories、add_definitions；
能否区分 configure 阶段和 build 阶段；
能否使用 imported target 而不是手写库路径；
能否写出可安装、可导出、可被 find_package 使用的库；
能否解释 vcpkg toolchain 为什么必须在 project() 前生效；
能否根据 CMake debug 输出判断包搜索失败原因；
```

## 1. CMake 到底解决什么问题

CMake 不是编译器，也不是一个单一的构建工具。

更准确地说：

```text
CMake 是跨平台构建系统生成器。
```

它读取 `CMakeLists.txt`，根据平台和生成器生成实际构建文件：

```text
Ninja 文件；
Unix Makefiles；
Visual Studio solution；
Xcode project；
其他后端构建文件；
```

然后真正编译、链接、归档、安装的工作，交给后端构建工具和编译器完成。

### 1.1 CMake 的生命周期

一个典型 CMake 项目经历：

```text
configure 阶段：执行 CMakeLists，检查编译器、依赖、选项，写入缓存；
generate 阶段：根据 target 和属性生成 Ninja/Makefile/VS 工程；
build 阶段：调用后端工具实际编译和链接；
install 阶段：按 install 规则复制头文件、库、CMake 配置文件等；
test 阶段：用 CTest 运行测试；
package 阶段：用 CPack 或外部工具打包；
```

很多错误来自混淆阶段。

例如：

```cmake
message("hello")
```

它在 configure 阶段输出，不是在每次编译源文件时输出。

再如：

```cmake
add_custom_command(...)
```

它通常是在 generate 阶段写入构建规则，真正执行在 build 阶段。

### 1.2 CMake 为什么难

CMake 难，不是因为语法复杂，而是因为它同时处理：

```text
多平台；
多编译器；
多构建后端；
多配置 Debug/Release；
依赖发现；
安装和导出；
工具链和交叉编译；
历史兼容；
```

如果只从命令表学习，会很快迷失。

更好的理解方式是抓住一个主线：

```text
现代 CMake 用 target 表达构建对象，用 target 之间的依赖关系传播构建需求。
```

## 2. 一句话理解现代 CMake

现代 CMake 可以理解为：

```text
现代 CMake = target + usage requirements + imported target + package config + presets
```

对应解释：

```text
target：我要构建或引用的东西；
usage requirements：使用这个东西时必须继承的 include、宏、编译特性、链接库；
imported target：外部库在当前项目里的目标表示；
package config：让 find_package 能创建 imported target 的配置文件；
presets：把配置命令、生成器、toolchain、构建类型固化成可复现配置；
```

现代 CMake 的正确思维不是：

```text
把 include 目录和库路径塞到全局变量里；
```

而是：

```text
声明 myapp 链接 mylib；
mylib 自己说明它的使用需求；
CMake 沿依赖图自动把需求传给 myapp。
```

## 3. 现代 CMake 的核心原则

### 3.1 以 target 为中心

推荐写法：

```cmake
add_library(core src/core.cpp)

target_include_directories(core
  PUBLIC
    include
)

target_compile_features(core
  PUBLIC
    cxx_std_20
)

add_executable(app src/main.cpp)
target_link_libraries(app PRIVATE core)
```

不推荐把规则写成目录全局：

```cmake
include_directories(include)
add_definitions(-DUSE_CORE)
link_directories(/some/path)
```

原因是：

```text
全局规则容易误伤不相关目标；
依赖传播关系不清楚；
子目录顺序会影响结果；
安装导出时很难恢复真实使用需求；
```

### 3.2 让依赖自己携带使用要求

如果 `core` 的公共头文件里包含了 `fmt/core.h`：

```cpp
// include/core/log.hpp
#include <fmt/core.h>
```

那么 `core` 对 `fmt` 的依赖就是公共使用需求。

应该写：

```cmake
find_package(fmt CONFIG REQUIRED)

target_link_libraries(core
  PUBLIC
    fmt::fmt
)
```

这样使用 `core` 的目标也能继承 `fmt` 的 include 和链接需求。

如果 `fmt` 只在 `core.cpp` 内部使用，公共头文件不暴露它：

```cmake
target_link_libraries(core
  PRIVATE
    fmt::fmt
)
```

判断 `PUBLIC` 还是 `PRIVATE` 的核心，不是“我喜欢隐藏依赖”，而是：

```text
这个依赖是否出现在我的公共 ABI、公共头文件或使用者编译所需信息里？
```

### 3.3 使用 imported target

现代包通常提供 imported target：

```cmake
find_package(ZLIB REQUIRED)
target_link_libraries(app PRIVATE ZLIB::ZLIB)
```

不要手写：

```cmake
include_directories(${ZLIB_INCLUDE_DIRS})
target_link_libraries(app PRIVATE ${ZLIB_LIBRARIES})
```

变量式写法在老包里还会遇到，但 imported target 更完整：

```text
包含 include 目录；
包含链接库路径；
包含编译定义；
包含平台特定系统库；
包含 Debug/Release 不同库；
包含传递依赖；
```

## 4. 最小项目：从 configure 到 build

目录：

```text
hello/
  CMakeLists.txt
  src/
    main.cpp
```

`src/main.cpp`：

```cpp
#include <iostream>

int main() {
  std::cout << "hello cmake\n";
  return 0;
}
```

`CMakeLists.txt`：

```cmake
cmake_minimum_required(VERSION 3.24)

project(hello
  VERSION 0.1.0
  LANGUAGES CXX
)

add_executable(hello_app src/main.cpp)

target_compile_features(hello_app
  PRIVATE
    cxx_std_20
)
```

构建：

```bash
cmake -S . -B build -G Ninja
cmake --build build
```

### 4.1 cmake_minimum_required

```cmake
cmake_minimum_required(VERSION 3.24)
```

它的作用不只是检查版本。

它还会设置 CMake policy 的默认行为，让项目在指定版本语义下解释。

建议：

```text
不要写过低版本只是为了“兼容”；
根据团队实际工具链选择一个明确版本；
新项目可优先考虑 3.20+、3.24+ 或更高版本；
如果使用新特性，例如 FILE_SET、Presets、高版本策略，应相应提高版本；
```

### 4.2 project

```cmake
project(hello VERSION 0.1.0 LANGUAGES CXX)
```

`project()` 会启用语言、检测编译器、初始化大量项目信息。

重要规则：

```text
toolchain file 必须在 project() 启用语言前生效；
vcpkg 的 CMake 集成就是通过 CMAKE_TOOLCHAIN_FILE 在这里介入；
```

这就是为什么 vcpkg 通常要求：

```bash
cmake -S . -B build -DCMAKE_TOOLCHAIN_FILE=/path/to/vcpkg/scripts/buildsystems/vcpkg.cmake
```

而不是在 `project()` 之后再设置。

### 4.3 out-of-source build

推荐：

```bash
cmake -S . -B build
```

不要把生成文件放进源码目录。

原因：

```text
源码树保持干净；
可以同时有 build-debug、build-release、build-windows 等多个构建目录；
删除构建产物只需删除 build 目录；
```

## 5. target：现代 CMake 的基本单位

target 是 CMake 构建图中的节点。

常见 target 类型：

| 类型 | 命令 | 说明 |
| --- | --- | --- |
| 可执行程序 | `add_executable` | 生成可执行文件 |
| 普通库 | `add_library` | 静态库、动态库、模块库 |
| 接口库 | `add_library(name INTERFACE)` | 不编译源码，只携带使用需求 |
| 对象库 | `add_library(name OBJECT)` | 编译对象文件，可被其他目标复用 |
| imported target | `add_library(name IMPORTED)` 或包提供 | 表示外部已有库 |
| alias target | `add_library(Alias ALIAS real)` | 给已有 target 起命名空间别名 |

### 5.1 可执行目标

```cmake
add_executable(tool
  src/main.cpp
  src/cli.cpp
)
```

后续所有规则都围绕 `tool` 写：

```cmake
target_compile_features(tool PRIVATE cxx_std_20)
target_compile_definitions(tool PRIVATE TOOL_VERSION="1.0")
target_link_libraries(tool PRIVATE core)
```

### 5.2 库目标

```cmake
add_library(core STATIC
  src/core.cpp
)
```

也可以让 `BUILD_SHARED_LIBS` 决定默认库类型：

```cmake
add_library(core src/core.cpp)
```

是否使用这种方式取决于项目策略。

库作者要考虑：

```text
静态库和动态库导出符号是否一致；
Windows 上是否需要 __declspec(dllexport/dllimport)；
依赖传播是否正确；
安装和包配置是否支持两种构建；
```

### 5.3 INTERFACE 库

头文件库：

```cmake
add_library(math_headers INTERFACE)

target_include_directories(math_headers
  INTERFACE
    include
)

target_compile_features(math_headers
  INTERFACE
    cxx_std_20
)
```

`INTERFACE` 库不编译任何源文件。

它只表达：

```text
使用这个库的人需要哪些 include、宏、编译选项、链接依赖。
```

适合：

```text
header-only 库；
配置聚合目标；
平台抽象开关；
编译选项集合；
```

### 5.4 ALIAS 目标

```cmake
add_library(project_core src/core.cpp)
add_library(MyProject::core ALIAS project_core)
```

好处：

```text
内部和外部使用同一命名风格；
调用方写 target_link_libraries(app PRIVATE MyProject::core)；
如果目标名拼错，CMake 更容易报错；
```

命名空间 `Name::Target` 是现代 CMake 常见约定。

## 6. PUBLIC、PRIVATE、INTERFACE：使用需求传播模型

这是现代 CMake 最重要的一章。

### 6.1 三个关键词的直觉

| 关键词 | 当前 target 自己需要 | 使用当前 target 的别人需要 | 常见场景 |
| --- | --- | --- | --- |
| `PRIVATE` | 需要 | 不需要 | `.cpp` 内部依赖 |
| `PUBLIC` | 需要 | 也需要 | 公共头文件暴露的依赖 |
| `INTERFACE` | 不需要 | 需要 | header-only 或纯使用需求 |

### 6.2 include 目录示例

项目：

```text
core/
  include/core/api.hpp
  src/api.cpp
  src/detail.hpp
```

CMake：

```cmake
add_library(core src/api.cpp)

target_include_directories(core
  PUBLIC
    include
  PRIVATE
    src
)
```

含义：

```text
core 自己编译 api.cpp 时能包含 include 和 src；
链接 core 的 app 只能继承 include；
app 不应该看到 core/src/detail.hpp；
```

### 6.3 链接依赖示例

```cmake
find_package(fmt CONFIG REQUIRED)
find_package(spdlog CONFIG REQUIRED)

add_library(core src/log.cpp)

target_link_libraries(core
  PUBLIC
    fmt::fmt
  PRIVATE
    spdlog::spdlog
)
```

如果 `core` 的公共头文件里出现 `fmt::format_string`，那么 `fmt` 是公共依赖。

如果 `spdlog` 只在 `log.cpp` 里使用，那么 `spdlog` 是私有依赖。

### 6.4 错误使用的后果

把公共依赖写成 `PRIVATE`：

```text
core 自己能编译；
app include core/api.hpp 时找不到第三方头文件；
错误出现在使用者项目里；
```

把私有依赖写成 `PUBLIC`：

```text
使用者不必要地继承 include、宏和链接库；
公共接口被污染；
包导出后传递依赖变多；
ABI 和依赖管理更难；
```

所以判断依赖传播时，要看公共接口，而不是只看当前目标能不能编译过。

## 7. 变量、缓存、属性与作用域

CMake 里最容易混淆的是变量和属性。

### 7.1 普通变量

```cmake
set(MY_NAME "demo")
message(STATUS "name=${MY_NAME}")
```

普通变量有目录和函数作用域。

子目录可以看到父目录变量的值，但子目录里重新设置不会自动改回父目录。

### 7.2 cache 变量

```cmake
option(ENABLE_TESTS "Build tests" ON)
set(MY_FEATURE "auto" CACHE STRING "Feature mode")
```

cache 变量会保存在构建目录的 `CMakeCache.txt`。

用户可以通过命令行设置：

```bash
cmake -S . -B build -DENABLE_TESTS=OFF
```

理解 cache 很关键：

```text
第一次 configure 的值会缓存；
之后修改 CMakeLists 默认值，不一定覆盖用户缓存；
排查奇怪行为时要看 CMakeCache.txt；
必要时重新配置或删除 build 目录；
```

### 7.3 target 属性

现代 CMake 更推荐把构建规则放进 target 属性。

```cmake
target_compile_definitions(core PUBLIC CORE_ENABLE_LOG)
```

内部会影响类似：

```text
COMPILE_DEFINITIONS；
INTERFACE_COMPILE_DEFINITIONS；
```

你不需要频繁手写 `set_target_properties`，但要理解 target 命令本质上是在设置属性。

### 7.4 目录属性和全局状态

旧式命令通常改目录状态：

```cmake
include_directories(include)
add_compile_options(-Wall)
```

它们会影响当前目录及后续子目录或目标，顺序和作用范围容易混乱。

现代项目中应优先：

```cmake
target_include_directories(core PRIVATE include)
target_compile_options(core PRIVATE -Wall)
```

例外是：你确实要给整个目录树设置统一策略，并且团队能清楚理解作用范围。

## 8. 头文件、源文件与目录组织

### 8.1 推荐目录结构

```text
project/
  CMakeLists.txt
  include/
    project/
      core.hpp
  src/
    core.cpp
    detail.hpp
  tests/
    core_test.cpp
```

公共头文件放 `include/project/`，私有实现放 `src/`。

这样 include 语句更稳定：

```cpp
#include <project/core.hpp>
```

### 8.2 target_sources

可以在目标创建后追加源文件：

```cmake
add_library(project_core)

target_sources(project_core
  PRIVATE
    src/core.cpp
    src/detail.hpp
  PUBLIC
    FILE_SET HEADERS
    BASE_DIRS include
    FILES
      include/project/core.hpp
)
```

`FILE_SET HEADERS` 适合较新 CMake 项目，用来表达公共头文件集合，并可配合安装规则。

如果团队 CMake 版本较老，也可以简单写：

```cmake
add_library(project_core
  src/core.cpp
)

target_include_directories(project_core
  PUBLIC
    include
)
```

### 8.3 不建议 glob 收集源文件

不推荐：

```cmake
file(GLOB_RECURSE SOURCES src/*.cpp)
add_library(core ${SOURCES})
```

原因：

```text
新增文件时，构建系统不一定自动重新 configure；
代码审查中不容易看到构建目标实际新增了什么；
不同生成器行为可能让问题更隐蔽；
```

现代项目更推荐显式列出源文件。

如果项目确实要用 glob，应理解其重新配置行为，并统一团队约定。

## 9. 编译特性、编译选项与宏定义

### 9.1 target_compile_features

推荐表达 C++ 标准需求：

```cmake
target_compile_features(core
  PUBLIC
    cxx_std_20
)
```

这表示：

```text
core 需要 C++20；
使用 core 的目标也至少需要 C++20；
```

如果 C++20 只用于 `.cpp`，公共头文件不需要：

```cmake
target_compile_features(core PRIVATE cxx_std_20)
```

### 9.2 CXX_STANDARD 与 compile features

也可以设置：

```cmake
set_property(TARGET core PROPERTY CXX_STANDARD 20)
set_property(TARGET core PROPERTY CXX_STANDARD_REQUIRED ON)
```

但 `target_compile_features` 更适合表达使用需求传播。

工程建议：

```text
库目标优先用 target_compile_features；
需要全项目强制标准时，可结合项目选项或函数封装；
不要依赖编译器默认 C++ 标准；
```

### 9.3 target_compile_options

```cmake
target_compile_options(core
  PRIVATE
    $<$<CXX_COMPILER_ID:GNU,Clang>:-Wall>
    $<$<CXX_COMPILER_ID:GNU,Clang>:-Wextra>
    $<$<CXX_COMPILER_ID:GNU,Clang>:-Wpedantic>
    $<$<CXX_COMPILER_ID:MSVC>:/W4>
)
```

编译选项通常是 `PRIVATE`。

不要轻易把 warning 选项设为 `PUBLIC`，否则使用者会继承你的警告策略。

### 9.4 target_compile_definitions

```cmake
target_compile_definitions(core
  PUBLIC
    CORE_USE_FAST_PATH
  PRIVATE
    CORE_BUILDING_LIBRARY
)
```

判断标准：

```text
公共头文件需要看到的宏，用 PUBLIC；
只影响库内部实现的宏，用 PRIVATE；
header-only 库需要使用者看到的宏，用 INTERFACE；
```

## 10. 链接规则：库、传递依赖与 imported target

### 10.1 target_link_libraries 的真正含义

```cmake
target_link_libraries(app PRIVATE core)
```

这不只是把 `libcore.a` 加到链接命令。

它还表示：

```text
app 依赖 core；
构建顺序中 core 先于 app；
app 继承 core 的 INTERFACE 使用需求；
生成器根据平台生成正确链接命令；
```

### 10.2 imported target

外部库常以 imported target 表示：

```cmake
find_package(ZLIB REQUIRED)
target_link_libraries(app PRIVATE ZLIB::ZLIB)
```

`ZLIB::ZLIB` 是一个 target，虽然它不是当前项目编译出来的。

它的属性可能包括：

```text
IMPORTED_LOCATION；
INTERFACE_INCLUDE_DIRECTORIES；
INTERFACE_LINK_LIBRARIES；
INTERFACE_COMPILE_DEFINITIONS；
IMPORTED_CONFIGURATIONS；
```

这就是为什么 imported target 比变量更可靠。

### 10.3 不要优先使用 link_directories

不推荐：

```cmake
link_directories(/opt/lib)
target_link_libraries(app PRIVATE foo)
```

问题：

```text
库名解析不明确；
容易链接到错误配置或错误架构；
全局影响范围难控；
无法携带 include、宏、传递依赖；
```

更好：

```cmake
find_package(foo CONFIG REQUIRED)
target_link_libraries(app PRIVATE Foo::foo)
```

如果确实只有裸库路径，也应尽量创建 imported target 封装。

## 11. 生成器表达式：按配置、平台和上下文生成规则

生成器表达式在 generate 阶段求值，用 `$<...>` 表示。

常见用途：

```cmake
target_compile_definitions(core
  PRIVATE
    $<$<CONFIG:Debug>:CORE_DEBUG_BUILD>
)
```

含义：

```text
只有 Debug 配置下定义 CORE_DEBUG_BUILD；
Release 不定义；
```

### 11.1 BUILD_INTERFACE 和 INSTALL_INTERFACE

库安装导出时常见：

```cmake
target_include_directories(core
  PUBLIC
    $<BUILD_INTERFACE:${CMAKE_CURRENT_SOURCE_DIR}/include>
    $<INSTALL_INTERFACE:include>
)
```

含义：

```text
在源码构建树中使用 core 时，include 路径是源码目录；
安装后使用 core 时，include 路径是安装前缀下的 include；
```

这对可重定位安装包非常重要。

不要把构建机上的绝对源码路径写进安装后的接口属性。

### 11.2 什么时候不要滥用生成器表达式

生成器表达式很强，但过度使用会让 CMake 变得难读。

建议：

```text
简单平台分支可以用 if；
与 target 属性、配置、安装接口强相关时用 generator expression；
复杂表达式应封装成函数或拆开说明；
```

## 12. 选项、配置类型与 CMake Presets

### 12.1 option

```cmake
option(PROJECT_BUILD_TESTS "Build tests" ON)
option(PROJECT_ENABLE_ASAN "Enable AddressSanitizer" OFF)
```

命名建议：

```text
加项目前缀，避免和依赖包选项冲突；
默认值保守；
顶层项目可以打开测试，作为子项目时可默认关闭；
```

### 12.2 单配置和多配置生成器

Ninja、Makefile 通常是单配置：

```bash
cmake -S . -B build-debug -DCMAKE_BUILD_TYPE=Debug
cmake --build build-debug
```

Visual Studio、Xcode、Ninja Multi-Config 是多配置：

```bash
cmake -S . -B build -G "Ninja Multi-Config"
cmake --build build --config Debug
cmake --build build --config Release
```

不要假设 `CMAKE_BUILD_TYPE` 在所有生成器下都有意义。

### 12.3 CMakePresets.json

Presets 用来记录可复现配置。

示例：

```json
{
  "version": 5,
  "configurePresets": [
    {
      "name": "ninja-debug",
      "generator": "Ninja",
      "binaryDir": "${sourceDir}/build/debug",
      "cacheVariables": {
        "CMAKE_BUILD_TYPE": "Debug",
        "CMAKE_CXX_STANDARD": "20"
      }
    }
  ],
  "buildPresets": [
    {
      "name": "ninja-debug",
      "configurePreset": "ninja-debug"
    }
  ]
}
```

使用：

```bash
cmake --preset ninja-debug
cmake --build --preset ninja-debug
```

vcpkg 项目里尤其推荐把 toolchain、triplet、构建目录写进 preset，而不是让每个开发者手打命令。

## 13. 安装、导出与让自己的库可被 find_package 找到

如果你写的是库，目标不只是自己能 build。

还应该让别人能：

```cmake
find_package(MyLib CONFIG REQUIRED)
target_link_libraries(app PRIVATE MyLib::core)
```

### 13.1 安装目标和头文件

```cmake
add_library(core src/core.cpp)
add_library(MyLib::core ALIAS core)

target_include_directories(core
  PUBLIC
    $<BUILD_INTERFACE:${CMAKE_CURRENT_SOURCE_DIR}/include>
    $<INSTALL_INTERFACE:include>
)

install(TARGETS core
  EXPORT MyLibTargets
  ARCHIVE DESTINATION lib
  LIBRARY DESTINATION lib
  RUNTIME DESTINATION bin
)

install(DIRECTORY include/
  DESTINATION include
)
```

### 13.2 导出 targets

```cmake
install(EXPORT MyLibTargets
  NAMESPACE MyLib::
  DESTINATION lib/cmake/MyLib
)
```

这会安装一个 targets 文件，里面描述 `MyLib::core` 这个 imported target。

### 13.3 生成 package config

常见做法是使用 `CMakePackageConfigHelpers`：

```cmake
include(CMakePackageConfigHelpers)

configure_package_config_file(
  cmake/MyLibConfig.cmake.in
  "${CMAKE_CURRENT_BINARY_DIR}/MyLibConfig.cmake"
  INSTALL_DESTINATION lib/cmake/MyLib
)

write_basic_package_version_file(
  "${CMAKE_CURRENT_BINARY_DIR}/MyLibConfigVersion.cmake"
  VERSION "${PROJECT_VERSION}"
  COMPATIBILITY SameMajorVersion
)

install(FILES
  "${CMAKE_CURRENT_BINARY_DIR}/MyLibConfig.cmake"
  "${CMAKE_CURRENT_BINARY_DIR}/MyLibConfigVersion.cmake"
  DESTINATION lib/cmake/MyLib
)
```

`cmake/MyLibConfig.cmake.in`：

```cmake
@PACKAGE_INIT@

include("${CMAKE_CURRENT_LIST_DIR}/MyLibTargets.cmake")
```

安装后，使用者的 `find_package(MyLib CONFIG REQUIRED)` 就能找到配置文件，并获得 `MyLib::core`。

### 13.4 可重定位原则

可重定位包的关键：

```text
安装后的 INTERFACE_INCLUDE_DIRECTORIES 不应包含开发机源码绝对路径；
安装后的 target 不应引用构建目录里的库文件；
依赖应通过 find_dependency 或 imported target 传播；
```

这就是为什么要用：

```cmake
$<BUILD_INTERFACE:...>
$<INSTALL_INTERFACE:...>
```

## 14. find_package 的工作原理

`find_package` 的目标是查找外部包，并把包的信息提供给当前项目。

现代项目希望它最终提供 imported target。

### 14.1 基本用法

```cmake
find_package(fmt CONFIG REQUIRED)
target_link_libraries(app PRIVATE fmt::fmt)
```

含义：

```text
查找 fmt 包配置；
如果找不到就报错；
找到后创建或导入 fmt::fmt target；
app 链接 fmt::fmt，并继承其使用需求；
```

### 14.2 Module mode

Module mode 查找 `Find<PackageName>.cmake`。

例如：

```cmake
find_package(ZLIB REQUIRED)
```

CMake 可能使用内置的 `FindZLIB.cmake`。

老式 Find 模块常设置变量：

```text
ZLIB_FOUND；
ZLIB_INCLUDE_DIRS；
ZLIB_LIBRARIES；
```

现代 Find 模块通常也提供 imported target：

```text
ZLIB::ZLIB；
```

### 14.3 Config mode

Config mode 查找包自己安装的配置文件：

```text
<PackageName>Config.cmake；
<package-name>-config.cmake；
```

例如：

```cmake
find_package(fmt CONFIG REQUIRED)
```

可能找到：

```text
<prefix>/lib/cmake/fmt/fmtConfig.cmake
```

Config mode 的优势：

```text
包作者最了解自己的 target、依赖和配置；
能提供准确 imported target；
更适合现代 CMake 包；
```

### 14.4 REQUIRED、QUIET、COMPONENTS、版本

```cmake
find_package(Qt6 6.5 REQUIRED COMPONENTS Core Widgets)
```

含义：

```text
需要 Qt6；
版本至少满足请求规则；
必须找到 Core 和 Widgets 组件；
找不到则报错；
```

`QUIET` 表示减少查找消息，但不要用它掩盖构建问题。

### 14.5 搜索路径来自哪里

`find_package` 会综合多类路径。

常见来源包括：

```text
CMAKE_PREFIX_PATH；
<PackageName>_DIR；
系统默认安装前缀；
环境变量；
CMake 用户包注册表；
toolchain 文件注入的路径；
包管理器生成的路径；
```

如果你手动设置：

```bash
cmake -S . -B build -DCMAKE_PREFIX_PATH=/opt/mylibs
```

CMake 会在这个前缀下寻找包配置。

如果你设置：

```bash
cmake -S . -B build -Dfmt_DIR=/opt/fmt/lib/cmake/fmt
```

则直接告诉 CMake `fmtConfig.cmake` 所在目录。

### 14.6 Debug 模式排查 find_package

CMake 提供调试包查找的能力。

常用：

```bash
cmake -S . -B build --debug-find
```

或只调试某个包：

```bash
cmake -S . -B build --debug-find-pkg=fmt
```

排查时关注：

```text
CMake 在找 Module 还是 Config；
它检查了哪些路径；
哪个路径下缺少 Config 文件；
是否混用了错误 triplet、架构、Debug/Release；
是否缓存了旧的 <PackageName>_DIR；
```

## 15. vcpkg 解决什么问题

vcpkg 是 C/C++ 依赖管理器。

它解决的问题包括：

```text
下载第三方源码；
按指定 triplet 构建库；
安装头文件、库文件和 CMake package config；
让 CMake 通过 find_package 使用这些依赖；
用 manifest 记录项目依赖；
用版本基线提高依赖解析可复现性；
```

它不是 CMake 的替代品。

更准确地说：

```text
vcpkg 负责获取、构建和安装依赖；
CMake 负责描述你的项目如何使用这些依赖；
vcpkg 的 CMake toolchain 负责把二者接起来。
```

### 15.1 经典模式和 manifest 模式

经典模式：

```bash
vcpkg install fmt:x64-linux
```

依赖安装在 vcpkg 根目录下的 installed tree。

manifest 模式：

```json
{
  "dependencies": [
    "fmt",
    "spdlog",
    "zlib"
  ]
}
```

项目根目录放 `vcpkg.json`，CMake configure 时 vcpkg 可以自动安装依赖。

现代项目更推荐 manifest 模式，因为依赖声明随项目走。

### 15.2 triplet

triplet 描述构建维度，例如：

```text
x64-windows；
x64-windows-static；
x64-linux；
arm64-osx；
```

它通常决定：

```text
目标架构；
操作系统；
静态库还是动态库；
运行库选择；
编译选项；
自定义工具链；
```

同一个包在不同 triplet 下是不同构建产物。

因此：

```text
app 的架构、编译器和运行时必须与 vcpkg triplet 匹配；
不要把 x64-windows 的库拿去给 arm64 或 Linux 链接；
不要混用 static 和 dynamic triplet；
```

## 16. vcpkg 与 CMake 集成的核心原理

vcpkg 与 CMake 集成的关键是 toolchain file：

```text
scripts/buildsystems/vcpkg.cmake
```

配置时传入：

```bash
cmake -S . -B build \
  -DCMAKE_TOOLCHAIN_FILE=/path/to/vcpkg/scripts/buildsystems/vcpkg.cmake
```

### 16.1 toolchain file 为什么必须早

CMake toolchain file 会在 `project()` 启用语言时参与工具链和平台设置。

因此必须在第一次 configure 时传入。

错误做法：

```cmake
project(myapp LANGUAGES CXX)
set(CMAKE_TOOLCHAIN_FILE "/path/to/vcpkg.cmake")
```

此时太晚。

正确做法：

```bash
cmake -S . -B build -DCMAKE_TOOLCHAIN_FILE=/path/to/vcpkg.cmake
```

或写入 `CMakePresets.json`：

```json
{
  "version": 5,
  "configurePresets": [
    {
      "name": "vcpkg-debug",
      "generator": "Ninja",
      "binaryDir": "${sourceDir}/build/vcpkg-debug",
      "cacheVariables": {
        "CMAKE_BUILD_TYPE": "Debug",
        "CMAKE_TOOLCHAIN_FILE": "$env{VCPKG_ROOT}/scripts/buildsystems/vcpkg.cmake",
        "VCPKG_TARGET_TRIPLET": "x64-linux"
      }
    }
  ]
}
```

### 16.2 vcpkg toolchain 做了什么

概念上，vcpkg toolchain 会：

```text
确定 vcpkg 根目录；
确定目标 triplet；
在 manifest 模式下解析 vcpkg.json；
必要时安装依赖；
把 vcpkg installed 目录接入 CMake 搜索路径；
调整 find_package、find_library、find_path 等查找行为；
让 CMake 优先找到 vcpkg 安装的 package config；
```

这解释了一个常见现象：

```text
同一个 CMakeLists.txt，不带 vcpkg toolchain 找不到 fmt；
带 vcpkg toolchain 后 find_package(fmt CONFIG REQUIRED) 成功；
```

不是 `find_package` 自己下载了 fmt。

而是 vcpkg 在 configure 期间把 fmt 安装好，并让 CMake 的搜索路径能看到它。

## 17. vcpkg find_package 的真实流程

以：

```cmake
find_package(fmt CONFIG REQUIRED)
target_link_libraries(app PRIVATE fmt::fmt)
```

为例。

### 17.1 配置命令

```bash
cmake -S . -B build \
  -DCMAKE_TOOLCHAIN_FILE=$VCPKG_ROOT/scripts/buildsystems/vcpkg.cmake \
  -DVCPKG_TARGET_TRIPLET=x64-linux
```

### 17.2 CMake configure 时发生什么

概念流程：

```text
1. CMake 读取顶层 CMakeLists.txt；
2. cmake_minimum_required 设置策略；
3. CMake 进入 project()，准备启用 CXX；
4. vcpkg toolchain 在语言和平台初始化过程中生效；
5. vcpkg 识别 manifest 和 triplet；
6. vcpkg 安装或复用 fmt:x64-linux；
7. vcpkg installed tree 中出现 fmtConfig.cmake；
8. vcpkg toolchain 把 installed/x64-linux 接入 CMake 搜索前缀；
9. find_package(fmt CONFIG REQUIRED) 搜索 config；
10. CMake 找到 fmtConfig.cmake；
11. fmtConfig.cmake 定义 imported target fmt::fmt；
12. target_link_libraries(app PRIVATE fmt::fmt) 继承 include 和 link 信息；
13. generate 阶段生成后端构建文件；
14. build 阶段编译 app 并链接 vcpkg 构建出的 fmt；
```

### 17.3 find_package 不等于安装包

严格说：

```text
CMake 的 find_package 负责查找包；
vcpkg 的 manifest/toolchain 负责在查找前准备包和搜索路径；
```

在 vcpkg manifest 模式下，二者看起来像一个动作，是因为 configure 阶段自动发生了依赖安装。

这点很重要。

如果包没在 `vcpkg.json` 里声明，vcpkg 未必会安装它。

如果 toolchain 没传入，CMake 也不会知道 vcpkg installed 目录。

如果 triplet 不一致，CMake 可能看到的是另一个架构或根本看不到。

### 17.4 为什么建议写 CONFIG

```cmake
find_package(fmt CONFIG REQUIRED)
```

`CONFIG` 表示优先使用包自己提供的 config 文件。

vcpkg 安装的现代包通常会提供 config 文件和 imported target。

好处：

```text
避免 CMake 内置 Find 模块和包自己 config 之间产生歧义；
更容易得到包作者或 vcpkg 修正后的 target；
和 target_link_libraries(app PRIVATE fmt::fmt) 的现代写法匹配；
```

但并非所有包都只能用 CONFIG。

有些包仍然使用 CMake 内置 Find 模块，或 vcpkg 提供兼容逻辑。实际写法应参考 vcpkg 安装后输出的 usage 信息和包文档。

## 18. manifest、triplet、overlay 与版本基线

### 18.1 vcpkg.json

示例：

```json
{
  "name": "demo",
  "version-string": "0.1.0",
  "dependencies": [
    "fmt",
    {
      "name": "spdlog",
      "features": ["fmt"]
    },
    "zlib"
  ]
}
```

它表达：

```text
项目依赖 fmt；
项目依赖带 fmt feature 的 spdlog；
项目依赖 zlib；
```

manifest 文件应该提交到仓库。

### 18.2 vcpkg-configuration.json

常用于声明 registry、baseline、overlay 等配置。

示意：

```json
{
  "default-registry": {
    "kind": "git",
    "repository": "https://github.com/microsoft/vcpkg",
    "baseline": "..."
  }
}
```

baseline 的意义：

```text
锁定一组端口版本解析基线；
减少不同机器、不同时间解析到不同依赖版本的概率；
```

### 18.3 overlay ports 和 overlay triplets

overlay ports 用于提供或覆盖端口定义。

overlay triplets 用于提供自定义 triplet。

典型场景：

```text
公司内部库；
官方端口尚未合并的补丁；
特殊编译选项；
特定平台 ABI 策略；
```

工程建议：

```text
overlay 应纳入版本控制；
triplet 名称要明确表达 ABI 策略；
CI 和本地使用同一套 preset；
```

## 19. 完整示例：用 vcpkg 查找 fmt、spdlog、zlib

目录：

```text
demo/
  CMakeLists.txt
  CMakePresets.json
  vcpkg.json
  src/
    main.cpp
```

### 19.1 vcpkg.json

```json
{
  "name": "cmake-vcpkg-demo",
  "version-string": "0.1.0",
  "dependencies": [
    "fmt",
    "spdlog",
    "zlib"
  ]
}
```

### 19.2 CMakePresets.json

```json
{
  "version": 5,
  "configurePresets": [
    {
      "name": "default",
      "generator": "Ninja",
      "binaryDir": "${sourceDir}/build/default",
      "cacheVariables": {
        "CMAKE_BUILD_TYPE": "Debug",
        "CMAKE_TOOLCHAIN_FILE": "$env{VCPKG_ROOT}/scripts/buildsystems/vcpkg.cmake",
        "VCPKG_TARGET_TRIPLET": "x64-linux"
      }
    }
  ],
  "buildPresets": [
    {
      "name": "default",
      "configurePreset": "default"
    }
  ]
}
```

Windows 可根据需要改成：

```text
x64-windows
x64-windows-static
```

不要把 Linux triplet 直接复制到 Windows。

### 19.3 CMakeLists.txt

```cmake
cmake_minimum_required(VERSION 3.24)

project(cmake_vcpkg_demo
  VERSION 0.1.0
  LANGUAGES CXX
)

find_package(fmt CONFIG REQUIRED)
find_package(spdlog CONFIG REQUIRED)
find_package(ZLIB REQUIRED)

add_executable(demo_app
  src/main.cpp
)

target_compile_features(demo_app
  PRIVATE
    cxx_std_20
)

target_link_libraries(demo_app
  PRIVATE
    fmt::fmt
    spdlog::spdlog
    ZLIB::ZLIB
)
```

### 19.4 main.cpp

```cpp
#include <fmt/core.h>
#include <spdlog/spdlog.h>
#include <zlib.h>

int main() {
  auto message = fmt::format("zlib version: {}", zlibVersion());
  spdlog::info("{}", message);
  return 0;
}
```

### 19.5 构建

```bash
cmake --preset default
cmake --build --preset default
```

如果第一次 configure，vcpkg 可能下载和构建依赖。

后续构建会复用已安装或二进制缓存中的包。

## 20. 常见问题与排查方法

### 20.1 找不到包

错误：

```text
Could not find a package configuration file provided by "fmt"
```

检查：

```text
CMAKE_TOOLCHAIN_FILE 是否在第一次 configure 时传入；
VCPKG_ROOT 是否正确；
vcpkg.json 是否声明 fmt；
VCPKG_TARGET_TRIPLET 是否正确；
build 目录是否缓存了旧配置；
fmt_DIR 是否被错误设置到旧路径；
```

排查命令：

```bash
cmake -S . -B build --debug-find-pkg=fmt
```

### 20.2 明明安装了包，CMake 还是找不到

常见原因：

```text
包安装在 classic mode 的某个 triplet 下，但项目使用另一个 triplet；
使用 manifest 模式时没有在当前项目 vcpkg.json 声明；
忘记传 CMAKE_TOOLCHAIN_FILE；
第一次 configure 没传 toolchain，后面再传但缓存已经固定；
设置了 CMAKE_FIND_ROOT_PATH 或其他交叉编译路径影响搜索；
```

处理：

```text
删除或重建 build 目录；
使用 CMakePresets 固化 toolchain 和 triplet；
查看 configure 日志中的 vcpkg triplet；
使用 --debug-find 查看实际搜索路径；
```

### 20.3 链接到 Debug/Release 错误库

多配置和单配置要分清。

Visual Studio：

```bash
cmake --build build --config Debug
```

Ninja 单配置：

```bash
cmake -S . -B build-debug -DCMAKE_BUILD_TYPE=Debug
cmake --build build-debug
```

vcpkg 包会根据 triplet 和配置准备库。

不要手写绝对库文件路径绕过 imported target，否则很容易链接错配置。

### 20.4 找到了系统库而不是 vcpkg 库

可能原因：

```text
没有使用 CONFIG；
CMAKE_PREFIX_PATH 或系统路径优先；
CMAKE_FIND_PACKAGE_PREFER_CONFIG 未按预期；
包名大小写或模块模式导致走了内置 Find 模块；
```

处理：

```text
优先使用 find_package(pkg CONFIG REQUIRED)；
查看 --debug-find 输出；
查看找到的 <PackageName>_DIR；
确认 imported target 的 LOCATION 和 include 属性；
```

### 20.5 target 名称不知道怎么写

vcpkg 安装包时通常会输出 usage 信息。

也可以查看：

```text
installed/<triplet>/share/<port>/usage；
installed/<triplet>/share/<port>/*Config.cmake；
包官方文档；
```

不要猜 target 名。

例如有些包 target 名是：

```text
fmt::fmt；
spdlog::spdlog；
ZLIB::ZLIB；
unofficial-sqlite3::sqlite3；
```

不同包命名风格可能不同。

## 21. 现代 CMake 代码审查清单

检查一个 `CMakeLists.txt` 时，可以按这张表看。

| 问题 | 推荐判断 |
| --- | --- |
| 是否有 `cmake_minimum_required` | 必须有，版本要与使用特性匹配 |
| 是否明确 `project(... LANGUAGES ...)` | 应明确启用语言 |
| 是否使用 target 命令 | 优先使用 `target_*` |
| 是否滥用全局 include/link/definitions | 一般应避免 |
| 公共头文件需要的依赖是否 `PUBLIC` | 是则必须传播 |
| 私有实现依赖是否误写 `PUBLIC` | 能私有就私有 |
| 是否链接 imported target | 优先链接 `Pkg::Target` |
| 是否手写绝对库路径 | 一般应避免 |
| 是否支持 Debug/Release | 不要假设单配置生成器 |
| 是否有 Presets | 团队项目建议有 |
| vcpkg toolchain 是否在 configure 开始传入 | 必须早于 `project()` 生效 |
| triplet 是否统一 | 本地、CI、文档应一致 |
| 安装接口是否可重定位 | 不应暴露源码绝对路径 |

## 22. 学习路线与练习项目

### 22.1 练习一：静态库和可执行程序

目标：

```text
写一个 core 静态库；
写一个 app 链接 core；
core 暴露 include/core/*.hpp；
app 只能看到公共头文件；
```

重点：

```text
target_include_directories；
PUBLIC/PRIVATE；
target_link_libraries；
```

### 22.2 练习二：header-only 库

目标：

```text
创建 INTERFACE 库；
传播 include 目录；
传播 cxx_std_20；
让 app 链接这个 INTERFACE 库；
```

重点：

```text
INTERFACE target；
INTERFACE usage requirements；
```

### 22.3 练习三：用 vcpkg 管理 fmt

目标：

```text
创建 vcpkg.json；
添加 fmt；
用 CMakePresets 传入 vcpkg toolchain；
find_package(fmt CONFIG REQUIRED)；
链接 fmt::fmt；
```

重点：

```text
manifest 模式；
toolchain file；
CONFIG mode；
imported target；
```

### 22.4 练习四：安装和导出自己的库

目标：

```text
install(TARGETS)；
install(EXPORT)；
生成 MyLibConfig.cmake；
另一个项目 find_package(MyLib CONFIG REQUIRED)；
```

重点：

```text
BUILD_INTERFACE；
INSTALL_INTERFACE；
CMakePackageConfigHelpers；
可重定位包；
```

### 22.5 练习五：排查一个找不到包的问题

目标：

```text
故意删除 toolchain；
故意设置错误 triplet；
故意设置错误 fmt_DIR；
分别观察错误；
用 --debug-find-pkg=fmt 解释搜索路径；
```

重点：

```text
find_package 搜索模型；
CMakeCache；
vcpkg installed tree；
triplet；
```

## 23. 官方资料

- CMake Buildsystem Manual：https://cmake.org/cmake/help/latest/manual/cmake-buildsystem.7.html
- CMake `find_package` 命令：https://cmake.org/cmake/help/latest/command/find_package.html
- CMake Packages Manual：https://cmake.org/cmake/help/latest/manual/cmake-packages.7.html
- CMake Presets Manual：https://cmake.org/cmake/help/latest/manual/cmake-presets.7.html
- `target_include_directories` 文档：https://cmake.org/cmake/help/latest/command/target_include_directories.html
- vcpkg CMake integration：https://learn.microsoft.com/en-us/vcpkg/users/buildsystems/cmake-integration
- vcpkg manifest mode：https://learn.microsoft.com/en-us/vcpkg/concepts/manifest-mode
- vcpkg triplets：https://learn.microsoft.com/en-us/vcpkg/concepts/triplets

建议阅读顺序：

```text
先读 cmake-buildsystem(7)，理解 target 和 usage requirements；
再读 target_* 命令文档，确认 PUBLIC/PRIVATE/INTERFACE 行为；
再读 find_package 和 cmake-packages，理解 Module mode 与 Config mode；
然后读 vcpkg CMake integration，理解 toolchain 如何接入；
最后用 --debug-find 和一个真实项目练习排查；
```

真正掌握现代 CMake 和 vcpkg 的标志，不是记住所有命令，而是能解释：

```text
一个依赖为什么应该是 PUBLIC 还是 PRIVATE；
一个 include 目录为什么应属于某个 target；
一个包为什么能通过 find_package 变成 imported target；
vcpkg 为什么必须通过 toolchain 早期介入；
triplet 为什么会影响 ABI 和搜索路径；
为什么删除 build 目录常常能排除缓存导致的假问题；
为什么安装导出的库不能泄漏构建机绝对路径。
```
