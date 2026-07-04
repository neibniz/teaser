# Sample_SoloMesh.cpp 逻辑梳理

本文整理 `RecastDemo/Source/Sample_SoloMesh.cpp` 的主要逻辑，重点解释单块导航网格从输入三角网格到 Detour 可查询数据的每个阶段，以及每个阶段涉及的设计、算法和数据结构。

`Sample_SoloMesh` 是 RecastDemo 中最基础的导航网格构建样例。它构建的是一个单块 navmesh，不做瓦片切分，不支持增量重建，也不处理超大场景流式加载。因此它很适合作为理解 Recast 构建流水线的入口。

## 总体职责

`Sample_SoloMesh` 继承自 `Sample`，主要负责四件事：

1. 管理 Recast 构建过程中产生的中间数据。
2. 根据 UI 参数执行完整 navmesh 构建流水线。
3. 把 Recast 生成的多边形网格打包成 Detour 的 `dtNavMesh`。
4. 提供调试绘制和工具交互，例如路径测试、off-mesh connection、convex volume、crowd。

核心流程可以概括为：

```text
InputGeom::Mesh
  -> rcHeightfield
  -> rcCompactHeightfield
  -> rcContourSet
  -> rcPolyMesh
  -> rcPolyMeshDetail
  -> dtNavMeshCreateParams
  -> dtNavMesh + dtNavMeshQuery
```

这个流程体现了 Recast 的核心思想：不要直接在原始三角网格上做寻路网格，而是先把几何体素化，再在体素空间中过滤、分区、简化，最后重新生成适合寻路的凸多边形图。

## 根源视角：数据如何一步步改变语义

`Sample_SoloMesh` 不是单纯把一种 mesh 格式转换成另一种 mesh 格式。它每一步都在改变数据的语义：

```text
原始三角形
  表达视觉/碰撞几何
      |
      v
rcHeightfield
  表达每个 xz cell 中有哪些高度占用
      |
      v
过滤后的 spans
  表达 agent 脚下能不能站、头顶有没有空间、边缘是否安全
      |
      v
rcCompactHeightfield
  表达可走 span 之间的邻接图
      |
      v
regions
  表达连通可走空间被分成哪些区域
      |
      v
contours
  表达每个区域的边界
      |
      v
rcPolyMesh
  表达可用于路径搜索的凸多边形拓扑图
      |
      v
rcPolyMeshDetail
  给 coarse polygon 补回地形高度细节
      |
      v
dtNavMesh
  表达运行时可查询、可保存、可加载的导航图
```

这条链路里最重要的变化是：

1. **从连续几何变成离散体素。**

   原始三角形可以任意细、任意斜、任意重叠。Recast 先把它们投进规则网格，让后续算法可以用邻接、距离变换、区域增长等离散算法处理。

2. **从“几何能站”变成“agent 中心能走”。**

   坡度过滤、净空过滤、ledge 过滤、半径腐蚀共同作用后，剩下的不是地面本身，而是 agent 中心点可达的安全区域。

3. **从 cell 图变成 convex polygon 图。**

   compact heightfield 很适合构建，但运行时逐 cell 搜索太重。Recast 把连通区域边界简化成凸多边形，让 Detour 在更小的图上做 A*。

4. **从构建数据变成运行时数据。**

   `rcHeightfield`、`rcCompactHeightfield`、`rcContourSet` 等是构建中间态。最终游戏运行时通常只需要 `dtNavMesh` 和 `dtNavMeshQuery`。

### 每个阶段不能省的原因

| 阶段 | 如果省掉会怎样 | 保留它解决什么根问题 |
| --- | --- | --- |
| 体素化 | 原始三角形噪声和重叠会直接影响寻路 | 把复杂几何统一成规则采样空间 |
| 过滤 | 坡面、悬崖、低矮空间会被误认为可走 | 把 agent 的物理限制烘焙进可走性 |
| compact | 链表 span 遍历慢，邻接关系隐含 | 把可走 spans 变成适合图算法的数据 |
| 腐蚀 | navmesh 边界贴墙，运行时还要处理半径碰撞 | 构建阶段把 agent 半径转成中心点安全区 |
| 分区 | 所有可走 span 只是一团 cell，无法稳定提轮廓 | 把连通空间拆成可轮廓化的区域 |
| 轮廓简化 | 边界呈楼梯状，polygon 数量爆炸 | 用误差阈值保留形状，同时减少顶点 |
| poly mesh | cell/contour 不适合 A* | 生成凸 polygon 拓扑图 |
| detail mesh | coarse polygon 丢失高度细节 | 查询时能贴合真实地形高度 |
| Detour 打包 | Recast 输出不是运行时查询结构 | 固化 ref、link、BVTree、off-mesh 等运行时信息 |

## 主要对象和数据结构

| 名称 | 所在位置 | 作用 | 为什么需要 |
|---|---|---|---|
| `Sample_SoloMesh` | `Sample_SoloMesh.h/.cpp` | 单块 navmesh 样例类 | 把 UI、构建、渲染、工具状态组织在一个 sample 内 |
| `InputGeom` | `InputGeom.h` | 输入几何和用户标注数据 | 统一保存 mesh、bounds、off-mesh connections、convex volumes |
| `InputGeom::Mesh` | `InputGeom.h` | 原始顶点、三角形、法线 | Recast 的输入是三角形 soup，构建开始时需要直接访问数组 |
| `rcConfig` | `Recast.h` | 构建参数集合 | 把世界单位参数转换为体素单位，并传给各个 Recast 阶段 |
| `triAreas` | `Sample_SoloMesh` 成员 | 每个输入三角形的 area id | 体素化前标记哪些三角形可走，哪些不可走 |
| `rcHeightfield` | `Recast.h` | 原始体素高度场，按格子列存 spans | 保留三角形体素化后的遮挡和高度信息 |
| `rcCompactHeightfield` | `Recast.h` | 紧凑高度场，只保留可走 span | 更适合后续邻接、距离场、分区等算法，缓存友好 |
| `rcContourSet` | `Recast.h` | 每个区域的轮廓集合 | 将体素区域边界抽象成可简化的多边形边界 |
| `rcPolyMesh` | `Recast.h` | 粗略导航多边形网格 | 形成 Detour 的基础导航图节点和连接 |
| `rcPolyMeshDetail` | `Recast.h` | 每个多边形内部的高度细节三角网格 | 粗 navmesh 负责拓扑，detail mesh 负责贴近真实地形高度 |
| `dtNavMeshCreateParams` | `DetourNavMeshBuilder.h` | Recast 到 Detour 的打包参数 | 把 Recast 输出和 off-mesh connection 等数据组装成 Detour tile data |
| `dtNavMesh` | Detour | 运行时导航网格 | 保存多边形图、边连接、BVTree、off-mesh connection |
| `dtNavMeshQuery` | Detour | 查询接口 | 路径搜索、最近多边形、raycast、findPath 等运行时查询都经由它完成 |

