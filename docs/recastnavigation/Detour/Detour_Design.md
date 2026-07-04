# Detour 实现逻辑详解

本文整理 `Detour` 库的核心实现。`Detour` 的职责不是生成导航网格，而是把已经生成好的多边形网格组织成可查询的运行时数据结构，并提供寻路、射线、局部移动、邻域查询、墙面查询等算法。

相关源码入口：

- `Include/DetourNavMesh.h`：运行时导航网格、瓦片、引用编码、边连接。
- `Include/DetourNavMeshQuery.h`：寻路和空间查询 API。
- `Include/DetourNode.h`：A* 节点池和开放列表。
- `Include/DetourCommon.h`：几何测试、向量运算、随机点和端序工具。
- `Source/DetourNavMeshBuilder.cpp`：把构建期数据打包成 `dtNavMesh` 可加载的 tile data。
- `Source/DetourNavMesh.cpp`：tile 管理、内部和外部连接、off-mesh connection。
- `Source/DetourNavMeshQuery.cpp`：A*、funnel、BV 查询、raycast、局部 Dijkstra 查询等。

校验依据：

- 本文按 `Detour/Include` 和 `Detour/Source` 中的当前源码逐段对齐，重点核对 `dtCreateNavMeshData()`、`dtNavMesh::addTile()`、`dtNavMeshQuery::findPath()`、`findStraightPath()`、`raycast()`、`moveAlongSurface()` 和局部 Dijkstra 查询。
- 官方 API 文档确认 `dtNavMesh` 是“基于凸多边形 tile 的 navigation mesh”，solo mesh 只是单 tile navmesh；`dtNavMeshQuery` 是基于该 navmesh 的路径和空间查询接口。
- 官方集成文档说明 Detour 是运行时 navmesh interface 和 query system，通常可以独立作为 game runtime 依赖使用。
- 官方社区讨论中，Mikko Mononen 明确指出 Recast/Detour 对“直立角色 + 重力方向”有大量假设，不能把不同 up axis 的 navmesh 自动连接到一起。这意味着 Detour 的查询虽保存 y 高度，但不是任意 3D 曲面导航系统。

参考链接：

- https://recastnav.com/classdtNavMesh.html
- https://recastnav.com/classdtNavMeshQuery.html
- https://recastnav.com/md_Docs_2__2__BuildingAndIntegrating.html
- https://groups.google.com/g/recastnavigation/c/43XlbrGVIdM

## 一、整体设计

`Detour` 把导航网格拆成多个 tile，每个 tile 内包含若干凸多边形。运行时查询时，`Detour` 不直接在三角形上搜索，而是在“多边形图”上搜索：

```text
Recast 输出
  rcPolyMesh / rcPolyMeshDetail / off-mesh connections
        |
        v
dtCreateNavMeshData()
  把顶点、多边形、detail mesh、BV tree、off-mesh 数据打成连续内存
        |
        v
dtNavMesh::addTile()
  接管 tile data，重建运行时 link，挂到 tile hash
        |
        v
dtNavMeshQuery
  A* 寻路、funnel 拉直、nearest、raycast、moveAlongSurface、邻域查询
```

这个设计把“生成”和“查询”拆开。生成阶段可以较慢，可以做体素化、区域划分、轮廓提取；查询阶段只读紧凑的 tile 数据和少量临时节点，目标是稳定、低分配、可逐帧调用。

## 根源视角：Detour 到底在维护什么

理解 Detour 最重要的一点是：它维护的不是“几何表面”，而是一张带几何约束的有向图。

```text
几何层：tile 内的顶点、detail triangles、BVTree
拓扑层：dtPoly + dtLink 组成的 polygon graph
语义层：area、flags、dtQueryFilter 决定谁能走、代价是多少
身份层：dtPolyRef / salt 保证运行时引用安全
```

这四层解决的问题不同：

| 层次 | 解决的问题 | 为什么不能混在一起 |
| --- | --- | --- |
| 几何层 | 点是否在面上、高度是多少、边界在哪里 | 几何精度高，但直接在三角形上寻路图太大 |
| 拓扑层 | 从哪个 polygon 能走到哪个 polygon | 搜索只需要连通关系，不需要每次遍历三角形 |
| 语义层 | 这个角色能不能走、走这里贵不贵 | 同一 navmesh 要服务不同 agent，不应复制几何 |
| 身份层 | 引用是否仍指向同一个运行时对象 | tile 可卸载重载，裸指针和数组下标都不安全 |

Detour 的大部分实现都可以归结为：在这四层之间转换时保持不变量。

### 关键不变量

1. **tile data 加载后几何基本不可变。**

   `dtNavMesh` 不会在原 tile data 内原地移动顶点或改 polygon 形状。动态变化通常通过删除 tile、重新构建 tile、再添加 tile 完成。这让查询可以把 tile 当作只读数据，避免并发和缓存一致性问题。

2. **所有可搜索连接都必须进入 `dtLink`。**

   `dtPoly::neis` 只是构建期邻接提示。A*、raycast、wall segment 等运行时逻辑都以 `dtLink` 链表为准。原因是连接不只来自 tile 内邻接，还来自跨 tile portal、off-mesh connection，而且这些连接需要保存 side、partial portal interval 等运行时信息。

3. **polygon ref 是唯一身份，不是内存地址。**

   任何 API 都用 `dtPolyRef` 暴露 polygon。调用者拿到 ref 后可以跨帧保存，但每次使用时 Detour 会通过 salt 校验它是否还有效。

4. **filter 是查询的一部分，不是后处理。**

   A* 扩展邻居时就调用 `passFilter()`，cost 计算时就乘 area cost。这样不可走区域不会进入搜索树，代价偏好也会影响最短路，而不是路径算完后再筛。

