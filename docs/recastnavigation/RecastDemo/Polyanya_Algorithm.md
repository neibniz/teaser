# Polyanya 算法详解

本文以 Polyanya 原论文为规范来源，并用作者公开实现核对工程细节：

- Michael L. Cui, Daniel D. Harabor, Alban Grastien, "Compromise-free Pathfinding on a Navigation Mesh", IJCAI 2017.
- [IJCAI 2017 原论文 PDF](https://www.ijcai.org/proceedings/2017/0070.pdf)
- [IJCAI 论文页面](https://www.ijcai.org/proceedings/2017/70)
- [作者公开 Pathfinding 仓库](https://bitbucket.org/dharabor/pathfinding/src/master/anyangle/polyanya/)

辅助参考：

- Shizhe Zhao, David Taniar, Daniel D. Harabor, "Faster and More Robust Mesh-based Algorithms for Obstacle k-Nearest Neighbour", arXiv 2018.
- [arXiv 论文](https://arxiv.org/pdf/1808.04043)

后续论文不是 Polyanya 原始提出文献，但第 4 节对 Polyanya 的节点、successor 和启发式做了较清晰的复述，可用于交叉理解。

阅读时要区分三层内容：

| 层次 | 本文如何处理 |
| --- | --- |
| 论文定义 | `node = (I,r)`、successor、`h_p`、剪枝和正确性以 IJCAI 2017 为准。 |
| 作者实现 | `next_polygon`、端点方向、退化几何、root history 的更新时机用于解释如何落地。 |
| Recast/Detour 接入 | 只讨论适配边界；Polyanya 原论文没有 area cost、off-mesh connection 或任意三维曲面语义。 |

## 一、Polyanya 解决什么问题

Polyanya 解决的是二维平面中导航网格上的 Euclidean Shortest Path Problem，简称 ESPP。

输入：

- 一个 navigation mesh。
- 一个起点 `s`。
- 一个终点 `t`。

输出：

- 从 `s` 到 `t` 的最短几何路径。
- 路径不能穿过障碍。
- 路径长度按欧氏距离计算。

这里的“最短”有严格边界：每段可见线段的代价就是欧氏长度，所有可走区域代价相同。若应用还要求 area cost、单向门、跳跃连接、转弯半径、坡度能耗或动态避障，问题已经不是原论文证明的 ESPP，不能直接沿用“全局最优”的结论。

这里的 navigation mesh 是一组凸多边形，它们共同覆盖所有可通行空间。障碍物不是搜索图的一部分，而是可通行多边形集合之外的区域。

Polyanya 的目标是同时满足三个性质：

| 性质 | 含义 |
| --- | --- |
| exact / optimal | 返回真实最短路径，不只是近似路径。 |
| online | 查询时不依赖昂贵的预处理结果。 |
| fast | 搜索效率接近或优于许多近似算法。 |

原论文称它为 compromise-free，是因为当时许多导航网格算法需要在速度、最优性和动态灵活性之间取舍：

- Channel Search 快，但通常只近似最短。
- TA* 可以求最优，但可能要重复搜索和缓存。
- TRA* 快，但依赖离线预处理，更适合静态环境。

Polyanya 的核心思路是：不在 polygon center、edge midpoint 或 mesh vertex 这些离散点上搜索，而是在边上的连续区间上搜索。

初学者可以先记住一句话：

> Polygon A* 搜索“经过哪个面”；Polyanya 搜索“同一个最后转折点能够看见下一条边上的哪一整段”。

## 二、为什么普通 A* 不够

在导航网格上做传统 A*，通常会把每个 polygon 当成图节点，把相邻 polygon 之间的共享边当成连接。这样可以很快得到一条 polygon corridor，然后再用 funnel / string pulling 拉直路径。

这个流程的问题是：

```text
A* 只决定经过哪些 polygon。
funnel 只在这条 corridor 内求最短折线。
```

如果 A* 选错 corridor，即使 funnel 在该 corridor 内最优，最终路径也可能不是全局最短。

根本原因是：几何最短路径不是纯拓扑问题。它取决于路径在哪里穿过每条 shared edge，而这些穿越点是连续变量。

Polyanya 的设计就是把这些连续变量保留下来。它不说“我到了某个 polygon”，而是说：

```text
我已经有一组路径前缀，它们共享同一个最后转折点 root，
并且可以从 root 直线看见某条 mesh edge 上的一段连续 interval。
```

这个 interval 代表无穷多个可能的穿越点。Polyanya 一次搜索一整段点，而不是一个点。

## 三、几何定义

### 1. Polygon 和 edge

论文中的 polygon 是平面中的有界图形，由顶点和边组成。

一条 edge 是两个不同顶点之间的闭区间：

```text
e = (a, b)
```

论文假设：

- edge 之间最多只在端点重合。
- polygon 之间最多只共享 edge 或 vertex。
- 两个 polygon 如果共享一条 edge，则它们相邻。
- navigation mesh 中所有可走 polygon 都是凸多边形。

凸性是 Polyanya 的核心前提。因为只有在凸 polygon 内，两个可走点之间的直线才一定仍在该 polygon 内，这样“把 interval 推过一个 polygon”的操作才成立。

### 2. Mesh 和 obstacle

一个 map 中，每个点要么 traversable，要么 not traversable。

navigation mesh `M` 是所有可通行凸多边形的集合。剩余区域被视为 obstacle。

Polyanya 不要求 mesh 必须是三角网格。只要 polygon 是凸的，可以是：

- CDT triangulation。
- 合并后的 convex polygon mesh。
- rectangle mesh。
- 人工创建的 convex navigation mesh。

### 3. Visibility

两个可通行点 `p` 和 `q` co-visible，当且仅当线段 `pq` 只穿过 navigation mesh 中的可通行点。

如果 `p` 和 `q` co-visible，agent 可以从 `p` 直线走到 `q`。

visibility 是 Polyanya 的基础，因为搜索节点 `(I, r)` 要求 interval 中所有点都能被 root 看见。

论文还排除了“两个障碍边界只在一个点接触，agent 到底能否从缝中挤过”这类拓扑歧义。其处理方法是在接触点附近放置一个任意小但非零的 polygon，并明确把它标成 traversable 或 obstacle。工程实现也必须先固定这条边界规则；仅靠浮点 epsilon 临时猜测会让 point location、corner 标记和 visibility 得出互相矛盾的结果。

### 4. Corner point

corner point 是可能出现在最短路径中的转折点。

直觉上，它通常是障碍物边界上的拐角。如果一条最短路径在某个点转弯，那么这个转弯点必须有几何原因：它绕过了一个障碍角点。

论文中的定义是：如果某个顶点 `c` 出现在一条最优路径 `<p1, c, p2>` 中，并且 `p1` 与 `p2` 不可见，那么 `c` 是 corner point。

这个定义的意义是：

- 不是所有 mesh vertex 都值得作为转折点。
- 内部 triangulation 产生的辅助点通常不是 corner point。
- non-observable successor 只有在 interval 端点是 corner point 时才需要生成。

这是 Polyanya 减少无意义搜索的重要依据。

### 5. Path 和 optimal path

一条 path 是一串可通行点：

```text
pi = <p1, p2, ..., pk>
```

任意相邻点 `pi` 与 `pi+1` 必须 co-visible。

路径长度是所有相邻点之间欧氏距离之和：

```text
len(pi) = sum distance(pi, pi+1)
```

optimal path 是所有可行路径中长度最短的一条。

## 四、核心思想：用 interval 表示一族路径

Polyanya 的节点表示一段连续区间：

```text
node = (I, r)
```

其中：

- `I = [a, b]` 是某条 mesh edge 上的连续 interval。
- `r` 是 root。
- `I` 中每个点都能从 `r` 直线看见。

`r` 是当前这族路径的最后共同转折点。可以把 `(I, r)` 理解成：

```text
从 start 到 root 的路径前缀已经确定。
从 root 可以直线走到 interval I 上的任意点。
之后路径还没有确定。
```

这意味着 `(I, r)` 不是一条路径，而是一族路径：

```text
start -> ... -> r -> p, 其中 p 属于 I
```

![Polyanya 节点用一个 root 和一段 interval 表示连续路径族](https://oss.euler.icu/teaser/recastnavigation/Docs/Images/Polyanya/polyanya-node-interval-family.png)

> **图 1：搜索节点 `(I,r)` 的信息压缩。** 深蓝线是已经确定的共同前缀 `s -> ... -> r`；青色 interval `I=[a,b]` 上每个 `p` 都与 `r` 共视，因此扇形中的无穷多条 `r -> p` 不需要拆成离散节点。`g` 只累计到 root，interval 之后仍是未决定的连续自由度。

二维障碍物环境中的最短路径有一个关键性质：

```text
最短路径只会在必要的障碍角点转弯。
两个连续转折点之间是直线。
```

Polyanya 不需要提前构建 visibility graph 来枚举所有可见角点对。它通过 interval search 在线发现这些转折点：

- observable successor 继续保持同一个 root，表示路径还没有必要转弯。
- non-observable successor 把 root 改成当前 interval 的某个端点，表示最短路径必须在这里绕过障碍角点。

一个合法普通节点始终满足四个不变量：

1. `I` 是一条 mesh edge 上的闭区间；只有虚拟 start node 和 target singleton 例外。
2. `I` 的每一点都能从 `r` 直线看见。
3. 从 `s` 到 `r` 的 parent 链描述一条具体路径，`g` 是这条路径的长度。
4. 节点还必须知道 interval 要推入哪一个 opposite polygon；同一几何区间朝相反 polygon 推进不是同一搜索状态。

## 五、搜索节点的数据结构

论文只用 `(I,r)` 描述数学状态。可执行实现还要保存 interval 的拓扑归属和方向。作者实现中的核心字段可以整理为：

| 字段 | 语义 | 为什么需要 |
| --- | --- | --- |
| `left/right` | interval 的两个几何端点。 | 做 orientation、射线与 edge 求交以及 `h_p` 计算。 |
| `left_vertex/right_vertex` | 承载 interval 的完整 mesh edge 两端 vertex id。 | interval 可能只覆盖 edge 子段，仅靠坐标无法知道 perimeter 的下一条边。 |
| `next_polygon` | 本节点即将 push 进入的 opposite polygon。 | 同一 shared edge 有两个方向，展开方向必须是状态的一部分。 |
| `root` | 最后共同转折点的 vertex id；start 常用特殊 id。 | root history 需要稳定 identity，浮点坐标不适合做 key。 |
| `g` | `s` 到 root 的已确定路径长度。 | observable propagation 不改变它，真正转弯时才增加。 |
| `f` | `g + h_p`。 | open list 的排序键；`h_p` 可以即时计算，不一定单独存储。 |
| `parent` | 产生该节点的父链。 | 恢复 corner roots；intermediate pruning 时可只保留发生转弯的节点。 |

`left/right` 不能随意交换。作者实现采用的约定是：站在 root 看向 left 时，right 位于视线右侧，即 `orient(root,left,right)` 不为逆时针。不同实现可以采用相反 convention，但 successor 的 perimeter 遍历、left/right corner 判定和 edge 邻接必须全部一致。

不要把 `next_polygon` 理解成“interval 当前所在 polygon”。interval 位于两个 polygon 的公共边上；它本身没有唯一所属面。`next_polygon` 明确的是下一次展开要进入哪一侧。

### `g` 的含义

`g(n)` 不是到 interval 的距离，而是到 root 的距离。

如果 successor 是 observable：

```text
root 不变
g(child) = g(parent)
```

因为路径前缀没有增加新的转折点，仍然只确定到同一个 root。

如果 successor 是 non-observable，root 会变成当前 interval 的某个端点 `e`：

```text
root(child) = e
g(child) = g(parent) + distance(root(parent), e)
```

这表示路径前缀现在确定到新的转折点 `e`。

这里还有两个容易漏掉的前提：

- `e` 必须恰好是承载 edge 的 mesh vertex，interval 内部的裁剪交点不是 corner。
- 该 vertex 必须被标记为 corner point；否则在这里转弯不可能形成 taut shortest path。

为什么不把 `g` 定义为到 interval 的距离？因为 interval 中有无穷多个点。如果把 `g` 定义为到 interval 的某个点，就必须选定具体点，interval 就失去了一次表示一族路径的能力。

## 六、初始化

### 1. 论文中的虚拟 start node

论文不是直接把 polygon edge 称为 start node，而是先定义一个统一的虚拟节点：

```text
N_start = ([s], r_outside)
```

- interval 是 singleton `[s]`。
- `r_outside` 被假设位于平面外，只能看见 `s`。
- 从 `r_outside` 到 `s` 的代价按定义为 0。

这样 Definition 2 对 start 也成立：把 `[s]` push 进包含 `s` 的 polygon，才得到真正位于 mesh edge 上的初始 successors。工程代码通常不会真的创建三维平面外坐标，而是用特殊 root id，例如 `-1`，并在几何计算时把它解释成 start 坐标。

### 2. 起点在 polygon 内部

若 `s` 位于凸 polygon `P` 内部，push 虚拟节点后得到 `P` 的所有边：

```text
I = P 的一条完整 edge
root = s
g = 0
next_polygon = edge 另一侧的 traversable polygon
```

凸性保证 `s` 能看见 `P` 边界上的每一点。若某条边外侧是 obstacle，则它没有可展开的 `next_polygon`，可以直接作为 cul-de-sac 省略。

### 3. 起点在 edge 或 vertex 上

如果 start 落在 edge 或 vertex 上，包含 start 的 polygon 不唯一。

分类处理比“任选一个 polygon”更稳健：

| 起点位置 | 初始展开 |
| --- | --- |
| shared edge 内部 | 对两侧 traversable polygons 分别展开，并跳过这条 entry edge。 |
| 非 corner vertex | 对所有 incident traversable polygons 展开，跳过以 `s` 为退化入口的边。 |
| corner vertex | 先按 mesh 的 squeeze/边界规则确定可进入扇区；歧义点不能用任意 epsilon 方向偷偷选一侧。 |
| mesh border | 只展开存在的 traversable 一侧。 |

作者实现把这些情况编码成若干 lazy start nodes，再复用 collinear successor 逻辑生成初始 intervals。这样普通展开代码不必充斥 start 特判。

### 4. 终点 singleton 与直接可见情况

如果 target 是 polygon 内部点，它通常不在任何 mesh edge 上。

但 Polyanya 的普通节点 interval 都在 edge 上。为了解决这个问题，论文规定：

```text
当搜索 push 进入包含 target 的 polygon 时，
显式生成一个只包含 target 的特殊 successor。
```

这个 successor 的 interval 是 singleton：

```text
I = [target]
```

root 的确定方式和普通 successor 一样：

- 如果 target 从当前 root observable，则 root 不变。
- 如果不可见，则 root 变成当前 interval 的某个端点。

“进入 target polygon”不等于无条件直达 target。仍要用当前 root 与 interval 的 cone 判断：

- `r -> t` 穿过 interval：target observable，singleton 的 root 仍是 `r`。
- `t` 落在 cone 左/右侧：最短 taut 局部路径经对应 interval endpoint；仅当该 endpoint 是 corner 时生成 non-observable target singleton。

作者实现做了等价优化：当 `next_polygon == target_polygon` 时不再把 `[t]`压入 heap，而是立即用 orientation 判断最终 root 并构造 final node。论文语义仍然是“生成 target singleton，并在它按最小 `f` 被展开时终止”。若 start 与 target 位于同一凸 polygon，则二者直接共视，答案就是线段 `s -> t`。

![Polyanya 从虚拟起点、interval 推进、corner 转根到 target singleton 的搜索链](https://oss.euler.icu/teaser/recastnavigation/Docs/Images/Polyanya/polyanya-search-lifecycle-v2.png)

> **图 2：从初始化到终点的状态变化。** 第一格严格画出 `[s]`与平面外 dummy root，代价按定义为 0；push 到初始 edges 后，工程实现才用特殊 id 映射到坐标 `s`。Observable push 保持 root 和 `g`不变；在 corner `c`发生 non-observable 转换后，root 变为 `c`且 `g=|s-c|`；最终 `[t]`仍以最后转折点 `c`为 root，而不是把 target 自身写入 root history。

## 七、Successor 的根本定义

给定父节点 `n=(I,r)`，successor `n'=(I',r')` 必须同时满足：

1. 对每个 `p' ∈ I'`，存在某个 `p ∈ I`，使局部路径 `<r,p,p'>` 是 taut。
2. `r'` 是这些局部路径最后一个共同点。
3. 在满足节点可见性与前两条的前提下，`I'` 已经最大，不能继续沿承载 edge 延长。

### Taut 到底是什么意思

`taut` 不是简单的“没有碰撞”，而是“把路径像绳子一样拉紧后，局部不能再缩短”。对三点 `<r,p,p'>`：

- 若 `r` 与 `p'` 共视，经过 `p` 的折线可被直线替换，不 taut。
- 若在 `p` 转弯但 `p` 只是 shared edge 内部点，没有障碍角支撑，轻微移动 `p`即可缩短，也不 taut。
- 只有当转弯被 obstacle corner 卡住时，`p`才可能成为必要转折点。

因此 observable successor 的路径实际上在局部被拉直为 `r -> p'`；non-observable successor 才保留 `r -> endpoint -> p'` 这个由 corner 支撑的折点。

## 八、Push 操作：从 interval 穿过 polygon

### 1. opposite polygon

当前 interval 位于 entry edge `E` 上。节点保存的 `next_polygon` 是 `E` 另一侧、沿搜索前进方向要进入的 traversable polygon，论文称它为 opposite polygon `Y`。

successor generation 的核心是：

```text
把 interval I 从 root 的方向推入 opposite polygon Y，
并投影到 Y 的 perimeter 上。
```

因为 `Y` 是凸多边形，从 `E` 上一点到 `Y` 内一点的线段不会离开 `Y`。所以 successor 不需要在 polygon 内采样，只要把从 root 通过 interval 的可见锥投影到 `Y` 的 perimeter。

### 2. 投影可见锥

设当前节点已经规范化方向：

```text
I = [a, b]
root = r
```

从 `r` 发出两条边界射线：

```text
r -> a
r -> b
```

这两条射线和 interval 之间形成 projection cone。沿 `Y` 的 perimeter 从 entry edge 的 right vertex 走向 left vertex，边界会依次落入三个连续区域：

```text
右侧非可见链 | cone 内可见链 | 左侧非可见链
```

两个 transition point 是边界射线与 perimeter edge 的交点。它们可能恰好是 vertex，也可能位于 edge 内部。实现不需要把整个 polygon 和无限 cone 做通用布尔运算；利用 convex perimeter 上 orientation 的单调变化，可以线性扫描，作者实现还用二分查找定位两处 transition。

### 3. 沿 polygon perimeter 切分 interval

可见链可能覆盖 `Y` 的多条边。每条被覆盖 edge 的完整段或子段分别成为一个 observable successor：

```text
I' = covered segment on an edge of Y
r' = r
```

如果 cone 只覆盖某条 edge 的一部分，就用 transition point 截出子 interval。即使两条相邻 edge 共线，也不能用一个 interval 跨越它们的公共 vertex，因为下一次 push 所需的承载 edge 和 opposite polygon 已经不同。

完整 push 可以按以下顺序实现：

1. 取得 `Y` 的有序 vertices、edges 和每条 edge 外侧邻居。
2. 找到 entry edge 的 right/left vertex，并确定 perimeter 的前进方向。
3. 用 `orient(r,right,p)` 找右侧非可见链到可见链的 transition。
4. 用 `orient(r,left,p)` 找可见链到左侧非可见链的 transition。
5. 对 transition 之间每条 edge 取 cone 覆盖部分，生成同 root 的 observable intervals。
6. 若 entry interval 的 right endpoint 恰好是 corner vertex，对右侧链逐 edge 生成 root=right 的 non-observable intervals。
7. 对 left endpoint 对称处理左侧链。
8. 跳过 entry edge 本身、外侧为 obstacle 的 edge，以及被 dead-end/cul-de-sac 规则剪掉的 successor。
9. 若 `Y` 是 target polygon，按同样可见/转角判定额外产生 `[t]`。

![Polyanya 把 interval 投影到 opposite polygon perimeter 并切分两类 successor](https://oss.euler.icu/teaser/recastnavigation/Docs/Images/Polyanya/polyanya-successor-projection.png)

> **图 3：一次 push 的 perimeter 分区。** 青色 cone 内边界按承载 edge 切成多个 observable intervals，root 仍为 `r`；cone 外的橙色边界只有在 entry endpoint（图中 `b`）恰好是 obstacle corner 时才生成 non-observable intervals，其共同 root 变为 `b`。被叉掉的 entry interval 不返回，否则会立即走回父 polygon。

### 4. 退化和共线情况

若 `r`、left、right 共线，projection cone 退化成一条射线，无法再用“左右 transition”区分可见链。作者实现的处理是：

1. 比较 root 到两个 interval endpoint 的距离，确定绳子会先碰到哪一端。
2. 只有该端是 corner 时，才把它设为新 root。
3. 绕 `Y` perimeter 生成对应方向的 non-observable intervals，同时排除 entry edge。

`root == left/right` 也归入这一类。这个分支不是罕见补丁：连续 intermediate pruning 后，root 很容易恰好落在新 interval 的端点。

## 九、Observable successor

observable successor 表示：从当前 root `r` 到 successor interval 中每个点都可见。

形式：

```text
parent = (I, r)
child  = (I', r)
```

root 不变，`g` 不变。

几何含义：

```text
从 start 到 r 的路径前缀已经确定。
从 r 仍然可以直线走到更远的 interval。
当前还不需要转弯。
```

observable successor 是 Polyanya 高效的原因之一。它可以跨越多个 polygon 而不引入新的 path vertex。

如果一片开阔区域被划分成很多 polygon，只要 root 能一直看见前方 interval，搜索就可以保持同一个 root 推进，不会像 polygon A* 那样被每个 polygon center 或 shared edge 人为打断。

注意“observable”是从 root **通过父 interval** 的可见性，不只是 `r` 与 child endpoint 做两次独立 segment test。整个 child interval 都必须落在 projection cone 内；只检查端点在浮点误差下可能漏掉中间越界。

## 十、Non-observable successor

non-observable successor 表示：successor interval 中的点无法从当前 root 直接看见。要到达它们，最短 taut path 必须先经过当前 interval 的某个端点。

形式：

```text
parent = ([a, b], r)
child  = (I', a) 或 (I', b)
```

只有当 `a` 或 `b` 是 corner point 时，才生成对应的 non-observable successor。

原因：

- 如果端点不是 corner point，那么经过该点转弯不可能形成最优路径。
- 生成这种 successor 只会引入冗余搜索。

non-observable successor 的 `g` 会增加：

```text
g(child) = g(parent) + distance(r, endpoint)
```

几何含义：

```text
路径已经确定需要在 endpoint 转弯。
endpoint 成为新的 root。
从 endpoint 可以看见 child interval。
```

一条 perimeter 链会被拆成若干 non-observable successors，因为每个 interval 仍必须落在一条 mesh edge 上。它们可以共享同一个新 root 和同一个 `g`，但各自拥有不同 `next_polygon`，不能合成一个跨 edge 状态。

## 十一、为什么 successor 要求 maximal interval

successor 定义要求 `I'` 是最大 interval。

这不是形式主义，而是效率和正确性的关键。

假设某条 edge 上有一段连续点都满足同样的可见性和 root 条件。如果把它拆成很多小段，搜索结果不变，但节点数会膨胀。

最大 interval 表示：

```text
只要一段连续点共享同一个路径前缀和同一个 root，
就把它们放到同一个节点中。
```

“最大”是相对于**同一承载 edge、同一 root、同一 push 方向**而言，不是沿整个 polygon perimeter 无限合并。transition point、polygon vertex、root 类型变化和邻接 polygon 变化都是 interval 的合法分界。这样既避免把同质区间切碎，又保留下一次展开所需的拓扑身份。

## 十二、启发式 `h_p`

Polyanya 是 best-first search，评价函数类似 A*：

```text
f(n) = g(n) + h_p(n, t)
```

其中：

- `g(n)` 是从 start 到 root `r` 的已知路径长度。
- `h_p(n, t)` 是从 root `r` 经 interval `I` 到 target `t` 的乐观估计。

对节点 `n=([a,b],r)`，定义：

```text
n = (I, r)
I = [a, b]
```

启发式本质是求：

```text
h_p(n,t) = min_{p in [a,b]} ( |r-p| + |p-t| )
```

这个最小化强制候选路径穿过 interval，但忽略 interval 之后的障碍，因此是比单纯 `|r-t|` 更有信息的 lower bound。

![Polyanya interval heuristic 的直达、端点和镜像三种几何情况](https://oss.euler.icu/teaser/recastnavigation/Docs/Images/Polyanya/polyanya-interval-heuristic-v2.png)

> **图 4：`h_p` 的三个分支。** 若 `r -> t`与 interval 相交，最小折线退化为直线；若与支撑线的交点落在 `[a,b]`外，约束最小值只能在端点取得；若 `r`与 `t`同侧，先把 `t`关于 interval 支撑线镜像为 `t'`，利用 `|p-t|=|p-t'|`再执行前两种判定。图中虚线是无限支撑线，粗青线才是合法 interval。

论文把几何情况分成三类。

### 1. 异侧且直线穿过 interval

如果线段 `r -> t` 穿过 interval `I`，则最小值就是直线距离：

```text
h = distance(r, t)
```

因为最优的 `p` 就是线段 `r -> t` 与 `[a,b]` 的交点，三角不等式在三点共线时取等号。

### 2. 异侧但直线错过 interval

若 `r -> t`与 interval 的**无限支撑线**相交，但交点参数落在 `[a,b]`外，受约束函数的最小值出现在较优端点：

所以：

```text
h = min(
  distance(r, a) + distance(a, t),
  distance(r, b) + distance(b, t)
)
```

### 3. 同侧时先镜像

如果 `t` 和 `r` 在 interval 支撑线的同一侧，那么路径必须先到达 interval，再折回到 target 所在侧。

这个情况可以用镜像法处理：

1. 把 `t` 关于 interval 所在直线镜像，得到 `t'`。
2. 对 `t'` 使用前两种情况。

因为对任意 `p in I`：

```text
distance(p, t) = distance(p, t')
```

所以：

```text
min distance(r,p)+distance(p,t)
= min distance(r,p)+distance(p,t')
```

镜像法把“同侧折线路径”转化成异侧问题。注意此时若镜像后的 `r -> t'`穿过 interval，启发式值是 `|r-t'|`，不是原始的 `|r-t|`；若仍错过 interval，则对 `t'`计算两个端点候选。

### 4. 可执行的判定顺序

不需要显式构造角度或 cone：

```text
heuristic(r, a, b, t):
    if r == a or r == b:
        return |r-t|

    if orient(a, b, r) 与 orient(a, b, t) 同号:
        t_eval = reflect(t, line(a,b))
    else:
        t_eval = t

    q = intersection(line(r,t_eval), line(a,b))

    if q 在线段 [a,b] 内:
        return |r-t_eval|
    else:
        return min(|r-a|+|a-t_eval|,
                   |r-b|+|b-t_eval|)
```

实际代码要单独处理两条直线近似平行或四点共线。作者实现用有界线交参数区分 `q` 在 left 外、interval 内或 right 外；完全共线时直接比较经两个端点的候选。所有 side、collinear 和 `q in [a,b]` 判断必须共用同一套尺度化 epsilon。

### 5. 为什么这个启发式可采纳

`h_p` 只考虑从 root 经 interval 到 target 的几何最短折线，不考虑 interval 之后的障碍。

障碍只会让真实路径更长，不会让真实路径更短。

因此：

```text
h_p(n, t) <= true remaining cost from node n to target
```

所以它是 admissible。

原论文明确依赖的是 admissibility：最优路径代表节点的 `f` 不超过最优解长度，因此次优 target node 不可能先于它被弹出。不要在没有额外证明时把这一点扩写成“任意实现下启发式一定 consistent”；尤其当 epsilon、非欧氏 area cost 或高度投影改变代价模型时，这个结论需要重新分析。

## 十三、主搜索流程

Polyanya 是按 `f=g+h_p` 排序的 best-first search。完整流程应把 point location、不可达检查、target singleton 和支配检查放在明确位置：

```text
findPath(mesh, start, target):
    S = locate all traversable start incident polygons
    T = locate target polygon

    if S or T is empty:
        return no path

    if no polygon in S shares a connected component with T:
        return no path

    if start and target are co-visible in one convex polygon:
        return [start, target]

    open = priority queue ordered by f
    bestG[root] = infinity

    virtualStart = ([start], outsideRoot), g = 0
    initial = push virtualStart into every valid start polygon
    enqueue(initial)

    while open not empty:
        n = pop minimum f, tie-break as configured

        if g(n) is strictly worse than bestG[root(n)]:
            continue    // stale heap entry

        if interval(n) contains target:
            return reconstruct path from n

        current = n
        loop:
            successors = push current through current.nextPolygon
            add target singleton when entering T
            apply cul-de-sac and dead-end pruning

            if successors.count != 1 or successor contains target:
                break

            current = the only successor
            if root changed:
                materialize a parent node for path recovery

        for each successor:
            child.g = parent.g                         // observable
                   or parent.g + |parent.root-endpoint| // non-observable

            if child.g is strictly worse than bestG[child.root]:
                discard
            else:
                bestG[child.root] = min(bestG[child.root], child.g)
                child.f = child.g + h_p(child, target)
                set child.parent
                push child into open

    return no path
```

这里的 `enqueue(initial)` 也要执行启发式计算和 obstacle/dead-end 过滤。open list 中可能保留后来被更优 root path 支配的旧节点，二叉堆通常不方便删除任意元素，所以在 pop 后再检查一次 `bestG`。

Intermediate pruning 的循环不能丢 parent 信息。Observable 唯一后继不改变 root，可以完全压缩；若唯一后继是 non-observable，必须至少记录一次 root change，否则最终路径长度正确但无法恢复转折点。

### 用一条节点链串起所有字段

假设最短路只需要绕过一个 corner `c`，搜索中可能出现：

| 阶段 | 节点 | `g` | `h_p` 的含义 | parent 中新增的路径点 |
| --- | --- | --- | --- | --- |
| 初始 | `(I0,s)` | `0` | 从 `s` 经 `I0` 到 `t` 的无障碍下界 | `s` |
| observable | `(I1,s)` | `0` | interval 已前移，但共同前缀仍只到 `s` | 无 |
| observable | `(I2,s)` | `0` | 继续保持同 root | 无 |
| non-observable | `(I3,c)` | `|s-c|` | 从 `c` 经 `I3` 到 `t` 的下界 | `c` |
| target singleton | `([t],c)` | `|s-c|` | `|c-t|` | 最终追加 `t` |

最后的 `f = |s-c| + |c-t|`就是该完整候选路径长度。这个例子也解释了为什么 observable 节点可以很多、输出 waypoint 却很少：搜索 frontier 在移动，已确定的折线路径并没有同步增长。

## 十四、Root history

如果 navigation mesh 存在环，并且 `s` 与 `t` 不连通，搜索可能不断生成 `g` 更大的节点。

论文借鉴 Anya，使用 root history：

```text
bestG[root] = 已知到达该 root 的最小 g
```

当准备生成一个新节点时：

```text
if g(newNode) > bestG[root(newNode)] + EPS:
    discard newNode
else:
    bestG[root] = min(bestG[root], g(newNode))
    keep it
```

root history 的作用不是替代 closed list，而是阻止同一个 root 被越来越差的路径反复访问。

为什么可以这么做：

- root 是路径最后共同转折点。
- 如果同一个 root 已经有更短路径前缀，那么更长的前缀不可能导出更短的完整路径。

这和 Dijkstra/A* 中“到同一离散节点的劣路径可丢弃”类似，但有一个关键差异：root 不是完整 search-state identity。同一个 root、相同 `g`可以对应多条不同 edge 上的 interval，它们覆盖不同未来方向。因此 root history 只能丢弃**严格更差的 g**；不能看到 root 相同就关闭所有其他 intervals，也不能把等价 g 的 interval 任意合并。

start 的特殊 root 通常不写入 vertex-indexed history。普通 root 应用 mesh vertex id 做 key，而不是量化后的坐标。作者实现还在生成 child 和 pop node 两处执行支配检查：前者减少入堆，后者清理已经变 stale 的 heap entry。

## 十五、Pruning 策略

### 1. Cul-de-sac pruning

cul-de-sac 是没有 successors 且不包含 target 的节点。

这种节点不可能到达目标，可以直接剪掉。

如果 entry edge 外侧是 obstacle，或者 push 后 perimeter 上没有任何合法 traversable successor，该节点就是 trivially cul-de-sac。实现可以在生成 `next_polygon == invalid` 时提前消掉它，不必真的入堆再展开。

### 2. Dead-end pruning

dead-end polygon 是只相邻一个 traversable polygon 的 polygon。

如果节点 push 进入一个 dead-end polygon，并且这个 polygon 不包含 target，那么可以直接剪掉。

原因：

```text
进入 dead-end 后，除了原路返回没有其他出口。
如果 target 不在里面，继续搜索只会产生 cul-de-sac 或回头路径。
```

dead-end polygon 可以在读取 mesh 时预计算：

```text
degreeTraversable(poly) == 1
```

### 3. Intermediate pruning

如果经过 target、cul-de-sac、dead-end 等处理后只剩一个 successor，并且它不包含 target，则可以不把它放入 open list，而是立即继续展开。

这类似 Jump Point Search 中跳过中间强制无分支节点。

论文还把这个策略推广到 non-observable successor：只要当前节点只有一个 successor，就可以连续推进，直到：

- 分支数增加。
- 到达 obstacle。
- 到达 target。
- 触发其他终止或剪枝条件。

若这一步 non-observable propagation 改变了 root，`g`与 parent history 必须立即提交；“不入 open”不等于“这个转弯没有发生”。作者实现只在发生转弯时为 intermediate node 分配持久节点，observable 中间节点则可以留在栈上。

### 4. 为什么 pruning 不破坏最优性

这些 pruning 都只删除不可能成为最优路径一部分的节点：

- cul-de-sac 没有出口。
- dead-end 没有目标时只能返回。
- intermediate pruning 没有删除路径，只是把唯一后继立即展开，改变的是调度方式而不是搜索空间。

因此它们不改变最优路径是否会被搜索到。

## 十六、路径恢复

Polyanya 的搜索节点不一定每一步都对应一个路径点。

observable successor 不改变 root，所以它只是把可见 interval 推得更远，不引入新的转折点。

non-observable successor 改变 root，才引入新的路径转折点。

因此恢复路径时不能收集 interval endpoint，更不能把每个 crossing point 当 waypoint。正确思路是：

1. 从目标节点沿 `parent` 回溯。
2. 对每个节点取得 root 对应坐标；与上一个输出点不同才收集。
3. 加上 start 和 target。
4. 反转顺序。

结果类似：

```text
start -> corner1 -> corner2 -> ... -> target
```

如果最终目标节点是 observable singleton：

```text
solution length = g(node) + distance(root(node), target)
```

如果最终目标节点是 non-observable singleton，`g` 已经包含到新 root 的路径，长度同样由：

```text
g(node) + distance(root(node), target)
```

得到。

作者实现采用一个很简洁的反向过程：先放入 `target`，沿 parent 链把不同的 root 坐标依次放入，再整体反转。start 的特殊 root id 在取坐标时映射回 `s`。如果 intermediate pruning 压缩了多个同-root observable nodes，恢复结果不受影响；若压缩跨过 root change 却未保存 parent，则会漏掉必要 corner。

恢复后可以做两类断言：

1. 相邻 waypoint 必须共视，且线段落在 traversable mesh 内。
2. 除 start/target 外，每个 waypoint 必须是 corner root；路径长度应等于 final node 的 `g + |root-target|`（容许浮点误差）。

## 十七、正确性直觉

论文的正确性证明可以拆成几个部分。

### 1. 最优路径节点会继续产生最优路径节点

如果某个节点 `(I, r)` 表示一条最优路径上的一段前缀，并且它还没有包含 target，那么它 push 进入 opposite polygon 后，至少会生成一个 successor，继续表示这条最优路径。

直觉：

- 如果 target 在 opposite polygon 中，直接生成 target successor。
- 否则最优路径必须从 opposite polygon 的某条边离开。
- 如果离开点从 `r` 可见，生成 observable successor。
- 如果不可见，最优路径必须在当前 interval 的某个端点转弯，而这个端点必须是 corner point，于是生成 non-observable successor。

### 2. open list 中始终保留一条最优路径的代表

初始节点表示从 start 出发的最优路径前缀。

每次弹出一个表示最优路径的节点，如果它不是目标节点，根据上一条性质，它会生成另一个表示最优路径的 successor。

所以 open list 中始终存在某个节点代表最优路径尚未展开的部分。

### 3. 第一个包含 target 的展开节点必然最优

假设第一个包含 target 的展开节点不是最优路径。

那么它的 `f` 值大于最优路径长度。

但 open list 中仍有一个表示最优路径的节点，而启发式不高估，所以这个节点的 `f` 不超过最优路径长度。

这就和 priority queue 先弹出当前最小 `f` 的规则矛盾。

因此第一个展开的 target 节点对应最优路径。

### 4. target 最终会被展开

论文说明，每个节点的 successor 数量有界；同一 `g` 值的连续节点数量有限；否则 `g` 会以非零距离增长。

论文证明使用的关键量是：每个节点 successor 数有界；同一 root、同一 `g` 的连续推进数量有限；一旦 root 改变，`g` 至少增加两个不同 corner points 之间的某个非零距离。连通性预检排除无解分量，root history 阻止环上更差前缀无限增长，因此若路径存在，最终会展开包含 target 的节点。

这份证明依赖理想几何假设：有限 mesh、无前述 squeeze 歧义、精确的 corner/visibility 判定和原始欧氏代价。工程 epsilon 若把两个不同 corner 合并、把零长度 interval 当成普通边，或错误删除等价 `g` 的不同 intervals，都会破坏证明所依赖的搜索空间。

## 十八、为什么 Polyanya 不需要 visibility graph

visibility graph 的做法通常是：

1. 找出所有障碍物角点。
2. 判断哪些角点两两可见。
3. 构建图。
4. 在图上搜索。

问题是 visibility graph 在最坏情况下有二次级边数，而且查询时如果 start/target 变化，还要插入新点并做 visibility checks。

Polyanya 避免了这一步。它只依赖 navigation mesh 的局部邻接：

```text
当前 interval -> adjacent polygon -> projected edge intervals
```

可见性不是通过全局两两检查得到，而是在 push 过程中局部维护。

这就是它 online 且 fast 的主要原因。

## 十九、为什么必须是 convex polygon mesh

Polyanya 对 polygon convexity 的依赖非常强。

在 convex polygon 中：

- 任意两点之间的线段都在 polygon 内。
- 从 interval push 到 polygon perimeter 的投影区域容易计算。
- observable successor 可以用连续 edge segment 表示。

如果 polygon 非凸：

- interval 上一点到 polygon 内另一点的线段可能穿出 polygon。
- 可见区域可能断裂。
- successor 不再是简单的最大连续 interval。

因此非凸区域必须先拆成凸 polygon。

除了单个 polygon 凸，原论文还要求整个问题是**同一个二维平面中的 polygon partition**：

- traversable polygons 覆盖可走空间。
- polygon interior 不互相重叠。
- 相邻 polygon 只通过公共 edge/vertex 接触。
- obstacle boundary 与 traversable boundary 的拓扑没有歧义。

这比“每个面单独看起来是凸的”更强。两个楼层在 xz 投影上重叠时，各面可能都凸，但投影后已经不是合法平面 partition。

## 二十、和 Detour A* + funnel 的关系

Detour 常见路径查询流程是：

```text
polygon graph A*
  -> polygon corridor
  -> funnel 拉直
```

这个流程在给定 corridor 内可以得到很好的折线路径，但全局最优性取决于 A* 找到的 corridor 是否包含真实最短路径。

Polyanya 的流程是：

```text
interval A*
  -> 在线搜索可能 corridor 和穿越区间
  -> 直接得到全局最短折线路径
```

差异在于：

| 方面 | Detour A* + funnel | Polyanya |
| --- | --- | --- |
| 搜索节点 | polygon ref | edge interval + root |
| 几何穿越点 | funnel 阶段才处理 | 搜索阶段就保留 |
| 最优性 | corridor 内最优 | 原论文平面、统一欧氏代价模型下全局最优 |
| 实现复杂度 | 较低 | 较高 |
| 对 mesh 要求 | convex polygons | convex polygons，且需要更精细的 interval successor |

如果要把 Polyanya 接到 RecastNavigation 中，它在职责上更像替换或扩展 `dtNavMeshQuery::findPath()`，但不能只把 `dtPolyRef` 换成 interval node。至少要解决以下合同差异：

### 1. 2.5D 高度与重叠楼层

原论文在一个平面内计算欧氏最短路。常见游戏适配是在 xz 平面搜索、再用 Detour polygon/detail mesh 查询 y，但只有在搜索区域的 xz 投影不自重叠时才安全。桥上桥下、多层室内和垂直堆叠 tile 会让两个不可相通表面投影到同一区域，必须分 layer 搜索或采用真正的曲面路径算法。

### 2. Portal 不一定是完整 shared edge

Detour 跨 tile `dtLink` 的 `bmin/bmax` 可以把 polygon edge 裁成 portal 子区间。Polyanya 能表示子 interval，但适配层必须同时保留：

- carrier edge 的完整两个 vertex id。
- portal 可穿越子段的 left/right 坐标。
- portal 另一侧 `dtPolyRef`。
- 进入邻 polygon 后的有序 perimeter 和方向。

只从 `dtLink`拿一个 portal midpoint 会重新退化成离散搜索，失去 Polyanya 的核心性质。

### 3. Area cost 与过滤器

`dtQueryFilter::getCost()`允许不同 area 具有不同单位代价。此时最短路径可能在没有 obstacle corner 的 area boundary 上折转，`g`和 `h_p`也不再是简单欧氏长度；原论文“只在 corner 转弯”的证明失效。include/exclude flags 可以在查询前删除 polygon，但删除后必须重新建立 obstacle boundary 和 corner 标记。

### 4. Off-mesh connection 和单向拓扑

跳跃、传送、梯子等 off-mesh connection 不是平面内连续可见区间，应该作为额外离散动作与 interval search 组合，其代价和启发式要另行证明。单向 link 同样超出论文的无向平面可见性模型；可以工程化支持，但不能直接声称仍有原证明保证。

### 5. Tile seam 的数值一致性

相邻 tile 的 portal endpoints 必须在统一坐标与 epsilon 下得到相同拓扑 identity。若同一点在两 tile 中有略不同坐标，root history、corner 判断和 maximal interval 切分会产生裂缝或重复节点。

所以更准确的结论是：Recast 生成的 convex polygons 提供了有用起点，但还需要一个面向 Polyanya 的平面 half-edge/portal 视图；不能直接把现有 Detour A* 循环局部改写成 interval A*。

## 二十一、实现数据结构建议

### 1. Mesh 表示

推荐使用 half-edge，或者至少使用“polygon 顶点数组 + 与每条边一一对应的邻居数组”：

```text
Vertex:
    position
    cyclic incident polygons
    isCorner
    isAmbiguous

Edge:
    v0, v1
    leftPoly, rightPoly
    traversable sub-interval / portal

Polygon:
    vertices in CCW order
    neighbour polygon per edge
    connectedComponent
    traversableDegree
```

必须能快速回答：

- 某个 point 落在哪个 polygon。
- 某条 edge 的另一侧 polygon 是谁。
- 某个 edge 子段是否是 obstacle boundary。
- 某个 vertex 是否 corner point。

incident polygons 最好按顶点周围的环序保存，这样才能判断 obstacle sector、边界歧义和 start 位于 vertex 时可进入哪些 polygon。作者实现的 mesh 输入在 incident polygon 列表中用 `-1`表示 obstacle sector，并据此标记 `is_corner/is_ambig`；若从 Detour 转换，应显式重建同等拓扑，而不是仅凭“vertex 是否位于 tile 边界”猜 corner。

### 2. Search node

```cpp
struct PolyanyaNode
{
    NodeId parent;
    VertexId root;          // special id for start
    Point left, right;      // interval geometry
    VertexId leftVertex;    // carrier edge endpoints
    VertexId rightVertex;
    PolyId nextPolygon;     // polygon to push into
    float g;
    float f;
};
```

root 坐标可通过 vertex id 查询，start root 用特殊 id 映射到 `start`。这样避免在每个节点重复存坐标，也让 root history 有稳定 key。若 mesh 支持运行时变更，vertex id 的生命周期必须覆盖整个查询。

不建议只存一个 `edgeId`而丢掉 `leftVertex/rightVertex`：interval 端点可能是 edge 内部交点，生成 non-observable successor 时需要区分“几何端点恰好落在 corner vertex”和“只是裁剪点”。

实际实现中的 root identity：

- start root 是特殊 id。
- corner root 可以用 vertex id。
- target singleton 的**interval**可用特殊标记，但它的 root 仍是 start 或 corner，不能把 target 当成新 root。

这样 root history 可以稳定 key。`h`是纯函数，可在入堆前计算后只存 `f`；调试版本可以额外存 `h`验证 `f≈g+h`。

### 3. Priority queue

open list 按 `f` 从小到大弹出。

如果 `f` 相等，常见 tie-break 可以是：

- 较小 `h` 优先，更接近目标。
- 较大 `g` 优先，更深入搜索。
- 稳定 insertion order，方便调试。

作者实现采用较大 `g`优先，也就是更深入的节点优先。论文的最优性不依赖特定 tie-break，但必须保证真正按非递减 `f`弹出；用近似 bucket queue 时要重新考虑精度。

### 4. Geometry robustness

实现中必须谨慎处理浮点误差：

- 点在线段上。
- ray 与 edge 几乎平行。
- target 恰好在 interval 端点。
- start 在 edge 或 vertex 上。
- interval 长度接近 0。
- polygon 共享边方向相反。

不要简单把所有比较写成固定 `EPS=1e-5`。更稳健的分层策略是：

1. topology identity 使用整数 vertex/poly id，不用坐标 epsilon。
2. orientation 优先使用 robust predicate 或尺度化误差界。
3. line intersection 使用 double，并检查 denominator 的相对量级。
4. interval 参数统一 clamp 到 `[0,1]`，只在接近端点时 snap 到已知 vertex。
5. `bestG`比较使用代价尺度 epsilon，与几何 orientation epsilon 分开。

所有 interval 还要保持统一方向 convention；否则同一 edge 从不同 polygon 观察时 left/right 颠倒，会把 left non-observable 错生成为 right non-observable。

## 二十二、Successor 生成的实现思路

下面的伪代码把论文的 perimeter 描述与作者实现的 orientation 分区合在一起。`emitEdgeSegment()`每次只产生一条 carrier edge 上的 interval：

```text
expand(node):
    Y = node.nextPoly
    if Y is invalid: return []

    orient interval so rightVertex -> ... -> leftVertex
    exclude entry edge from candidate perimeter

    if root, left, right are collinear:
        e = nearer endpoint to root
        if e is not the matching corner vertex: return []
        emit every non-entry perimeter edge as non-observable(root=e)
        return filtered successors

    A = first perimeter position where
        orient(root, right, position) becomes strictly inside cone
    qRight = ray(root,right) intersect edge(A-1,A)

    B = last perimeter position where
        orient(root, left, position) remains inside cone
    qLeft = ray(root,left) intersect edge(B,B+1)

    if right endpoint equals rightVertex and rightVertex is corner:
        for each edge from rightVertex to qRight:
            emit non-observable segment with root=rightVertex

    for each edge segment from qRight to qLeft:
        emit observable segment with root=node.root

    if left endpoint equals leftVertex and leftVertex is corner:
        for each edge from qLeft to leftVertex:
            emit non-observable segment with root=leftVertex

    if Y contains target:
        emit [target] using the same cone/endpoint rule

    for each emitted segment:
        set nextPolygon from the carrier edge's outer neighbour
        discard obstacle, entry-backtrack, cul-de-sac and dead-end cases
        compute root, g, h_p and f
        apply root-history dominance

    return successors
```

关键实现点：

1. **Transition 是 edge 内点。** `qRight/qLeft` 不一定是 vertex，必须保存几何坐标和 carrier edge vertex ids。
2. **严格/非严格 orientation 要成对设计。** 在恰好共线时，两侧都抢同一个 vertex 会产生重叠节点，两侧都不收又会漏路径。
3. **不能跨 vertex 合并。** 只允许把同一 carrier edge、同 root、同 `nextPolygon` 的数值碎片合并成 maximal interval。
4. **普通 singleton 不可盲删。** cone 恰好擦过 vertex 时，零长度 interval 可能是连接最优路径的唯一状态；应把“vertex singleton”作为明确类型处理，而不是按长度 epsilon 一律丢弃。
5. **Triangle fast path 只是优化。** 三角形中只有两条非 entry edges，可用少量 orientation 分支直接生成；它必须和通用 convex perimeter 算法产生相同 successors。
6. **Target 不是 perimeter edge。** `[t]`是额外特殊 successor，不能参与 edge 邻居查找，也不能把 target id 写进 root history。

## 二十三、复杂度和性能来源

原论文主要给出实验比较，没有给出一个只由 polygon 数量决定的紧致最坏复杂度。节点数取决于 mesh 几何、corner 数量、同一 edge 被多少 root 投影到，以及 pruning 效果；不能简单写成“每个 polygon 只访问一次”。

单次通用 polygon expansion 若线性扫描 `k`条边，几何工作量是 `O(k)`；利用 convex orientation 单调性可二分定位 transition，再线性输出实际 successors。但总搜索成本仍由生成的 interval nodes 和 heap 操作主导。

论文实验显示 Polyanya 在测试 mesh 上通常比当时多种 online/近似方法快，同时保持原问题定义下的最优性。性能来源主要有：

1. **Interval compression。**

   一个节点表示一整段连续穿越点，而不是单个点。

2. **Observable propagation。**

   开阔区域中 root 可以跨多个 polygon 保持不变，避免为每个 polygon 人为增加路径点。

3. **无需全局 visibility graph。**

   查询时只做局部 push，不做所有 corner pair 的可见性预处理。

4. **Pruning。**

   dead-end、cul-de-sac 和 intermediate pruning 减少了明显无用的扩展。

5. **强启发式。**

   `h_p` 不是简单的 root 到 target 直线距离，而是考虑了必须穿过 interval 的几何下界。

影响性能的 mesh 特性：

- polygon 数量。
- 平均 polygon 度数。
- dead-end 数量。
- obstacle corner 数量。
- polygon 是否合并得足够大。

通常，过细的 triangulation 会增加需要 push 的边；适当合并 convex polygons 能减少搜索节点。

但 polygon 越大，单次 expansion 的 degree 和 perimeter 扫描成本也可能升高。原论文的实验正是比较 CDT、合并 CDT 和 rectangle mesh，结论不是“越大越好”，而是在 polygon 数量、degree、corner 暴露和几何分支之间找平衡。

## 二十四、常见误解

### 1. Polyanya 不是 funnel 的替代品吗

它可以直接生成最短折线路径，因此不需要先得到 corridor 再 funnel。

但从几何思想看，Polyanya 和 funnel 都在利用可见通道。区别是：

- funnel 在已知 corridor 内拉直。
- Polyanya 在搜索过程中同时决定 corridor 和可见 interval。

### 2. interval endpoint 都能成为 root 吗

不能。endpoint 必须同时满足“恰好是 carrier edge 的 mesh vertex”和“该 vertex 是 corner point”，才能成为 non-observable root。cone 与 edge 的普通裁剪交点不能转根。

把所有 mesh vertex 都当 root 不再符合论文 successor 的 taut 定义，会引入大量局部可拉直路径；即使某个实现仍碰巧找到最短路，也不能沿用论文的节点界和正确性论证。

### 3. Polyanya 能处理任意 3D navmesh 吗

原论文讨论的是二维平面 navigation mesh。

Recast/Detour 常用于 2.5D 游戏场景：polygon 有 xz 投影和高度。只有搜索区域投影后仍是无重叠的平面 partition 时，才能在 xz 平面求路径后再查询高度。桥上桥下等重叠楼层不能直接压到同一个 Polyanya mesh。

如果要求真实 3D 曲面上的 geodesic shortest path，Polyanya 原论文的几何假设不够，需要不同算法。

### 4. root history 是 closed list 吗

不是。closed list 通常按搜索节点 identity 去重；root history 按 root 的最优 `g` 剪枝。

同一个 root 可能对应多个不同 interval，这些 interval 不能简单全部合并或关闭。root history 只用于丢弃到同一 root 的更差路径前缀。

### 5. 为什么 target 要做 singleton successor

因为 target 通常在 polygon 内部，不在 edge 上。普通 push 只产生 edge interval。如果不把 target 显式加入 successor，搜索会一直在 mesh edge 上推进，无法自然终止在内部点。

### 6. Maximal interval 可以跨 polygon vertex 吗

不能。maximal 的范围只限于同一 carrier edge、同一 root 和同一 `nextPolygon`。polygon vertex 两侧 edge 的拓扑邻居不同，即使几何共线也必须拆开。

### 7. 换成 area-weighted distance 仍然最优吗

不能直接保证。单位代价改变后，最短路可能像光线折射一样在 area boundary 改变方向，而不经过 obstacle corner；`g`、`h_p`、taut successor 和证明都要重建。原算法保证的是统一欧氏长度目标。

## 二十五、实现检查清单

实现 Polyanya 时可以按下面顺序检查：

1. 所有 polygon 是否位于同一平面并构成无 interior overlap 的 partition。
2. 所有 polygon 是否 convex，vertices 是否使用统一 winding。
3. 每条 shared edge 的两侧邻接、方向和 carrier vertex ids 是否一致。
4. obstacle sector、corner 和 ambiguous squeeze point 是否显式标记。
5. start/target 在 polygon、shared edge、vertex 和 border 上的 point location 是否都有测试。
6. 是否用虚拟 `[s]`语义生成所有合法初始 intervals，并排除 start 所在 entry edge。
7. 普通 interval 是否只位于一条 carrier edge，target/start singleton 是否显式区分。
8. node 是否保存 left/right 方向、carrier vertices 和下一步 opposite polygon。
9. observable projection 的两个 transition 是否能落在 edge 内部，并按每条 edge 切分。
10. non-observable successor 是否只从“interval endpoint == corner vertex”生成。
11. entry edge 是否不会作为 successor 原样返回。
12. collinear、root==endpoint、cone 擦过 vertex 和零长度 interval 是否有明确语义。
13. `g`是否表示到 root；observable 时不变，转根时只增加旧 root 到 corner 的距离。
14. `h_p`是否覆盖异侧直达、异侧端点、同侧镜像、平行和共线情况。
15. target polygon 是否按相同 cone 规则生成 `[t]`，且 target 不会成为 root-history key。
16. root history 是否按 root identity 只删除严格更差的 `g`，保留等价 g 的不同 intervals。
17. heap pop 后是否再次过滤 stale root path。
18. intermediate pruning 跨 non-observable 转弯时是否保留 parent/root change。
19. recovered waypoints 是否两两共视，长度是否与 final cost 一致。
20. 与 Detour 接入时是否明确拒绝或单独处理重叠楼层、area cost、off-mesh 和单向 link。

## 二十六、设计取舍总结

1. **连续 interval 替代离散点。**

   保留穿越 edge 的连续自由度，因此能保持 exact shortest path。

2. **root 表示最后共同转折点。**

   一组路径共享同一个 root 时，可以作为一个节点统一搜索。

3. **observable / non-observable 分裂。**

   能直线看见就保持 root，不可见时只在 corner point 转 root。

4. **启发式考虑必须穿过 interval。**

   比单纯直线距离更强，同时仍然 admissible。

5. **不构建全局 visibility graph。**

   用局部 push 在线发现必要可见关系，减少预处理和内存。

6. **依赖凸 navigation mesh。**

   凸性让 projection、visibility 和 maximal interval 都变得可控。

7. **实现复杂度高于 Detour A*。**

   需要稳健几何、interval 切分、corner 判断和 root history，但换来全局最优路径。

这里的“全局最优”始终指本文开头定义的平面、统一欧氏代价 ESPP。只要改变代价模型或几何模型，就应把 Polyanya 看作可复用的 interval-search 思想，而不是未经修改仍自动成立的证明。
