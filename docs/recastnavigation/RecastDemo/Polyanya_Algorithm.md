# Polyanya 算法详解

本文根据 Polyanya 原论文整理：

- Michael L. Cui, Daniel D. Harabor, Alban Grastien, "Compromise-free Pathfinding on a Navigation Mesh", IJCAI 2017.
- 论文地址：https://www.ijcai.org/proceedings/2017/0070.pdf
- IJCAI 页面：https://www.ijcai.org/proceedings/2017/70

辅助参考：

- Shizhe Zhao, David Taniar, Daniel D. Harabor, "Faster and More Robust Mesh-based Algorithms for Obstacle k-Nearest Neighbour", arXiv 2018.
- 论文地址：https://arxiv.org/pdf/1808.04043

后续论文不是 Polyanya 原始提出文献，但第 4 节对 Polyanya 的节点、successor 和启发式做了较清晰的复述，可用于交叉理解。

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

二维障碍物环境中的最短路径有一个关键性质：

```text
最短路径只会在必要的障碍角点转弯。
两个连续转折点之间是直线。
```

Polyanya 不需要提前构建 visibility graph 来枚举所有可见角点对。它通过 interval search 在线发现这些转折点：

- observable successor 继续保持同一个 root，表示路径还没有必要转弯。
- non-observable successor 把 root 改成当前 interval 的某个端点，表示最短路径必须在这里绕过障碍角点。

## 五、搜索节点的数据结构

工程实现中，一个 Polyanya 节点通常需要以下字段：

| 字段 | 说明 |
| --- | --- |
| `intervalLeft` | interval 左端点坐标。 |
| `intervalRight` | interval 右端点坐标。 |
| `edgeId` | interval 所在 mesh edge。 |
| `polygonId` | interval 即将 push 进入的 opposite polygon。 |
| `root` | 最后共同转折点坐标或 root vertex id。 |
| `g` | 从 start 到 root 的已知路径长度。 |
| `h` | 从 root 经 interval 到 target 的乐观估计。 |
| `f` | `g + h`。 |
| `parent` | 用于恢复路径。 |

论文把节点抽象为 `(I, r)`，但实现必须额外知道 interval 属于哪条 edge、下一步要推入哪个 polygon，否则无法生成 successors。

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

为什么不把 `g` 定义为到 interval 的距离？因为 interval 中有无穷多个点。如果把 `g` 定义为到 interval 的某个点，就必须选定具体点，interval 就失去了一次表示一族路径的能力。

## 六、初始化

### 1. 起点在 polygon 内部

如果 start 在某个 convex polygon `P` 内部，初始 successors 是 `P` 的所有边。

每个初始节点：

```text
I = polygon edge
r = start
g = 0
```

因为 `P` 是凸的，start 能看见 `P` 边界上的每一点。

### 2. 起点在 edge 或 vertex 上

如果 start 落在 edge 或 vertex 上，包含 start 的 polygon 不唯一。

工程实现中通常需要：

1. 找出所有 incident traversable polygons。
2. 对每个 polygon 遍历边。
3. 跳过包含 start 的边，或把它拆分成非退化 interval。
4. 对剩余边生成 root 为 start 的初始节点。

### 3. 终点特殊处理

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

搜索终止条件就是：弹出并展开一个 interval 包含 target 的节点。

## 七、Successor 的根本定义

论文给出的 successor 定义可以拆成三条。

给定父节点 `(I, r)`，它的 successor `(I', r')` 必须满足：

1. 对每个 `p' in I'`，存在某个 `p in I`，使得路径 `<r, p, p'>` 是 taut。
2. `r'` 是所有这些 `<r, p, p'>` 路径共享的最后共同点。
3. `I'` 是满足以上条件的最大 interval。

这里的 taut 很关键，意思是局部最优、没有可以拉直的冗余折线。

如果 `<r, p, p'>` 能被替换成更短的直线或更短的局部折线，那么这个 successor 不应该生成。

## 八、Push 操作：从 interval 穿过 polygon

### 1. opposite polygon

当前 interval 位于某条 mesh edge 上。沿着从 root 远离的方向，edge 的另一侧有一个相邻 traversable polygon，论文称为 opposite polygon `Y`。

successor generation 的核心是：