5. **coarse polygon 决定拓扑，detail mesh 只补几何细节。**

   寻路不在 detail triangles 上扩展。detail triangles 主要用于高度、最近点和表面贴合。这是 Detour 能保持查询轻量的关键。

6. **查询的拓扑判断主要在 xz 平面上进行。**

   `queryPolygons()`、`findPolysAroundCircle()`、`findPolysAroundShape()`、`raycast()` 等函数的交叉和邻域判断大多把 polygon 投影到 xz 平面。y 值不会让 Detour 变成完整 3D 导航：它主要用于高度、closest point、路径 cost、walkableClimb 附近的最近点偏好等。这个设计和 Recast 的 Y-up / XZ ground plane 假设一致。

## 二、核心数据结构

### 1. `dtNavMesh`

`dtNavMesh` 是整个运行时导航图的容器。

它主要维护：

| 字段 | 作用 |
| --- | --- |
| `m_tiles` | 固定大小的 `dtMeshTile` 数组。tile 不用 `new/delete` 零散分配，而是从这个数组取槽位。 |
| `m_posLookup` | 按 `(tileX, tileY)` 哈希到 tile 链表。用于快速找到某个平面坐标下的所有层。 |
| `m_nextFree` | 空闲 tile 槽位链表。添加 tile 时从这里取，删除 tile 时还回去。 |
| `m_tileBits / m_polyBits / m_saltBits` | 用于把 tile index、poly index、salt 打包成 `dtPolyRef`。 |
| `m_params` | tile 尺寸、原点、最大 tile 数、最大 poly 数等全局参数。 |

`dtNavMesh` 不是简单数组查找。它同时需要解决几个问题：

1. 按世界坐标快速找到 tile。
2. 支持 tile 动态加载、卸载。
3. 卸载后，旧的 polygon 引用不能误指向新 tile。
4. tile 之间要能跨边搜索。

这些都通过“固定槽位 + salt 引用 + 运行时 link”完成。

官方文档强调一个容易忽略的点：所有 `dtNavMesh` 技术上都是 tiled navmesh。所谓 solo mesh 只是初始化时只有一个 tile。这个事实解释了为什么单 tile 路径也走 `dtCreateNavMeshData()`、tile header、tile ref、link free list 和 salt 校验这一套机制，而不是另写一个“非 tile”查询实现。

### 2. `dtPolyRef` 和 `dtTileRef`

`Detour` 的 polygon 引用是一个整数，而不是裸指针。它把三部分打包在一起：

```text
dtPolyRef = [ salt | tile index | polygon index ]
```

设计目的：

- `tile index` 找到 `m_tiles` 中的槽位。
- `polygon index` 找到 tile 内的 `dtPoly`。
- `salt` 防止悬空引用。tile 被删除后 salt 会递增，旧引用虽然 tile index 一样，但 salt 不匹配，会被判无效。

删除 tile 时的关键步骤：

1. 从 `m_posLookup` 哈希链中摘掉 tile。
2. 断开邻接 tile 指向该 tile 的外部 link。
3. 释放或返回 tile 原始数据。
4. 清空 `dtMeshTile`。
5. `salt = salt + 1`，并避免 salt 为 0。
6. 把 tile 槽位放回 free list。

这样，游戏对象或上层系统缓存的旧 `dtPolyRef` 不会悄悄命中一个新 tile。

32 位构建下，`m_tileBits` 和 `m_polyBits` 由 `maxTiles`、`maxPolys` 计算，剩下的位数给 salt。源码要求 salt/tile/poly 总位数不超过 31，保留一位避免有符号转换问题。定义 `DT_POLYREF64` 时会改用固定的 64 位布局，适合非常大的世界，但 32 位 ref 构建出的 tile data 不能和 64 位 ref 混用。

### 3. `dtMeshTile`

`dtMeshTile` 表示一个已加载 tile。它既保存构建期打包出来的数据，也保存运行时重建的连接。

重要字段：

| 字段 | 说明 |
| --- | --- |
| `header` | `dtMeshHeader`，记录 tile 坐标、poly 数、vert 数、BV tree 数、off-mesh 数等。 |
| `verts` | tile 内基础多边形顶点。 |
| `polys` | `dtPoly` 数组，是 A* 搜索的节点图。 |
| `links` | `dtLink` 数组，是运行时邻接边链表。 |
| `detailMeshes/detailVerts/detailTris` | 细节三角形，用于高度、closest point 等几何查询。 |
| `bvTree` | polygon AABB 的量化 BVH，用于 `queryPolygons` 加速。 |
| `offMeshCons` | 跳跃、门、传送点等非普通边连接。 |
| `data` | 一整块原始 tile data。其他指针大多指向这块内存中的偏移。 |

`Detour` 的 tile data 是连续内存。这让加载 tile 时只需要一块内存，并且 cache locality 比许多小对象更好。

### 4. `dtPoly`

`dtPoly` 是寻路图上的节点。

重要字段：

| 字段 | 说明 |
| --- | --- |
| `verts[]` | 多边形顶点索引。 |
| `neis[]` | 每条边的静态邻接信息。内部邻居存 poly index + 1，外部边带 `DT_EXT_LINK` 和 side。 |
| `firstLink` | 运行时 link 链表头。真正搜索时遍历 `dtLink`，不是直接遍历 `neis`。 |
| `flags` | 由上层定义的通行标志，配合 `dtQueryFilter`。 |
| `areaAndtype` | area id 和 polygon 类型，普通地面或 off-mesh connection。 |

`neis` 是构建期的紧凑邻接提示，`links` 是运行时可遍历连接。这样内部邻接、跨 tile 邻接、off-mesh 邻接可以统一进入 A*。

### 5. `dtLink`