## 类级逻辑

### 构造与析构

构造函数默认启用 `NavMeshTesterTool`：

```cpp
Sample_SoloMesh::Sample_SoloMesh()
{
    setTool(new NavMeshTesterTool);
}
```

这是为了让用户一构建出 navmesh 就可以点击场景测试路径。析构函数调用 `cleanup()`，释放所有构建中间数据和 `navMesh`。

### cleanup

`cleanup()` 会释放：

```text
triAreas
rcHeightfield
rcCompactHeightfield
rcContourSet
rcPolyMesh
rcPolyMeshDetail
dtNavMesh
```

这些对象有不同的分配器：

- `triAreas` 用 `new[]` 分配，所以用 `delete[]`。
- Recast 中间结构用 `rcAlloc*` 分配，所以用 `rcFree*`。
- Detour navmesh 用 `dtAllocNavMesh` 分配，所以用 `dtFreeNavMesh`。

这样做的原因是 Recast/Detour 支持替换自定义 allocator。调用对应的释放函数可以避免跨 allocator 释放带来的错误。

### UI 和调试模式

`drawSettingsUI()` 负责通用构建参数、保存、加载和构建耗时显示。

`drawToolsUI()` 负责切换工具：

- Test Navmesh
- Prune Navmesh
- Create Off-Mesh Connections
- Create Convex Volumes
- Create Crowds

`drawDebugUI()` 负责切换中间结果可视化。它不是额外逻辑，而是把构建流水线中的每个阶段暴露出来，例如 voxels、compact heightfield、regions、contours、poly mesh、detail mesh。

这类可视化非常关键，因为 navmesh 构建错误通常发生在中间阶段。如果只看最终 navmesh，很难判断问题来自体素化、过滤、分区还是轮廓简化。

## build() 流程总览

`build()` 是整个文件的核心。它从当前 `inputGeometry` 读取三角形数据，依次执行 Recast 的构建步骤，最后初始化 Detour 查询对象。

伪代码如下：

```text
validate input
cleanup old build data
read mesh bounds, verts, tris

Step 1: fill rcConfig
Step 2: rasterize triangles into rcHeightfield
Step 3: filter walkable surfaces
Step 4: build compact heightfield and partition regions
Step 5: trace and simplify contours
Step 6: build rcPolyMesh
Step 7: build rcPolyMeshDetail
Step 8: create Detour navmesh data and dtNavMesh

record build time
initialize tools
```

## Step 0: 输入检查和旧数据清理

```cpp
if (!inputGeometry || inputGeometry->mesh.verts.empty())
{
    buildContext->log(RC_LOG_ERROR, "buildNavigation: Input mesh is not specified.");
    return false;
}

cleanup();
```

### 关键设计

构建前先清掉旧的中间数据，保证每次构建都是从当前 mesh 和当前 UI 参数出发。这样 UI 调参后点击 Build，不会混入上一轮的高度场、轮廓或 Detour 数据。

### 数据来源

```cpp
const float* boundsMin = inputGeometry->getNavMeshBoundsMin();
const float* boundsMax = inputGeometry->getNavMeshBoundsMax();
const float* verts = inputGeometry->mesh.verts.data();
const int* tris = inputGeometry->mesh.tris.data();
```

`InputGeom` 既可以来自 `.obj`，也可以来自 `.gset`。`.gset` 可能保存过用户指定的 navmesh bounds 和构建参数，所以这里通过 `getNavMeshBoundsMin/Max()` 间接读取，而不是直接使用 mesh bounds。

## Step 1: 初始化 rcConfig

`rcConfig` 是 Recast 构建参数的集中入口。

```cpp
config.cs = cellSize;
config.ch = cellHeight;
config.walkableSlopeAngle = agentMaxSlope;
config.walkableHeight = static_cast<int>(ceilf(agentHeight / config.ch));
config.walkableClimb = static_cast<int>(floorf(agentMaxClimb / config.ch));
config.walkableRadius = static_cast<int>(ceilf(agentRadius / config.cs));
```

### 关键设计

UI 里的参数大多是世界单位，例如 agent height、agent radius、max climb。Recast 后续算法运行在体素网格上，所以这里必须把它们转换成体素单位：

- `walkableHeight = ceil(agentHeight / ch)`
  向上取整，确保空间高度至少能容纳 agent。

- `walkableClimb = floor(agentMaxClimb / ch)`
  向下取整，避免把略高于最大跨越高度的台阶误判为可走。

- `walkableRadius = ceil(agentRadius / cs)`
  向上取整，确保障碍边缘收缩距离不会小于 agent 半径。

这些取整方向很重要。它们共同偏向保守结果，宁可少一点可走空间，也不要让 agent 进入几何上无法通过的位置。

### 算法细节：从连续世界到离散体素

`rcConfig` 的初始化本质上是在定义一个离散采样空间。原始场景是连续坐标，Recast 后续算法却运行在规则网格上，所以这里要决定三件事：

1. xz 平面每个格子覆盖多大世界空间。
2. y 轴每个高度层代表多大世界高度。
3. agent 的连续尺寸映射成多少个格子。

这个离散化会直接影响所有后续算法。假设 `cellSize = 0.3`，`agentRadius = 0.6`，那么 `walkableRadius = ceil(0.6 / 0.3) = 2`，后续腐蚀算法会把障碍边缘向内收缩 2 个格子。也就是说 navmesh 边界表示的是 agent 中心点能到达的位置，而不是原始墙体位置。

`walkableHeight` 用 `ceil`，是因为只要 agent 高度不是 cell height 的整数倍，就必须多占一层体素。例如 agent 高 2.0，`cellHeight = 0.3`，实际需要 `ceil(6.666) = 7` 层，也就是 2.1 的净空。少算一层会让 agent 穿过本来不够高的洞口。

`walkableClimb` 用 `floor`，是因为跨台阶能力不能被夸大。例如最大跨越高度 0.9，`cellHeight = 0.4`，只能安全认为可跨 2 层，也就是 0.8。向上取整到 3 层会允许 1.2 高度差，被错误判定为可通行。

`minRegionArea` 和 `mergeRegionArea` 也会从 UI 的“长度感知参数”转成面积：

