# DetourTileCache 实现逻辑详解

`DetourTileCache` 用于动态障碍物场景。它不重新体素化完整场景，而是保存一份压缩的 tile layer。当障碍物添加或移除时，只解压受影响的 tile layer，在 layer 上把障碍区域标成不可走，然后重新生成该 tile 的 Detour navmesh data，并替换 `dtNavMesh` 中对应 tile。

相关源码入口：

- `Include/DetourTileCache.h`：tile cache、障碍物、引用、更新状态机。
- `Include/DetourTileCacheBuilder.h`：压缩 layer、区域、轮廓、多边形网格构建 API。
- `Source/DetourTileCache.cpp`：tile 管理、障碍物请求、tile 重建、与 `dtNavMesh` 交互。
- `Source/DetourTileCacheBuilder.cpp`：layer 解压、区域划分、轮廓提取、三角化、polygon 合并、障碍物栅格标记。

校验依据：

- 本文按仓库提交 `9f4ce64` 的 `DetourTileCache/Include` 和 `DetourTileCache/Source` 核对，重点检查 layer 格式、tile/obstacle 引用、请求和更新队列、三类障碍物标记、regions/contours/poly mesh 重建、Detour tile 替换和各固定容量的截断语义。
- 官方集成文档把 `DetourTileCache` 定位为 runtime navmesh dynamic obstacle and re-baking system；它不是查询系统，最终查询仍发生在 `dtNavMeshQuery` 上。
- 官方 `dtTileCache` API 文档确认 `update()` 的语义是重建尚未完成的 obstacle request 所触碰的 tile，并通过 `upToDate` 告知是否追平。
- 官方 Google Group 讨论中，Mikko Mononen 说明 TileCache 假设小 tile，典型约 32 到 64 cell square，并为快速更新做了取舍；不能期望 TileCache 生成的 mesh 与 SoloMesh 完全一致。
- 官方论坛还明确区分了两类动态变化：箱子、桶这类“从既有可走层扣除空间”的半静态障碍适合 TileCache；房屋、多层结构或会新增可走表面的复杂几何，应收集触碰 tile 的完整静态与动态三角形后走常规 Recast tile 重建。

参考链接：