`dtLink` 是一条有向邻接边。

字段含义：

| 字段 | 说明 |
| --- | --- |
| `ref` | 连接到的 polygon ref。 |
| `edge` | 当前 polygon 的哪条边。 |
| `side` | 如果跨 tile，记录连接在 tile 的哪一侧。 |
| `bmin/bmax` | 边门户的可通行区间，压缩到 0..255。 |
| `next` | 同一个 polygon 的下一条 link。 |

为什么需要 `bmin/bmax`：

跨 tile 的两个边不一定完全重合，可能只部分重叠。`Detour` 会用 slab overlap 找到重叠区间，并把重叠段压缩成 `bmin/bmax`。之后 `getPortalPoints()`、`raycast()`、`getPolyWallSegments()` 都可以知道真正可穿过的 portal 子段。

## 三、tile data 构建：`dtCreateNavMeshData`

`dtCreateNavMeshData()` 位于 `Source/DetourNavMeshBuilder.cpp`。它的任务是把 Recast 的构建结果转换为 `dtNavMesh::addTile()` 可加载的二进制数据。

### 输入

典型输入来自：

- coarse polygon mesh：顶点、凸多边形、area、flags。
- detail mesh：每个 polygon 内的细节顶点和三角形。
- off-mesh connections：非地面连接。
- tile 坐标、cell size、cell height、walkable 参数。

### 核心步骤

1. 参数校验。

   它会检查顶点数、多边形数、`nvp`、tile 参数等是否合理。`nvp` 是每个 polygon 最大顶点数，Detour 默认和 Recast 的 `DT_VERTS_PER_POLYGON` 对齐。

2. 分类 off-mesh connection 端点。

   off-mesh 连接的起点和终点可能在当前 tile 内，也可能跨 tile。构建器会判断端点落在 tile 的哪一侧，并只存储需要由当前 tile 持有的连接。

   分类不是只看 xz 边界。源码会先从 detail vertices 或基础 poly vertices 中求 tile 的紧高度范围，再按 `walkableClimb` 扩张，用这个高度范围剔除明显不可能接触本 tile 的 off-mesh 起点。只有起点被判定在本 tile 内部的连接才会被当前 tile 存成 off-mesh polygon；终点分类会保存在 `dtOffMeshConnection::side`，用于加载后跨 tile 连接。

3. 统计需要的数组大小。

   构建器会计算：

   - 顶点数量。
   - polygon 数量，包括 off-mesh poly。
   - detail mesh 数据量。
   - BV tree 节点数。
   - link 预留数量。

   注意：link 只是预留空间。真正的 `dtLink` 内容在 `addTile()` 时根据邻接关系重建。

   `maxLinkCount` 来自普通边数、portal 边数和 off-mesh 端点数：普通内部边至少需要一条 link；tile portal 需要为两侧重叠连接预留额外 link；off-mesh 的起点和终点也会创建普通 poly 到 off-mesh poly、off-mesh poly 到普通 poly 的 link。这里预留偏保守，是为了 `addTile()` 不再分配。

4. 分配一整块 aligned 内存。

   tile data 中的 header、verts、polys、detail mesh、BV tree、off-mesh connection 等都顺序放入同一块 buffer。这样 `addTile()` 只需要把指针按偏移修好。

5. 坐标转换。

   Recast 的 poly mesh 顶点通常是体素坐标。构建器会按：

   ```text
   world.x = bmin.x + voxel.x * cs
   world.y = bmin.y + voxel.y * ch
   world.z = bmin.z + voxel.z * cs
   ```

   转换为世界坐标。

6. 创建普通 polygon 和 off-mesh polygon。

   普通 polygon 使用 mesh 的顶点索引和邻接信息。off-mesh connection 会被表示成特殊类型的二顶点 polygon，这让 A* 可以把它当作图中的普通边处理。

7. 写入 detail mesh。

   detail mesh 负责补充高度和细节三角形。寻路图只用 coarse polygon，但 `closestPointOnPoly()`、`getPolyHeight()` 等需要 detail triangles 给出更准确的表面。

   如果输入没有 detail mesh，构建器不会失败，而是为每个 convex polygon 生成一个简化 detail：用 polygon 自身顶点扇形三角化，detail extra vertex 数为 0。这种 fallback 高度只来自 coarse polygon，精度较低，但保证 `dtNavMesh` 的 detail 查询结构始终存在。

8. 可选生成 BV tree。

   如果开启 `buildBvTree`，构建器会为 tile 内 polygon 生成量化 AABB 树。查询 AABB 与 polygon 重叠时，走 BV tree 比逐 polygon 检查更快。

9. 端序转换。

   `dtNavMeshHeaderSwapEndian()` 只交换 `dtMeshHeader`。`dtNavMeshDataSwapEndian()` 假设 header 已是当前端序，然后交换 vertices、polys、detail mesh、detail vertices、BV tree 和 off-mesh connection。`dtLink` 不需要交换，因为加载 tile 时会全部重建；detail triangles 是 byte 数组，也不需要交换。

### 设计重点

构建器做的不是“高级算法寻路”，而是数据布局。它的关键目标是：

- 运行时只读紧凑数据。
- tile 可独立加载。
- tile data 可序列化。
- 查询需要的几何信息都能从 tile data 恢复。

## 四、tile 加载和连接：`dtNavMesh::addTile`

`addTile()` 的职责是把二进制 tile data 挂到运行时 navmesh，并建立可搜索的图连接。

### 1. 验证和占位

流程：

1. 检查 magic/version。
2. 检查同一 `(x, y, layer)` 位置是否已有 tile。
3. 从 free list 取一个 `dtMeshTile`。
4. 如果是恢复历史 tile ref，可以用指定 salt 和槽位；否则用当前空槽。
5. 把 tile 插入 `m_posLookup` 的哈希链。