```text
把 interval I 从 root 的方向推入 opposite polygon Y，
并投影到 Y 的 perimeter 上。
```

因为 `Y` 是 convex polygon，从 interval 上任意点到 `Y` 内任意点的线段都在 `Y` 内。因此可见区域在 `Y` 的边界上可以通过几何投影得到。

### 2. 投影可见锥

设当前节点：

```text
I = [a, b]
root = r
```

从 `r` 发出两条射线：

```text
r -> a
r -> b
```

这两条射线形成一个可见锥。把这个锥穿过 polygon `Y`，与 `Y` 的边界相交。落在这个投影区域内的边界点，都是从 `r` 通过 `I` 可见的点。

这些点会形成 observable successors。

### 3. 沿 polygon perimeter 切分 interval

投影区域可能覆盖 `Y` 的多条边。

每条被覆盖的 edge 或 edge 子段，都会成为一个 observable successor interval：

```text
I' = covered segment on an edge of Y
r' = r
```

如果投影只覆盖某条边的一部分，就生成子 interval，而不是整条边。

这就是 Polyanya 能保持几何精确的原因：它不会把 edge 简化成 midpoint，也不会强行选择某个离散穿越点。

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

## 十一、为什么 successor 要求 maximal interval

successor 定义要求 `I'` 是最大 interval。

这不是形式主义，而是效率和正确性的关键。

假设某条 edge 上有一段连续点都满足同样的可见性和 root 条件。如果把它拆成很多小段，搜索结果不变，但节点数会膨胀。

最大 interval 表示：

```text
只要一段连续点共享同一个路径前缀和同一个 root，
就把它们放到同一个节点中。
```

这样 Polyanya 才能用少量节点覆盖连续空间。

## 十二、启发式 `h_p`

Polyanya 是 best-first search，评价函数类似 A*：

```text
f(n) = g(n) + h_p(n, t)
```

其中：

- `g(n)` 是从 start 到 root `r` 的已知路径长度。
- `h_p(n, t)` 是从 root `r` 经 interval `I` 到 target `t` 的乐观估计。

对节点：

```text
n = (I, r)
I = [a, b]
```

启发式本质是求：

```text
h_p(n, t) = min over p in I of distance(r, p) + distance(p, t)
```

但是它忽略 interval 之后可能存在的障碍。因此它是 lower bound，不会高估真实剩余代价。

论文把几何情况分成三类。

### 1. target 在 interval 另一侧，并且位于可见锥内

如果线段 `r -> t` 穿过 interval `I`，则最小值就是直线距离：

```text
h = distance(r, t)
```

因为最优的 `p` 就是 `r -> t` 与 `I` 的交点。

### 2. target 在 interval 另一侧，但在可见锥外

如果 `t` 在 interval 的另一侧，但不在 `r` 通过 `[a,b]` 形成的 cone 内，那么从 `r` 经 interval 到 `t` 的最短折线会经过 interval 的某个端点。

所以：

```text
h = min(
  distance(r, a) + distance(a, t),
  distance(r, b) + distance(b, t)
)
```

### 3. target 和 root 在 interval 同侧

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

镜像法把“同侧折线路径”转化成“异侧直线或端点路径”问题。

### 4. 为什么这个启发式可采纳

`h_p` 只考虑从 root 经 interval 到 target 的几何最短折线，不考虑 interval 之后的障碍。

障碍只会让真实路径更长，不会让真实路径更短。

因此：

```text
h_p(n, t) <= true remaining cost from node n to target
```

所以它是 admissible。

论文还指出这三种几何情况足以保证 estimator consistent。直观理解是：当搜索从一个 interval push 到下一个 interval 时，`g` 的增加和 `h` 的减少不会违反三角不等式，因此 best-first 顺序具备 A* 式的最优性保证。

## 十三、主搜索流程

Polyanya 的主流程可以写成：

```text
findPath(mesh, start, target):
    locate polygon(s) containing start
    locate polygon containing target

    if start and target are not in same connected component:
        return no path

    open = priority queue ordered by f
    rootHistory = best g per root

    generate initial nodes from start polygon(s)
    push initial nodes into open

    while open not empty:
        n = pop node with smallest f

        if interval(n) contains target:
            return reconstruct path from n

        successors = push interval through opposite polygon
        apply pruning

        for each successor:
            compute g, h, f
            if rootHistory says successor root was reached cheaper:
                discard
            else:
                update rootHistory if needed
                push successor

    return no path
```

