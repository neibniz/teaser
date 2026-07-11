# Recast 实现逻辑详解

本文整理 `Recast` 核心库的设计和实现。`Recast` 的职责是把输入三角形场景转换成适合运行时寻路的导航多边形网格。它不负责运行时 A* 查询，运行时查询由 `Detour` 完成。

相关源码入口：

- `Include/Recast.h`：公共数据结构、配置、构建 API。
- `Source/Recast.cpp`：对象分配、基础工具、heightfield 创建、三角形坡度标记、compact heightfield 构建。
- `Source/RecastRasterization.cpp`：三角形体素化和 span 合并。
- `Source/RecastFilter.cpp`：低障碍、悬崖、净空过滤。
- `Source/RecastArea.cpp`：半径腐蚀、area 标注、area 中值滤波。
- `Source/RecastRegion.cpp`：距离场、watershed/monotone/layer 区域分区、区域合并过滤。
- `Source/RecastContour.cpp`：轮廓追踪、简化、洞合并。
- `Source/RecastMesh.cpp`：轮廓三角化、凸多边形合并、邻接构建。
- `Source/RecastMeshDetail.cpp`：detail mesh 高度采样和三角化。
- `Source/RecastLayers.cpp`：为 tile cache 构建 heightfield layers。

校验依据：

- 本文按仓库提交 `9f4ce64` 的 `Recast/Include` 和 `Recast/Source` 核对，覆盖所有公共构建阶段及其核心内部算法：rasterization、三类 filter、compact 邻接、area 操作、distance field、三种 region partition、heightfield layers、contour、poly mesh、detail mesh、分配/日志/计时和合并复制接口。
- 官方 Recast 模块文档把 Recast 定位为生成 Detour navmesh 输入数据的构建系统，并列出典型 pipeline：准备三角形、构建 heightfield、compact heightfield、contours、poly mesh、detail mesh，再交给 Detour 打包。
- 官方 `rcConfig` 文档强调单位混用关系：`cs/ch` 定义体素尺度，很多参数使用 voxel 或 world unit；先决定 agent 的逻辑圆柱尺寸，再换算构建参数。
- 官方社区讨论补充了几个源码外的设计动机：Recast/Detour 深度假设 Y-up / XZ ground plane；常见做法是把 agent radius 烘焙进 navmesh，而不是让运行时 pathfinding 处理半径；`rcMarkConvexPolyArea()` 用 xz 投影再按高度挤出，是为了粗略标注道路、泥地等区域，常常足够但并非精确实体几何布尔。
- 官方 FAQ 还明确了输入合同：右手 Y-up 坐标系、三角形顺时针 winding。坡度标记只看有向法线的 y 分量，winding 错误会把本应可走的地面法线翻到负 y。

参考链接：