### 2. 指针修复

tile data 是连续内存。`addTile()` 会根据 header 中的数量，把 `tile->verts`、`tile->polys`、`tile->detailMeshes` 等指针指到 `tile->data` 的正确偏移。

### 3. 构建 link free list

`tile->links` 是数组。`addTile()` 会先把所有 link 组织成空闲链表。之后创建连接时从这个链表取 link，删除连接时还回去。

### 4. 连接 tile 内部 polygon

`connectIntLinks()` 遍历 tile 内所有普通 polygon 的边：

1. 如果某条边的 `neis[j]` 表示内部邻居，则得到邻居 polygon index。
2. 从 link free list 取一个 `dtLink`。
3. `link.ref` 设置为邻居 polygon ref。
4. `link.edge` 设置为当前边 index。
5. 把 link 插到当前 polygon 的 `firstLink` 链表头。

内部连接不需要 portal 区间，因为两个 polygon 共享完整边。

### 5. 连接 off-mesh connection

off-mesh connection 分两步：

1. `baseOffMeshLinks()`：把 off-mesh 起点连接到附近地面 polygon。
2. `connectExtOffMeshLinks()`：把 off-mesh 终点连接到当前 tile 或邻接 tile 的地面 polygon。

核心算法：

- 对 off-mesh 端点做 `findNearestPolyInTile()`。
- 搜索半径使用 connection 自身 radius。
- 如果找到地面 polygon，就创建双向或单向 link。
- off-mesh polygon 自身是图节点，所以 A* 可以进入它，再从另一端出去。

这种设计让跳跃、门、梯子、传送点不需要特殊寻路算法，只是特殊 polygon 类型。

### 6. 连接邻接 tile

跨 tile 连接由 `connectExtLinks()` 完成。

算法步骤：

1. 遍历当前 tile 中所有 polygon 的边。
2. 找出带 `DT_EXT_LINK` 的边，也就是 tile 边界。
3. 根据边的 side 找到邻接 tile。
4. 在邻接 tile 中调用 `findConnectingPolys()`。
5. `findConnectingPolys()` 用 slab overlap 判断两条边是否在边界方向上重叠。
6. 对每个重叠邻居创建 link。
7. 把重叠区间压缩为 `bmin/bmax`。

为什么不用简单“坐标相等”：

- tile 边上的轮廓可能被简化。
- 两边 polygon 边段数量可能不一致。
- 实际通道可能只是部分重叠。

slab overlap 的结果更稳健，可以表达“这条边只有中间一段能穿过”。

## 五、查询过滤：`dtQueryFilter`

`dtQueryFilter` 决定某个 polygon 是否可走，以及走过不同 area 的代价。

默认逻辑：

```text
pass = (poly.flags & includeFlags) != 0
       && (poly.flags & excludeFlags) == 0

cost = distance(pa, pb) * areaCost[poly.area]
```

设计意义：

- 同一份 navmesh 可以支持不同角色，例如人、车、怪物。
- area cost 可以表达泥地、道路、水面等偏好，而不需要改拓扑。
- include/exclude flags 可以快速禁用区域。

如果启用虚函数过滤，用户可以重写 filter 逻辑，但默认实现避免虚调用，查询更轻。

`getCost()` 的输入不是简单的两个 polygon id，而是前一段、当前段、后一段 polygon 及其端点位置。源码这样设计，是为了让 cost 可以表达“从上一个 portal 进入当前 polygon，再从下一个 portal 离开”的局部上下文。默认实现只按当前 polygon area cost 乘距离；自定义 filter 可以利用前后 polygon 做更复杂的通行代价，但需要定义 `DT_VIRTUAL_QUERYFILTER` 才能通过继承覆盖默认逻辑。

## 六、A* 寻路：`dtNavMeshQuery::findPath`

`findPath()` 在 polygon 图上执行 A*。它返回的是 polygon ref 序列，不是空间中的折线路径。折线由后续 `findStraightPath()` 计算。

### 1. 临时结构

A* 使用两个结构：

| 结构 | 作用 |
| --- | --- |
| `dtNodePool` | 固定容量节点池，用 hash 从 `dtPolyRef` 找 `dtNode`。 |
| `dtNodeQueue` | 二叉堆开放列表，按 `node.total` 从小到大弹出。 |

`dtNode` 存储：

- `id`：polygon ref。
- `pos`：该节点代表的位置，通常是 portal 中点。
- `cost`：从起点到当前节点的实际代价。
- `total`：`cost + heuristic`。
- `pidx`：父节点在 node pool 中的索引，1-based。
- `flags`：open/closed 状态。

### 2. 初始化

1. 清空 node pool 和 open list。
2. 为 `startRef` 创建起点 node。
3. 起点位置设为 `startPos`。
4. `cost = 0`。
5. `total = distance(startPos, endPos) * H_SCALE`。
6. 把起点推入 open list。

`H_SCALE` 小于或接近 1，避免启发式过强导致不稳定。启发式是到终点的欧氏距离。

### 3. 主循环

每轮：

1. 从 open list 弹出 `total` 最小的 node。
2. 标记为 closed。
3. 如果 node 是 `endRef`，搜索结束。
4. 遍历当前 polygon 的所有 `dtLink`。
5. 对每个邻居：
   - 跳过无效 ref。
   - 跳过父 polygon，避免立即回头。
   - 通过 `dtQueryFilter::passFilter()` 检查通行。
   - 计算从当前 node 到邻居的 portal 代价。
   - 计算 heuristic。
   - 如果新代价更优，更新或插入邻居 node。

