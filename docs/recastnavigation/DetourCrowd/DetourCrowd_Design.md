# DetourCrowd 实现逻辑详解

`DetourCrowd` 是基于 `Detour` 的多人群局部移动系统。它不负责生成 navmesh，也不替代完整的行为系统。它解决的是：多个 agent 在 navmesh 上朝目标移动时，如何持续维护路径、避开墙和其他 agent、处理 off-mesh connection，并把最终位置约束回导航网格。

相关源码入口：

- `Include/DetourCrowd.h`：crowd、agent、参数、状态机。
- `Include/DetourPathCorridor.h`：路径走廊维护。
- `Include/DetourPathQueue.h`：异步分片寻路队列。
- `Include/DetourObstacleAvoidance.h`：速度采样避障。
- `Include/DetourLocalBoundary.h`：agent 附近墙段缓存。
- `Include/DetourProximityGrid.h`：agent 邻居空间哈希。
- `Source/DetourCrowd.cpp`：每帧 update 主流程。
- `Source/DetourPathCorridor.cpp`：corridor 合并、局部移动、off-mesh 推进。
- `Source/DetourPathQueue.cpp`：分片路径请求队列。
- `Source/DetourObstacleAvoidance.cpp`：速度采样、碰撞时间和调试采样数据。

校验依据：

- 本文按仓库提交 `9f4ce64` 的 `DetourCrowd/Include` 和 `DetourCrowd/Source` 核对，重点检查 `dtCrowd::update()` 的真实执行顺序、两套状态机、corridor 三种合并、固定容量 path queue、局部边界、邻居 broad phase、速度采样和 off-mesh 动画。
- 官方 `dtCrowd` API 文档把它定义为 agent group 的 local steering behaviors，并明确提醒：Crowd 拥有 active agent 的位置，不能直接更新 active agent position；如果必须从外部回写位置，需要 remove 后重新 add。
- 官方文档还说明 Crowd 是 local movement 系统，path corridor 有 256 polygon 限制，不应被理解为无限长距离自动寻路服务。
- 官方 roadmap 把 formations、group behaviors、flowfield 等列为未来/额外方向，说明当前 DetourCrowd 的实现范围主要是局部 steering、避障和 corridor 维护，而不是完整群体战术系统。
- 官方 Crowd 页面仍保留“所有 agent 共用一个 filter”的旧说明，但当前源码已经有 `m_filters[16]` 和 `queryFilterType`。遇到官方叙述与当前实现不一致时，本文以当前头文件和实现为准，并明确标出这种版本漂移。

参考链接：