```cpp
config.minRegionArea = static_cast<int>(rcSqr(regionMinSize));
config.mergeRegionArea = static_cast<int>(rcSqr(regionMergeSize));
```

这是因为区域过滤处理的是 span 数量近似面积，而 UI 用边长更容易理解。`regionMinSize = 8` 对应的不是 8 个体素，而是约 `8 x 8 = 64` 个 span 的小岛阈值。

### 网格范围

```cpp
rcVcopy(config.bmin, boundsMin);
rcVcopy(config.bmax, boundsMax);
rcCalcGridSize(config.bmin, config.bmax, config.cs, &config.width, &config.height);
```

`bmin/bmax` 定义要构建的世界空间 AABB。`rcCalcGridSize` 根据 cell size 计算 xz 平面的体素宽高。

### 参数对结果的影响

| 参数 | 影响 | 取舍 |
|---|---|---|
| `cellSize` | xz 平面分辨率 | 越小越精细，但体素数量和构建时间快速增加 |
| `cellHeight` | y 轴分辨率 | 越小越能表达台阶和高度变化，但数据更多 |
| `agentHeight` | 可站立空间要求 | 过大导致低矮区域不可走，过小可能穿过低天花板 |
| `agentRadius` | 离障碍边缘的安全距离 | 过大使走廊变窄或消失，过小会贴边 |
| `agentMaxClimb` | 可跨越台阶高度 | 过大会跨过不该跨的障碍，过小会上不了台阶 |
| `agentMaxSlope` | 最大可走坡度 | 控制斜坡是否参与导航 |

## Step 2: 三角网格体素化

这一阶段把输入三角形写入 `rcHeightfield`。

```cpp
heightfield = rcAllocHeightfield();
rcCreateHeightfield(...);
```

### rcHeightfield 的结构

`rcHeightfield` 是一个 xz 网格，每个格子列里有一个或多个 `rcSpan`。一个 span 表示该列中一段被几何占据或影响的高度范围。

```text
cell(x, z)
  span 0: [smin, smax), area
  span 1: [smin, smax), area
```

### 标记可走三角形

```cpp
triAreas = new unsigned char[numTris];
memset(triAreas, 0, numTris * sizeof(unsigned char));
rcMarkWalkableTriangles(..., triAreas);
```

`rcMarkWalkableTriangles` 根据三角形法线和 `walkableSlopeAngle` 判断哪些三角形可走，并把结果写入 `triAreas`。

这里不是直接删除不可走三角形，而是先把 area 记录下来。原因是体素化时还需要完整几何作为障碍来源。不可走面虽然不能成为地面，但仍然会影响空间占用和过滤。

### 算法细节：坡度判定

`rcMarkWalkableTriangles` 的核心是法线与世界上方向的夹角判断。Recast 使用 Y-up 坐标系，所以三角面法线的 `y` 分量越大，表面越接近水平。

概念上可以理解为：

```text
normal = normalize(cross(v1 - v0, v2 - v0))
walkableThreshold = cos(walkableSlopeAngle)

if normal.y > walkableThreshold:
    triAreas[i] = RC_WALKABLE_AREA
else:
    triAreas[i] = RC_NULL_AREA
```

例如 `agentMaxSlope = 45` 度时，阈值约为 `cos(45) = 0.707`。法线 y 分量大于 0.707 的面会被认为坡度不超过 45 度，可以成为候选地面。

这个判断只回答“这个三角形本身是否像地面”，还没有考虑 agent 高度、半径、台阶、悬空、边缘安全等问题。那些约束在后续过滤和腐蚀阶段处理。

### 体素化三角形

```cpp
rcRasterizeTriangles(..., *heightfield, config.walkableClimb)
```

`rcRasterizeTriangles` 将三角形投影到体素网格覆盖的 cell 中，并生成对应 spans。

### 算法细节：三角形如何变成 spans

体素化阶段不是简单地“每个三角形中心落在哪个格子，就填哪个格子”。它会处理三角形覆盖到的所有 xz cell。整体思路是：

1. 计算三角形的世界空间包围盒。
2. 将包围盒投影到 xz 网格，得到可能覆盖的 cell 范围。
3. 对每个候选 cell，用 cell 的 xz 边界裁剪三角形。
4. 如果裁剪后多边形非空，计算它在 y 轴上的最小和最大高度。
5. 将 y 范围转换为 `smin/smax` 体素高度。
6. 调用 span 插入逻辑写入 `rcHeightfield`。

裁剪这一步很重要。一个大三角形可能横跨很多 cell，如果只看三角形顶点或中心，会漏掉它经过的格子。Recast 通过在 xz 平面上逐格裁剪三角形，保证每个被三角形覆盖的 cell 都能产生高度信息。

span 插入时会按高度顺序合并重叠或相邻的 span。这样多个三角形落在同一 cell、并形成连续实体时，不会产生大量碎片 span。`flagMergeThreshold` 也就是这里传入的 `walkableClimb`，用于决定高度接近的 span 在合并时 area 标记如何处理。这个合并行为让 voxel 数据更稳定，也减少后续遍历成本。

最终 `rcHeightfield` 保存的是“每个 xz cell 里哪些高度区间被几何占用或影响”。它还不是可走空间图，更像是从三角形世界压缩出来的高度柱状表示。

### 为什么先体素化

直接从原始三角形生成 navmesh 会遇到很多复杂问题：

- 输入 mesh 可能有很多细碎三角形。
- 几何可能有缝隙、重叠、薄片和非流形结构。
- 寻路需要的是可站立区域，不是视觉模型本身。

体素化把几何转换为规则网格，后续的过滤、腐蚀、连通区域分析都会变得稳定且易调试。

## Step 3: 过滤可走表面

体素化后的数据是保守的，可能包含不适合站立的 spans。过滤阶段通过 agent 参数清理它们。

```cpp
rcFilterLowHangingWalkableObstacles(...)
rcFilterLedgeSpans(...)
rcFilterWalkableLowHeightSpans(...)
```

### Low Hanging Obstacles

低悬障碍过滤会把低矮障碍附近仍然可跨越的 spans 继承为可走。

设计意图：如果一个小台阶或低障碍在 `walkableClimb` 范围内，agent 可以跨过去，不应该因此切断导航。

算法上，它会按每个 cell 的 span 从低到高遍历。如果当前 span 与下方上一个可走 span 的高度差小于等于 `walkableClimb`，当前 span 可以继承可走 area。这样小台阶、路缘、矮障碍顶部不会因为三角形本身坡度或体素化细节而被误判为断路。