代价不是简单“polygon 中心到 polygon 中心”。`Detour` 会通过 `getEdgeMidPoint()` 等逻辑把节点位置放在跨边的 portal 附近，让代价更接近实际路径长度。

### 4. 找不到终点时的部分路径

如果 open list 耗尽仍未到达终点，`findPath()` 会返回“离终点启发式最近”的节点链，并带 `DT_PARTIAL_RESULT`。

这样上层能得到一条尽量接近目标的路径，而不是完全失败。对于动态障碍、局部 tile 缺失、目标不可达，这很实用。

官方 API 文档还说明两个细节：

- 如果 `endRef` 通过导航图不可达，返回 path 的最后一个 polygon 会是搜索中最接近终点的 polygon。
- 如果调用者提供的 path buffer 太小，结果会从起点开始尽量填满，并通过 status flag 表达 buffer 不足。

源码中的 `lastBestNode` 和 `lastBestNodeCost` 正是为第一个细节服务。它不是“随便返回最后扩展的节点”，而是持续记录启发式意义上离 `endPos` 最近的已发现节点。

### 5. 路径回溯

找到终点或 best node 后：

1. 沿 `pidx` 从目标 node 回到起点。
2. 反转节点顺序。
3. 输出 polygon ref 数组。

返回结果仍是 corridor，不是最终转角列表。

## 七、分片寻路：sliced find path

`initSlicedFindPath()`、`updateSlicedFindPath()`、`finalizeSlicedFindPath()` 把 A* 拆成多帧执行。

它保存：

- 起点、终点、filter。
- open list。
- node pool。
- best node。
- 当前状态。

`updateSlicedFindPath(maxIter)` 每次只扩展有限数量节点。这样 crowd 或游戏主循环可以避免某一帧因为长路径搜索卡顿。

`finalizeSlicedFindPathPartial(existingPath, ...)` 还有一个额外用途：如果 agent 正在沿旧 corridor 移动，新搜索可以尽量和旧路径合并，避免路径突然大幅跳动。

分片搜索还支持 `DT_FINDPATH_ANY_ANGLE`。启用后，`updateSlicedFindPath()` 会在扩展邻居时尝试从父节点向邻居做 `raycast()`，如果 line of sight 成立，就把邻居的父节点改成祖先节点，并用 raycast 的路径代价更新 cost。源码用 `DT_NODE_PARENT_DETACHED` 标记这种“父子不直接相邻”的节点；`finalizeSlicedFindPath()` 回溯时会再次 raycast，把被跳过的 polygon 补回输出 corridor，避免输出断开的 ref 序列。

## 八、拉直路径：`findStraightPath` 和 funnel 算法

A* 返回 polygon 序列后，角色不能按 polygon 中心走，需要得到可行的转角点。`findStraightPath()` 使用 funnel，也叫 string pulling。

### 1. 输入

- `startPos`
- `endPos`
- polygon path

对相邻两个 polygon，`Detour` 可以求出它们之间的 portal，也就是共享边或 off-mesh connection 端点。

### 2. funnel 状态

算法维护三个点：

| 名称 | 说明 |
| --- | --- |
| `portalApex` | 当前漏斗顶点，已经确定路径会经过这里。 |
| `portalLeft` | 当前可见通道的左边界。 |
| `portalRight` | 当前可见通道的右边界。 |

初始时三者都在 `startPos`。

### 3. 遍历 portal

对每个 portal：

1. 得到该 portal 的左右端点。
2. 尝试收紧右边界。
   - 如果新的右端点仍在漏斗内，就更新 `portalRight`。
   - 如果右端点跨过了左边界，说明漏斗闭合，必须输出左边界作为转角。
3. 尝试收紧左边界。
   - 如果新的左端点仍在漏斗内，就更新 `portalLeft`。
   - 如果左端点跨过了右边界，说明漏斗闭合，必须输出右边界作为转角。
4. 输出转角后，把 apex 移到该转角，并从对应 portal 重新开始。

### 4. 为什么 funnel 正确

polygon path 给出一条由 convex polygon 组成的通道。任意可行路径都必须依次穿过这些 portal。funnel 维护“从 apex 能直接看见的所有未来 portal 的交集”。当左右边界交叉时，说明已经没有直线可以继续穿过后续 portal，交叉前的边界点就是最早必须转弯的位置。

这比沿 polygon 中心走短得多，也避免了在宽阔区域里出现不必要折线。

### 5. off-mesh connection 处理

如果路径中遇到 off-mesh connection，`findStraightPath()` 会输出带 `DT_STRAIGHTPATH_OFFMESH_CONNECTION` 的点。上层可以在这个点触发跳跃、爬梯、传送等动画。

`findStraightPath()` 还支持两个 crossing 选项：

- `DT_STRAIGHTPATH_AREA_CROSSINGS`：只在 area 变化的 portal 上插入额外点。
- `DT_STRAIGHTPATH_ALL_CROSSINGS`：每个 polygon portal 都插入额外点。

这两个选项通过 `appendPortals()` 实现。它会拿当前 straight segment 和每个 portal 求 2D 交点，把交点作为 straight path 点写出。默认不启用时，funnel 只输出真正需要转弯的点。

## 九、空间查询：`queryPolygons` 和 BV tree

`queryPolygons(center, halfExtents, filter, ...)` 用 AABB 找可能重叠的 polygons，常用于：

- `findNearestPoly()`
- agent 放置
- 局部邻域搜索
- tile cache 障碍物影响范围

源码有两个 public 版本：一个把结果写入 `dtPolyRef` 数组，另一个接收 `dtPolyQuery` 回调。回调版本按最多 32 个 polygon 一批调用 `process()`，可以避免调用者必须提供一个足够大的结果数组，适合流式处理大范围查询。

### 1. tile 范围计算