- [Crowd 官方模块文档](https://recastnav.com/group__crowd.html)
- [`dtCrowd` 官方 API](https://recastnav.com/classdtCrowd.html)
- [官方集成文档](https://recastnav.com/md_Docs_2__2__BuildingAndIntegrating.html)
- [官方 Roadmap：formations、group behavior 与 flowfield 不属于当前实现](https://recastnav.com/md_Docs_2__99__Roadmap.html)
- [官方论坛：位置所有权和 velocity request 的历史背景](https://groups.google.com/g/recastnavigation/c/OFYx4RDwJuo)
- [官方论坛：静止 agent、到达判定与 arrival behavior 的边界](https://groups.google.com/g/recastnavigation/c/IHO0CMbO3b4)

源码覆盖关系：

| 源码 | 本文对应内容 |
| --- | --- |
| `DetourCrowd.cpp` | 初始化容量、agent 生命周期、请求状态机和完整逐帧更新闭环 |
| `DetourPathCorridor.cpp` | corridor 合并、corner 提取、可视/拓扑优化、失效修复和 off-mesh 推进 |
| `DetourPathQueue.cpp` | 8 槽请求队列、sliced A* 预算、结果保活和句柄代数 |
| `DetourObstacleAvoidance.cpp` | circle/segment 预计算、TOI penalty、grid/adaptive sampling 和 early-out |
| `DetourLocalBoundary.cpp` / `DetourProximityGrid.cpp` | 最近墙段缓存与 agent 空间哈希；固定容量和截断规则在对应章节说明 |

## 一、整体定位

`DetourCrowd` 的核心思想是：

![DetourCrowd 从路径走廊到位置约束的逐帧控制闭环](https://oss.euler.icu/teaser/recastnavigation/DetourCrowd/Docs/Images/crowd-control-loop.png)

*图 1：Crowd 先从 corridor 提取局部意图，再进行速度避障、积分和碰撞修正，最后把实际位置约束回 navmesh 并维护 corridor。*

它不建议用于超长距离的全自动寻路。源码注释中也说明 crowd 更偏“local movement”，默认 corridor 长度是 256 个 polygon。长距离目标通常应由上层拆成阶段性目标，或在需要时重新请求目标。

官方 API 文档中的使用流程也体现了这个定位：先分配并初始化 crowd，设置 avoidance 参数，添加 agent 并提交移动请求，然后每帧调用 `update()`，再读取 active agent 状态。也就是说，Crowd 的主入口是逐帧维护已有 agent 状态，而不是一次性“给所有 agent 算完整全局计划”。

## 根源视角：Crowd 是一个分层控制闭环

`DetourCrowd` 不是“多人 A*”，也不是完整物理模拟。它是一个在 navmesh 上运行的分层控制闭环：

上图同时给出了 Crowd 的分层控制关系：corridor 到 corners、dvel、nvel、integrate、resolve 和 constrain，最终合法位置通过 visited polygons 反馈给下一帧 corridor。

每一层只解决一个问题：

| 层 | 输入 | 输出 | 为什么单独存在 |
| --- | --- | --- | --- |
| 路径层 | 目标 polygon 和当前位置 | corridor | 全局拓扑变化慢，不应每帧重算 |
| 意图层 | corridor corners | `dvel` | 把“要去哪里”转成局部速度目标 |
| 避障层 | `dvel`、邻居、墙段 | `nvel` | 在速度空间预测碰撞，先避免再移动 |
| 运动层 | `nvel`、当前 `vel` | 初步 `npos` | 加速度限制让运动连续 |
| 修正层 | 初步 `npos`、邻居重叠、navmesh | 合法 `npos` | 处理采样没避免掉的重叠和越界 |
| 维护层 | 实际移动经过的 polygons | 新 corridor | 让路径跟随真实运动，而不是跟随理想路线 |

这个闭环解释了很多实现细节：为什么 agent 位置不能由外部直接改，为什么要保留 corridor，为什么避障后还要碰撞解算，为什么最后还要 `moveAlongSurface()`。

### 关键不变量

1. **Crowd 拥有 active agent 的位置。**

   `npos`、`vel`、`corridor`、`boundary` 是一组同步状态。外部如果直接改 `npos`，corridor 第一个 polygon、boundary center、邻居网格都会过期。源码因此只提供添加、移除、更新参数、请求目标等接口，不提供直接 set position。

2. **corridor 的第一个 polygon 必须能解释当前位置。**

   每帧最后调用 `corridor.movePosition()`，目的就是让 `corridor.getFirstPoly()` 和 `corridor.getPos()` 重新匹配。如果这个不变量断掉，下一帧 corners、boundary、path validity 都会基于错误拓扑。

3. **target state 是异步寻路的协议。**

   `REQUESTING`、`WAITING_FOR_QUEUE`、`WAITING_FOR_PATH`、`VALID` 等状态不是简单标志，而是在“快速局部路径”和“分片长路径”之间传递责任。没有这个状态机，agent 要么卡住等完整 A*，要么拿不到稳定 corridor。

4. **避障只承诺选择较优速度，不承诺绝对无碰撞。**

   速度采样是有限样本的预测，邻居也在同时移动。源码后面保留 4 轮重叠投影，就是承认采样不是硬约束。

5. **局部边界是缓存，不是真理。**

   `dtLocalBoundary` 只在 agent 移动超过阈值或 polygon 失效时更新。它服务于避障，不参与最终合法性判断。最终位置合法性仍由 navmesh query 决定。

## 二、核心数据结构

### 1. `dtCrowd`

`dtCrowd` 是 agent 池和更新流水线的拥有者。

重要字段：

| 字段 | 作用 |
| --- | --- |
| `m_agents` | 固定大小 agent 池，slot 可复用。 |
| `m_activeAgents` | 每帧收集的 active agent 指针数组。 |
| `m_agentAnims` | off-mesh connection 动画状态。 |
| `m_grid` | `dtProximityGrid`，用于快速查附近 agent。 |
| `m_obstacleQuery` | `dtObstacleAvoidanceQuery`，用于速度采样避障。 |
| `m_pathq` | `dtPathQueue`，用于较长路径的分片 A*。 |
| `m_navquery` | 小 node pool 的 `dtNavMeshQuery`，用于局部查询。 |
| `m_filters` | 多套 `dtQueryFilter`，不同 agent 可使用不同过滤规则。 |

`dtCrowd::init()` 会一次性分配这些结构。之后每帧 update 基本不做动态内存分配。

初始化时有几个固定容量选择：

- `m_agentPlacementHalfExtents = (maxAgentRadius*2, maxAgentRadius*1.5, maxAgentRadius*2)`，同时用于添加 agent 时吸附到 navmesh 和失效恢复。
- `dtProximityGrid` 的 pool size 是 `maxAgents * 4`，cell size 是 `maxAgentRadius * 3`。
- `dtObstacleAvoidanceQuery` 默认容量是 6 个 circle obstacles 和 8 个 segment obstacles，对应 agent 最多 6 个邻居和最多 8 个局部墙段。
- `m_pathResult` 和每个 corridor 的 path buffer 默认 256 个 polygon。
- `m_navquery` 使用较小 node pool，主要服务局部查询；长路径交给 `dtPathQueue` 内部 query。

这些容量不是文档层面的建议，而是源码里的硬边界。超过这些边界时，库通常选择截断、保留最近/最重要的数据，或让请求失败，而不是临时扩容：

- 邻居最多保留 `DT_CROWDAGENT_MAX_NEIGHBOURS = 6` 个。
- corner lookahead 最多 `DT_CROWDAGENT_MAX_CORNERS = 4` 个，但注释说明实际可用角点通常少一个。
- obstacle avoidance 配置最多 8 套。
- query filter 最多 16 套。
- 默认 path queue 请求槽位有限，结果也存入固定 path buffer。

这和 Detour 的整体风格一致：运行时成本可预估，容量由集成方提前规划。

对象使用 Detour allocator 家族创建：`dtAllocCrowd()/dtFreeCrowd()`、`dtAllocProximityGrid()/dtFreeProximityGrid()`、`dtAllocObstacleAvoidanceQuery()/dtFreeObstacleAvoidanceQuery()` 和对应 debug-data API。这样才能与 `dtAllocSetCustom()` 的全局分配策略一致。

`getAgent()` / `getEditableAgent()`、`getGrid()`、`getPathQueue()`、`getNavMeshQuery()` 等访问器主要用于读取状态和调试。`getEditableAgent()` 返回可写指针并不取消位置所有权不变量：直接改 `npos`、corridor 或 target 字段不会自动重建 boundary、grid 和请求状态。稳定集成应优先使用 `updateAgentParameters()` 与 request API。

### 2. `dtCrowdAgent`

`dtCrowdAgent` 是单个 agent 的运行时状态。

关键字段：

| 字段 | 说明 |
| --- | --- |
| `active` | 该 slot 是否正在使用。 |
| `state` | walking、offmesh、invalid。 |
| `corridor` | `dtPathCorridor`，当前 polygon 路径走廊。 |
| `boundary` | `dtLocalBoundary`，附近墙段缓存。 |
| `neis` | 最近邻 agent 列表。 |
| `npos` | crowd 维护的当前位置。用户不能直接改 active agent 的位置。 |
| `dvel` | desired velocity，期望速度。 |
| `nvel` | new velocity，避障采样后的速度。 |
| `vel` | 当前实际速度。 |
| `cornerVerts/cornerFlags/cornerPolys` | corridor 拉直后的前几个转角。 |
| `targetState` | 移动请求状态机。 |
| `targetRef/targetPos` | 目标 polygon 和目标位置。 |
| `targetPathqRef` | 异步 path queue 请求句柄。 |

`state` 和 `targetState` 是两套状态：

- `state` 描述 agent 当前在哪种运动模式：invalid、walking、offmesh。
- `targetState` 描述移动请求进展：none、requesting、waiting queue、waiting path、valid、failed、velocity。

这两者不能合并。一个 agent 可能处于 walking，但目标请求还在 waiting path；也可能处于 offmesh 动画，但 corridor 和 target 仍然有效。

### 3. `dtCrowdAgentParams`

这些参数决定 agent 的几何、运动和更新策略：

| 参数 | 含义 |
| --- | --- |
| `radius/height` | agent 碰撞圆柱。 |
| `maxAcceleration/maxSpeed` | 速度积分约束。 |
| `collisionQueryRange` | 查询邻居 agent 和墙段的范围。 |
| `pathOptimizationRange` | corridor 可视优化 raycast 的最大距离。 |
| `separationWeight` | 分离力权重。 |
| `updateFlags` | 是否启用转向预判、避障、分离、可视优化、拓扑优化。 |
| `obstacleAvoidanceType` | 使用哪一套避障采样参数。 |
| `queryFilterType` | 使用哪一套 navmesh filter。 |

### 4. `dtProximityGrid`

`dtProximityGrid` 是 agent broad phase。它不是二维数组，而是“哈希桶 + item pool”：

```text
bucket hash(x, y) -> item index -> next item -> ...
```

每个 item 存 agent id、覆盖的 grid cell 坐标，以及同 bucket 链表中的 next。`addItem()` 会把 agent 的 2D AABB 覆盖到多个 cell 中；`queryItems()` 查询 AABB 覆盖的 cell，并在输出时做 id 去重。这个结构避免每个 agent 扫描所有其他 agent；真正的距离、高度重叠和排序仍在 `getNeighbours()` 中完成。

### 5. `dtLocalBoundary`

`dtLocalBoundary` 缓存 agent 附近的墙段和对应的局部 polygon 集合：

- 最多 `MAX_LOCAL_POLYS = 16` 个 polygon。
- 最多 `MAX_LOCAL_SEGS = 8` 条墙段。
- `m_center` 记录缓存对应的位置。

`update()` 先调用 `findLocalNeighbourhood()` 找 collision range 内可达 polygon，再对每个 polygon 调 `getPolyWallSegments()`。墙段会按距离插入固定数组，只保留最近的几条。`isValid()` 只检查缓存 polygon 是否仍通过 filter，不检查墙段本身；只要 polygon 失效，就会触发重建。

### 6. agent 生命周期和请求入口

`dtCrowd` 使用固定 agent 池。`addAgent()` 不分配新对象，而是找第一个 inactive slot：

1. 拷贝 `dtCrowdAgentParams`。
2. 用 `findNearestPoly()` 把请求位置吸附到 navmesh。
3. `corridor.reset(ref, nearest)`，把 corridor 初始化成单 polygon。
4. 清空 boundary、邻居、速度、target 状态。
5. 如果找到 ref，状态设为 walking；否则设为 invalid。
6. 标记 slot active 并返回 index。

`removeAgent()` 只把 slot 标为 inactive，不销毁 corridor buffer。之后这个 slot 可被新 agent 复用。

移动请求有三类：

- `requestMoveTarget()`：提交目标 ref/pos，状态变成 `REQUESTING`，下一次 `updateMoveRequest()` 处理。
- `requestMoveVelocity()`：把 `targetPos` 当作速度向量，状态变成 `TARGET_VELOCITY`，绕过路径请求但仍参与避障、积分和 navmesh 约束。
- `resetMoveTarget()`：清空目标、目标速度和 path queue ref，状态回到 `NONE`。

`resetMoveTarget()` 和新的 request 只让 agent 忘掉旧 `targetPathqRef`，`dtPathQueue` 没有 cancel API；已经进入队列的旧 sliced query 仍会占用槽位直到完成后保活超时。频繁改目标不会破坏 agent 状态，但可能短期消耗 8 个 queue slot，所以上层应避免每帧无条件重复提交目标。

`updateAgentParameters()` 只更新参数，不重算 corridor 或当前位置。因此改变 `queryFilterType`、半径、速度等参数后，实际路径有效性要等后续 update 的 validity 检查和局部逻辑来修正。

`queryFilterType` 和 `obstacleAvoidanceType` 在热路径里直接作为固定数组下标使用，源码不会在每次访问时自动 clamp。集成层应保证它们分别位于 `[0, 15]` 和 `[0, 7]`。这也解释了为什么当前实现可以让不同 agent 选择不同 filter，而官方模块页末尾“所有 agent 共用一个 filter”的旧段落已经不再准确。

官方文档提醒：新添加 agent 的 path 相关信息要在至少一次 `update()` 后才可用；agent 对象来自池并会复用，所以外部缓存 `dtCrowdAgent*` 时必须检查 `active`。这和源码一致：`addAgent()` 只初始化 corridor 和状态，真正的 corners、邻居、boundary、path request 推进都发生在后续帧的 update 流水线中。

## 三、路径走廊：`dtPathCorridor`

Crowd 不会每帧从当前位置到目标完整 A*。它维护一个 corridor：

```text
当前位置所在 poly -> ... -> 目标所在 poly
```

`reset(ref, pos)` 把 corridor 变成单 polygon；`setCorridor(target, path, npath)` 可以直接安装调用方已有路径，但它只拷贝目标和 ref 数组，不验证每对 polygon 是否相邻、是否通过当前 filter。直接使用后者时，路径合法性责任属于调用方。

### 1. corridor 的意义

agent 的真实运动由避障和碰撞修正影响，可能不会严格沿全局路径中心移动。如果每帧重新寻路，成本高且路径容易抖动。corridor 允许 agent 在局部偏离后，通过局部 navmesh 查询把路径头部修正回来。

### 2. `movePosition()`

每帧 agent 积分和碰撞修正后，`dtCrowd` 调用：

```text
ag->corridor.movePosition(ag->npos, navquery, filter)
```

内部算法：

1. 从 corridor 第一个 polygon 和旧位置开始。
2. 调 `dtNavMeshQuery::moveAlongSurface()`，把希望的新位置限制在 navmesh 可达表面上。
3. 得到局部访问过的 polygon 列表 `visited`。
4. 调 `dtMergeCorridorStartMoved()` 把 `visited` 合并到 corridor 开头。
5. 用 `getPolyHeight()` 修正 y 高度。
6. 更新 corridor 位置。

这样 agent 被避障推出原路径边缘时，corridor 开头会跟着实际位置重新贴合。

`moveTargetPosition()` 是对称操作：从 corridor 最后一个 polygon 和旧 target 出发调用 `moveAlongSurface()`，再用 `dtMergeCorridorEndMoved()` 修正 corridor 尾部。它适合目标点小范围移动的场景，不适合目标跨很远位置；远目标应重新请求路径。

### 3. 三种 corridor 合并

`DetourPathCorridor.cpp` 中有三个合并函数：

| 函数 | 场景 | 核心逻辑 |
| --- | --- | --- |
| `dtMergeCorridorStartMoved` | agent 当前位置移动了 | 找 `visited` 和旧 path 最远公共 polygon，把 visited 反向接到 path 前面。 |
| `dtMergeCorridorEndMoved` | 目标位置移动了 | 找旧 path 和 visited 的公共点，把 visited 后半接到 path 尾部。 |
| `dtMergeCorridorStartShortcut` | 发现可见捷径或局部拓扑优化 | 用新搜索结果替换 path 开头，保留后面仍然有效的旧 path。 |

它们的共同目标是避免整条路径重算，只替换局部发生变化的一段。

![Corridor 局部合并与异步目标状态](https://oss.euler.icu/teaser/recastnavigation/DetourCrowd/Docs/Images/crowd-path-state.png)

*图 2：左侧展示三种 corridor 局部替换如何在公共 polygon 处保持连通；右侧展示目标请求从提交、等待到有效或失败的异步状态。*

### 4. `findCorners()`

`findCorners()` 调用 Detour 的 `findStraightPath()`，从 corridor 中提取前几个转角。Crowd 默认最多只关心 `DT_CROWDAGENT_MAX_CORNERS` 个角点，因为局部 steering 不需要知道很远的路径。

它还会做两个裁剪：

1. 删除离 agent 太近的首个角点。
2. 如果遇到 off-mesh connection，只保留到该 connection 为止，便于触发特殊移动。

### 5. corridor 优化

Crowd 有两种 corridor 优化：

1. 可视优化 `optimizePathVisibility()`。

   它从当前位置向下一个或下下个角点做 navmesh raycast。如果 raycast 几乎没有被墙阻挡，就说明 corridor 开头可以走捷径，于是用 raycast 经过的 polygon 替换 corridor 开头。

2. 拓扑优化 `optimizePathTopology()`。

   它执行一个小步数的 sliced A*，从当前 poly 到 corridor 末端重新找一段局部路径。如果找到更好的局部拓扑，就用 `dtMergeCorridorStartShortcut()` 合并。

可视优化解决“明明直线可见却还绕旧 corridor”的问题；拓扑优化解决“局部 polygon 拓扑因为偏移或 tile 边界导致路径不理想”的问题。

### 6. 路径修复和 off-mesh 推进

`fixPathStart()` 用于当前位置 poly 失效后的恢复。它把 corridor 位置改成安全位置，并强制把 path 开头替换为 `safeRef`。源码在路径太短时会构造一个至少 3 个槽位的形状，目的是让后续合并逻辑有稳定的 sentinel。

`trimInvalidPath()` 从 path 开头向后扫描，保留仍通过 filter 的最长前缀：

- 如果第一个 poly 就失效，用 `safeRef/safePos` 重置成单 polygon。
- 如果中间失效，截断到失效前。
- 最后把 target clamp 到新的最后一个 polygon 边界。

`moveOverOffmeshConnection()` 在 agent 触发 off-mesh connection 时使用。它在 path 中找到 off-mesh poly，把 path 前缀剪掉，然后通过 `dtNavMesh::getOffMeshConnectionPolyEndPoints()` 取得按行进方向排序的 start/end，并把 corridor position 直接推进到 end。

## 四、异步路径队列：`dtPathQueue`

当新目标很远时，Crowd 不会在 `requestMoveTarget()` 里立即完整寻路。请求会进入 agent 的目标状态机，并在 `updateMoveRequest()` 中处理。

### 1. 目标状态机

主要状态：

| 状态 | 含义 |
| --- | --- |
| `DT_CROWDAGENT_TARGET_NONE` | 无移动目标。 |
| `DT_CROWDAGENT_TARGET_REQUESTING` | 新目标刚提交，等待处理。 |
| `DT_CROWDAGENT_TARGET_WAITING_FOR_QUEUE` | 快速搜索只得到部分路径，需要进入异步队列。 |
| `DT_CROWDAGENT_TARGET_WAITING_FOR_PATH` | path queue 正在分片 A*。 |
| `DT_CROWDAGENT_TARGET_VALID` | corridor 已能朝目标移动。 |
| `DT_CROWDAGENT_TARGET_FAILED` | 请求失败。 |
| `DT_CROWDAGENT_TARGET_VELOCITY` | 不按目标寻路，直接按给定速度移动。 |

图 2 右侧把 NONE、请求、等待、队列、有效、失败和 velocity 模式放在同一状态视图中；具体枚举名与逐条转换条件见上表和下文。

这些状态的设计重点是“可继续运动”。例如快速搜索只得到 partial path 时，agent 不是停在原地等 path queue，而是先进入 `VALID` 或等待队列相关状态，沿当前可用 corridor 移动。异步结果回来后再把新 path 接到仍有效的旧 corridor 上。这个状态机避免了长路径请求带来的帧尖峰，也避免 agent 在等待完整路径时完全无响应。

### 2. 快速搜索

当状态是 `REQUESTING`：

1. 从当前 corridor 第一个 polygon 到目标 polygon 初始化 sliced A*。
2. 只执行少量迭代，源码中是 `MAX_ITER = 20`。
3. 如果是 replan，优先 `finalizeSlicedFindPathPartial()`，尽量沿用旧 path。
4. 如果是新目标，直接 `finalizeSlicedFindPath()`。
5. 如果结果已经到目标，状态设为 `VALID`。
6. 如果只是部分路径，先让 agent 沿部分 corridor 动起来，再进入 path queue 补全。

这个设计让 agent 对新目标响应很快，即使完整路径还没算完。

### 3. path queue

`dtPathQueue` 内部有固定大小请求数组，默认最多 8 个请求。每个请求保存：

- start/end ref。
- start/end pos。
- filter。
- 当前 status。
- 输出 path buffer。
- keepAlive。

`update(maxIters)` 每帧最多消耗一定 A* 迭代数。`dtCrowd` 当前每帧给整个 queue 共享 `MAX_ITERS_PER_UPDATE = 100` 次扩展预算，而 queue 内部 query 的 node pool 容量是 `MAX_PATHQUEUE_NODES = 4096`。队列从 `m_queueHead` 轮转处理槽位，预算用尽就停；因此 100 是整条队列的帧预算，不是每个请求各 100。请求完成后结果最多保活 2 个额外 update tick，第三次仍未读取便释放槽位。

队列保存的是调用方传入的 `const dtQueryFilter*`，不是 filter 副本；头文件也明确把它标为潜在生命周期风险。`dtCrowd` 传入的是自身固定 `m_filters[]` 中的地址，所以对象生命周期稳定。直接使用 `dtPathQueue` 时，调用方必须保证 filter 在请求完成并取走结果前仍然存在，且不要并发修改其 flags/cost，否则同一次 sliced A* 的前后帧可能采用不同规则。

请求 slot 的生命周期是：

1. `request()` 找空 slot，写入 start/end/filter/path buffer，状态置为 waiting。
2. `update()` 第一次看到 waiting 请求时，调用 `initSlicedFindPath()` 并转为 in-progress。
3. 后续 update 按迭代预算调用 `updateSlicedFindPath()`。
4. 完成后 `finalizeSlicedFindPath()` 写入请求自己的 path buffer。
5. `getPathResult()` 拷贝结果并释放 slot。
6. 如果结果迟迟不取，`keepAlive` 递减到 0 后自动释放。

### 4. 合并 path queue 结果

当异步路径完成：

1. 读取 path queue 结果。
2. 检查结果开头是否等于旧 corridor 的末尾。因为请求发出后 agent 可能已经移动，必须确保还能拼接。
3. 把旧 path 前段拷到结果前面。
4. 删除 `A, B, A` 这种 trackback。
5. 如果结果仍未到目标，把目标位置夹到最后一个 polygon 上，并标记 partial。
6. 更新 corridor。

这段逻辑非常关键：异步搜索返回的是“从发起请求时 corridor 尾部到目标”的路径，而 agent 在等待期间继续移动，所以必须把“已经走过或仍有效的旧 corridor 前段”和“异步新结果”拼起来。

## 五、每帧更新主流程：`dtCrowd::update`

`dtCrowd::update(dt)` 是整个库的核心。源码中的执行顺序很有讲究，因为后续步骤依赖前面生成的数据。

图 1 按源码顺序概括了 update 的运动闭环；路径有效性、异步请求和拓扑优化发生在图示的 corridor 阶段，proximity grid、local boundary 和邻居查询为 nvel 采样准备障碍输入。

下面逐步展开。

### 1. 检查路径有效性

`checkPathValidity()` 做三类恢复：

1. 当前 polygon 是否仍有效。

   如果当前位置所在 polygon 被动态修改或 filter 排除，Crowd 会尝试在附近找最近 polygon。找到后用 `corridor.fixPathStart()` 修复路径开头；找不到则 agent 进入 invalid 状态。

2. 目标 polygon 是否仍有效。

   如果目标 ref 失效，尝试在目标位置附近重新找 polygon。找不到则移动请求失败。

3. corridor 前几个 polygon 是否仍有效。

   默认检查前 10 个 polygon。如果有无效 poly，触发 replan。

另外，如果 agent 接近一个 partial path 的尾部，但尾部不是目标，也会按延迟触发重新规划。

### 2. 处理移动请求

`updateMoveRequest()` 先做快速搜索，再把需要补全的请求放入 path queue。这个步骤必须早于 steering，因为 steering 需要 corridor 和 corners。

### 3. 拓扑优化

`updateTopologyOptimization()` 不会每帧优化所有 agent。它按 `topologyOptTime` 排队，每次只优化少量 agent，源码默认一次 1 个。这样把成本摊到多帧，避免 crowd 数量大时尖峰。

### 4. 构建 proximity grid

`dtProximityGrid` 是 2D 均匀网格哈希：

1. 每帧清空。
2. 把每个 active agent 的 AABB 加入覆盖的格子。
3. 查询邻居时只查附近格子。
4. 返回 id 时去重。

哈希函数使用两个大质数混合 x/y，然后按 bucket mask 截断。网格只用于 broad phase，精确距离和高度重叠仍在 `getNeighbours()` 里检查。

源码里每个 agent 加入 grid 的 AABB 是：

```text
[npos.x - radius, npos.z - radius] -> [npos.x + radius, npos.z + radius]
```

因此 grid 只表达水平圆盘的包围盒，不表达高度。高度过滤放在邻居查询阶段，用 agent 的 `height` 判断两个圆柱是否重叠。

### 5. 更新局部边界

每个 walking agent 有一个 `dtLocalBoundary`。当 agent 离上次 boundary center 超过一定阈值，或缓存 polygon 已失效时，重新更新。

更新算法：

1. 调 `findLocalNeighbourhood()` 找 collision range 内可达 polygon。
2. 对每个 polygon 调 `getPolyWallSegments()`。
3. 过滤掉距离太远的 segment。
4. 按距离插入固定大小数组，只保留最近墙段。

这些墙段会在避障阶段作为 segment obstacles。

`dtLocalBoundary::addSegment()` 按距离插入有序数组。数组满时，只有更近的墙段才会替换更远墙段。这样 avoidance query 容量固定时，优先使用最可能影响当前 agent 的墙。

### 6. 查询邻居 agent

`getNeighbours()` 用 proximity grid 查候选 id，然后过滤：

1. 跳过自己。
2. 检查高度区间是否重叠。
3. 只看 2D 距离是否在 `collisionQueryRange` 内。
4. 按距离从近到远插入，最多保留 `DT_CROWDAGENT_MAX_NEIGHBOURS` 个。

排序很重要，因为后面避障 query 默认容量有限，优先使用最近邻。

### 7. 提取 corners 和可视优化

对每个有目标的 walking agent：

1. `corridor.findCorners()` 提取局部转角。
2. 如果启用 `DT_CROWD_OPTIMIZE_VIS`，取第 1 或第 2 个 corner 做 raycast。
3. 如果 raycast 几乎可达，则缩短 corridor 开头。

这一步在计算 steering 前执行，因为 steering 直接朝 corner 走。

### 8. 触发 off-mesh connection

如果最后一个 corner 标记为 off-mesh connection，并且 agent 离它小于 `radius * 2.25`：

1. `corridor.moveOverOffmeshConnection()` 把 corridor 推进到 connection 之后。
2. 从 navmesh 取 connection 的起点和终点。
3. 创建 `dtCrowdAgentAnimation`。
4. agent 状态切到 `DT_CROWDAGENT_STATE_OFFMESH`。

off-mesh 动画期间 agent 不参与普通 walking 更新，位置由动画插值控制。

源码还会在触发 off-mesh 时清理普通 steering 状态：agent 状态切为 `DT_CROWDAGENT_STATE_OFFMESH`，动画结构记录 `initPos/startPos/endPos/polyRef/t/tmax`。这意味着 off-mesh 不是 obstacle avoidance 的特殊速度，而是 Crowd update 主流程中的另一种运动模式。真实项目如果要接入动画系统，应把这个状态当成“开始特殊移动”的信号，而不是继续依赖普通 `dvel/nvel`。

## 六、期望速度计算

### 1. 直线转向和预判转向

如果没有 `DT_CROWD_ANTICIPATE_TURNS`，Crowd 使用直线转向：

```text
dvel = normalize(firstCorner - npos) * speed
```

如果启用预判转向，`calcSmoothSteerDirection()` 会同时看前两个 corner：

1. `dir0 = corner0 - npos`
2. `dir1 = normalize(corner1 - npos)`
3. `dir = dir0 - dir1 * len(dir0) * 0.5`
4. normalize

直觉是：不要死盯第一个拐点，而是提前向第二段路径偏一点，让转弯更平滑。

### 2. 到达终点减速

如果最后一个 corner 是 end-of-path，Crowd 会计算 agent 到目标的距离。距离小于 `radius * 2` 时按比例降低速度：

```text
speedScale = distanceToGoal / slowDownRadius
dvel = steerDir * maxSpeed * speedScale
```

这样 agent 靠近终点时不会以最大速度冲过目标。

### 3. 分离力

如果启用 `DT_CROWD_SEPARATION`，Crowd 会对邻居施加分离修正：

1. 遍历邻居。
2. 计算从邻居指向自己的 `diff`。
3. 距离越近，权重越大：

   ```text
   weight = separationWeight * (1 - (dist / separationDist)^2)
   ```

4. 把 `diff / dist * weight` 累加到 `disp`。
5. 用平均分离向量调整 `dvel`。
6. 如果新速度超过期望速度，源码按 `desiredSpeed^2 / speed^2` 缩放。

分离力是软偏好，后面还有真正的避障采样和重叠解算。

第 6 步不是常见的 `desiredSpeed / speed` 标准归一化 clamp，而是平方比缩放，因此会把超速结果压到不高于目标速度、并可能压得更低。文档这里保留源码的真实公式，避免读者按“普通限幅”去推导却和运行结果对不上。

## 七、速度采样避障：`dtObstacleAvoidanceQuery`

Crowd 的避障不是直接修改位置，而是在速度空间里挑一个代价最低的候选速度。

### 1. 障碍物输入

对每个 agent：

1. 邻居 agent 作为 circle obstacles：

   ```text
   position, radius, current velocity, desired velocity
   ```

2. 附近墙段作为 segment obstacles。

### 2. `prepare()`

对 circle obstacle，预计算：

- `dp`：从当前 agent 指向障碍 agent 的方向。
- `np`：倾向绕行的侧向法线。

侧向法线根据当前 agent 和邻居的 desired velocity 相对方向选择，目的是让双方更一致地从某一侧错开。

对 segment obstacle，预计算 agent 是否已经非常靠近 segment。如果已经贴墙，后续碰撞时间会特殊处理。

### 3. 单个速度的代价：`processSample()`

给定候选速度 `vcand`，总代价由四部分组成：

```text
penalty =
  weightDesVel * distance(vcand, dvel) / vmax
+ weightCurVel * distance(vcand, vel) / vmax
+ weightSide   * sideBias
+ weightToi    * timeOfImpactPenalty
```

每一项含义：

| 项 | 作用 |
| --- | --- |
| desired velocity penalty | 不要偏离想走的方向太多。 |
| current velocity penalty | 不要突然改变当前速度，减少抖动。 |
| side penalty | 鼓励从一致的侧面绕过邻居。 |
| time of impact penalty | 越快撞到障碍，惩罚越大。 |

![速度空间避障采样与 off-mesh 特殊运动](https://oss.euler.icu/teaser/recastnavigation/DetourCrowd/Docs/Images/crowd-avoidance-offmesh.png)

*图 3：左侧把 desired/current/side/TOI 四类代价投射到速度空间候选；右侧展示 walking agent 进入 off-mesh 后由独立插值状态推进。*

#### circle obstacle 的碰撞时间

源码使用类似 RVO 的相对速度：

```text
vab = 2 * vcand - self.vel - obstacle.vel
```

然后用 sweep circle vs circle 求未来一段时间内是否相撞。如果 `htmin` 是首次碰撞时间：

- 没有交点：不产生 TOI 惩罚。
- 已经重叠：把退出时间转换成更强惩罚。
- 越早碰撞，`timeOfImpactPenalty` 越大。

#### segment obstacle 的碰撞时间

对墙段，候选速度被看成一条 ray：

- 如果 ray 与 segment 相交，交点参数就是碰撞时间近似。
- 如果 agent 已贴近 segment，并且速度指向墙外侧，视为立即碰撞。
- 墙段的碰撞时间会乘 2，表示面对墙时惩罚相对柔和一些。

#### early out

`processSample()` 会根据当前最小 penalty 估算一个碰撞时间阈值。如果仅 desired/current velocity 两项已经不可能优于当前最优，就提前返回。这能明显减少采样成本。

### 4. 网格采样

`sampleVelocityGrid()` 在速度圆盘内做规则网格采样：

1. 采样中心偏向 desired velocity：`dvel * velBias`。
2. 网格覆盖剩余速度范围。
3. 跳过速度圆外的点。
4. 调 `processSample()`。
5. 取 penalty 最低速度。

网格采样简单稳定，但采样数量随 `gridSize^2` 增长。

### 5. 自适应采样

`sampleVelocityAdaptive()` 是 Crowd 默认使用的方式。

流程：

1. 构造围绕 desired velocity 方向的采样 pattern。
2. 初始搜索半径为 `vmax * (1 - velBias)`。
3. 在当前中心周围采样一圈候选速度。
4. 找到本轮最佳速度。
5. 把搜索中心移到最佳速度，半径减半。
6. 重复 `adaptiveDepth` 层。

这相当于在速度空间做粗到细搜索。相比固定大网格，它用较少样本集中搜索更可能有用的区域。

`dtObstacleAvoidanceDebugData` 可以记录每个候选速度及其 total/desired/current/side/TOI penalty。通过 `normalizeSamples()` 可把记录的 penalty 分量归一化到便于显示的 0..1 范围；它只服务分析和可视化，不参与最佳速度选择。`dtCrowd::update()` 只会为 `dtCrowdAgentDebugInfo::idx` 指定的 agent 把这些样本写入调用方提供的 debug buffer，避免为所有 agent 支付记录成本。

## 八、速度积分和碰撞解算

### 1. 加速度约束

`integrate()` 不会直接令 `vel = nvel`，而是按最大加速度限制速度变化：

```text
dv = nvel - vel
if length(dv) > maxAcceleration * dt:
    dv = normalize(dv) * maxAcceleration * dt
vel += dv
npos += vel * dt
```

这样运动更像物理对象，避免速度瞬间跳变。

### 2. 重叠碰撞解算

速度采样是预测性的，但仍可能发生 agent 圆盘重叠。Crowd 后面还有 4 轮位置级修正：

1. 对每个 agent 清空 `disp`。
2. 遍历邻居。
3. 如果两个圆盘重叠，计算 penetration。
4. 沿二者连线方向推开。
5. 如果完全重合，就根据 agent index 和 desired velocity 选一个稳定的侧向方向。
6. 对所有邻居平均 displacement。
7. 一轮结束后统一加到 `npos`。
8. 重复 4 次。

这是一个轻量投影式解算，不追求严格物理正确，但足够让 crowd 不互相穿插。

### 3. 约束回 navmesh

碰撞解算可能把 agent 推到 navmesh 外。最后必须：

```text
corridor.movePosition(npos)
npos = corridor.getPos()
```

如果 agent 当前没有路径目标，或处于 velocity 模式，还会把 corridor 重置为当前位置单 polygon，避免残留旧路径。

## 九、off-mesh connection 动画

Crowd 把 off-mesh movement 做成一个简单时间插值：

图 3 右侧对应 WALKING 到 OFFMESH 再回到 WALKING 的主状态转换；INVALID 属于当前位置无法恢复到有效 polygon 的异常分支，不参与普通 traversal 插值。

1. `moveOverOffmeshConnection()` 得到 `initPos`、`startPos`、`endPos`。
2. `tmax = distance(start, end) / maxSpeed * 0.5`。
3. 前 15% 时间从当前位置插到 connection start。
4. 后 85% 时间从 start 插到 end。
5. 结束后 agent 状态切回 walking。
6. 动画期间 velocity、desired velocity 清零。

这段插值只是示例级 traversal：它不知道抛物线、梯子姿态、门交互或 root motion。由于 active agent 的位置属于 Crowd，应用层不应一边让该内置插值运行、一边每帧把外部动画位置写回 `npos`。需要完整动画控制时，应替换/扩展这段 off-mesh 状态处理，或暂停 Crowd 对该角色的所有权并在 traversal 结束后以一致方式重新放置 agent，而不是形成两个位置权威。

`tmax` 由 `distance / maxSpeed * 0.5` 计算，所以这也不是“严格按 maxSpeed 穿越”的通用运动学模型。使用 off-mesh traversal 的 agent 应给出正的 `maxSpeed`；`maxSpeed == 0` 虽满足头文件的非负参数注释，却无法得到有意义的 traversal 时长。

还有一个逐行读源码才能看到的边界：当 `anim.t > anim.tmax` 时，结束分支只关闭 animation 并把 state 改回 walking，没有在该分支强制 `npos = endPos`。通常上一帧插值已非常接近终点，下一帧又会被 corridor 约束到 connection 后的 polygon，但它不是严格的“结束帧必等于 endPos”合同。需要精确落点的动画集成应自行补齐这一语义。

## 十、源码阅读路径和实现因果

想从源码根源理解 Crowd，建议按 `dtCrowd::update()` 的真实执行顺序读，而不是先读所有类定义。

### 1. `checkPathValidity()`：先保证旧状态还能用

Crowd 每帧首先检查已有 corridor 和 target。原因是动态 navmesh 或 filter 变化可能让上一帧的 ref 失效。如果不先检查，后面的 corners、boundary、避障都会建立在坏 ref 上。

这里有三个恢复层级：

1. 当前 poly 失效：用 `findNearestPoly()` 把 agent 拉回最近可走面。
2. target poly 失效：尝试重新吸附 target。
3. corridor 前段失效：触发 replan，但尽量保留当前仍可用路径。

这个步骤体现了 Crowd 的容错策略：能修局部就修局部，不能修才让请求失败或 agent invalid。

### 2. `updateMoveRequest()`：先给短路径，再补长路径

新目标不会直接进入完整 A*。源码先用 20 次迭代做快速 sliced search：

- 如果已经到目标，马上得到 valid corridor。
- 如果只得到部分路径，agent 先沿部分路径走，同时进入 path queue 补全。

这样设计的原因是交互响应。用户点击远处目标后，agent 不应该原地等长路径搜索结束；它应该先朝已知可达方向移动。

### 3. `updateTopologyOptimization()`：把昂贵优化摊开

拓扑优化每次只处理少量 agent，默认一次 1 个。因为它会跑局部 sliced A*，如果 crowd 很大，全员每帧优化会产生尖峰。源码用 `topologyOptTime` 排序，谁最久没优化，谁优先。

这是一种“公平但限流”的实时系统设计。

### 4. proximity grid 和 local boundary：先 broad phase，再 narrow phase

邻居 agent 查询分两层：

1. `dtProximityGrid` 用格子 hash 粗筛候选。
2. `getNeighbours()` 再检查高度和 2D 距离。

墙段也是两层：

1. `findLocalNeighbourhood()` 找附近可达 polygon。
2. `getPolyWallSegments()` 提取真正墙段。

这样避免每个 agent 对所有 agent、所有墙段做 O(n²) 或全图扫描。

### 5. steering 和 avoidance：先表达意图，再找可行速度

`dvel` 是“如果没有障碍，我想怎么走”。`nvel` 是“考虑障碍后，我这帧最好怎么走”。这两个变量必须分开：

- 分离力修改的是意图。
- obstacle avoidance 在速度空间选择实际候选。
- debug 可视化也能分别看意图和避障结果。

如果直接把所有规则混进一个速度，调参会很难判断到底是路径转向、分离、避墙还是加速度约束造成的问题。

### 6. integrate 和 collision：预测不够时做投影修正

速度采样是前瞻式，碰撞解算是后验式。两者组合的原因是：

- 只做速度采样，有限样本下仍会穿插。
- 只做位置投影，agent 会互相挤压抖动，且没有提前绕行。

源码先采样再投影，能得到较自然的绕行，同时有最后的重叠兜底。

### 7. 最后 `moveAlongSurface()`：把局部物理结果还给拓扑系统

这是 update 中非常关键的一步。碰撞投影只知道圆盘重叠，不知道 navmesh 边界。它可能把 agent 推到不可走区。`moveAlongSurface()` 把结果夹回 navmesh，并把实际经过的 polygon 合并进 corridor。

也就是说，Crowd 的最终状态不是由速度积分单独决定的，而是：

```text
velocity integration
+ collision projection
+ navmesh surface constraint
+ corridor merge
```

这四者共同决定。

## 十一、参数调节背后的原因

### `collisionQueryRange`

它越大，agent 越早感知邻居和墙，但每帧候选障碍越多，避障更保守也更贵。通常它应大于 `radius * 2`，否则 agent 太晚发现碰撞，只能靠位置投影硬推开。

### `pathOptimizationRange`

它决定可视优化 raycast 能看多远。过小会错过明显捷径；过大会让 raycast 成本变高，也可能让 corridor 在复杂空间里频繁改头。

### `separationWeight`

它是软分离，不是硬碰撞半径。值太低时 agent 容易贴得很近，值太高时 crowd 会像互相排斥的粒子，可能偏离路径和门洞。

### `maxAcceleration`

它控制 `vel` 追上 `nvel` 的速度。过低时 agent 避障反应慢，容易被投影修正推开；过高时速度变化突兀，避障采样的 current velocity penalty 也更难稳定。

### `obstacleAvoidanceType`

不同参数组本质上是在选择速度空间搜索策略。更高的 `adaptiveDivs/rings/depth` 能找到更好的速度，但样本数更多。对大量 agent，通常要在视觉质量和 CPU 成本之间分层配置。

## 十二、常见误解

1. **Crowd 不是完整动态寻路系统。**

   它不会因为另一个 agent 站在门口就把全局路径改到另一条走廊。agent 之间主要通过局部速度避让。真正改变 navmesh 拓扑的临时障碍，应使用 TileCache 或上层逻辑。

2. **`requestMoveVelocity()` 不等于物理控制。**

   velocity 模式绕过目标路径，但仍会经过避障、积分、碰撞、navmesh 约束。它适合短时外部控制，不适合强制瞬移。

3. **partial path 不是错误。**

   对 crowd 来说，partial path 仍然有价值。agent 可以先走到当前可达边界，同时 path queue 或后续 replan 尝试补路径。

4. **off-mesh connection 需要上层语义。**

   Crowd 只提供简单插值和状态切换。真正跳跃、爬梯、开门动画，以及何时允许通过，通常由游戏逻辑根据 poly flags、area 或 user data 处理。

5. **Crowd 不能直接同步外部物理位置。**

   官方 API 文档明确说明 active agent 的位置由 Crowd 拥有。原因不是接口缺失，而是 `npos`、corridor、boundary、target、邻居网格共同组成一致状态。如果外部物理系统把角色推走，正确做法通常是移除并重新添加 agent，或在上层把物理控制转成 `requestMoveVelocity()` / 目标请求，再让 Crowd 自己通过 `moveAlongSurface()` 维护 corridor。

6. **没有单独的“已到达”状态。**

   `targetState == VALID` 只说明 corridor 可用，不代表已经到终点；`ncorners == 0` 也可能在刚提交请求、尚未生成 corners 时短暂出现。官方论坛建议同时检查最后一个 corner 的 `DT_STRAIGHTPATH_END` 标志和 agent 到该 corner 的 2D 距离。到达阈值、排队占位、多个 agent 共享目标点等策略很依赖游戏规则，所以库没有替应用层决定。

7. **`resetMoveTarget()` 不保证当前帧瞬间静止。**

   源码会清零 `targetPos` 和 `dvel`，但不会直接把实际 `vel` 清零。后续 `integrate()` 仍按 `maxAcceleration * dt` 向新速度收敛，所以 agent 可能继续滑行一小段。这是加速度连续性的结果；需要硬停必须在上层明确处理运动模式，而不能把 reset 当作瞬移式制动。

## 十三、设计取舍总结

1. **全局路径低频，局部速度高频。**

   A* 成本被摊薄，agent 每帧主要做局部 steering 和避障。

2. **corridor 是核心抽象。**

   它允许 agent 实际运动和全局路径不完全一致，同时持续保持路径有效。

3. **避障在速度空间求解。**

   不直接强推位置，而是选择“既接近期望速度又不容易碰撞”的速度。运动更平滑。

4. **仍保留位置级重叠修正。**

   速度避障不能保证完全无重叠，后处理保证视觉上不互穿。

5. **固定容量和分片搜索。**

   适合实时系统，但也意味着 crowd 有容量上限、path queue 上限和 corridor 长度上限。

6. **用户不能直接改 active agent 位置。**

   Crowd 拥有 `npos` 和 corridor。如果外部随意改位置，corridor、boundary、邻居缓存都会失效。需要瞬移时通常移除并重新添加 agent。