这个过滤器解决的是“低障碍可跨越”问题，但它不是万能的。它不会让过高障碍变可走，也不会处理水平空间不足的问题。它只在同一列或局部高度相近的 spans 上做可走性传播。

### Ledge Spans

`rcFilterLedgeSpans` 移除靠近过高落差的 spans。

设计意图：一个平面本身可能坡度可走，但如果旁边是超过 `walkableClimb` 的垂直落差，agent 不应该站在这个危险边缘并认为可以跨过去。

算法上，它会检查每个可走 span 的四邻域连接情况：

1. 取当前 span 的地面高度。
2. 查看东南西北四个相邻 cell 中可到达的 span。
3. 计算相邻 span 与当前 span 的高度差。
4. 如果某个方向没有可站立邻居，或高度落差超过 `walkableClimb`，这个 span 就可能是 ledge。
5. 如果邻域内可达高度的最大差异过大，也会被标记为不可走。

这一步不仅处理悬崖边，也处理“高度变化太剧烈”的区域。它避免 navmesh 在垂直断层或高台边缘紧贴生成，从而减少 agent 在边缘卡住、掉落或选择不合理路径的可能。

### Walkable Low Height Spans

`rcFilterWalkableLowHeightSpans` 移除头顶空间不足的 spans。

设计意图：agent 不是一个点。即使脚下地面可走，如果上方净空小于 `walkableHeight`，就不能通行。

算法上，对同一 cell 中的相邻 spans，当前 span 的顶部和上方下一个 span 的底部之间形成可站立空间。Recast 会计算这个间隙：

```text
clearance = nextSpan.smin - currentSpan.smax
```

如果 `clearance < walkableHeight`，说明 agent 站在当前 span 上会撞到上方几何，于是当前 span 被标记为不可走。没有上方 span 时，可以理解为头顶空间无限大。

这个过滤器处理的是桥底、门洞、低天花板、楼梯下方等场景。它发生在体素空间中，因此对细碎三角形和复杂上方结构比较稳定。

## Step 4: 构建紧凑高度场并分区

### 构建 rcCompactHeightfield

```cpp
compactHeightfield = rcAllocCompactHeightfield();
rcBuildCompactHeightfield(..., *heightfield, *compactHeightfield);
```

`rcHeightfield` 使用链表 spans，适合动态写入。后续算法需要大量遍历和邻接访问，因此 Recast 会把它转换成 `rcCompactHeightfield`。

`rcCompactHeightfield` 只保留可走空间，并用数组保存：

- `cells`: 每个 xz cell 对应的 span 起始索引和数量。
- `spans`: 所有可走 spans。
- `areas`: 每个 span 的 area id。
- `dist`: 到边界的距离场数据。

### 为什么要 compact

紧凑结构有三个好处：

1. 遍历更快，数组比链表更缓存友好。
2. 只保留后续需要的可走空间，减少数据量。
3. 生成邻接连接，分区和轮廓提取需要知道 span 之间是否连通。

### 算法细节：压缩和邻接连接

`rcBuildCompactHeightfield` 大致做两件事：压缩可走 spans，建立邻接关系。

压缩过程会先统计 `rcHeightfield` 中 area 不为 `RC_NULL_AREA` 的 spans，然后分配连续数组。每个 xz cell 在 `rcCompactCell` 中只保存两个信息：

```text
index: 这个 cell 的第一个 compact span 在 spans 数组里的位置
count: 这个 cell 有多少个 compact span
```

每个 `rcCompactSpan` 保存：

```text
y: span 底部高度
h: 可用高度
reg: 后续分区产生的 region id
con: 四个方向的邻接连接，打包存储
```

邻接连接建立时，Recast 会检查四个方向相邻 cell 的 spans。只有当两个 span 的高度差不超过 `walkableClimb`，并且它们之间的垂直净空至少有 `walkableHeight`，才会认为它们连通。

也就是说，从这里开始，“可走”不只是单个 span 的属性，还变成了 span 之间的图连接关系。后续分区算法本质上就是在这个 compact span 图上做区域划分。

### 腐蚀可走区域

```cpp
rcErodeWalkableArea(buildContext, config.walkableRadius, *compactHeightfield)
```

腐蚀会按 agent 半径向内收缩可走区域。

这是 Recast 的重要设计：在构建阶段把 agent 半径烘焙进 navmesh，让运行时的 agent 可以被近似为一个点在 navmesh 上运动。

这样做的好处是运行时查询更简单：

- 不需要每次路径查询都拿 agent 圆柱去和障碍边界做碰撞。
- navmesh 边界已经天然代表 agent 中心不能越过的位置。

代价是一个 navmesh 通常只适合一种 agent 尺寸。不同半径或高度的角色，往往需要不同 navmesh。

### 算法细节：距离变换和形态学腐蚀

`rcErodeWalkableArea` 可以理解为体素网格上的形态学腐蚀。它要找出距离障碍太近的可走 spans，并把它们标记为不可走。

概念过程是：

1. 找到所有边界 spans：不可走区域旁边、缺少邻接连接、或者靠近障碍的 spans。
2. 从这些边界开始计算每个可走 span 到最近边界的网格距离。
3. 如果距离小于 `walkableRadius`，把该 span 的 area 置为不可走。

这和图像处理里的“腐蚀”非常像：白色可走区域会从边缘向内缩小，缩小距离就是 agent 半径。

它的关键价值是把运行时几何碰撞前移到构建期。经过腐蚀后，Detour 查询得到的是 agent 中心线可走区域。路径规划只需要保证中心点在 navmesh 内，agent 的圆柱体就不会穿墙。

腐蚀也解释了为什么窄门和走廊会消失。如果走廊宽度小于 `2 * agentRadius`，两侧腐蚀会相遇，中间不再有可走 span。这不是错误，而是表示该 agent 物理上过不去。

### Convex Volume 标注

```cpp
for (ConvexVolume& vol : inputGeometry->convexVolumes)
{
    rcMarkConvexPolyArea(..., (unsigned char)vol.area, *compactHeightfield);
}
```

Convex volume 是用户在 demo 中画出的凸多边形区域。它可以把某些空间标记为不同 area，例如 water、grass、road、door。

设计意图：导航不只需要知道“能不能走”，还需要知道“走这里的代价或能力要求”。area id 后续会映射到 Detour poly flags 和 query filter。

算法上，`rcMarkConvexPolyArea` 会把 convex volume 的 xz 投影当作一个凸多边形，再用 `hmin/hmax` 限制高度范围。所有落在这个体积内的 compact spans 会被改写为指定 area。