先根据 world AABB 和 tile size，算出 AABB 覆盖哪些 tile 坐标。然后从 `m_posLookup` 查这些 tile。

### 2. tile 内查询

如果 tile 有 BV tree：

1. 把查询 AABB 量化到 tile 的 BV 坐标系。
2. 从 BV tree 根节点开始遍历。
3. 如果当前 node 的 bounds 与查询 AABB 重叠：
   - 如果是 leaf，取 polygon。
   - 如果是 internal node，继续下一个 node。
4. 如果不重叠：
   - 如果是 internal node，使用 negative escape index 跳过整个子树。
   - 如果是 leaf，跳过一个节点。

如果没有 BV tree，就遍历 tile 内所有 polygon，计算 polygon AABB 后测试重叠。

### 3. 为什么 BV node 用 escape index

BV tree 存在数组里，而不是指针树。内部节点如果不重叠，可以通过 escape index 一次跳过整个子树。这减少分支和内存访问，适合只读查询。

## 十、最近点：`findNearestPoly`

`findNearestPoly()` 的逻辑：

1. 调 `queryPolygons()` 找 AABB 范围内候选 polygon。
2. 对每个候选调用 `closestPointOnPoly()`。
3. 计算候选最近点到输入点的距离。
4. 选择距离最小者。

特殊点在于高度：

- 如果点在 polygon 水平投影内，会用 detail mesh 求高度。
- 如果点在边界外，会求 polygon 边界上的最近点。
- 如果点投影在 polygon 内，距离评估会扣掉 `walkableClimb`。这让垂直方向上略高或略低、但仍在可爬范围内的 polygon 更容易被选中。

因此它不只是 2D 最近 polygon，还会考虑 navmesh 表面高度。

相关的点投影 API 有三个：

- `closestPointOnPoly()`：使用 detail mesh，最准确；点在 polygon 外时先夹到边界，再用 detail triangles 求高度。
- `closestPointOnPolyBoundary()`：只用 coarse polygon 边界，速度更快；点在 xz 投影内时直接返回输入点，不查询 detail 高度。
- `getPolyHeight()`：要求输入点在 polygon xz 范围内，用 detail triangles 求高度；off-mesh connection 会按二端点线段插值高度。

## 十一、局部移动：`moveAlongSurface`

`moveAlongSurface(startRef, startPos, endPos, ...)` 用于“把移动限制在 navmesh 表面上”。它不是完整寻路，而是短距离局部搜索。

典型用途：

- crowd 每帧移动后把 agent 约束回 navmesh。
- corridor 维护当前位置。
- 鼠标拖动目标时保持目标在可走区域。

算法流程：

1. 从 `startRef` 开始，把起点 polygon 放入一个小队列。
2. 用 `startPos` 到 `endPos` 的线段和一个局部半径定义搜索范围。
3. 遍历邻接 polygon。
4. 只扩展与移动段附近区域重叠的 polygon。
5. 如果 `endPos` 落在某个已访问 polygon 内，则返回 end。
6. 如果不能直接到达 end，则在访问到的 polygon 边界上找离 end 最近的点。
7. 输出 visited polygon 列表。

这个 visited 列表会被 `dtPathCorridor::movePosition()` 用来合并 corridor 开头，让 agent 即使因为避障稍微偏离原 corridor，也能把路径修回来。

## 十二、射线：`raycast`

`raycast()` 在 navmesh 上沿 2D 线段前进，报告是否撞到墙，以及经过哪些 polygon。

算法步骤：

1. 从起始 polygon 开始。
2. 在当前 polygon 内调用 `dtIntersectSegmentPoly2D()`，求线段从哪条边离开。
3. 如果没有离开边，说明终点在当前 polygon 内，成功。
4. 如果离开边是墙，返回 hit：
   - `t` 是碰撞在线段上的比例。
   - `hitNormal` 是墙法线。
5. 如果离开边有 link：
   - 检查 filter。
   - 如果是跨 tile partial portal，还要检查穿越点是否落在 `bmin/bmax` 区间内。
   - 进入邻接 polygon，继续循环。

它和物理引擎射线不同：它只在 navmesh 可走表面上穿行，适合做可视优化、简单 line of sight、局部路径截短。

## 十三、邻域 Dijkstra 查询

`Detour` 有一组从某个 polygon 出发的局部 Dijkstra 搜索：

- `findPolysAroundCircle()`
- `findPolysAroundShape()`
- `findLocalNeighbourhood()`
- `findDistanceToWall()`
- `findRandomPointAroundCircle()`

共同特征：

1. 使用 node pool 和队列。
2. 从起点 polygon 开始按代价扩展。
3. 只遍历通过 filter 的 link。
4. 用几何测试限制搜索范围，例如 circle、convex shape、wall distance。

### `findPolysAroundCircle`

这个函数找圆形范围内可到达的 polygons。它不是简单 AABB 查询，而是考虑拓扑连通：

1. 从起点入队，代价为 0。
2. 弹出当前代价最小 polygon。
3. 遍历 portal。
4. 判断 portal segment 是否与圆相交或在圆内。
5. 如果相交，计算到邻居的代价并入队。
6. 输出 polygon、父 polygon、累计 cost。

所以结果是“从起点不穿墙能到达的圆形邻域”，不是“空间上和圆重叠的所有 polygon”。

### `findPolysAroundShape`

这个函数和 circle 版本结构相同，但限制条件从“portal 是否触碰圆”换成“portal segment 是否和凸多边形重叠”。输入 shape 必须是 xz 平面上的 CCW convex polygon。搜索代价仍通过 `filter->getCost()` 计算，输出也包含 result refs、parent refs 和累计 cost。

### `getPathFromDijkstraSearch`