论文强调 Polyanya 不像普通 A* 那样维护所有 search node 的 closed list。它使用 root history 解决循环和终止问题。

## 十四、Root history

如果 navigation mesh 存在环，并且 `s` 与 `t` 不连通，搜索可能不断生成 `g` 更大的节点。

论文借鉴 Anya，使用 root history：

```text
bestG[root] = 已知到达该 root 的最小 g
```

当准备生成一个新节点时：

```text
if g(newNode) worse than bestG[root(newNode)]:
    discard newNode
else:
    keep it
```

root history 的作用不是替代 closed list，而是阻止同一个 root 被越来越差的路径反复访问。

为什么可以这么做：

- root 是路径最后共同转折点。
- 如果同一个 root 已经有更短路径前缀，那么更长的前缀不可能导出更短的完整路径。

这和 Dijkstra/A* 中“到同一离散节点的劣路径可丢弃”类似，但 Polyanya 的离散身份不是 interval，而是 root。

## 十五、Pruning 策略

### 1. Cul-de-sac pruning

cul-de-sac 是没有 successors 且不包含 target 的节点。

这种节点不可能到达目标，可以直接剪掉。

如果一个节点 push 进入 obstacle，自然不会产生 traversable successor，因此是 trivially cul-de-sac。

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

如果一个节点展开后只有一个 successor，并且该 successor 不包含 target，则可以不把 successor 放入 open list，而是立即继续展开它。

这类似 Jump Point Search 中跳过中间强制无分支节点。

论文还把这个策略推广到 non-observable successor：只要当前节点只有一个 successor，就可以连续推进，直到：

- 分支数增加。
- 到达 obstacle。
- 到达 target。
- 触发其他终止或剪枝条件。

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

因此恢复路径时不能简单收集每个节点的 interval。正确思路是：

1. 从目标节点沿 `parent` 回溯。
2. 收集 root 发生变化的节点。
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

在 root history 和连通性检查的配合下，搜索不会无限循环。若路径存在，最终会展开包含 target 的节点。

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

这也是 Recast/Detour 输出 convex polygons 对 Polyanya 有潜在价值的原因。

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
| 最优性 | corridor 内最优 | navmesh 上全局最优 |
| 实现复杂度 | 较低 | 较高 |
| 对 mesh 要求 | convex polygons | convex polygons，且需要更精细的 interval successor |

如果要把 Polyanya 接到 RecastNavigation 中，它更像是替换或扩展 `dtNavMeshQuery::findPath()` 的查询算法，而不是替换 Recast 构建流程。

## 二十一、实现数据结构建议

### 1. Mesh 表示

推荐使用 half-edge 或 winged-edge 结构：

```text
Vertex:
    position
    incident edges / polygons
    isCorner

Edge:
    v0, v1
    leftPoly, rightPoly
    isObstacleBoundary

Polygon:
    vertices in CCW order
    edges in CCW order
    neighbour polygons
```

必须能快速回答：

- 某个 point 落在哪个 polygon。
- 某条 edge 的另一侧 polygon 是谁。
- 某个 edge 子段是否是 obstacle boundary。
- 某个 vertex 是否 corner point。

### 2. Search node

```cpp
struct PolyanyaNode
{
    Point left;
    Point right;
    EdgeId edge;
    PolyId nextPoly;
    Point root;
    float g;
    float h;
    float f;
    NodeId parent;
};
```

实际实现中 `root` 最好既有坐标，也有可比较的 identity：

- start root 是特殊 id。
- corner root 可以用 vertex id。
- target singleton 可用特殊标记。

这样 root history 可以稳定 key。

### 3. Priority queue

open list 按 `f` 从小到大弹出。

如果 `f` 相等，常见 tie-break 可以是：

- 较小 `h` 优先，更接近目标。
- 较大 `g` 优先，更深入搜索。
- 稳定 insertion order，方便调试。

论文不依赖某个 tie-break 保证正确性，但 tie-break 会影响扩展节点数。

### 4. Geometry robustness