这里要求 volume 是凸的，是为了让点在多边形内的测试更简单稳定。凸多边形可以用所有边的半空间测试判断包含关系，不需要处理复杂自交或凹多边形分解。

### 区域分区算法

代码支持三种分区方式：

```cpp
WATERSHED
MONOTONE
LAYERS
```

分区的目标是把可走 spans 分组成连通区域，每个区域之后会被抽轮廓和三角化。

#### Watershed

```cpp
rcBuildDistanceField(...)
rcBuildRegions(...)
```

Watershed 先构建距离场，再从远离边界的地方扩张区域。

算法细节：

1. `rcBuildDistanceField` 先计算每个可走 span 到最近障碍边界的距离。距离越大，说明越处在开阔区域内部。
2. `rcBuildRegions` 从距离场的高值位置开始创建种子区域。
3. 区域像水流填充一样向低距离位置扩张，但会受连通性、area、边界和最小区域面积约束。
4. 扩张后，小于 `minRegionArea` 的孤立区域会被删除。
5. 小区域如果能和相邻大区域合并，并且面积低于 `mergeRegionArea`，会被合并。

Watershed 的直觉是：开阔区域中心先成为区域核心，然后向边界扩张。这样生成的区域通常更圆润、更符合房间和走廊结构，后续轮廓和多边形也更自然。

特点：

- 生成的区域形状通常最好。
- 多边形质量较高。
- 速度通常最慢。
- 在某些狭窄螺旋或障碍贴近开阔区域的场景可能出现重叠或洞。

适合离线构建、小中型单块 navmesh、追求较好拓扑质量的场景。

#### Monotone

```cpp
rcBuildRegionsMonotone(...)
```

Monotone 不需要距离场，速度快，并且能保证没有区域重叠和洞。

算法细节：

1. 按扫描线遍历 compact heightfield。
2. 在扫描过程中把连续可走 spans 归入单调区域。
3. 当相邻扫描线的 span 连续性发生变化时，创建、延续或合并 region。
4. 最终得到的区域在某个扫描方向上保持单调性，因此轮廓三角化更不容易失败。

它的稳定性来自算法约束：单调区域不会形成复杂洞和重叠。但这个约束也会牺牲形状质量。遇到自然弯曲空间时，它可能沿扫描方向切出很长的区域，后续 polygon 也更容易长而薄。

特点：

- 最快。
- 结果稳定。
- 可能产生细长多边形，路径可能出现不自然绕行。

适合快速构建或对质量要求较低的场景。

#### Layers

```cpp
rcBuildLayerRegions(...)
```

Layer 分区在速度和质量之间折中，常用于 tiled navmesh。

算法细节：

1. 先根据高度和连通关系把 compact spans 分成不重叠的层。
2. 每一层内部再做区域生成。
3. 层与层之间在 xz 上可以重叠，但在 y 方向分离，因此适合桥、楼层、坡道上下交叠的结构。
4. 生成区域时避免 watershed 的部分距离场扩张问题，但仍能比 monotone 产生更自然的边界。

Layer 的名字容易误解。它不是简单按固定高度切片，而是按可走 span 的连通层次组织区域。对于 tiled navmesh，它常常更稳，因为每个 tile 内需要处理被边界截断的局部几何，layer 分区对这类局部构建更友好。

特点：

- 比 watershed 快。
- 避免 watershed 的部分 corner cases。
- 质量通常好于 monotone。
- 大开阔区域中有小障碍时可能不够漂亮。

## Step 5: 追踪并简化区域轮廓

```cpp
contourSet = rcAllocContourSet();
rcBuildContours(..., config.maxSimplificationError, config.maxEdgeLen, *contourSet);
```

### 数据结构

`rcContourSet` 包含多个 `rcContour`。每个 `rcContour` 代表一个区域边界：

- `rverts`: 原始轮廓点。
- `verts`: 简化后的轮廓点。
- `reg`: region id。
- `area`: area id。

### 算法作用

这个阶段把体素区域的边界追踪成二维轮廓，并根据参数简化：

- `maxSimplificationError`: 简化轮廓允许偏离原始体素边界的最大误差。
- `maxEdgeLen`: 轮廓边的最大长度，控制长边是否需要插入额外顶点。

### 算法细节：边界追踪

`rcBuildContours` 首先在 compact heightfield 的 region 标记上找边界。一个 span 的某条边如果满足以下条件之一，就属于轮廓边：

- 边外没有连通 span。
- 边外 span 属于不同 region。
- 边外 span 的 area 类型不同。

找到边界后，算法会沿着 region 外边缘行走，收集一串原始轮廓点，也就是 `rverts`。这些点仍然在体素格子边界上，所以通常呈现明显的楼梯状。

一个轮廓点不只保存 x/y/z，还会保存额外连接和 region 信息。这些信息用于后续处理洞、门户边、区域连接以及调试绘制。

### 算法细节：轮廓简化

原始轮廓可能有很多点。简化过程类似 Douglas-Peucker 线段简化：

1. 先用少量关键点近似原始轮廓。
2. 对每条简化线段，计算原始轮廓点到该线段的最大偏差。
3. 如果最大偏差超过 `maxSimplificationError`，把偏差最大的点加入简化轮廓。
4. 反复执行，直到所有边的误差都在阈值内。
5. 如果某条边长度超过 `maxEdgeLen`，再插入额外点限制边长。

`maxSimplificationError` 越大，轮廓越简洁，但越可能削掉细节。`maxEdgeLen` 则主要控制长边。长边过多时，某些区域连接、后续三角化和路径贴边表现可能不够理想。

### 为什么要简化

体素边界是格子化的，可能非常锯齿。如果直接三角化，最终 navmesh 会有大量无意义顶点和边。

简化轮廓的目标是减少多边形复杂度，同时保留足够形状精度。navmesh 面向寻路，不是渲染模型，所以通常不需要完全贴合原始几何。

## Step 6: 构建 rcPolyMesh

```cpp
polyMesh = rcAllocPolyMesh();
rcBuildPolyMesh(buildContext, *contourSet, config.maxVertsPerPoly, *polyMesh);
```

`rcBuildPolyMesh` 将简化后的轮廓转成凸多边形网格。

### 算法细节：从轮廓到可导航凸多边形

`rcBuildPolyMesh` 的输入是一组区域轮廓。每个轮廓描述一个 region 的边界。算法大致分为几步：