`findPolysAroundCircle()` 和 `findPolysAroundShape()` 都会把搜索树留在 `dtNodePool` 中。`getPathFromDijkstraSearch(endRef, ...)` 依赖这个内部状态，沿 `pidx` 回溯出从起点到某个已探索 polygon 的路径。它只能紧跟在 Dijkstra 查询之后调用；任何会清空 node pool 的查询都会破坏这个上下文。

### `findLocalNeighbourhood`

这个函数类似局部 flood fill，但会避免重叠 polygon 的重复扩展，适合 crowd 的局部边界采样。它返回附近 polygon 集合，后续可以调用 `getPolyWallSegments()` 收集墙段。

### `findDistanceToWall`

这个函数在半径范围内找最近墙面：

1. 从起点 polygon 扩展。
2. 对每个 polygon 的边判断是否为墙。
3. 对墙段求点到线段距离。
4. 如果找到更近墙，更新距离、hitPos、hitNormal。
5. 只扩展仍可能存在更近墙的邻接 polygon。

它常用于 steering 或 debug，回答“当前位置离最近不可通行边界多远”。

### `findRandomPoint` 和 `findRandomPointAroundCircle`

`findRandomPoint()` 先用 reservoir sampling 随机选一个已加载 tile，再在该 tile 内按 polygon 面积加权选择 ground polygon，最后用 `dtRandomPointInConvexPoly()` 在 polygon 内采样，并调用 `closestPointOnPoly()` 贴到表面。

`findRandomPointAroundCircle()` 则从 `startRef` 做局部 Dijkstra，只访问通过 filter 且 portal 与搜索圆相交的 polygon，再按 polygon 面积 reservoir sampling。注意源码注释明确说：结果受可达 polygon 集合限制，但最终随机点不保证严格落在圆内，因为采样是在被访问 polygon 内进行。

这些局部查询有一个共同的坐标语义：相交测试在 xz 平面中完成，但 cost 计算使用传入点的 3D 坐标，y 值会影响距离代价。官方 API 文档对 circle、shape、local neighbourhood 都强调了这一点。工程上不要把它们当作纯 2D 查询，也不要把它们当作完整 3D 空间查询；它们是“投影判定 + 表面代价”的 navmesh 查询。

## 十四、墙段枚举：`getPolyWallSegments`

`getPolyWallSegments()` 返回某个 polygon 的墙边或 portal 边。

关键逻辑：

- 如果边没有可通过 link，输出为 wall segment。
- 如果边有通过 filter 的 link，默认可以跳过，或在需要 portal 信息时输出 portal segment。
- 对跨 tile partial portal，使用 `dtLink::bmin/bmax` 把边裁剪成真正可通行的子区间。

因此它能正确表达：

- 完整墙。
- 完整通道。
- 只有一部分重叠的跨 tile 通道。

Crowd 的 `dtLocalBoundary` 会用它收集附近墙段，交给 obstacle avoidance 作为 segment obstacles。

## 十五、内存和状态设计

`Detour` 很少在查询过程中临时分配内存。常见模式是：

- 初始化时创建 `dtNavMeshQuery(maxNodes)`。
- 每次查询复用 node pool 和 open list。
- 输出数组由调用者提供。
- 返回 `dtStatus`，用 bit flag 同时表达成功、失败、进行中、部分结果、buffer 不足等。

这种设计适合游戏循环，因为：

- 分配次数可控。
- 查询耗时可估计。
- 失败原因可以通过 status detail 判断。

`dtNavMesh` 还有一组“非结构状态”接口：

- `setPolyFlags()` / `getPolyFlags()`
- `setPolyArea()` / `getPolyArea()`
- `getTileStateSize()`
- `storeTileState()`
- `restoreTileState()`

这里的状态只包含每个 polygon 的 flags 和 area id，不包含顶点、polygon 形状、links、BV tree 或 off-mesh connection。`storeTileState()` 会写入 `DT_NAVMESH_STATE_MAGIC`、版本号和当前 tile ref；`restoreTileState()` 会检查 magic、version 和 tile ref 是否完全匹配。这样可以保存运行时门禁、代价区域等轻量状态，同时不改变 `dtTileRef` 和 `dtPolyRef`。

`dtNavMeshQuery` 的杂项接口也围绕内部搜索状态设计：

- `isValidPolyRef(ref, filter)`：先用 navmesh salt/ref 校验取出 tile/poly，再检查 filter。Crowd 用它判断 corridor 和 local boundary 是否仍可用。
- `isInClosedList(ref)`：查询上一次 A* 或 Dijkstra 中某个 ref 是否进入 closed list。它依赖当前 node pool 内容，所以只适合紧跟搜索之后做调试或局部逻辑。
- `getNodePool()` / `getAttachedNavMesh()`：暴露 query 当前复用的 node pool 和 navmesh，主要给 Crowd、PathCorridor 或调试代码使用。

## 十六、源码阅读路径和因果链

如果想从源码根源理解 Detour，可以按下面顺序读。这个顺序和运行时数据的生命线一致，比按文件名读更容易建立整体模型。

### 1. 从 `dtCreateNavMeshData()` 看数据如何被固化

重点看它如何把输入拆成几类数据：

```text
dtMeshHeader
dtPoly vertices/neighbours/flags/area
detail meshes
BVTree
off-mesh connections
预留 links 空间
```

读这段代码时要注意两个问题：

1. 哪些数据是构建期就能确定的。
2. 哪些数据必须等 tile 加入 navmesh 后才能确定。

例如 tile 内部 polygon 邻接可以在构建期写入 `neis`，但跨 tile link 不能，因为相邻 tile 可能尚未加载。于是构建器只写“这是外部边、在某个 side”，真正邻居由 `addTile()` 阶段连接。