- [`dtTileCache` 官方 API](https://recastnav.com/classdtTileCache.html)
- [官方集成文档](https://recastnav.com/md_Docs_2__2__BuildingAndIntegrating.html)
- [官方论坛：TileCache 假设小 tile，结果不等价于 SoloMesh](https://groups.google.com/g/recastnavigation/c/oB5VFwJoB0o)
- [官方论坛：简单障碍用 TileCache，复杂/多层几何用常规 Recast 重建](https://groups.google.com/g/recastnavigation/c/F9mRO_eimEI)
- [官方论坛：TileCache 导出与 off-mesh connection 容易遗漏](https://groups.google.com/g/recastnavigation/c/QBH_yGOnwvs)

源码覆盖关系：

| 源码 | 本文对应内容 |
| --- | --- |
| `DetourTileCache.cpp` | cache 初始化、tile hash/free list、obstacle 请求、update 状态机、受影响 tile 查询、重建与替换 |
| `DetourTileCacheBuilder.cpp` | layer 压缩/解压、monotone regions、轮廓、ear clipping、凸合并、顶点删除、邻接与三类 area marking |
| `DetourTileCache.h` | 两套 salt 引用、固定请求/更新/touched 容量、allocator/compressor/mesh-process 扩展点 |
| `DetourTileCacheBuilder.h` | layer 二进制合同、临时构建数据结构和固定六边形上限 |

## 一、整体设计

TileCache 的数据流：

![TileCache 从压缩高度层到替换 Detour navmesh tile 的运行时模型](https://oss.euler.icu/teaser/recastnavigation/DetourTileCache/Docs/Images/tilecache-runtime-model.png)

> **图 1：TileCache 的持久源数据与重建闭环。** 构建期 `rcHeightfieldLayer` 被压缩为 header 与 `heights/areas/cons`，由 `dtTileCache` 长期保存；障碍变化时只解压受影响 layer、在 area 网格上重放全部 active obstacles，再执行 region、contour、poly mesh 和 `dtCreateNavMeshData()`。最右侧运行时 tile 由 `dtNavMesh` 查询，但不是下一次重建的输入，更新时会被新 tile 整体替换。

关键点是：动态变化发生在“压缩高度层”的 area 标记上，而不是重新跑完整 Recast pipeline。这样适合门、箱子、临时圆柱障碍等局部变化。

因此 TileCache 的目标不是“在运行时重现完整 Recast SoloMesh 构建结果”。SoloMesh 可以在全局范围内使用完整 Recast pipeline 和更自由的区域划分；TileCache 为了动态更新，把输入固定成小 tile 的 heightfield layer，并使用适合快速重建的 monotone-like region、contour 和 poly mesh 流程。官方讨论明确指出两者结果不可能保证相同。

## 根源视角：TileCache 保存的是“可重新建模的可走层”

普通 `dtNavMesh` 保存的是最终 polygon graph。它适合查询，但不适合局部修改，因为 polygon、link、BV tree、off-mesh 等数据互相依赖。TileCache 选择保存更早一层的数据：heightfield layer 的紧凑网格。

![TileCache 的压缩源、临时 layer、临时 mesh 和 Detour tile 四层模型](https://oss.euler.icu/teaser/recastnavigation/DetourTileCache/Docs/Images/tilecache-four-layer-model.png)

> **图 2：四层数据模型及其生命周期。** `Compressed Source` 是持久的 header 与三个 byte grid；`Temporary Layer` 是一次重建中解压并被 obstacle area marking 改写的 `dtTileCacheLayer`；`Tile Mesh` 是由 region/contour 推导出的临时凸多边形；`Detour Tile` 是加入 `dtNavMesh`、支持 link 与查询的运行时结果。图中回箭头表示变化后从压缩源重新推导并替换 Detour tile，而不是对旧 polygon graph 做原地布尔修改。

这就是 TileCache 的核心设计：把动态变化放在“areas 网格”上表达，再用固定 pipeline 重新推导出 polygon mesh。

### 四个层次

| 层次 | 数据 | 作用 | 生命周期 |
| --- | --- | --- | --- |
| 压缩源数据 | header + compressed heights/areas/cons | 保存可重建输入 | 长期存在于 `dtTileCache` |
| 临时 layer | `dtTileCacheLayer` | 解压后被 obstacle 改写 | 每次重建临时分配 |
| 临时 mesh | contours、poly mesh | 从 layer 推导 navmesh 输入 | 每次重建临时分配 |
| Detour tile | `dtCreateNavMeshData()` 输出 | 运行时查询 | 加入 `dtNavMesh`，直到下一次替换 |

### 关键不变量

1. **compressed tile 是重建源，不直接查询。**

   `dtNavMeshQuery` 完全不知道 TileCache 的存在。TileCache 修改后必须重新生成 Detour tile，并替换到 `dtNavMesh`，查询结果才会变化。

2. **障碍物不直接修改压缩源数据。**

   添加障碍物时不会把 compressed data 改坏，而是在每次重建时先解压原始 layer，再把所有 active obstacles 栅格化进去。这样移除障碍物不需要“恢复原 area”，只要下一次从干净源数据重建即可。

3. **`touched[]` 记录影响范围，`pending[]` 记录处理进度。**

   touched 是 obstacle 的长期影响集合；pending 是当前 add/remove 请求还没完成重建的 tile 集合。两者分开，才能在逐 tile update 时正确判断 obstacle 何时从 processing 变成 processed，或从 removing 变回 empty。

4. **navmesh 以完整 tile 为更新单位，但 remove/add 不是事务。**

   `buildNavMeshTile()` 会先生成完整新 tile data，再 remove 旧 tile，再 add 新 tile，不会让查询看到“只改了一半数组”的 tile。Detour 的 salt 和 link 重建机制会让旧引用自然失效。但源码没有 rollback：如果旧 tile 已删除而新 tile `addTile()` 失败，该坐标会暂时没有 navmesh。应用必须检查 `update()`/`buildNavMeshTile()` 的 `dtStatus`，不能把“完整 tile 替换”误解成数据库式原子提交。

5. **动态障碍物只改变可走性，不改变原始高度。**

   `dtMarkCylinderArea()` 等函数只是写 `areas[idx]`。它不会新增几何高度，也不会改变 `heights` 或 `cons`。所以 TileCache 适合“占据空间导致不可走”的障碍，不适合运行时改变地形高度或打开新的几何连接。

6. **tile 尺寸是算法假设的一部分。**

   `DetourTileCacheBuilder.cpp` 里的区域、轮廓、三角化和顶点删除都围绕小 layer 工作。tile 太大会放大固定数组、region id、contour 顶点、重建耗时和单障碍 touched tile 管理问题。官方社区建议的 32 到 64 cell square 不是硬编码限制，但反映了这个模块的真实设计目标。

## 二、核心数据结构

### 1. `dtCompressedTile`

`dtCompressedTile` 是 tile cache 保存的原始动态重建数据。

| 字段 | 说明 |
| --- | --- |
| `salt` | tile 引用防悬空计数。 |
| `header` | `dtTileCacheLayerHeader`，记录 tile 坐标、层号、尺寸、高度范围、紧边界。 |
| `compressed` | 压缩后的 grids 数据起始地址。 |
| `compressedSize` | 压缩数据大小。 |
| `data/dataSize` | 完整 tile cache data，包含 header 和 compressed bytes。 |
| `flags` | 是否由 tile cache 负责释放 data。 |
| `next` | 哈希链或 free list 指针。 |

和 `dtNavMesh` 类似，TileCache 也用固定数组加 free list 管理 tile。

### 2. `dtTileCacheLayerHeader`

header 描述一个压缩 layer：

| 字段 | 说明 |
| --- | --- |
| `tx/ty/tlayer` | tile 平面坐标和层号。一个 `(tx, ty)` 下可以有多个高度层。 |
| `bmin/bmax` | tile world bounds。 |
| `hmin/hmax` | layer 高度范围。 |
| `width/height` | 网格尺寸，通常对应 tile 的 compact heightfield layer 尺寸。 |
| `minx/maxx/miny/maxy` | 实际有用区域的紧边界，用于 queryTiles 快速排除。 |

### 3. `dtTileCacheParams`

`dtTileCacheParams` 是压缩 layer 与最终 Detour tile 共享的空间合同：

| 字段 | 进入哪一步 | 根本含义 |
| --- | --- | --- |
| `orig/cs/ch/width/height` | tile 坐标换算、障碍物栅格化、顶点回写 | 必须与生成 layer 时的体素网格一致，不能在运行时随意改 |
| `walkableHeight/walkableRadius/walkableClimb` | 填入 `dtNavMeshCreateParams`；climb 还参与 layer region/contour 连通 | 它们描述这份 cache 对应的 agent 能力；原始 layer 通常已经按该半径腐蚀 |
| `maxSimplificationError` | `dtBuildTileCacheContours()` | 动态重建时轮廓允许偏离 raw cell 边界的世界单位误差 |
| `maxTiles/maxObstacles` | 固定池和引用 bit 分配 | 初始化后决定容量和 salt/index 编码，不能无代价扩容 |

TileCache poly mesh 的 `nvp` 不是参数化的：builder 固定最多 6 个顶点，与 `DT_VERTS_PER_POLYGON` 对齐。这是它能直接交给 `dtCreateNavMeshData()` 的合同之一。

### 4. `dtTileCacheLayer`

解压后得到 `dtTileCacheLayer`：

| 字段 | 说明 |
| --- | --- |
| `heights` | 每个 cell 的高度，byte 数组。 |
| `areas` | 每个 cell 的 area id。0 表示不可走。 |
| `cons` | 每个 cell 的连接和 portal 信息，低 4 bit 表示四方向连接，高 4 bit 表示 tile 边 portal。 |
| `regs` | 构建时临时写入的 region id。 |
| `regCount` | 区域数量。 |

TileCache 的构建算法几乎都围绕这四个网格数组展开。

### 5. `dtTileCacheObstacle`

动态障碍物支持三种形状：

- cylinder
- AABB box
- Y 轴旋转的 oriented box

每个 obstacle 还保存：

| 字段 | 说明 |
| --- | --- |
| `touched[]` | obstacle 影响到哪些 compressed tile。 |
| `pending[]` | 当前还有哪些 touched tile 尚未重建完成。 |
| `salt` | obstacle ref 防悬空。 |
| `type` | cylinder、box、oriented box。 |
| `state` | empty、processing、processed、removing。 |
| `next` | free list。 |

`DT_MAX_TOUCHED_TILES` 默认是 8。这是一个重要限制：单个障碍物影响的 tile 不能太多，否则超出的 tile 不会进入该 obstacle 的 touched/pending 数组。

![TileCache 障碍从空闲槽到添加、稳定、移除并回收的状态机](https://oss.euler.icu/teaser/recastnavigation/DetourTileCache/Docs/Images/tilecache-obstacle-lifecycle.png)

> **图 3：`dtTileCacheObstacle` 生命周期。** `addObstacle()`只把空闲槽置为 `PROCESSING` 并写入请求；`update()`计算 `touched[]`，以 `pending[]`追踪尚未重建的 tile，清空后进入 `PROCESSED`。在 processing 或 processed 状态请求删除都会进入 `REMOVING`，旧 `touched[]`重新成为 pending；全部完成后递增 `salt`并回到 `EMPTY`，使旧 `dtObstacleRef = [salt | index]`失效。四个等轴 tile 分别表示空闲槽、等待重建、稳定挖洞和等待恢复；状态机箭头表达请求与完成条件，tile 图只是对应状态的数据快照。

同时源码还有两个和帧预算相关的固定上限：

- `MAX_REQUESTS = 64`：一次积压的 add/remove obstacle 请求数量。
- `MAX_UPDATE = 64`：全局待重建 tile 列表容量。

这三个上限的失败语义并不相同：请求数达到 64 时 add/remove API 明确返回 `DT_BUFFER_TOO_SMALL`；`queryTiles()` 超过 `DT_MAX_TOUCHED_TILES` 时只写前 8 个且仍返回成功；全局 `m_update` 达到 64 后，后续 touched tile 也会被跳过而没有额外错误码。后两种属于静默截断，甚至可能让 obstacle 的 `pending[]` 提前清空并进入 processed 状态。因此大面积动态变化不能只“检查 API 是否成功”，还必须从设计上限制 obstacle 与 tile 的尺寸关系，或改走常规 Recast tile 重建。

### 6. 引用编码

TileCache 有两种引用：

```text
dtCompressedTileRef = [ salt | tile index ]
dtObstacleRef       = [ salt | obstacle index ]
```

tile ref 的 salt bits 由 `maxTiles` 决定，obstacle ref 固定高 16 bit 为 salt、低 16 bit 为 index。

删除 tile 或 obstacle 时 salt 递增，避免旧引用误命中新对象。

### 7. allocator、compressor 和 mesh process

TileCache 把三个工程相关能力做成接口：

| 接口 | 作用 | 设计原因 |
| --- | --- | --- |
| `dtTileCacheAlloc` | 临时分配 layer、contour、poly mesh 等构建数据 | 调用者可以接入帧分配器或 scratch allocator，`buildNavMeshTile()` 每次会先 `reset()`。 |
| `dtTileCacheCompressor` | 压缩/解压 `[heights|areas|cons]` 三个 byte grid | 库不绑定 FastLZ/LZ4 等具体压缩实现。 |
| `dtTileCacheMeshProcess` | 在生成 Detour tile data 前修改 `polyAreas/polyFlags` | 应用层可把 area 映射成 Detour flags、成本或业务语义。 |

压缩 layer 本身不保存 off-mesh connections。`process()` 接收可修改的 `dtNavMeshCreateParams*`，所以应用应在每次 tile 重建时按 tile 范围重新附加 `offMeshConVerts/Rad/Dir/Areas/Flags/UserID/Count`。RecastDemo 的 `Sample_TempObstacles` 就在 `MeshProcess::process()` 中这样做。若只序列化 cache layer 而没有保存并重新提供这些业务连接，重建出的 Detour tile 会缺少 off-mesh connection；这正是官方论坛中常见的导出误区。

`NavMeshTileBuildContext` 是源码中的 RAII 临时上下文，持有解压 layer、contour set 和 poly mesh。`buildNavMeshTile()` 任一步失败或返回时，它都会释放这些临时对象，避免中途失败泄漏 scratch 内存。

## 三、压缩 layer：`dtBuildTileCacheLayer`

TileCache 的输入不是完整 `dtNavMesh` 数据，而是更早阶段的 compact layer 数据。

`dtBuildTileCacheLayer()` 做的事：

1. 计算 header aligned size。
2. 计算 grid size：

   ```text
   gridSize = width * height
   ```

3. 创建临时 buffer，把三个 byte 网格顺序拼接：

   ```text
   [ heights | areas | cons ]
   ```

4. 调用户提供的 `dtTileCacheCompressor::compress()` 压缩。
5. 输出：

   ```text
   [ dtTileCacheLayerHeader | compressed grids ]
   ```

`dtTileCacheCompressor` 是虚接口，工程可接入 FastLZ、LZ4 或自定义压缩。DetourTileCache 不绑定具体压缩库。

对应的解压入口是 `dtDecompressTileCacheLayer()`。它会分配一整块临时内存：

```text
[ dtTileCacheLayer | dtTileCacheLayerHeader | heights | areas | cons | regs ]
```

其中 `heights/areas/cons` 来自压缩数据，`regs` 是本次构建的临时输出，不会写回 compressed tile。这个布局让构建阶段只有一次临时分配，并且所有网格数组连续存放。

`dtTileCacheHeaderSwapEndian()` 只交换 `dtTileCacheLayerHeader`。layer grid 全是 byte 数组，不需要端序转换；压缩 payload 的字节顺序由 compressor 自己定义。

### 为什么只压缩 heights/areas/cons

动态重建需要的信息主要是：

- 每个 cell 的高度。
- 每个 cell 当前是否可走以及 area 类型。
- cell 间连接和 tile 边 portal。

有了这些，就可以重新划分区域、提取轮廓、生成 polygon mesh。它比保存完整体素场小，也比保存最终 navmesh 更适合动态障碍重建。

## 四、tile cache 管理

### 1. `dtTileCache::init`

初始化做以下事情：

1. 保存参数、allocator、compressor、mesh process。
2. 分配 obstacle 池，并把所有 obstacle 放入 free list。
3. 分配 compressed tile 池和 tile 位置哈希表。
4. 把所有 tile 放入 free list。
5. 根据 `maxTiles` 计算 tile ref 的 `tileBits` 和 `saltBits`。
6. 要求 salt 至少 10 bit，否则返回 invalid param。

### 2. `addTile`

添加 compressed tile：

1. 检查 layer header magic/version。
2. 检查 `(tx, ty, tlayer)` 是否已存在。
3. 从 tile free list 取槽位。
4. 插入 `(tx, ty)` 哈希链。
5. 记录 `header/data/compressed` 指针和 flags。
6. 返回 `dtCompressedTileRef`。

这里并不会立即生成 navmesh tile。通常之后调用 `buildNavMeshTilesAt()` 或让 update 流程重建。

`addTile()` 保存的 `header`、`compressed` 指针都指向 `data` 这块连续内存内部：

```text
data -> dtTileCacheLayerHeader
     -> compressed bytes
```

如果传入 `DT_COMPRESSEDTILE_FREE_DATA`，TileCache 析构或 remove tile 时会负责释放这块 data；否则删除 tile 时可把 data/dataSize 还给调用者。

### 3. `removeTile`

删除 compressed tile：

1. 解码 ref 并检查 salt。
2. 从位置哈希链摘除。
3. 根据 flags 决定释放 data 还是把 data 还给调用者。
4. 清空 tile 字段。
5. tile salt 递增。
6. 放回 tile free list。

`getTilesAt(tx, ty, ...)` 会返回同一平面 tile 坐标下的所有 layer；`getTileAt(tx, ty, tlayer)` 才要求 layer 精确匹配。这个设计对应 Recast layer set：同一个 xz tile 可能有多层互不连通的可走高度平台。

`getTileRef()` / `getTileByRef()` 和 `getObstacleRef()` / `getObstacleByRef()` 都执行指针或 salt/index 之间的转换；ref 版本应作为跨帧身份，裸指针只在池未重建且对象仍有效时使用。`calcTightTileBounds()` 使用 header 的 `minx/maxx/miny/maxy` 得到有效 cell 的世界 AABB，`getObstacleBounds()` 则为三种 obstacle 生成候选 AABB，它们共同服务于 broad phase。

compressed cache 与 `dtNavMesh` 是两个独立容器：`addTile()` 不会自动生成 Detour tile，`removeTile()` 也不会自动删除对应 Detour tile。初次加载通常在全部 compressed layers 加入后调用 `buildNavMeshTilesAt()`；流式卸载则必须由应用同时协调 cache tile、Detour tile 和仍引用它的 obstacle 请求，不能只删其中一边。

TileCache 本体和 builder 临时对象同样使用成对 API：`dtAllocTileCache()/dtFreeTileCache()`、`dtAllocTileCacheContourSet()/dtFreeTileCacheContourSet()`、`dtAllocTileCachePolyMesh()/dtFreeTileCachePolyMesh()`。`dtFreeTileCacheLayer()` 通过创建它时的 `dtTileCacheAlloc` 释放，不能改用普通 `free()`。

## 五、障碍物请求和状态机

### 1. 添加障碍物

`addObstacle()`、`addBoxObstacle()` 都只做请求登记：

1. 检查请求队列是否有空间。
2. 从 obstacle free list 取一个槽位。
3. 写入形状参数。
4. state 设为 `DT_OBSTACLE_PROCESSING`。
5. 在 `m_reqs` 追加 `REQUEST_ADD`。
6. 返回 obstacle ref。

它不会马上改 navmesh。真正处理发生在下一次 `update()`。

### 2. 移除障碍物

`removeObstacle(ref)` 也只是追加 `REQUEST_REMOVE`。这样添加和移除都被串行化到 `update()`，避免调用者在任意时刻直接修改正在重建的数据结构。

返回成功只表示 remove 请求成功入队，不表示 ref 当前一定有效：`ref == 0` 被当作成功的 no-op；过期 salt 或越界 index 会在 `update()` 消费请求时被忽略。需要业务层确认对象是否仍存在时，应先用 `getObstacleByRef()` 检查，而不能把 `removeObstacle()` 的返回值当作存在性证明。

### 3. `update()` 的请求处理

当 `m_nupdate == 0` 时，`update()` 会先处理所有请求。

添加请求：

1. 解码 obstacle ref，验证 salt。
2. 调 `getObstacleBounds()` 得到 obstacle AABB。
3. `queryTiles()` 找和 AABB 重叠的 compressed tiles。
4. 写入 obstacle 的 `touched[]`。
5. 把这些 tile 去重后加入全局 `m_update[]`。
6. 同时写入 obstacle 的 `pending[]`。

如果单个障碍物触碰的 tile 超过 `DT_MAX_TOUCHED_TILES`，多余 tile 不会写入 obstacle 的 touched/pending 数组。源码不会动态扩容；这是调用者需要控制障碍物尺寸或 tile 尺寸的原因。

移除请求：

1. obstacle state 设为 `DT_OBSTACLE_REMOVING`。
2. 把它之前记录的 `touched[]` tile 加入 `m_update[]`。
3. 写入 `pending[]`。

![dtTileCache update 将障碍请求汇入去重重建队列并逐帧处理](https://oss.euler.icu/teaser/recastnavigation/DetourTileCache/Docs/Images/tilecache-update-queue.png)

> **图 4：`dtTileCache::update()` 的请求队列与重建工作集。** 只有 `m_nupdate == 0` 时才批量消费 `m_reqs`：ADD 通过 `queryTiles()`生成 obstacle 的 `touched[]/pending[]`，REMOVE 把旧 `touched[]`复制为 pending；两者都把 tile ref 去重写入全局 `m_update[]`。图中的 `m_update` 卡片只代表 compressed tile ref，不携带 ADD/REMOVE 类型。每次调用只重建 `m_update[0]`，随后从所有 processing/removing obstacle 的 `pending[]`删除该 ref，并在 pending 清空时推进状态；只有 `m_reqs`和 `m_update`同时为空时 `upToDate`才为 true。

### 4. 一次 update 只重建一个 tile

源码中 `update()` 每次只处理 `m_update[0]`：

1. 调 `buildNavMeshTile(ref, navmesh)`。
2. 从 `m_update` 移除该 ref。
3. 对所有 processing/removing obstacle，从 pending 中删除该 ref。
4. 如果某个 obstacle 的 pending 清空：
   - processing -> processed。
   - removing -> empty，salt 递增，回到 obstacle free list。

这个设计把动态重建成本摊到多帧，避免一帧内重建大量 tile。

`update(dt, navmesh, upToDate)` 的 `dt` 参数在当前源码中没有实际使用，保留它主要是为了接口形态和未来扩展。是否继续处理更多 tile 不是由 `dt` 自动决定，而是由调用者是否循环调用 `update()` 决定。需要“本帧强制追平”时，可以在外层循环直到 `upToDate == true`，但这等价于主动接受当前帧的重建成本。

`upToDate` 只表示请求队列和更新队列已经排空，不表示此前每个重建都成功。当前实现即使 `buildNavMeshTile()` 返回失败，也会把该 ref 从 `m_update` 弹出并推进 pending 状态。因此调用者必须逐次检查 `update()` 返回的 `dtStatus`；只循环到 `upToDate == true` 会遗漏失败。

## 六、查询受影响 tile：`queryTiles`

`queryTiles(bmin, bmax)` 的流程：

1. 根据 tile width/height 和 cache origin，把 AABB 转成 tile 坐标范围。
2. 遍历范围内的 `(tx, ty)`。
3. 用 `getTilesAt()` 找该坐标下所有 layer。
4. 对每个 compressed tile 计算 tight bounds：

   ```text
   tight bmin.x = header.bmin.x + minx * cs
   tight bmax.x = header.bmin.x + (maxx + 1) * cs
   tight bmin.z = header.bmin.z + miny * cs
   tight bmax.z = header.bmin.z + (maxy + 1) * cs
   ```

5. 与 obstacle AABB 做 bounds overlap。
6. 重叠则加入结果。

使用 tight bounds 可以避免障碍物碰到 tile 的空白边缘时误重建。

这里还有两层固定截断：`queryTiles()` 对每个 `(tx,ty)` 最多先取 32 个 layer，对最终结果只写 `maxResults` 个，而且无论是否截断都返回 `DT_SUCCESS`。`buildNavMeshTilesAt()` 同样最多构建该坐标下前 32 个 layer。正常地图通常远低于 32 层，但垂直堆叠极端场景不能依赖 status 自动报警；初始化/导入阶段应统计每个 tile 坐标的 layer 数量。

## 七、重建 navmesh tile：`buildNavMeshTile`

这是 TileCache 的核心。

![压缩高度层重放障碍并重建为新的 Detour navmesh tile](https://oss.euler.icu/teaser/recastnavigation/DetourTileCache/Docs/Images/tilecache-rebuild-pipeline.png)

> **图 5：`dtTileCache::buildNavMeshTile()` 的单 tile 重建。** 验证 compressed tile ref 后重置临时 allocator、解压 layer，并在原始 area 网格上重放所有触碰该 tile 且处于 processing/processed 的障碍；随后依次调用 `dtBuildTileCacheRegions()`、`dtBuildTileCacheContours()`和 `dtBuildTileCachePolyMesh()`。若 `npolys == 0`，只删除旧 Detour tile；否则先让 `dtTileCacheMeshProcess`映射 flags 和业务数据，再由 `dtCreateNavMeshData()`序列化，最后 remove 旧 tile、add 新 tile 并由 `dtNavMesh`重建运行时 links。

### 1. 解压 layer

`dtDecompressTileCacheLayer()`：

1. 检查 magic/version。
2. 分配一整块临时 buffer：

   ```text
   [ dtTileCacheLayer | dtTileCacheLayerHeader | heights | areas | cons | regs ]
   ```

3. 拷贝 header。
4. 调 compressor 解压 grids。
5. 设置 `layer->heights/areas/cons/regs` 指针。

`regs` 不在压缩数据中保存，它是本次构建临时区域 id 输出。

### 2. 栅格化当前 active obstacles

遍历 obstacle 池：

1. 跳过 empty 和 removing。
2. 检查该 obstacle 的 `touched[]` 是否包含当前 tile ref。
3. 根据 obstacle 类型调用 area marking：
   - `dtMarkCylinderArea`
   - `dtMarkBoxArea` AABB
   - `dtMarkBoxArea` oriented box
4. 把覆盖 cell 的 `areas[idx]` 改成 0。

这里的 0 就是 `DT_TILECACHE_NULL_AREA`，后续区域构建会把它当作不可走。

### 3. 构建 regions

`dtBuildTileCacheRegions()` 把可走 cell 分成单调区域。

算法分两阶段。

#### 阶段 A：逐行 sweep 分区

对每一行 y：

1. 从左到右扫描 cell。
2. 跳过 `areas == NULL_AREA` 的 cell。
3. 如果左侧 cell 与当前 cell 可连接，就继承左侧 sweep id。
4. 否则创建新的本行 sweep id。
5. 检查上一行同列 cell 是否可连接。
6. 如果连接到上一行 region，记录该 sweep 的候选邻居 `nei`，并统计连接次数。
7. 一行结束后，为每个 sweep 决定最终 region id：
   - 如果它只和上一行一个连续 region 连接，就沿用那个 region。
   - 否则创建新 region。
8. 把本行临时 sweep id 重映射为最终 region id。

`isConnected()` 的条件：

```text
area 相同
height 差 <= walkableClimb
```

#### 阶段 B：合并相邻单调区域

初始 sweep 结果可能把同一块区域切得太碎。接着：

1. 统计每个 region 的面积、area id、邻居 region。
2. 对每个 region，寻找相同 area id 的最大邻居。
3. 调 `canMerge()` 判断合并后是否仍保持简单连接。
4. 可以合并则把 old reg id 重映射到 new reg id。
5. 最后 compact region id，写回 `layer.regs`。

这个算法比 watershed 简单，适合 tile cache 的快速局部重建。代价是区域形状可能不如完整 Recast pipeline 精细。

### 4. 构建 contours

`dtBuildTileCacheContours()` 把每个 region 的 cell 边界变成简化轮廓。

#### `walkContour()`

对某个 region 的起始 cell：

1. 找第一条邻居 region 不同的边作为起始方向。
2. 沿边界行走。
3. 如果当前方向外侧不是同 region：
   - 这是边界边。
   - 添加一个顶点。
   - 顺时针转向。
4. 如果外侧仍是同 region：
   - 前进到邻接 cell。
   - 逆时针转向。
5. 回到起点和起始方向时结束。

顶点第 4 个 byte 暂存邻居 region 或 portal 信息。`appendVertex()` 会合并同一直线上的连续边，减少 raw contour 顶点。

#### `simplifyContour()`

轮廓简化类似 Douglas-Peucker，但有区域边界约束：

1. 先把 area/region 发生变化的点作为必须保留点。
2. 如果保留点少于 2 个，选择左下和右上两个点作为初始点。
3. 对每条简化边，遍历 raw contour 中对应段。
4. 找到离该线段最远的 raw 点。
5. 如果最大误差超过 `maxSimplificationError`，插入该点。
6. 重复直到所有 raw 点误差都在阈值内。
7. 重新排列顶点，使轮廓从较小索引开始，保证结果更稳定。

#### corner height 和 portal 标记

保存 contour 时，`getCornerHeight()` 会检查角点周围最多 4 个 cell：

- 选择可走且高度差在 climb 内的最大高度。
- 判断是否是 tile portal 边。
- 判断某些 portal 上的多余边界顶点是否可以后续移除。

最终 `cont.verts` 的第 4 个 byte：

- 低 4 bit：portal 方向，`0xf` 表示不是 portal。
- 高 bit `0x80`：该顶点后续可尝试删除。

### 5. 构建 poly mesh

`dtBuildTileCachePolyMesh()` 把 contours 变成 Detour 可用的凸多边形网格。

#### 容量估算

先遍历所有 contour：

- 最大顶点数是所有 contour 顶点总数。
- 最大三角数是 `nverts - 2` 的总和。
- 最大单 contour 顶点数用于临时 triangulation buffer。

#### 顶点去重

`addVertex()` 使用 hash bucket 按 `(x, z)` 查重，并允许 y 差在 2 以内视为同一顶点：

```text
same if x equal, z equal, abs(y0 - y1) <= 2
```

这样相邻 contour 共用边时能合并顶点，减少裂缝。

#### ear clipping 三角化

`triangulate()` 使用 ear clipping：

1. 对每个顶点，检查前后顶点构成的 diagonal 是否有效。
2. 有效 diagonal 的顶点标记为可移除 ear。
3. 每轮选择 diagonal 最短的 ear。
4. 输出一个三角形。
5. 删除 ear 顶点。
6. 更新相邻顶点的 diagonal 标记。
7. 剩 3 个点时输出最后三角形。

diagonal 有效需要：

- 在顶点锥内。
- 不与其他边相交。

这保证三角形在多边形内部。

#### 三角形合并为凸多边形

初始三角形太碎。`getPolyMergeValue()` 尝试合并两个共享边 polygon：

1. 合并后顶点数不能超过 `MAX_VERTS_PER_POLY`，源码中 TileCache 为 6。
2. 两个 polygon 必须共享一条边。
3. 合并后的两个关键角必须保持 convex。
4. 返回共享边长度平方作为 merge value。

主循环每次选择 merge value 最大的一对合并，直到无法合并。这会尽量生成更大的凸 polygon，减少 Detour A* 节点数。

#### 删除 portal 上的多余边界顶点

某些 contour 顶点被标记了 `0x80`，表示可以尝试删除。删除前 `canRemoveVertex()` 会验证：

1. 删除该顶点后，周围剩余边数量足够形成 polygon。
2. 受影响边数量不超过固定 buffer。
3. open edges 不超过 2，避免两个不相邻区域错误连成一个洞。

真正删除由 `removeVertex()` 完成：

1. 删除所有包含该顶点的 polygon。
2. 移除顶点并修正所有索引。
3. 收集被删除 polygon 留下的洞边界。
4. 把洞边界重新三角化。
5. 再把三角形合并成凸多边形。
6. 写回 mesh。

这一步可以减少 tile 边界上的冗余点，让相邻 tile portal 更干净。

#### 构建 adjacency

`buildMeshAdjacency()` 做两件事：

1. 找内部共享边。

   它先把 `v0 < v1` 的边放入 hash-like 链，再用反向边匹配。匹配到说明两个 polygon 共享边，于是在 `polys[nvp + edge]` 写邻接 polygon index。

2. 标记 portal 边。

   遍历 contour 中带 portal direction 的边，在 mesh 边里找重叠边段。如果是开放边且和 portal 边重叠，就在邻接槽写：

   ```text
   0x8000 | dir
   ```

   后续 `dtCreateNavMeshData()` 会把这个信息转换成 Detour 的外部连接标记。

### 6. 生成 Detour navmesh data

poly mesh 构建完成后，`buildNavMeshTile()` 填充 `dtNavMeshCreateParams`：

- `verts`
- `polys`
- `polyAreas`
- `polyFlags`
- `polyCount`
- `nvp`
- walkable 参数
- tile 坐标和 layer
- `bmin/bmax`
- `buildBvTree = false`

然后调用可选的 `dtTileCacheMeshProcess::process()`。这是用户给 polygon areas 和 flags 做最终映射的扩展点。例如：

- walkable area -> `SAMPLE_POLYFLAGS_WALK`
- water area -> `SAMPLE_POLYFLAGS_SWIM`
- door area -> 特殊 flags

之后：

1. 调 `dtCreateNavMeshData()` 生成 Detour tile data。
2. 从 `dtNavMesh` 删除同坐标旧 tile。
3. 如果新 tile 非空，调用 `navmesh->addTile()` 加入新 tile。

如果 poly mesh 没有 polygon，则只删除旧 tile，让该位置变为空。

TileCache 生成的 Detour tile 通常不构建 BV tree，因为 tile cache layer 本身已经很小，而且动态重建追求低成本。`navmesh->addTile()` 使用 `DT_TILE_FREE_DATA`，表示新生成的 Detour tile data 交给 `dtNavMesh` 管理；如果 add 失败，TileCache 会立即释放这块 `navData`。

## 八、障碍物栅格标记算法

TileCache 动态障碍物的本质是把 layer 上某些 cell 的 `area` 改成 0。

### 1. Cylinder

`dtMarkCylinderArea()`：

1. 根据圆柱 position/radius/height 计算 world AABB。
2. 转成 layer 网格范围。
3. clamp 到 tile 尺寸内。
4. 对范围内每个 cell：
   - 计算 cell center 到圆心的 xz 平面距离。
   - 如果距离大于半径，跳过。
   - 检查 cell height 是否在圆柱 y 范围内。
   - 命中则 `areas[idx] = areaId`。

半径判断使用的是体素空间半径：

```text
r2 = (radius / cs + 0.5)^2
```

`+0.5` 让圆柱覆盖更保守，避免障碍物边缘因为 cell center 量化漏标。

添加 obstacle 时传入的 `areaId` 是 0，所以命中 cell 变不可走。

### 2. AABB box

`dtMarkBoxArea()` AABB 版本：

1. 把 box bmin/bmax 转成 grid min/max。
2. clamp 到 tile。
3. 遍历范围内 cell。
4. 只检查高度是否在 box y 范围内。
5. 命中则写 area。

它没有圆形距离测试，因为水平范围已经是矩形。

### 3. Oriented box

旋转 box 版本：

1. 先用最大半径算一个保守 AABB，确定候选 cell 范围。
2. 对每个 cell，把 cell 坐标转到 box 局部坐标。
3. 判断 `xrot` 是否在 `[-xhalf, xhalf]`。
4. 判断 `zrot` 是否在 `[-zhalf, zhalf]`。
5. 判断高度。
6. 命中则写 area。

源码里 obstacle 保存的不是完整 sin/cos，而是 `rotAux[2]`，由 half angle 计算，用于快速旋转测试。

`getObstacleBounds()` 对 oriented box 使用 `1.41 * max(halfExtents.x, halfExtents.z)` 得到保守水平半径。这个 bounds 只用于找 touched tiles；真正栅格标记时仍会做局部旋转后的 box 内测试。

## 九、和 Detour 的关系

TileCache 不直接服务查询。真正查询仍由 `dtNavMeshQuery` 在 `dtNavMesh` 上完成。

TileCache 负责：

- 保存可快速重建的压缩 tile layer。
- 根据动态障碍物局部重建 Detour tile data。
- 替换 `dtNavMesh` 中对应 tile。

Detour 负责：

- tile 加载后的 link 连接。
- A*、funnel、raycast、nearest 等运行时查询。

Crowd 如果使用同一个 `dtNavMesh`，下一帧 `checkPathValidity()` 会发现旧 polygon ref 失效或 corridor 不可用，然后触发重新规划。

## 十、源码阅读路径和实现因果

### 1. 先读 `dtBuildTileCacheLayer()`：看运行时为什么不需要原始三角形

这个函数把 `heights/areas/cons` 三个 byte grid 压缩保存。读它时要注意：运行时重建不再需要原始三角网格，也不再需要完整 `rcHeightfield`。因为 TileCache 假设静态场景的可走层已经算好，动态障碍物只是在这个可走层上扣掉一些 cell。

这解释了它为什么快，也解释了它的边界：如果运行时真的改变了楼梯高度、地面坡度、天花板净空，仅靠这三个 grid 不够表达。

### 2. 再读 `addObstacle()`：看为什么只是入队

`addObstacle()` 不调用 `buildNavMeshTile()`。它只分配 obstacle、写形状、添加 request。

原因有三个：

1. 调用者可能在一帧内添加多个障碍物，立即重建会重复处理同一 tile。
2. 重建可能较贵，需要由 `update()` 控制预算。
3. add/remove 顺序必须和 obstacle state 一起管理，否则 touched/pending 很容易乱。

所以 TileCache 把外部 API 做成“提交意图”，把实际执行集中在 `update()`。

### 3. 读 `update()`：看 add/remove 如何变成 tile 重建列表

`update()` 的第一阶段把请求转成 `m_update[]`：

```text
ObstacleRequest
  -> obstacle bounds
  -> queryTiles()
  -> obstacle.touched[]
  -> obstacle.pending[]
  -> global m_update[]
```

这里同时有 obstacle 维度和 tile 维度：

- obstacle 维度需要知道自己影响了哪些 tile，以及哪些 tile 还没处理。
- tile 维度需要去重，避免多个 obstacle 影响同一 tile 时重复重建。

这就是 `contains(m_update, ...)` 和 `pending[]` 同时存在的原因。

### 4. 读 `buildNavMeshTile()`：看重建 pipeline 如何闭环

`buildNavMeshTile()` 的本质是一个小型 Recast pipeline：

```text
decompress layer
  -> apply active obstacles to areas
  -> build monotone regions
  -> trace and simplify contours
  -> triangulate and merge convex polys
  -> build adjacency and portal marks
  -> dtCreateNavMeshData()
  -> replace dtNavMesh tile
```

它没有重新 rasterize 三角形，也没有重新计算静态 walkable slope。因为这些静态结果已经被保存在 layer 里。

### 5. 最后读 `dtBuildTileCachePolyMesh()`：看为什么能回到 Detour

Detour 需要的是凸 polygon mesh，包含顶点、poly、邻接和 area/flags。TileCache 从 grid region 出发，必须重新推导这些信息：

1. region 边界给出 contour。
2. contour 三角化得到初始三角形。
3. 三角形合并得到凸 polygon。
4. 共享边匹配得到内部邻接。
5. contour portal 标记得到 tile 边外部连接提示。

这一步是 TileCache 和 Detour 的桥。没有它，TileCache 只能知道哪些 cell 可走，不能被 Detour A* 查询。

## 十一、为什么这些限制存在

### 1. 为什么单个 obstacle touched tiles 有上限

`DT_MAX_TOUCHED_TILES` 是固定数组大小。TileCache 偏向实时和低分配，因此没有为每个 obstacle 动态扩容 touched/pending。上限要求动态障碍物尺寸适中。大范围动态改变应拆成多个 obstacle，或直接重建相关 tile。

### 2. 为什么每次 update 默认只重建一个 tile

tile 重建包含 contour、triangulation、poly merge、Detour tile data 创建。最坏情况下会有明显 CPU 成本。一次只重建一个 tile 可以让帧时间稳定。调用者如果想尽快追平，可以循环调用 `update()` 直到 `upToDate == true`，但这会把成本集中到当前帧。

### 3. 为什么移除 obstacle 也要重建 touched tile

障碍物不是以“增量 patch”存进 navmesh 的，而是通过 area=0 参与重建。移除时，旧 navmesh tile 里已经被扣掉的区域不会自动恢复，必须从原始 compressed layer 重新解压，再应用剩余 active obstacles。

### 4. 为什么 active obstacles 每次都重新标记

如果 tile 上有多个障碍物，删除其中一个时不能只把它覆盖的 cell 改回 walkable，因为这些 cell 可能仍被另一个障碍物覆盖。最稳妥的方式是从干净 layer 开始，把所有仍 active 的 obstacle 重新应用一遍。

### 5. 为什么 TileCache 不保存运行时 link

link 是 `dtNavMesh` 加载 tile 时根据当前邻居 tile 重建的运行时连接。TileCache 只负责产出 tile data。如果 TileCache 自己保存 link，就会和 `dtNavMesh` 的 tile salt、跨 tile 连接、off-mesh 连接产生两套状态，复杂且容易不一致。

### 6. 为什么 TileCache 结果和 SoloMesh 不完全一致

SoloMesh 的输入是完整场景三角形，后续会构建完整 heightfield、compact heightfield、距离场或其他 region partition，再提取全局轮廓。TileCache 的运行时输入则是已经切成小 tile/layer 的 byte grids：

```text
heights + areas + cons
```

运行时重建只从这些 layer 局部推导 region、contour 和 poly mesh，不会重新考虑原始三角形、全局区域形状或跨 tile 的完整轮廓。因此它更快、更适合动态障碍，但 mesh 形状、边界简化和 polygon 划分都可能和 SoloMesh 不同。官方讨论中也直接说明 TileCache 为了小 tile 快速更新做了技巧，不能得到和 solo mesh 一样的结果。

## 十二、调试和排错思路

1. **障碍物添加了但路径没变。**

   先看 `update()` 是否被调用到 `upToDate`，再看 obstacle bounds 是否查到了 touched tile，最后看 `dtMark*Area()` 是否真的把 layer areas 写成 0。

2. **障碍物移除后区域没恢复。**

   检查 obstacle state 是否从 removing 走到 empty，pending 是否清空，以及同一 tile 上是否还有其他 active obstacle。

3. **重建后 tile 消失。**

   `dtBuildTileCachePolyMesh()` 可能得到 `npolys == 0`。这通常表示障碍物覆盖了整个 layer，或 region/contour 构建没有产生有效区域。源码会 remove 旧 navmesh tile 并保持该位置为空。

4. **跨 tile 边界断开。**

   检查 contour portal direction 是否被保留下来，以及 `buildMeshAdjacency()` 是否把开放边标成 `0x8000 | dir`。最终跨 tile 连接仍由 `dtNavMesh::addTile()` 处理。

5. **障碍物边界精度不够。**

   TileCache 的精度受 `cs/ch` 限制。圆柱和 box 最终都落到 cell area 标记上。想更精细，需要更小 cell size，但会增加 tile layer 和重建成本。

## 十三、设计取舍总结

1. **局部重建，而不是完整重建。**

   只重建 obstacle touched tiles，成本低。代价是单个障碍物影响 tile 数有限，且障碍物适合近似为简单形状。

2. **保存压缩 layer，而不是保存完整中间 Recast 数据。**

   内存更小，重建更快。代价是结果受 tile cache layer 表达能力限制。

3. **请求队列和 update 列表。**

   添加/删除障碍物不会立即阻塞，而是在 update 中逐 tile 处理。代价是 navmesh 更新不是瞬时完成，上层需要看 `upToDate`。

4. **区域构建用 monotone sweep。**

   算法快、实现简单、适合小 tile。代价是区域质量不如完整构建流程中的更复杂分区。

5. **障碍物通过 area=0 表达。**

   简单直接，后续 pipeline 自然避开这些 cell。代价是障碍边界精度受 cell size 限制。

6. **最终仍生成标准 Detour tile。**

   TileCache 和普通 Detour 查询完全兼容。动态 tile 替换后，A*、funnel、crowd 都不需要特殊逻辑。