1. 对每个轮廓进行三角化，先得到三角形集合。
2. 在同一 region 内尝试合并相邻三角形。
3. 合并时必须保证结果仍然是凸多边形。
4. 合并后的 polygon 顶点数不能超过 `maxVertsPerPoly`。
5. 建立 polygon 之间的邻接关系。
6. 将 region id、area id 写入 polygon 元数据。

先三角化再合并，是因为任意简单多边形直接分解成最优凸多边形比较复杂；三角形天然是凸的，作为中间表示稳定可靠。然后再把三角形尽量合并，可以减少 Detour 图节点数量。

合并时的关键约束是凸性。两个 polygon 共享一条边后，如果合并结果出现凹角，就不能合并。凹多边形不适合 Detour 的运行时移动和 funnel 处理，因为 polygon 内部的直线可见性不再天然成立。

### rcPolyMesh 的结构

`rcPolyMesh` 保存：

- `verts`: 多边形顶点，使用体素坐标。
- `polys`: 每个 polygon 的顶点索引和邻接信息。
- `areas`: 每个 polygon 的 area id。
- `flags`: 用户定义的通行 flag。
- `nvp`: 每个 polygon 最多顶点数。

### 为什么是凸多边形

Detour 的路径搜索把每个 polygon 当作图节点，把相邻边当作图边。凸多边形有两个关键优势：

1. polygon 内任意两点之间的直线都在 polygon 内，局部移动和 funnel 算法更简单。
2. 节点数量比三角网格少，路径搜索图更小。

`vertsPerPoly` 默认是 6。它必须不超过 Detour 的 `DT_VERTS_PER_POLYGON`，否则不能打包成 Detour navmesh。

### 邻接关系如何形成导航图

`rcPolyMesh::polys` 不只存 polygon 顶点，也存邻接信息。对 Detour 来说，最终 navmesh 是一张图：

```text
polygon = graph node
shared edge = graph edge
```

如果两个 polygon 共享一条边，agent 就可以从一个 polygon 进入另一个 polygon。路径搜索不需要在三角形像素或体素上跑，而是在这个更小的 polygon 图上跑。

这也是 navmesh 相比网格寻路的重要优势：节点更少，路径更平滑，后续 string-pulling 可以在 polygon corridor 中生成自然折线路径。

## Step 7: 构建 rcPolyMeshDetail

```cpp
detailMesh = rcAllocPolyMeshDetail();
rcBuildPolyMeshDetail(
    buildContext,
    *polyMesh,
    *compactHeightfield,
    config.detailSampleDist,
    config.detailSampleMaxError,
    *detailMesh);
```

### 关键设计

`rcPolyMesh` 是粗略的导航拓扑，重点是平面连接关系。它可能不能精确表达原始地形高度。

`rcPolyMeshDetail` 为每个 polygon 附加局部三角形细节，用来查询更准确的地面高度。

### 为什么拆成 coarse mesh 和 detail mesh

如果直接让 navmesh polygon 完全贴合地形高度，会增加图复杂度。Recast 将问题拆开：

- 粗 polygon 负责寻路拓扑。
- detail mesh 负责高度采样。

这样路径搜索仍然运行在较小的图上，但 agent 贴地高度可以更准确。

### 参数含义

- `detailSampleDist`: 采样间距。值越大，采样越稀疏。
- `detailSampleMaxError`: detail mesh 与高度场允许的最大误差。

如果 `detailSampleDist < 0.9f`，代码把它设为 0，表示不额外采样，只保留基础细节。

### 算法细节：高度细节采样

`rcBuildPolyMeshDetail` 会为每个粗 polygon 构建一小片局部三角网格。它的工作不是改变可通行拓扑，而是恢复高度细节。

概念过程是：

1. 取一个粗 polygon 的边界。
2. 根据 `detailSampleDist` 在 polygon 内部生成采样点。
3. 从 `rcCompactHeightfield` 查询这些采样点对应的高度。
4. 根据 `detailSampleMaxError` 判断当前 detail mesh 与高度场偏差是否过大。
5. 在偏差大的地方保留或插入更多采样点。
6. 对 polygon 内的点做局部三角化，写入 `rcPolyMeshDetail`。

粗 polygon 顶点通常来自简化轮廓，可能跨过起伏地形。例如一个斜坡或弯曲地面在粗 mesh 中可能只是一个四边形。detail mesh 会在这个四边形内部补充高度三角形，让 `dtNavMeshQuery::getPolyHeight` 这类查询能返回更贴近原始高度场的 y 值。

这里要注意，detail mesh 不增加路径搜索节点。Detour 的 A* 仍然在粗 polygon 上执行；detail mesh 只参与高度和表面贴合查询。这就是它性能友好的原因。

## Step 8: 创建 Detour navmesh

Recast 到这里已经生成了可用的构建结果，但运行时寻路使用的是 Detour。因此还需要把 `rcPolyMesh` 和 `rcPolyMeshDetail` 打包成 Detour tile data。

### area 到 flags 的映射

```cpp
for (int i = 0; i < polyMesh->npolys; ++i)
{
    if (polyMesh->areas[i] == RC_WALKABLE_AREA)
        polyMesh->areas[i] = SAMPLE_POLYAREA_GROUND;

    if (ground/grass/road)
        polyMesh->flags[i] = SAMPLE_POLYFLAGS_WALK;
    else if (water)
        polyMesh->flags[i] = SAMPLE_POLYFLAGS_SWIM;
    else if (door)
        polyMesh->flags[i] = SAMPLE_POLYFLAGS_WALK | SAMPLE_POLYFLAGS_DOOR;
}
```

### area 与 flags 的职责区别

`area` 描述地表类型，`flags` 描述能力要求或通行能力。

例如：

- `SAMPLE_POLYAREA_GRASS`: 这里是草地。
- `SAMPLE_POLYFLAGS_WALK`: 可以走。
- `SAMPLE_POLYFLAGS_SWIM`: 需要游泳能力。
- `SAMPLE_POLYFLAGS_DOOR`: 需要门相关逻辑。

Detour 查询时可以用 filter 根据 flags 决定哪些 polygon 可通行，也可以根据 area 设置不同移动代价。

### 算法细节：area、flags 和 query filter 的关系

Detour 查询不是简单地“所有 polygon 都能走”。查询时通常会使用 `dtQueryFilter`：

- `includeFlags` 指定必须包含哪些能力位。
- `excludeFlags` 指定排除哪些能力位。
- 每个 area 可以有不同 cost。

例如普通地面、草地、道路都可以设置 `SAMPLE_POLYFLAGS_WALK`，这样普通地面 agent 都能通行。但 grass 的 area cost 可以更高，道路的 area cost 可以更低。结果是路径不一定完全避开草地，但会在有选择时偏向道路。