实现中必须谨慎处理浮点误差：

- 点在线段上。
- ray 与 edge 几乎平行。
- target 恰好在 interval 端点。
- start 在 edge 或 vertex 上。
- interval 长度接近 0。
- polygon 共享边方向相反。

建议统一使用 epsilon：

```text
EPS = 1e-5 或按场景尺度设置
```

并且所有 interval 都保持规范方向，例如按 polygon perimeter 的方向保存 left/right。

## 二十二、Successor 生成的实现思路

下面是一个工程化的 high-level 版本。

```text
expand(node):
    Y = node.nextPoly

    if Y is invalid:
        return no successors

    if target inside Y:
        generate target singleton if reachable through node interval

    cone = rays root->intervalLeft and root->intervalRight
    projected = intersect cone with perimeter(Y)

    successors = []

    for each edge segment S on perimeter(Y) covered by projected:
        if S is not the entry interval:
            successors.add(observable interval S, same root)

    for each side outside projected that is reachable only via interval endpoint:
        endpoint = intervalLeft or intervalRight
        if endpoint is corner point:
            successors.add(non-observable interval, root = endpoint)

    return maximal successors
```

关键细节：

1. 不要把 entry interval 原样返回，否则会产生回退。
2. projected region 要按 polygon perimeter 切到各条 edge 上。
3. 如果 covered segment 跨越 vertex，需要拆成多个 edge interval。
4. non-observable successor 必须检查 endpoint 是否 corner。
5. 生成后要合并同 edge、同 root、相邻或重叠的 interval，保证 maximal。

## 二十三、复杂度和性能来源

论文实验显示 Polyanya 在多种 mesh 上通常比 TA*、TRA* 等方法快，并且保持最优。

性能来源主要有：

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

## 二十四、常见误解

### 1. Polyanya 不是 funnel 的替代品吗

它可以直接生成最短折线路径，因此不需要先得到 corridor 再 funnel。

但从几何思想看，Polyanya 和 funnel 都在利用可见通道。区别是：

- funnel 在已知 corridor 内拉直。
- Polyanya 在搜索过程中同时决定 corridor 和可见 interval。

### 2. interval endpoint 都能成为 root 吗

不能。只有 corner point 才需要成为 non-observable root。

如果把所有 mesh vertex 都当 root，算法仍可能找到正确路径，但会产生大量冗余节点，甚至破坏论文中的效率假设。

### 3. Polyanya 能处理任意 3D navmesh 吗

原论文讨论的是二维平面 navigation mesh。

Recast/Detour 常用于 2.5D 游戏场景：polygon 有 xz 投影和高度。若直接应用 Polyanya，通常是在 xz 平面上求最短路径，再用 detail mesh 或高度查询贴地。

如果要求真实 3D 曲面上的 geodesic shortest path，Polyanya 原论文的几何假设不够，需要不同算法。

### 4. root history 是 closed list 吗

不是。closed list 通常按搜索节点 identity 去重；root history 按 root 的最优 `g` 剪枝。

同一个 root 可能对应多个不同 interval，这些 interval 不能简单全部合并或关闭。root history 只用于丢弃到同一 root 的更差路径前缀。

### 5. 为什么 target 要做 singleton successor

因为 target 通常在 polygon 内部，不在 edge 上。普通 push 只产生 edge interval。如果不把 target 显式加入 successor，搜索会一直在 mesh edge 上推进，无法自然终止在内部点。

## 二十五、实现检查清单

实现 Polyanya 时可以按下面顺序检查：

1. 所有 polygon 是否 convex。
2. polygon 邻接和 shared edge 方向是否一致。
3. start/target point location 是否稳定。
4. corner point 是否正确标记。
5. interval 是否总是位于 mesh edge 上。
6. node 是否知道下一步 push 的 opposite polygon。
7. observable projection 是否按 polygon perimeter 正确切分。
8. non-observable successor 是否只从 corner endpoint 生成。
9. `g` 是否表示到 root，而不是到 interval。
10. `h_p` 三种几何情况是否都处理。
11. target polygon 是否生成 singleton successor。
12. root history 是否按 root identity 比较。
13. pruning 是否不删除包含 target 的路径。
14. path reconstruction 是否只输出 root change 点和 target。

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