- [Recast 官方模块文档](https://recastnav.com/group__recast.html)
- [官方 Introduction：完整构建 pipeline](https://recastnav.com/md_Docs_2__1__Introduction.html)
- [`rcConfig` 官方 API](https://recastnav.com/structrcConfig.html)
- [官方 FAQ：坐标系、winding、内存与日志](https://recastnav.com/md_Docs_2__3__FAQ.html)
- [官方集成文档](https://recastnav.com/md_Docs_2__2__BuildingAndIntegrating.html)
- [官方论坛：agent radius 烘焙进 navmesh](https://groups.google.com/g/recastnavigation/c/XIjvm79w0U4)
- [官方论坛：ground plane 与垂直堆叠](https://groups.google.com/g/recastnavigation/c/s3s2xMi5TTI)
- [官方论坛：`rcMarkConvexPolyArea()` 的投影体积语义](https://groups.google.com/g/recastnavigation/c/m_JMXgmlIs4)
- [官方论坛：不同 up axis 与墙面/天花板限制](https://groups.google.com/g/recastnavigation/c/43XlbrGVIdM)
- [官方论坛：常规 tile 重建应收集所有触碰几何](https://groups.google.com/g/recastnavigation/c/F9mRO_eimEI)

源码覆盖关系：

| 源码 | 本文对应内容 |
| --- | --- |
| `Recast.cpp` / `RecastRasterization.cpp` | 对象生命周期、bounds/grid、坡度标记、heightfield、三角形裁剪、span 合并和 compact 邻接 |
| `RecastFilter.cpp` / `RecastArea.cpp` | 低障碍、ledge、净空、半径腐蚀、中值滤波、box/cylinder/convex area 标注与 polygon offset |
| `RecastRegion.cpp` / `RecastLayers.cpp` | distance transform、watershed/monotone/layer partition、区域过滤合并与可压缩 heightfield layers |
| `RecastContour.cpp` | 边界追踪、强制 portal 顶点、误差简化、长边切分、winding 和洞合并 |
| `RecastMesh.cpp` | 顶点去重、ear clipping、凸合并、边界点删除、邻接、mesh 合并/复制 |
| `RecastMeshDetail.cpp` | height patch、边界和内部误差采样、Delaunay-like triangulation、detail 合并和固定上限 |

## 一、整体定位

Recast 的核心目标不是“保留原始几何”，而是从复杂三角形场景中抽取出 agent 可通行的拓扑结构。

![Recast 从输入三角形到 detail mesh 的完整构建流水线](https://oss.euler.icu/teaser/recastnavigation/Recast/Docs/Images/recast-build-pipeline.png)

> **图 1：Recast 构建流水线及同一场景在各阶段的空间表达。** 灰色表示输入或不可走实体，青色表示可走表面，蓝色表示 coarse polygon，紫色表示 region contour。图中把过滤、腐蚀和 area 标注归入体素到 compact span 的语义转换；源码中的完整顺序是 `rcHeightfield` 占用 span、可走性过滤、`rcCompactHeightfield`、距离场与 region、`rcContourSet`、`rcPolyMesh`、`rcPolyMeshDetail`，最后才转换为 Detour tile 数据。

每个阶段都在改变数据语义：

| 阶段 | 数据表达 | 解决的问题 |
| --- | --- | --- |
| 三角形 | 原始世界几何 | 输入复杂、噪声多，不适合直接寻路 |
| `rcHeightfield` | 每个 xz cell 内的垂直 spans | 把连续几何离散化，统一到体素空间 |
| 过滤后 heightfield | 初步可走 span | 把坡度、台阶、悬崖、净空规则烘焙进去 |
| `rcCompactHeightfield` | 可走 span 数组和四向邻接 | 从链表结构转成适合图算法的紧凑结构 |
| distance field / regions | 可走空间的区域划分 | 把 span 图拆成可提取轮廓的简单区域 |
| `rcContourSet` | 区域边界轮廓 | 从 cell 边界变成可简化多边形边界 |
| `rcPolyMesh` | 凸多边形拓扑图 | 为 Detour 提供 A* 图节点和邻接 |
| `rcPolyMeshDetail` | 每个 polygon 内部高度三角网 | 给 coarse polygon 补回地形高度细节 |

Recast 的关键思想是：运行时 agent 可以近似为一个点在已经按 agent 半径收缩过的凸多边形图上移动。构建阶段越充分，运行时就越简单。

这个“点 agent”不是说游戏角色没有体积，而是说体积在构建期被提前转成了约束：坡度由 `walkableSlopeAngle` 标记，高度由 `walkableHeight` 过滤，台阶由 `walkableClimb` 控制，半径由 `walkableRadius` 腐蚀。官方社区讨论中也说明，业界更常见的是为 agent 半径烘焙 navmesh，必要时为少量半径档位生成多份 navmesh，而不是让每次运行时寻路都带连续半径做复杂规划。

## 二、根源设计：从几何问题变成图问题

原始三角形场景直接用于寻路会有几个问题：

1. 三角形数量通常远大于导航需要。
2. 渲染三角形可能包含细碎装饰、重叠面、非流形边。
3. agent 有高度、半径、可爬高度，不是几何点。
4. 运行时寻路需要稳定拓扑，而不是每帧处理复杂三角面。

Recast 的方案是分三步降维：

![Recast 将三维几何逐步降维为凸多边形拓扑图](https://oss.euler.icu/teaser/recastnavigation/Recast/Docs/Images/recast-dimensional-reduction.png)

> **图 2：Recast 的降维策略。** 三角形首先被离散为按 xz cell 分列、仍保留垂直层次的 2.5D span；过滤后的可站立 span 通过四向连接形成图；region 把图分区并抽取二维边界；最终凸多边形成为 Detour 的运行时图节点。降维没有直接丢掉高度：coarse mesh 的顶点高度以及后续 `rcPolyMeshDetail` 仍负责恢复地形表面。

这种设计的代价是精度受 `cs`、`ch` 和简化误差影响；收益是算法稳定、数据紧凑、运行时查询成本低。

### 关键不变量

1. **可走性存储在 area id 中。**

   `RC_NULL_AREA` 表示不可走，非 0 area 表示可走或有语义的可走区域。过滤、腐蚀、区域分区都会依赖 area。

2. **`rcHeightfield` 表示实心占用，`rcCompactHeightfield` 表示可站立空间。**

   `rcHeightfield` 的 span 来自三角形体素化，表示一段垂直占用。`rcCompactHeightfield` 只保留可走 span 的“地面高度 + 上方空间”，用于后续连通和区域算法。

3. **agent 的能力在构建期被转成体素约束。**

   `walkableHeight`、`walkableClimb`、`walkableRadius` 会在过滤、compact 邻接、腐蚀阶段生效。最终 mesh 边界表达的是 agent 中心能到达的位置。

4. **region 必须尽量形成简单轮廓。**

   后续 contour 和 polygon mesh 构建假设区域边界可以形成可处理的简单多边形。区域过碎、重叠或有复杂洞时，会导致轮廓合并和三角化困难。

5. **coarse mesh 负责拓扑，detail mesh 负责高度。**

   `rcPolyMesh` 的 polygon 数量应少，适合寻路；`rcPolyMeshDetail` 保留高度贴合，避免 coarse polygon 过度平面化。

6. **坐标系假设贯穿整个构建流程。**

   Recast 的体素网格是 xz 平面网格，y 是高度方向。`rcMarkWalkableTriangles()` 看三角形法线的 y 分量；heightfield cell 用 `(x,z)` 定位；`rcCompactSpan::con` 四方向也是 xz 平面邻接。官方 FAQ 要求右手 Y-up、顺时针三角形；官方社区讨论也明确说明直立角色假设已经深入库内部。想改成 YZ、墙面或天花板导航，不是替换几个轴名就能完成。

## 三、配置：`rcConfig`

`rcConfig` 是整个构建流程的参数合同。它把世界单位参数和体素单位参数放在一起。

| 参数 | 单位 | 作用 |
| --- | --- | --- |
| `cs` | world unit | xz 平面 cell size，决定水平采样精度。 |
| `ch` | world unit | y 轴 cell height，决定高度采样精度。 |
| `width/height` | voxel | heightfield 在 xz 平面的尺寸。 |
| `tileSize` | voxel | tiled build 中去掉 padding 后的有效 tile 边长。solo build 可为 0。 |
| `borderSize` | voxel | tiled build 在四周额外构建、最终再裁掉的 padding。 |
| `bmin/bmax` | world unit | 当前构建块的世界 AABB；tiled build 时应包含 border 扩张。 |
| `walkableSlopeAngle` | degree | 坡度阈值，标记输入三角形是否可走。 |
| `walkableHeight` | voxel | agent 需要的净空。 |
| `walkableClimb` | voxel | agent 可跨越的最大高度差。 |
| `walkableRadius` | voxel | agent 半径，用于腐蚀可走区域。 |
| `maxEdgeLen` | voxel | 轮廓长边切分限制。 |
| `maxSimplificationError` | voxel | 轮廓简化允许偏离 raw contour 的距离。 |
| `minRegionArea` | voxel span count | 小孤岛区域删除阈值。 |
| `mergeRegionArea` | voxel span count | 小区域合并阈值。 |
| `maxVertsPerPoly` | count | 输出 polygon 最大顶点数。Detour 通常要求不超过 `DT_VERTS_PER_POLYGON`。 |
| `detailSampleDist` | world unit | detail mesh 内部采样间距。 |
| `detailSampleMaxError` | world unit | detail mesh 最大高度误差。 |

### 参数之间的根本关系

`cs` 和 `ch` 是所有结果的根。它们越小，越接近原始几何，但构建成本和内存上升。

`walkableRadius` 必须按 `ceil(agentRadius / cs)` 转成体素半径。它不是运行时碰撞半径，而是构建期对可走区域做形态学腐蚀的半径。

RecastDemo 给出的标准换算是：

```text
walkableHeight = ceil(agentHeight / ch)
walkableClimb  = floor(agentMaxClimb / ch)
walkableRadius = ceil(agentRadius / cs)
```

height 和 radius 向上取整是为了不低估角色尺寸；climb 向下取整是为了不高估跨越能力。`ch` 太大时，台阶、低矮空间和坡面都会变得粗糙。

构建前通常先用 `rcCalcBounds()` 计算输入顶点 AABB，再用 `rcCalcGridSize()` 按 `cs` 算出 `width/height`。这两个工具看起来简单，但它们定义了整个体素网格的空间坐标系。后续 `rcCreateHeightfield()`、三角形 rasterization、area marking、poly mesh 顶点回写都会围绕同一个 `bmin/bmax/cs/ch` 做世界坐标和体素坐标转换。

官方 `rcConfig` 文档给出的实用顺序是：先确定 agent 的逻辑圆柱尺寸，再推导 `walkableHeight/walkableRadius/walkableClimb` 等体素参数。这个顺序很重要，因为 Recast 的构建结果回答的是“这个尺寸的 agent 中心能在哪里走”，不是“原始几何哪里看起来像地面”。

tiled build 还需要理解 border 的因果关系。RecastDemo 使用：

```text
borderSize = walkableRadius + 3
width      = tileSize + 2 * borderSize
height     = tileSize + 2 * borderSize
```

同时把 tile 的 `bmin/bmax` 在 xz 上各扩张 `borderSize * cs`。腐蚀、region 生长和 contour 简化因此能看到 tile 外的一圈邻域；输出 contour/poly mesh 时再减去 border offset。没有 padding 时，每个 tile 都会把边界外当作空白/悬崖，独立构建结果很容易在接缝处断开。

## 四、核心数据结构

### 1. `rcHeightfield`

`rcHeightfield` 是原始体素高度场。

```cpp
struct rcHeightfield
{
    int width, height;
    float bmin[3], bmax[3];
    float cs, ch;
    rcSpan** spans;
    rcSpanPool* pools;
    rcSpan* freelist;
};
```

每个 `(x, z)` cell 存一个按高度排序的 `rcSpan` 链表。一个 span 表示该列中 `[smin, smax)` 的垂直占用。

`rcSpan` 使用 bit field：

- `smin`：13 bit。
- `smax`：13 bit。
- `area`：6 bit。
- `next`：同列更高 span。

因此单个 heightfield 相对 `bmin.y` 的量化高度上限是 8191 个 `ch`。这不是世界绝对高度限制，而是当前构建块的垂直范围限制；超高世界应按 tile/高度区间组织构建，不能假设 bit field 会自动扩展。

为什么用链表：

- 三角形体素化时，每列可能有任意多个 span。
- 构建时不断插入和合并 span，链表方便。
- `rcSpanPool` 每次分配 2048 个 span，减少大量小分配。

### 2. `rcCompactHeightfield`

`rcCompactHeightfield` 是后续算法的主数据结构。

```cpp
struct rcCompactHeightfield
{
    int width, height;
    int spanCount;
    rcCompactCell* cells;
    rcCompactSpan* spans;
    unsigned short* dist;
    unsigned char* areas;
};
```

`rcCompactCell`：

- `index`：该 cell 第一个 compact span 在 `spans` 数组中的下标。
- `count`：该 cell 内 compact span 数量。

`rcCompactSpan`：

- `y`：可站立地面的高度。
- `h`：上方可用空间高度。
- `reg`：区域 id。
- `con`：四方向连接，每个方向 6 bit，`RC_NOT_CONNECTED` 表示无连接。

为什么 compact：

- 后续算法频繁遍历 span 和邻接，数组比链表 cache 友好。
- 只保留非 null area 的可走 span，数据量减少。
- 四方向连接预先算好，区域分区和轮廓追踪不必每次重新扫描邻接列。

![rcHeightfield 链表 span 到 rcCompactHeightfield 连续数组的结构转换](https://oss.euler.icu/teaser/recastnavigation/Recast/Docs/Images/recast-heightfield-data-structures.png)

> **图 3：`rcHeightfield` 与 `rcCompactHeightfield` 的数据布局。** 左侧 `rcSpan` 链表的 `smin/smax` 描述实心占用区间，`area` 保存可走语义；压缩时只为非 `RC_NULL_AREA` span 生成可站立记录。`rcCompactCell.index/count` 定位 cell 在连续 `rcCompactSpan` 数组中的切片；每个 compact span 以 `y` 表示地面、`h` 表示上方净空、`reg` 表示区域，并以 `con[4]` 保存四方向邻居在目标 cell 内的层索引。

### 3. `rcContourSet` 和 `rcContour`

`rcContour` 保存一个 region 的边界：

- `rverts`：raw contour，完整贴着 cell 边界。
- `verts`：simplified contour，经过误差阈值简化。
- 每个顶点 4 个整数：`x, y, z, flags/neighborReg`。

第 4 个分量同时保存：

- 邻接 region。
- `RC_BORDER_VERTEX`。
- `RC_AREA_BORDER`。

这些标记后续会影响 portal 对齐、边界顶点删除和 area 边界保留。

### 4. `rcPolyMesh`

`rcPolyMesh` 是 coarse navigation mesh。

| 字段 | 说明 |
| --- | --- |
| `verts` | 体素坐标顶点，3 个 `unsigned short` 一组。 |
| `polys` | polygon 顶点和邻接数据，长度为 `maxpolys * 2 * nvp`。 |
| `regs` | polygon 所属 region。 |
| `areas` | polygon area id。 |
| `flags` | 用户填充的运行时 flags。 |
| `nvp` | 每个 polygon 最大顶点数。 |

`polys` 前 `nvp` 个槽是顶点索引，后 `nvp` 个槽是邻接信息。内部邻接存相邻 polygon index，tile 边 portal 会写 `0x8000 | side`。

### 5. `rcPolyMeshDetail`

`rcPolyMeshDetail` 是 coarse polygon 的高度补充。

| 字段 | 说明 |
| --- | --- |
| `meshes` | 每个 coarse polygon 对应的 detail 子网格起始 vertex、vertex count、triangle 起始、triangle count。 |
| `verts` | detail 顶点，世界坐标。 |
| `tris` | detail 三角形，每个三角形 4 byte，最后一 byte 存边界 flag。 |

它不改变寻路拓扑，只让 Detour 做高度查询和最近点时更准确。

## 五、体素化：从三角形到 `rcHeightfield`

体素化入口：

- `rcCreateHeightfield()`
- `rcMarkWalkableTriangles()`
- `rcClearUnwalkableTriangles()`
- `rcRasterizeTriangles()`
- `rcRasterizeTriangle()`
- `rcAddSpan()`

### 1. 坡度标记

`rcMarkWalkableTriangles()` 对每个三角形计算法线：

```text
normal = normalize(cross(v1 - v0, v2 - v0))
walkable if normal.y > cos(walkableSlopeAngle)
```

如果三角面接近水平，`normal.y` 大；坡越陡，`normal.y` 越小。这个阶段只设置三角形 area id，不生成体素。

因为 normal 来自有向叉积，顺时针 winding 是可走地面得到正 y 法线的输入合同。若上游模型是逆时针 winding，应在导入时交换每个三角形的两个索引；用 `abs(normal.y)` 会把天花板底面也误标为可走，不能作为通用修复。

`rcClearUnwalkableTriangles()` 做相反操作：坡度超过阈值时把 triangle area 清成 `RC_NULL_AREA`。

### 2. 三角形裁剪到 cell

`rasterizeTri()` 是热路径。它的核心不是逐像素采样，而是精确裁剪三角形 footprint：

1. 计算三角形 AABB。
2. 如果三角形 AABB 不和 heightfield AABB 相交，跳过。
3. 计算三角形覆盖的 z 行范围。
4. 对每个 z 行，用 `dividePoly()` 把三角形裁剪到该行。
5. 对行内每个 x 列，再用 `dividePoly()` 裁剪到该 cell。
6. 得到 cell 内多边形后，计算它的 y 最小值和最大值。
7. 把 y 范围按 `ch` 量化成 `[spanMin, spanMax)`。
8. 调 `addSpan()` 加入 heightfield。

![三角形经双轴裁剪、垂直量化并合并为 heightfield span](https://oss.euler.icu/teaser/recastnavigation/Recast/Docs/Images/recast-triangle-rasterization.png)

> **图 4：`rasterizeTri()` 的几何裁剪与 span 写入。** 三角形 AABB 先限定覆盖范围；`dividePoly()` 依次把三角形裁到 z row 和 x cell，而不是只测试 cell 中心；cell 内裁剪多边形的 y 极值按 `ch` 量化并 clamp 到 heightfield；`addSpan()` 再按高度有序插入，合并重叠或足够接近的区间，并按 `flagMergeThreshold` 处理 area 优先级。

`dividePoly()` 是 Sutherland-Hodgman 风格的凸多边形裁剪。输入三角形被逐步裁到 row 和 column 内，避免只按 cell center 采样导致漏掉细三角形。

`rcRasterizeTriangles()` 有三个 public overload：

- indexed triangles，索引为 `int`。
- indexed triangles，索引为 `unsigned short`。
- triangle list，每三个连续顶点组成一个三角形。

这些 overload 最终都会进入同一套 triangle rasterization 逻辑。`rcRasterizeTriangle()` 是单三角形版本，适合少量动态或调试输入；大量三角形应使用批量版本减少调用开销。

### 3. span 合并

`addSpan()` 会把新 span 插入对应 cell 的高度有序链表。如果它和已有 span 重叠，会合并：

- `smin = min(old.smin, new.smin)`
- `smax = max(old.smax, new.smax)`
- 如果两个 span 的 `smax` 差在 `flagMergeThreshold` 内，area 取较大值。

“area id 越大优先级越高”这个规则让更具体的 area 标注可以覆盖普通 walkable area。

`rcAddSpan()` 是 public API，对应内部 `addSpan()`。它允许调用者不经过三角形 rasterization，直接向 heightfield 写 span。只要保证 `(x,z)`、`spanMin/spanMax` 和 area id 合法，后续过滤、compact 和区域构建会把这些手动 span 当作普通 rasterized span 处理。

### 为什么先体素化

体素化把复杂几何统一成规则网格。之后所有核心算法都可以用 cell 邻接、span 高度、距离变换和 flood fill 实现，而不必处理任意三角形拓扑。

## 六、过滤：把几何可站立变成 agent 可通行

过滤在 `rcHeightfield` 上进行，修改 span 的 `area`。

### 1. `rcFilterLowHangingWalkableObstacles`

目的：把 agent 可以迈上去的小障碍恢复为可走。

算法：

1. 对每个 cell 的 span 从低到高遍历。
2. 记录前一个 span 是否原本可走。
3. 如果当前 span 不可走，但前一个 span 可走，并且：

   ```text
   current.smax - previous.smax <= walkableClimb
   ```

   则当前 span 继承前一个 span 的 area。

注意源码用“原始 previousWasWalkable”而不是被修改后的结果，避免一串连续不可走 span 被级联标成可走。

### 2. `rcFilterLedgeSpans`

目的：移除靠近悬崖或不可跨越高度差的 span。

算法对每个可走 span 检查四邻域：

1. 当前 span 的 floor 是 `smax`。
2. ceiling 是上一个 span 的 `smin`，没有上方 span 时用最大高度。
3. 遍历邻居列中所有 span。
4. 只有当当前 span 与邻居 span 的垂直重叠空间足够 `walkableHeight` 时，才可能连通。
5. 如果邻居 floor 相对当前 floor 的下降超过 `walkableClimb`，标记为 ledge。
6. 如果所有可达邻居的最高/最低 floor 差超过 `walkableClimb`，也视为过陡。

这个过滤解决保守体素化在悬崖边缘留下“悬空可走面”的问题。

邻列越出 heightfield bounds，或邻列在当前 floor 下方有足够净空却没有承接 span，也会按“下降超过 climb”处理。这就是 tiled build 必须带 border padding 的另一原因：若只体素化有效 tile 的精确边界，边缘 span 会因为看不到相邻 tile 几何而被误判成 ledge。

### 3. `rcFilterWalkableLowHeightSpans`

目的：移除头顶空间不足的位置。

算法很直接：

```text
floor = span.smax
ceiling = nextSpan ? nextSpan.smin : MAX_HEIGHT
if ceiling - floor < walkableHeight:
    area = RC_NULL_AREA
```

这一步表达 agent 高度，不考虑半径。

## 七、构建 `rcCompactHeightfield`

`rcBuildCompactHeightfield()` 做两件事：

1. 只把非 `RC_NULL_AREA` 的 spans 拷贝到紧凑数组。
2. 预计算四方向可行连接。

拷贝前会先调用 `rcGetHeightFieldSpanCount()` 统计非 `RC_NULL_AREA` 的 spans 数量。这个值成为 `compactHeightfield.spanCount`，也决定 `cells/spans/areas` 等数组的分配规模。也就是说，过滤阶段清掉的 spans 不会进入 compact 表示，后续算法看不到它们。

### 1. 拷贝可走 spans

对每个 heightfield column：

1. `rcCompactCell.index` 指向当前写入位置。
2. 对每个非 null span：
   - `y = span.smax`
   - `h = next.smin - span.smax`，没有 next 时用最大高度。
   - `areas[current] = span.area`
3. `cell.count` 记录该 cell 写入了多少 compact span。

### 2. 建立邻接

对每个 compact span 和四个方向：

1. 找邻居 cell。
2. 遍历邻居 cell 的所有 compact spans。
3. 计算两者可通行空间：

   ```text
   bot = max(span.y, neighbor.y)
   top = min(span.y + span.h, neighbor.y + neighbor.h)
   ```

4. 如果：

   ```text
   top - bot >= walkableHeight
   abs(neighbor.y - span.y) <= walkableClimb
   ```

   则连接成立。
5. 把邻居 span 在邻居 cell 内的层索引写入 `con` 对应方向。

![compact span 按垂直净空和爬升高度建立四方向连接](https://oss.euler.icu/teaser/recastnavigation/Recast/Docs/Images/recast-compact-connectivity.png)

> **图 5：`rcCompactSpan.con[4]` 的连接判定。** 蓝色为当前 span，青色为满足 `min(ceiling0, ceiling1) - max(floor0, floor1) >= walkableHeight` 且 `abs(floor1 - floor0) <= walkableClimb` 的邻居；红色示例因共享净空不足被拒绝，橙色示例因台阶过高被拒绝。通过判定后写入的不是全局 span 下标，而是邻 cell 切片内最多 6 bit 的层索引；`RC_NOT_CONNECTED` 使用保留值 `0x3f`。

`con` 每方向只能存有限层索引，超出会记录错误。这个限制来自 `RC_NOT_CONNECTED = 0x3f`，也就是 6 bit 连接编码。

四方向坐标变换由 `rcGetDirOffsetX()`、`rcGetDirOffsetY()` 和 `rcGetDirForOffset()` 统一定义。源码里 `Y` 实际指 xz 网格中的 z 方向，这是历史命名；所有 compact neighbor、region flood、contour walk 都依赖同一套方向编号，避免不同阶段对 0..3 方向含义不一致。

### 为什么 compact 阶段重要

从这里开始，Recast 的问题已经变成“可走 span 图”问题。区域分区、轮廓提取、detail 高度采样都围绕这个图进行。

## 八、area 操作：腐蚀、标注和滤波

`RecastArea.cpp` 处理 compact heightfield 的 area。

### 1. `rcErodeWalkableArea`

目的：把 agent 半径烘焙进 navmesh，使运行时 agent 中心不会贴墙。

算法：

1. 创建 `distanceToBoundary`，初始为 255。
2. 所有不可走 span、缺少四向可走邻居的 span 标为距离 0。
3. 两遍距离变换：
   - 正向扫描检查左、下、左下、右下。
   - 反向扫描检查右、上、右上、左上。
   - 横竖代价为 2，斜向代价为 3。
4. 如果：

   ```text
   distanceToBoundary < erosionRadius * 2
   ```

   则设为 `RC_NULL_AREA`。

为什么乘 2：距离场中横竖一步的代价是 2，所以体素半径要转换到同一度量。

### 2. `rcMedianFilterWalkableArea`

目的：消除 area 标注噪声。

算法：

1. 对每个非 null span，收集 3x3 邻域 area。
2. 不可走邻居不参与替换，缺失时使用自身 area。
3. 对 9 个 area 做插入排序。
4. 取中位数 `neighborAreas[4]`。

它平滑的是 area 类型，不改变几何可走性。

### 3. 体积标注

`rcMarkBoxArea()`、`rcMarkCylinderArea()`、`rcMarkConvexPolyArea()` 会把指定空间内的 compact spans 改成某个 area id。

典型用途：

- 水面。
- 道路。
- 禁行区。
- 高代价区域。

area 只是语义，不直接等于 Detour flags。后续通常在应用层把 area 映射为 Detour 的 poly flags 和 cost。

`rcMarkConvexPolyArea()` 的设计容易被误解。源码和官方讨论都说明：它忽略输入 polygon 顶点的 y 值，把 convex polygon 投影到 xz 平面，再按 `hmin/hmax` 拉伸成一个竖直体积。它不是一般 3D point-in-polyhedron 或精确 mesh boolean。设计目的是给 navmesh 上的投影区域打 area id，例如道路低 cost、泥地高 cost、触发区或禁行区。对 capsule/cylinder agent 来说，把一个物体的水平外轮廓投影到可走面上通常已经足够；更精确的复杂体积标注应由应用层拆成多个 convex/box/cylinder 标记，或在输入几何阶段处理。

`rcOffsetPoly()` 是 `rcMarkConvexPolyArea()` 的辅助函数。它会把一个 xz 平面上的 convex polygon 沿边法线外扩或内缩，并在尖角处插入 bevel 顶点。这样应用层可以先把区域按 agent 半径、道路边缘或设计需求做几何偏移，再把偏移后的 convex polygon 标注到 compact heightfield 上。函数要求输出 buffer 至少能容纳扩展后的顶点；通常上限按 `2 * numVerts` 预留。

## 九、距离场和区域分区

区域分区的目标是给每个可走 span 分配 `reg`。一个 region 后续会形成一个或多个 contour。

### 1. `rcBuildDistanceField`

距离场用于 watershed 分区。

`calculateDistanceField()`：

1. 初始化所有距离为 `0xffff`。
2. 如果某 span 四邻域没有同 area 邻居，则是边界，距离设为 0。
3. 两遍扫描传播距离。
4. 横竖代价 2，斜向代价 3。
5. 得到 `maxDistance`。

`boxBlur()`：

- 对距离场做 3x3 blur。
- 距离小于阈值的边界附近保留原值。

blur 的目的不是几何精确，而是让 watershed 区域增长更平滑，减少细碎区域。

### 2. Watershed：`rcBuildRegions`

Watershed 是默认质量较高的分区。

核心直觉：

- 距离边界越远，越像区域中心。
- 先从高距离处创建区域种子。
- 再逐步向低距离处扩张。

源码流程：

![watershed 从距离场高值种子向边界扩张并合并区域](https://oss.euler.icu/teaser/recastnavigation/Recast/Docs/Images/recast-watershed-regions.png)

> **图 6：`rcBuildRegions()` 的 watershed 分区。** 蓝色高度表示离障碍边界的距离，`level = (maxDistance + 1) & ~1` 后每轮下降 2；`sortCellsByLevel()`/`appendStacks()`组织当前层 span，`expandRegions()`延伸已有区域，`floodRegion()`在剩余 span 上创建种子。level 0 的最终扩张填补未分配空间，`mergeAndFilterRegions()`再删除小孤岛、合并满足拓扑约束的小区域，并把压缩后的 id 写入 `rcCompactSpan.reg`。

1. 可选绘制 border regions，标记 `RC_BORDER_REG`。
2. `level = (maxDistance + 1) & ~1`，从高距离层开始。
3. 每次 level 降 2。
4. 用 `sortCellsByLevel()` 把当前层的未分配 span 放入 level stack。
5. `expandRegions()` 让已有 region 向当前层扩张。
6. 对仍未分配的 span，用 `floodRegion()` 创建新 region。
7. 所有 level 处理完后，再做一次大范围 expand 填满空洞。
8. `mergeAndFilterRegions()` 删除小孤岛、合并小区域、压缩 region id。

`floodRegion()` 会检查邻居是否已有不同 region，避免新 region 直接和已有 region 冲突。它只 flood 同 area 且距离层满足条件的 spans。

为什么 watershed 质量高：

- 开阔区域从中心向外扩张，边界更自然。
- 房间、平台等区域更容易被分成大而规整的 region。

代价：

- 需要 distance field。
- 算法比 monotone 慢。
- 对狭窄斜向通道可能产生小区域，需要 merge 参数修正。

### 3. Monotone：`rcBuildRegionsMonotone`

Monotone 分区按扫描线构建区域。

流程：

1. 可选绘制 border regions。
2. 从上到下逐行扫描。
3. 对每个 span：
   - 如果左侧同 area 且连通，沿用左侧 sweep id。
   - 否则创建本行新 sweep id。
   - 如果上方同 area 且连通，记录候选邻居 region。
4. 一行结束后，如果某 sweep 和上方 region 是唯一连续连接，则沿用上方 region；否则创建新 region。
5. 最后执行 `mergeAndFilterRegions()`。

monotone 的优势：

- 快。
- 不需要 distance field。
- 生成的 region 不容易产生复杂洞。

代价：

- 区域可能沿扫描方向拉长。
- 形状质量通常不如 watershed。

当前实现确实不读取 `chf.dist` 或 `chf.maxDistance`，所以从源码依赖看 monotone 不需要先调用 `rcBuildDistanceField()`。但当前 Doxygen 注释仍写着 distance field 必须先构建，这是沿用 watershed 文案造成的版本漂移。若应用只选择 monotone，可以省掉 distance-field 计算；若后续调试或自定义阶段需要 `chf.dist`，则仍可显式构建。

### 4. Layer regions：`rcBuildLayerRegions`

Layer partition 是生成 `rcContourSet -> rcPolyMesh` 之前的第三种 region 划分选择。它偏向把垂直重叠的表面分进不同、不重叠的 region layer，适合多层平台，并常被 tiled build 选用。

它也使用扫描线 sweep，但重点是避免垂直重叠的平台被合进同一 layer。`mergeAndFilterLayerRegions()` 会检查：

- area type 是否一致。
- 是否和 root region 垂直重叠。
- 是否连接到 border。
- 小区域是否应移除。

输出仍写入 `rcCompactSpan::reg`，供 `rcBuildContours()` 使用。它和下一章的 `rcBuildHeightfieldLayers()` 名称相近，但没有调用依赖：后者会从 compact spans 自己重新做一遍 byte region sweep，并不读取这里写入的 `span.reg`。这一区分很重要：

- `rcBuildLayerRegions()` 的产物是 contour/poly mesh pipeline 的 region 标号。
- `rcBuildHeightfieldLayers()` 的产物是给 TileCache 压缩的 `heights/areas/cons` layer set。

与 monotone 一样，当前 `rcBuildLayerRegions()` 实现也不读取 distance field。

### 5. 区域合并和过滤

`mergeAndFilterRegions()` 是区域质量控制的关键。

它会：

1. 构建 `rcRegion` 表。
2. 统计每个 region 的 spanCount、areaType、邻接环、floors、是否重叠、是否连接 border。
3. 删除小于 `minRegionArea` 且不连接 tile border 的孤岛。
4. 对小于 `mergeRegionArea` 或不连接 border 的 region，尝试合并到邻居。
5. 合并时要求：
   - area type 相同。
   - 两个 region 只通过一个连接相邻。
   - 不在同一 vertical floor 里重叠。
6. 压缩 region id。

不删除连接 tile border 的小区域，是因为单个 tile 看不到邻接 tile 中该区域的完整尺寸，贸然删除会破坏跨 tile 连通。

## 十、Heightfield layers

`rcBuildHeightfieldLayers()` 在 `RecastLayers.cpp` 中。它把 compact heightfield 转换成若干 2D layer：

```cpp
struct rcHeightfieldLayer
{
    unsigned char* heights;
    unsigned char* areas;
    unsigned char* cons;
};
```

构建流程：

![compact heightfield 经扫描分区和冲突消解生成可压缩高度层](https://oss.euler.icu/teaser/recastnavigation/Recast/Docs/Images/recast-heightfield-layers.png)

> **图 7：`rcBuildHeightfieldLayers()` 的层分配。** 扫描线先生成 monotone region；region overlap graph 表示同一 xz cell 内的垂直冲突，因此相互冲突的 region 不能放入同一 layer。高度范围相容且合并后仍无冲突的 layer 可继续合并；输出的 `heights/areas/cons` 都是二维 byte 网格，其中 `cons` 低 4 bit 表示本层四向连接，高 4 bit 表示通向层外或 tile 边界的 portal。

1. 扫描线分区，得到 monotone regions。
2. 统计 region 邻居、垂直重叠关系、高度范围。
3. 把不互相垂直重叠、且高度范围可放进 byte 的 regions 合并到同一 layer。
4. 合并相近高度范围的 layer，仍确保没有重叠冲突。
5. 为每个 layer 分配 `heights/areas/cons`。
6. 把 compact span 写成 2D 网格。
7. `cons` 低 4 bit 保存同 layer 四向连接，高 4 bit 保存跨 layer portal。

为什么 layer 数据用 byte：

- Tile cache 需要压缩和快速重建。
- layer 是局部 tile 内的数据，设计上要求高度范围不要超过 255。
- 简化数据结构可以降低动态重建成本。

byte 也带来硬边界：扫描 region id 达到 255 会失败；单 layer 的相对高度范围必须能落进 0..255。这里的限制是 TileCache 紧凑格式的一部分，不是 `rcCompactHeightfield` 本身只能有 255 层。

## 十一、轮廓提取：`rcBuildContours`

轮廓阶段把 region span 边界转成多边形轮廓。

![region 的体素边界经强制点、误差简化和洞合并生成 contour](https://oss.euler.icu/teaser/recastnavigation/Recast/Docs/Images/recast-contour-extraction.png)

> **图 8：`rcBuildContours()` 的边界追踪与简化。** `walkContour()`沿 `span.reg` 的边界 bit 生成 raw contour；portal 和 area transition 顶点是不能被误差简化删除的强制点。其余轮廓按 `maxError` 反复插入离当前线段最远的 raw 点，并按 `maxEdgeLen` 与 `buildFlags` 切分指定长边；winding 区分 outline 与 hole。`mergeRegionHoles()`选择位于合法 cone 内且不与现有边相交的对角线，`mergeContours()`再各复制一次桥两端顶点，使连接线可沿相反方向经过两次，从而把两个闭环编码成一个弱简单 contour 顶点序列。

### 1. 标记边界边

对每个非 border region span：

1. 检查四方向邻接。
2. 如果邻居 region 和当前 region 相同，说明该边不是边界。
3. 否则该方向是边界边。
4. `flags[i] = res ^ 0xf`，保存非连接边。

### 2. `walkContour`

从第一个边界边开始沿 region 边界行走：

1. 如果当前方向是边界边：
   - 根据方向选择 cell corner。
   - 用 `getCornerHeight()` 计算角点高度。
   - 写入邻居 region、area border、border vertex flag。
   - 清除该边界 flag，避免重复。
   - 顺时针转向。
2. 如果当前方向不是边界：
   - 进入邻接 span。
   - 逆时针转向。
3. 回到起点和起始方向时结束。

这个算法沿 cell 边界走，所以 raw contour 会精确贴着 region 的体素轮廓。

### 3. `getCornerHeight`

角点高度来自角点周围最多 4 个 spans 的最大 `y`。这样可以避免轮廓顶点低于相邻 span。

它还会判断：

- 是否是 tile border vertex。
- 是否是 area border。

`RC_BORDER_VERTEX` 后续在 `rcBuildPolyMesh()` 中可被删除，用于 tile 边界对齐。

### 4. `simplifyContour`

简化过程：

1. 如果 raw contour 上有 portal 或 area 切换，把 region/area 变化点作为强制点。
2. 如果没有强制点，选择左下和右上点作为初始简化点。
3. 对每条简化边，遍历对应 raw segment。
4. 找到距离该边最远的 raw 点。
5. 如果误差超过 `maxError`，插入该点。
6. 重复直到所有 raw 点误差都在阈值内。
7. 如果启用了 `maxEdgeLen`，对墙边或 area 边过长的边继续切分。
8. 去掉相邻重复顶点。

为什么 portal/area 变化点必须保留：

- 相邻 region 的 portal 需要对齐。
- 不同 area 的边界不能被简化掉，否则语义区域会被混合。

`rcBuildContours()` 的 `buildFlags` 控制第 7 步具体切哪些长边：

- `RC_CONTOUR_TESS_WALL_EDGES`：对 solid wall 边按 `maxEdgeLen` 细分。
- `RC_CONTOUR_TESS_AREA_EDGES`：对 area 边界按 `maxEdgeLen` 细分。

RecastDemo 默认只传 `RC_CONTOUR_TESS_WALL_EDGES`；公共 API 本身没有隐藏默认值，行为完全由 `buildFlags` 决定。墙边过长会影响后续 tile 边界和轮廓贴合；area 边界是否需要细分取决于应用是否需要更精细地保留语义边界。源码通过顶点第 4 分量中的 `RC_CONTOUR_REG_MASK`、`RC_AREA_BORDER`、`RC_BORDER_VERTEX` 判断一条 raw contour segment 属于哪类边。

### 5. 洞合并

一个 region 可能有洞。`rcBuildContours()` 根据 contour winding 判断：

- 正向 winding 是 outline。
- 反向 winding 是 hole。

`mergeRegionHoles()` 会：

1. 对每个 hole 找最左下顶点。
2. 在 outline 上找合法连接对角线。
3. 对角线必须在 cone 内，且不和 outline 或其他 holes 相交。
4. 选择较短合法对角线。
5. 用 `mergeContours()` 把 hole 并入 outline。

这样后续 poly mesh 只需要处理单个简单多边形。

## 十二、构建 coarse polygon mesh：`rcBuildPolyMesh`

`rcBuildPolyMesh()` 把 contours 转成凸多边形网格。

![simplified contour 经三角化和凸合并构建 rcPolyMesh](https://oss.euler.icu/teaser/recastnavigation/Recast/Docs/Images/recast-polymesh-build.png)

> **图 9：`rcBuildPolyMesh()` 的 coarse mesh 构建。** `addVertex()`以 `(x,z)` hash 去重并允许有限 y 偏差；`triangulate()`使用 ear clipping 得到三角形；随后按共享边长度排序候选，只合并仍保持凸性且不超过 `nvp` 的 polygon。删除带 `RC_BORDER_VERTEX` 的顶点时会重建局部洞，最后 `buildMeshAdjacency()`把共享无向边配对，并把结果编码进 `rcPolyMesh.polys` 的邻接半区。

### 1. 顶点去重

`addVertex()` 使用 hash bucket 按 `(x, z)` 查找已有顶点，并允许 y 差小于等于 2 时合并。

原因：

- 相邻 contour 共边时，顶点必须复用，否则邻接无法正确建立。
- y 方向允许小误差，可以缓解体素高度和轮廓简化带来的细微不一致。

### 2. Ear clipping 三角化

每个 contour 先通过 `triangulate()` 切成三角形：

1. 对每个顶点检查其前后顶点形成的 diagonal 是否有效。
2. diagonal 有效则该顶点可作为 ear 删除。
3. 每次选择 diagonal 长度最短的 ear。
4. 输出一个三角形。
5. 删除 ear 顶点。
6. 更新相邻顶点的 diagonal 标记。
7. 如果严格 diagonal 找不到，会使用 loose 版本尝试恢复。

检查 diagonal 的核心条件：

- 在多边形内。
- 不与其他边相交。

### 3. 三角形合并为凸多边形

如果 `nvp > 3`，Recast 会尽量把三角形合并成更大的凸 polygon。

`getPolyMergeValue()` 要求：

1. 两个 polygon 共享一条边。
2. 合并后顶点数不超过 `nvp`。
3. 合并后的关键角保持凸性。
4. 返回共享边长度平方作为优先级。

主循环每次选择 merge value 最大的一对合并，直到不能合并。

为什么优先合并长共享边：

- 合并长边通常能形成更规整、更大的 polygon。
- 更少 polygon 意味着 Detour A* 节点更少。

### 4. 删除边界顶点

`RC_BORDER_VERTEX` 标记的顶点可尝试删除，主要用于 tiled navmesh 边界简化和对齐。

删除前 `canRemoveVertex()` 会验证：

- 删除后剩余边足够形成 polygon。
- 涉及的 open edges 不超过 2，避免错误连接不相邻区域。

`removeVertex()` 的过程：

1. 删除所有包含该顶点的 polygon。
2. 移除顶点并修正所有索引。
3. 收集删除后形成的洞边界。
4. 重新三角化这个洞。
5. 再合并三角形为凸 polygon。
6. 写回 mesh。

这一步不是简单删点，它会维护 polygon mesh 的拓扑完整性。

### 5. 构建邻接

`buildMeshAdjacency()` 用边匹配建立 polygon 邻接：

1. 先收集 `v0 < v1` 的边。
2. 再遍历反向边 `v0 > v1`，查找匹配。
3. 匹配成功说明两个 polygon 共享边。
4. 在 `polys[nvp + edge]` 写邻接 polygon index。

如果构建时有 borderSize，还会检查开放边是否位于 tile 边界，并写：

```text
0x8000 | side
```

这为 Detour 后续跨 tile 连接提供提示。

### 6. 多 mesh 合并和拷贝

`rcMergePolyMeshes()` 用于把多个 tile 或分块构建出的 `rcPolyMesh` 合成一个 mesh：

1. 先计算合并后的整体 bounds、最大顶点数、最大 polygon 数。
2. 为目标 mesh 分配 verts、polys、regs、areas、flags。
3. 对每个输入 mesh，根据它的 `bmin` 相对总 `bmin` 计算体素偏移 `ox/oz`。
4. 通过 `addVertex()` 去重并建立 `vremap`。
5. 拷贝 polygon 顶点、region、area、flags。
6. 只有位于最终外边界的 portal 标记会保留；内部 tile 边界 portal 会被丢弃。
7. 重新调用 `buildMeshAdjacency()`，让合并后的内部共享边变成普通邻接。

这个函数不是简单拼接数组。它必须重映射顶点、去掉内部 portal、重建邻接，否则 Detour 会把已经合并到同一 mesh 内的边仍当作跨 tile portal。

`rcCopyPolyMesh()` 是深拷贝工具，要求目标 mesh 为空。它会复制 bounds、cell size、borderSize、maxEdgeError 和所有顶点/poly/area/flag 数组，常用于保存中间结果或把构建结果交给不同后处理流程。

## 十三、构建 detail mesh：`rcBuildPolyMeshDetail`

coarse polygon mesh 适合拓扑搜索，但高度精度不足。detail mesh 的目标是为每个 coarse polygon 建一个局部高度三角网。

![coarse polygon 根据 height patch 和误差采样细化为 detail submesh](https://oss.euler.icu/teaser/recastnavigation/Recast/Docs/Images/recast-detail-mesh.png)

> **图 10：`rcBuildPolyMeshDetail()` 的高度细化。** 每个 coarse polygon 先从 compact heightfield 提取 `rcHeightPatch`，复制边界后按 `sampleDist` 采样并简化边；`triangulateHull()`建立初始三角网，再寻找与当前网格高度差最大的内部候选点。只要误差超过 `sampleMaxError` 且未达到顶点上限，就加入该点并重新进行 Delaunay-like 三角化；最终三角形和 boundary edge flags 写入该 polygon 对应的 detail submesh。

### 1. Height patch

对每个 polygon，Recast 先建立 `rcHeightPatch`：

- 覆盖 polygon AABB 附近的 compact heightfield 区域。
- 保存每个 cell 的高度。
- 没有高度的位置用 `RC_UNSET_HEIGHT`。

`getHeightData()` 优先从同 region spans 拷贝高度，并从边界做 BFS 填充缺失数据。对于 `RC_MULTIPLE_REGS` 或特殊情况，会用 `seedArrayWithPolyCenter()` 从 polygon 中心附近找 seed。

### 2. 边界采样

`buildPolyDetail()` 首先把 coarse polygon 顶点拷贝到 detail 顶点。

如果 `sampleDist > 0`：

1. 对每条边按 `sampleDist` 采样。
2. 每个采样点用 `getHeight()` 从 height patch 查高度。
3. 对边上采样点做误差简化。
4. 按一致的词典序处理边，避免相邻 polygon 在共享边上生成不同采样点。

一致顺序是防止 detail mesh 在 polygon 边界产生裂缝的关键。

### 3. 内部采样

如果 polygon 最小宽度足够大：

1. 用 `triangulateHull()` 先对边界点三角化。
2. 在 polygon AABB 内按 `sampleDist` 生成网格采样点。
3. 过滤离 polygon 边界太近的点。
4. 对每个候选点，计算它到当前 detail 三角网的高度误差。
5. 选择误差最大的点加入。
6. 重新用 `delaunayHull()` 三角化。
7. 直到最大误差小于 `sampleMaxError` 或达到顶点上限。

这是一个逐步细化过程：只在当前 detail mesh 不能表达真实高度的位置增加点。

单个 coarse polygon 的 detail 构建有固定上限：`MAX_VERTS = 127`、`MAX_TRIS = 255`、每条边最多 32 个采样点。达到顶点上限会停止继续插入内部误差点；三角形超过 255 时源码会截断并记录 error 日志。它可能仍返回构建结果，所以集成层不应只看 `bool`，还应在离线构建中保留 `rcContext` 错误日志。

### 4. Delaunay 三角化

`delaunayHull()` 从 hull edges 开始，调用 `completeFacet()` 给未完成边寻找左侧最佳点。

选择点时使用 circumcircle 规则：

- 点在当前外接圆内时更新候选。
- 如果接近外接圆边界，还要检查新边是否和已有边相交。

输出三角形后，`setTriFlags()` 标记 detail 边界边。Detour 会使用这些 flag 区分边界和内部 detail 边。

`rcMergePolyMeshDetails()` 用于合并多个 `rcPolyMeshDetail`。它先统计总 `nmeshes/nverts/ntris`，再顺序拷贝每个 detail mesh。关键是修正每个子 mesh 的 `vertBase` 和 `triBase`：

```text
dst.vertBase = currentTotalVerts + src.vertBase
dst.triBase  = currentTotalTris  + src.triBase
```

detail triangle 的局部索引本身不需要改，因为它们仍然相对于对应子 mesh 的 base 解释。这个函数通常和 `rcMergePolyMeshes()` 配套，用于非 tiled 或导出前合并多个构建块。

坐标回写还有一个容易忽略的源码细节：detail 顶点从局部体素高度转世界坐标时使用 `orig.y + ch`，而 coarse polygon 顶点只加 `orig.y`。源码旁保留了 “Is this offset necessary?” 注释。阅读高度误差或做跨实现对比时，应以当前这一格 `ch` 的偏移为准，不能假设两套顶点执行完全相同的 y 变换。

## 十四、内存、日志和错误模型

Recast 使用自己的 allocator 接口：

- 永久数据用 `RC_ALLOC_PERM`。
- 临时构建数据用 `RC_ALLOC_TEMP`。

公共对象一般通过 `rcAlloc*` 和 `rcFree*` 管理，例如：

- `rcAllocHeightfield()` / `rcFreeHeightField()`
- `rcAllocCompactHeightfield()` / `rcFreeCompactHeightfield()`
- `rcAllocContourSet()` / `rcFreeContourSet()`
- `rcAllocPolyMesh()` / `rcFreePolyMesh()`
- `rcAllocPolyMeshDetail()` / `rcFreePolyMeshDetail()`
- `rcAllocHeightfieldLayerSet()` / `rcFreeHeightfieldLayerSet()`

`rcContext` 负责：

- 日志。
- 构建阶段计时。
- 统一错误输出。

大部分构建函数返回 `bool`。失败原因通常通过 `ctx->log()` 输出。常见失败是内存不足、region id 溢出、顶点/多边形数量过多、轮廓过度简化导致三角化失败。

`rcContext` 的默认 `doLog()/doStartTimer()/doStopTimer()` 不会替应用保存结果；RecastDemo 使用派生 context 才能收集日志和 profile。还要注意“返回 true”不一定表示没有质量退化：例如 compact 邻接层索引过多会记录错误但继续，detail triangle 超限会截断并记录错误。离线构建工具应把 `RC_LOG_WARNING/ERROR` 当作构建产物的一部分保存和审查。

### API 的所有权模式

Recast 没有一个包办全流程的 `buildNavMesh()`。调用方显式拥有每个中间对象并决定何时释放：

| API 类型 | 所有权/修改方式 |
| --- | --- |
| `rcAlloc*()` / `rcFree*()` | 只分配或释放对象壳及其内部数组；必须配对使用 |
| `rcCreateHeightfield()`、`rcBuildCompactHeightfield()`、`rcBuildContours()`、`rcBuildPolyMesh()`、`rcBuildPolyMeshDetail()` | 调用方先分配目标对象，函数填充内部数据；失败后仍应由对应 free API 清理 |
| rasterize/filter/area/region API | 原地修改 heightfield 或 compact heightfield；调用顺序会改变后续语义 |
| merge/copy API | 分配新的目标数组并重映射索引，不会自动销毁输入 mesh |

这种低层 API 让应用可以插入自定义 area 标注、调试快照、分区策略和 tile 调度，但也意味着顺序、单位、生命周期和失败清理由集成方负责。官方 Roadmap 中“Higher-Level APIs”仍是未来方向，说明直接复制 Sample pipeline 再按项目封装是预期用法。

### 并发边界

Recast 本身不创建线程，也不为共享对象加锁。独立 tile 的构建天然适合由应用并行调度，但每个任务应使用独立的 heightfield/compact/contour/mesh 中间对象和 `rcContext`，并在最终写入共享容器时同步。`rcAllocSetCustom()` 和自定义 assert handler 是进程级回调，运行中切换它们不能视为线程安全操作。官方 Roadmap 也把完整 threading story 列为待完善项，因此“算法可并行”不等于“任意共享对象可并发调用”。

## 十五、源码阅读路径

建议按数据生命线阅读，而不是按文件顺序。

### 1. `Recast.h`

先读所有 struct：

- `rcSpan`
- `rcHeightfield`
- `rcCompactSpan`
- `rcCompactHeightfield`
- `rcContour`
- `rcPolyMesh`
- `rcPolyMeshDetail`

目标是建立每个阶段的输入输出。

### 2. `RecastRasterization.cpp`

重点看：

- `dividePoly()`
- `rasterizeTri()`
- `addSpan()`

这里解释了三角形如何变成 spans，以及为什么 span 需要合并。

### 3. `RecastFilter.cpp`

重点看三个过滤函数。这里把“表面存在”变成“agent 能站和能走”。

### 4. `Recast.cpp`

重点看：

- `rcMarkWalkableTriangles()`
- `rcBuildCompactHeightfield()`

这里完成从原始 spans 到可走 span 图的转换。

### 5. `RecastArea.cpp`

重点看：

- `rcErodeWalkableArea()`
- `rcMedianFilterWalkableArea()`
- area marking 函数。

这里解释 agent 半径和语义区域如何进入构建结果。

### 6. `RecastRegion.cpp`

重点看：

- `rcBuildDistanceField()`
- `rcBuildRegions()`
- `rcBuildRegionsMonotone()`
- `mergeAndFilterRegions()`

这里是可走 span 图变成 region 的核心。

### 7. `RecastContour.cpp`

重点看：

- `walkContour()`
- `simplifyContour()`
- `mergeRegionHoles()`
- `rcBuildContours()`

这里解释 region 边界如何变成简化多边形边界。

### 8. `RecastMesh.cpp`

重点看：

- `triangulate()`
- `getPolyMergeValue()`
- `removeVertex()`
- `buildMeshAdjacency()`
- `rcBuildPolyMesh()`

这里把 contour 转成 Detour 可用的 coarse polygon graph。

### 9. `RecastMeshDetail.cpp`

重点看：

- `getHeightData()`
- `buildPolyDetail()`
- `delaunayHull()`
- `rcBuildPolyMeshDetail()`

这里解释高度细节如何被补回。

## 十六、常见问题和根因排查

### 1. 最终没有任何 polygon

按顺序检查：

1. 输入 mesh 是否有三角形。
2. `rcMarkWalkableTriangles()` 后是否有非 null area。
3. rasterization 后 `rcHeightfield` 是否有 spans。
4. 三个过滤函数是否把 area 全清了。
5. `rcBuildCompactHeightfield()` 的 `spanCount` 是否为 0。
6. `rcBuildRegions()` 是否因 `minRegionArea` 删除了所有区域。

### 2. 斜坡消失

检查：

- `walkableSlopeAngle` 是否太低。
- 三角形 winding 是否导致法线朝下。
- `ch` 是否太大，导致 compact 邻接时高度差超过 `walkableClimb`。

### 3. 台阶无法通过

检查：

- `walkableClimb` 是否小于台阶高度。
- `rcFilterLedgeSpans()` 是否把边缘判成 ledge。
- compact 邻接条件 `abs(neighbor.y - span.y) <= walkableClimb` 是否满足。

### 4. 门洞或窄走廊消失

通常是腐蚀导致：

```text
walkableRadius = ceil(agentRadius / cs)
```

如果通道宽度小于 `2 * agentRadius`，agent 中心确实无法通过。若视觉上应该能过，需要减小 agent radius 或重新评估模型尺寸。

### 5. 轮廓过碎

检查：

- `cs` 是否太小。
- `maxSimplificationError` 是否太小。
- `maxEdgeLen` 是否强制切分太多边。

### 6. 轮廓过度简化导致错误

如果日志出现 bad outline、bad triangulation，通常是：

- `maxSimplificationError` 太大。
- 复杂 region 洞被简化成自交。
- `minRegionArea` 和 `mergeRegionArea` 让 region 合并过度。

### 7. 高度贴合不好

检查：

- `detailSampleDist` 是否为 0 或过大。
- `detailSampleMaxError` 是否过大。
- coarse polygon 是否过大且地形起伏明显。

### 8. tiled navmesh 边界不连

检查：

- tile 构建时是否使用足够 `borderSize`。
- `RC_BORDER_VERTEX` 是否被保留并在 poly mesh 阶段正确处理。
- `rcBuildPolyMesh()` 是否把开放边标记为 `0x8000 | side`。
- 每个 tile 是否查询并栅格化了所有触碰其扩张后 `bmin/bmax` 的几何，而不只是本次新增/变化的物体。

官方论坛明确指出 RecastDemo 的 chunky tri mesh 只是“按 tile 快速查询输入三角形”的示例设施，并不是 Recast 的必需数据结构。实际引擎应使用自己的 world/spatial query 收集完整相关几何。重建 tile 时若只传新箱子而漏掉原地形，Recast 会忠实地生成“只有箱子”的 tile；它不会从旧 navmesh 反推出缺失三角形。

### 9. 想在墙面、天花板或不同重力方向上生成 navmesh

这是 Recast/Detour 的架构限制，不是某个参数问题。源码中大量地方把 xz 当作水平平面、y 当作高度方向，官方社区讨论也明确说直立角色和重力方向假设很多。可行的工程绕法通常是：

1. 把输入几何先旋转到 Recast 期望的 XZ ground plane。
2. 在这个“仿真空间”里构建 navmesh、运行 Detour 查询。
3. 把路径和 agent 位置再反向变换回游戏/渲染空间。

但这只适合同一套 up axis 的场景。多个不同 up axis 的 navmesh 不能由 Recast/Detour 自动连接成统一导航图；如果必须跨墙面、地面、天花板移动，通常需要上层用 off-mesh link、分区仿真或专门的 3D/表面导航系统处理。

## 十七、设计取舍总结

1. **体素化优先于直接几何处理。**

   体素化降低几何复杂度，让后续算法用规则网格和 span 图处理。代价是精度受 `cs/ch` 限制。

2. **构建期烘焙 agent 约束。**

   半径、高度、爬升都在构建期处理，运行时 agent 可以近似为点。代价是不同尺寸 agent 通常需要不同 navmesh 或不同构建层。

3. **compact heightfield 是核心中间表示。**

   它把链表 spans 转成数组和邻接图，是区域分区、轮廓、detail 采样的共同基础。

4. **区域分区服务于轮廓质量。**

   Watershed、monotone、layers 都不是最终结果，它们的目的都是让轮廓可提取、可简化、可三角化。

5. **coarse/detail 双层 mesh。**

   coarse mesh 保持拓扑简洁，detail mesh 保持高度精度。这样 Detour 查询既快又能贴地。

6. **大量固定上限是实时工程取舍。**

   span 高度 bit 数、连接 bit 数、region id、detail 顶点/三角形上限都限制了极端场景，但换来紧凑数据和可控构建成本。