flags 更适合表达硬约束，比如“不会游泳就不能进水”。area 更适合表达软约束，比如“草地可以走但代价高”。`Sample_SoloMesh` 在打包前做这层映射，就是把构建期的地表分类转成运行时查询语义。

### dtNavMeshCreateParams

```cpp
dtNavMeshCreateParams params;
params.verts = polyMesh->verts;
params.polys = polyMesh->polys;
params.detailMeshes = detailMesh->meshes;
params.detailVerts = detailMesh->verts;
params.detailTris = detailMesh->tris;
params.offMeshConVerts = inputGeometry->offmeshConnVerts.data();
params.walkableHeight = agentHeight;
params.walkableRadius = agentRadius;
params.walkableClimb = agentMaxClimb;
params.buildBvTree = true;
```

`dtNavMeshCreateParams` 是 Recast 和 Detour 之间的桥。它包含：

- 粗 polygon mesh。
- detail mesh。
- off-mesh connections。
- agent 参数。
- tile bounds。
- cell size 和 cell height。
- 是否构建 BVTree。

### 算法细节：dtCreateNavMeshData 打包了什么

`dtCreateNavMeshData` 会把 Recast 生成的数组转换成 Detour 的 tile 二进制布局。这个布局不是简单拷贝，而是生成运行时查询需要的一整套结构：

- `dtMeshHeader`: tile 元数据，例如 bounds、poly 数量、顶点数量、BVTree 数量。
- 顶点数组: 转成 Detour 使用的世界坐标或 tile 内坐标。
- `dtPoly`: 每个 polygon 的顶点索引、邻接边、area/type、flags。
- `dtPolyDetail`: 每个 polygon 对应 detail mesh 的范围。
- detail vertices 和 detail triangles。
- `dtBVNode`: 可选的包围盒树节点。
- `dtOffMeshConnection`: 自定义连接。

Detour 运行时追求快速查询，所以打包阶段会提前整理很多信息。例如 polygon 邻接、边界链接、BVTree 都是在这里固化的。这样游戏运行时加载 navmesh 后，不需要重新执行 Recast 的构建算法。

### Off-Mesh Connections

off-mesh connection 是自定义图边，连接两个空间点，例如跳跃、爬梯、传送门。

它们不是从普通 walkable surface 自动生成的，而是用户或工具显式添加。打包进 Detour 后，它们会成为路径搜索图的一部分。

算法上，每条 off-mesh connection 会尝试把起点和终点附着到附近的 navmesh polygon。附着成功后，它在 Detour 图中表现为一条特殊边。路径搜索可以经过这条边，但移动系统需要根据 off-mesh connection 的语义播放跳跃、攀爬或传送逻辑。

`offMeshConDir` 控制方向：单向连接只能从 A 到 B，双向连接允许来回。`offMeshConRad` 是端点吸附半径，用来决定连接端点附近多大范围内的 polygon 可以关联。

### BVTree

```cpp
params.buildBvTree = true;
```

BVTree 是 polygon 的空间加速结构。它不改变 navmesh 拓扑，但能加速查找附近 polygon、raycast、点投影等查询。

算法上，BVTree 是一棵包围盒层次树。叶子节点对应 polygon 的 AABB，内部节点包住一组子节点。查询某个点或 AABB 附近的 polygon 时，可以快速排除大量不相交的 polygon，而不是线性扫描全部 polygon。

在单块 navmesh 中，BVTree 对 `findNearestPoly` 这类查询很有价值。工具点击场景测试路径时，通常需要先把点击点投影到最近 polygon；BVTree 能显著减少候选 polygon 数量。

### 创建和初始化 Detour 对象

```cpp
dtCreateNavMeshData(&params, &navData, &navDataSize);

navMesh = dtAllocNavMesh();
navMesh->init(navData, navDataSize, DT_TILE_FREE_DATA);
navQuery->init(navMesh, 2048);
```

`dtCreateNavMeshData` 生成一块二进制 tile data。`dtNavMesh::init` 接管这块数据。

`DT_TILE_FREE_DATA` 表示 `dtNavMesh` 在销毁或移除 tile 时负责释放 `navData`。这避免调用者还要额外追踪这块内存。

`navQuery->init(navMesh, 2048)` 初始化查询对象。`2048` 是查询节点池大小，影响复杂路径查询能展开多少节点。

### 算法细节：Detour 查询如何使用这些数据

构建完成后，Detour 的典型路径查询会经历：

1. `findNearestPoly`: 将起点和终点吸附到 navmesh polygon。
2. `findPath`: 在 polygon 邻接图上执行 A*，得到 polygon corridor。
3. `findStraightPath`: 在 corridor 上执行 funnel/string-pulling，得到更自然的拐点路径。

`Sample_SoloMesh` 的构建结果必须满足这些查询的需求：

- polygon 邻接关系要正确，否则 A* 不能连通。
- polygon flags 和 area 要正确，否则 filter 会错误允许或拒绝路径。
- detail mesh 要能提供高度，否则路径点可能不能贴地。
- BVTree 要能加速空间查找，否则交互工具会变慢。

## 构建完成后的工具初始化

```cpp
if (tool)
{
    tool->init(this);
}
initToolStates(this);
```

构建完成后工具需要重新绑定新的 `navMesh`、`navQuery` 和 sample 状态。

这很重要，因为用户可能在同一个 mesh 上反复调参构建。每次构建都会释放旧 navmesh 并生成新对象，工具如果继续持有旧状态就会访问无效数据。

## 渲染和调试视图

`render()` 根据 `currentDrawMode` 绘制不同阶段的数据。

### 输入 mesh

```cpp
duDebugDrawTriMeshSlope(...)
```

按坡度绘制原始三角形，用来观察哪些三角形会被认为可走。

### navmesh

```cpp
duDebugDrawNavMeshWithClosedList(...)
duDebugDrawNavMeshBVTree(...)
duDebugDrawNavMeshNodes(...)
```

这些视图用于检查 Detour 数据：

- navmesh polygon 是否符合预期。
- BVTree 空间结构是否存在。
- navQuery closed list 如何展开路径搜索。

### Recast 中间结果