### 2. 从 `dtNavMesh::addTile()` 看拓扑如何活起来

`addTile()` 是从“静态数据”到“可搜索图”的分界线。源码里值得重点跟踪：

- tile 槽位如何从 free list 取出。
- tile 如何进入 `(x, y)` 位置 hash。
- tile data 内的偏移如何恢复成指针。
- link free list 如何建立。
- `connectIntLinks()` 如何把内部邻接变成 `dtLink`。
- `connectExtLinks()` 如何发现邻接 tile 的 portal。
- `baseOffMeshLinks()` 和 `connectExtOffMeshLinks()` 如何把 off-mesh 连接挂入图。

一旦这些步骤完成，A* 看到的就是统一的 `dtLink` 图，不再关心连接来自内部边、tile 边，还是 off-mesh connection。

### 3. 从 `findPath()` 看拓扑搜索

读 `findPath()` 时，不要把它理解成“在空间里找线”。它其实在 polygon 图上做：

```text
当前 poly
  -> 遍历 dtLink
  -> filter 检查
  -> cost + heuristic
  -> node queue
```

也就是说，它只回答“应该经过哪些 polygon”。为什么不直接输出转角？因为在同一串 polygon corridor 中，最短折线取决于 portal 几何，需要 funnel 单独处理。把搜索和拉直拆开后，A* 的状态更小，funnel 也可以被 corridor、Crowd 等系统复用。

### 4. 从 `findStraightPath()` 看几何约束如何回到路径

`findStraightPath()` 把 polygon corridor 还原成几何折线。这里要重点看 `getPortalPoints()`：

- 内部 polygon 边给出共享边两端点。
- 跨 tile link 会应用 `bmin/bmax` 裁剪 portal。
- off-mesh connection 会输出特殊点和 flag。

所以 funnel 不是在抽象图上运行，它是在“每一步拓扑连接对应的实际门洞”上运行。

### 5. 从 `moveAlongSurface()` 看局部移动为什么不等于 A*

`moveAlongSurface()` 的目标不是找全局最短路，而是回答：

```text
从当前 poly 出发，在小范围内，agent 想移动到 npos，最后合法位置在哪里？
```

它会返回 visited corridor 片段。这个片段的价值不只是位置结果，还能让上层把 corridor 头部修正到 agent 的真实移动轨迹。这正是 Crowd 能在避障后仍维护路径的基础。

## 十七、常见误解和设计原因

### 1. 为什么不直接在三角形网格上 A*

原始三角形数量通常远大于导航 polygon 数量，而且很多三角形只表达渲染细节。直接在三角形上 A* 会导致节点多、路径抖、转角碎。Recast 先把可走空间归并成凸 polygon，Detour 再在 polygon 图上搜索，这才是运行时高效的原因。

### 2. 为什么 `findPath()` 返回 partial path 仍算成功

运行时世界经常不完整：tile 可能没加载，动态障碍可能切断路径，目标可能在不可达岛上。返回离目标最近的可达 corridor 可以让 agent 先朝合理方向移动或给上层反馈，而不是完全没有结果。

### 3. 为什么 tile 边连接要保存 portal 子区间

相邻 tile 的边界轮廓可能不同步，例如一侧是一条长边，另一侧被拆成几条短边。如果只记录“这条边连到邻居 polygon”，raycast 和 funnel 会以为整条边都可通过。`bmin/bmax` 保存的是真实重叠通道，能防止路径穿过不存在的门洞。

### 4. 为什么 height 查询依赖 detail mesh

coarse polygon 通常是简化后的平面边界，它不一定准确表达地形起伏。如果 agent 只贴 coarse polygon，高度会悬空或穿地。detail mesh 保留了更细的三角面，既不增加 A* 节点，又能让位置查询贴近真实表面。

### 5. 为什么查询对象有固定 node pool

游戏运行时最怕隐式分配和不可控耗时。`dtNavMeshQuery` 初始化时给定最大节点数，之后查询只在池内工作。如果节点不够，返回 `DT_OUT_OF_NODES` 或 partial result。这是显式容量模型，调用者可以按项目规模调大，而不是让库偷偷扩容。

### 6. 为什么 Detour 不是任意 3D 曲面导航

Detour 的 polygon 可以有 y 高度，detail mesh 也能给出表面高度，但核心邻接、portal、raycast、邻域交叉大量使用 xz 投影。官方社区讨论中也明确说明 Recast/Detour 有直立角色和重力方向假设，不能把墙面、天花板或不同 up axis 的 navmesh 自动接成一个统一导航系统。

如果项目需要侧滚、墙面行走或球面世界，常见做法是把“仿真空间”变换到 Recast/Detour 期望的 XZ 地面空间中运行，然后把结果再变换回渲染/游戏空间；但这仍然不能解决多个不同 up axis navmesh 的自动连接问题。

## 十八、设计取舍总结

1. **以 polygon 为图节点，而不是以三角形或格子为节点。**

   图更小，A* 更快，路径更平滑。代价是需要 funnel 和 detail mesh 辅助几何查询。

2. **tile 化。**

   支持大世界流式加载和局部重建。代价是跨 tile 连接要额外处理，尤其 partial portal。

3. **整数 ref + salt。**

   上层可以安全缓存 polygon ref。代价是最大 tile/poly 数受 bit 分配限制。

4. **连续内存 tile data。**

   序列化和 cache locality 好。代价是构建时要精确计算大小，加载时要修指针。

5. **查询和过滤解耦。**

   同一 navmesh 可以服务多个角色。代价是 filter 需要贯穿几乎所有查询函数。

6. **A* 输出 corridor，funnel 输出转角。**

   这让寻路和几何拉直分工清晰。corridor 还能被 Crowd 持续维护，而不是每帧完整重算路径。