| DrawMode | 数据结构 | 用途 |
|---|---|---|
| `VOXELS` | `rcHeightfield` | 查看原始体素化结果 |
| `VOXELS_WALKABLE` | `rcHeightfield` | 查看可走 span 标记 |
| `COMPACT` | `rcCompactHeightfield` | 查看紧凑可走空间 |
| `COMPACT_DISTANCE` | `rcCompactHeightfield::dist` | 查看距离场 |
| `COMPACT_REGIONS` | `rcCompactHeightfield::spans.reg` | 查看区域分区结果 |
| `RAW_CONTOURS` | `rcContourSet::rverts` | 查看未简化轮廓 |
| `CONTOURS` | `rcContourSet::verts` | 查看简化轮廓 |
| `POLYMESH` | `rcPolyMesh` | 查看粗导航多边形 |
| `POLYMESH_DETAIL` | `rcPolyMeshDetail` | 查看高度细节网格 |

### 为什么保留所有中间结构

生产环境通常不需要保留所有中间结构。但 demo 保留它们是为了调试和教学：

- 构建失败时可以定位阶段。
- 参数变化的效果可以立即可视化。
- 学习者能把每个 Recast API 调用和对应输出联系起来。

## onMeshChanged 的作用

```cpp
Sample::onMeshChanged(geom);
dtFreeNavMesh(navMesh);
navMesh = nullptr;
tool->reset();
tool->init(this);
resetToolStates();
initToolStates(this);
```

当用户切换输入 mesh 时，旧 navmesh 不再可信，必须释放。工具和工具状态也必须重置，因为它们可能保存了旧 mesh 上的点击点、连接、路径、crowd agent 等状态。

这里调用基类 `Sample::onMeshChanged` 还有一个作用：如果输入 `.gset` 自带构建参数，基类会把这些参数恢复到 UI 设置中。

## Save 和 Load

`drawSettingsUI()` 中的 Save/Load 使用基类的 `saveAll` 和 `loadAll`。

保存的是 Detour navmesh tile data，而不是 Recast 的中间结构。也就是说：

- 保存结果适合运行时加载和查询。
- 保存后不能恢复 voxels、contours、poly mesh detail 调试阶段。
- 如果要重新可视化完整构建过程，需要重新执行 `build()`。

这符合常见工作流：编辑器或构建工具生成 navmesh，游戏运行时只加载 Detour 数据。

## 单块 navmesh 的设计边界

`Sample_SoloMesh` 的优点：

- 流程简单，适合理解 Recast。
- 参数少，调试直观。
- 单次构建完整，状态管理容易。

局限：

- 场景越大，`config.width * config.height` 越大，内存和构建时间会快速上升。
- 不能局部重建。
- 不适合流式开放世界。
- 动态障碍支持有限。

更大规模或更动态的场景应参考 `Sample_TileMesh.cpp`。tiled navmesh 把世界拆成 tile，可以按 tile 构建、加载、卸载和重建。

## 从根源排查构建问题

当最终 navmesh 看起来不对时，不要先看 Detour。Detour 只查询已经构建好的结果，绝大多数问题发生在 Recast 的中间阶段。可以按下面顺序定位。

### 1. 没有生成任何 navmesh

优先检查：

1. `inputGeometry` 是否为空。
2. `config.width/height` 是否合理。
3. `triAreas` 中是否有三角形被标为 walkable。
4. `rcRasterizeTriangles()` 后 heightfield 是否有 spans。

如果体素化阶段就没有 spans，后面所有阶段都会空。

### 2. 地面存在但被判不可走

看过滤阶段：

- `walkableSlopeAngle` 是否太小，导致斜面被剔除。
- `walkableHeight` 是否太大，导致净空不足。
- `walkableClimb` 是否太小，导致台阶被当作 ledge。
- 输入 mesh 的法线或三角 winding 是否异常。

Recast 的可走性不是只看“有没有地面”，而是看 agent 是否能站、能过、不会从边缘掉下去。

### 3. 门洞或走廊消失

优先看 `walkableRadius` 和 `cellSize`。

半径腐蚀会从障碍边界向内收缩：

```text
腐蚀格数 = ceil(agentRadius / cellSize)
```

如果走廊宽度小于 `2 * agentRadius`，或者离散后只剩一两格，就会在腐蚀时消失。这不是构建错误，而是 agent 物理尺寸无法通过。

### 4. navmesh 边缘锯齿太多或过度简化

看 contour 参数：

- `maxSimplificationError` 太小，轮廓会贴着体素楼梯走，顶点很多。
- `maxSimplificationError` 太大，边界会过度拉直，窄区域可能被吞掉。
- `maxEdgeLen` 会限制长边，有助于 tiled navmesh 边界匹配，也会增加顶点。

轮廓简化是在“形状保真”和“polygon 数量”之间取平衡。

### 5. A* 路径能走但高度不贴地

看 detail mesh：

- `detailSampleDist` 为 0 时细节较少。
- `detailSampleMaxError` 太大时高度贴合较粗。
- 输入地形起伏很大但 coarse polygon 很大时，更依赖 detail mesh。

Detour 的路径拓扑来自 `rcPolyMesh`，但高度查询依赖 `rcPolyMeshDetail`。

### 6. Detour 查询失败

如果 Recast 中间结果都正确，再看 Detour 打包：

1. `polyFlags` 是否非 0。
2. `dtQueryFilter` 的 include/exclude flags 是否允许这些 polygon。
3. `dtCreateNavMeshData()` 是否成功。
4. `dtNavMesh::init()` 和 `addTile()` 是否成功。
5. 起点终点是否先通过 `findNearestPoly()` 吸附到 navmesh。

很多“路径找不到”的问题其实是 flags/filter 不匹配，而不是 navmesh 没连通。

## 源码阅读建议

阅读 `Sample_SoloMesh.cpp` 时，可以把每段代码放进三个问题里：

1. **这一步的输入数据结构是什么？**

   例如 `rcBuildContours()` 的输入是 `rcCompactHeightfield`，不是原始三角形。

2. **这一步新增了什么语义？**

   例如 `rcErodeWalkableArea()` 新增的是 agent 半径安全约束；`rcBuildRegions()` 新增的是连通区域 id。

3. **这一步的输出被谁消费？**

   例如 `rcContourSet` 只服务于 `rcBuildPolyMesh()`；`rcPolyMesh` 和 `rcPolyMeshDetail` 一起服务于 `dtCreateNavMeshData()`。

按这三个问题读，源码就不再是一串 API 调用，而是一条数据语义逐步收敛的流水线。

## 一句话总结

`Sample_SoloMesh.cpp` 展示的是 Recast 的最小完整构建闭环：把原始三角网格转换为体素高度场，在体素空间中得到安全、连通、按 agent 尺寸收缩后的可走区域，再把这些区域简化成适合 Detour 查询的凸多边形导航图。
