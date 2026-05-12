# Immich 时间线与 Memory 查询逻辑分析 (Round3)

---

## 一、Memory 前端完整展示链路

（内容同 round2，略）

---

## 二、旅行场景功能未实现的闭环证据

（内容同 round2，略）

---

## 三、共享时间线的两条独立路径分析

### 3.1 关键代码证据：双模式参数定义

**文件**: `web/src/routes/(user)/albums/[albumId=id]/[[photos=photos]]/[[assetId=id]]/+page.svelte:221-230`

```typescript
const options = $derived.by(() => {
  if (viewMode === AlbumPageViewMode.SELECT_ASSETS) {
    // 选图模式时间线参数
    return {
      visibility: AssetVisibility.Timeline,
      withPartners: true,
      timelineAlbumId: albumId,  // 前端标记字段
    };
  }
  // 普通相册浏览模式参数
  return { albumId, order: album.order };
});
```

**核心发现**：相册页面存在两条完全不同的时间线路径，由 `viewMode` 控制。

### 3.2 路径一：普通相册时间线（albumId 模式）

#### 3.2.1 触发条件
`viewMode === AlbumPageViewMode.VIEW`（默认状态）

#### 3.2.2 请求参数
传递给 `getTimeBuckets` 和 `getTimeBucket` 的参数：
```typescript
{
  albumId: '相册ID',    // 后端 API 标准参数
  order: 'desc/asc'     // 相册排序配置
}
```

#### 3.2.3 单次请求逻辑

**文件**: `web/src/lib/managers/timeline-manager/internal/load-support.svelte.ts:8-47`

```typescript
export async function loadFromTimeBuckets(
  timelineManager: TimelineManager,
  timelineMonth: TimelineMonth,
  options: TimelineManagerOptions,
  signal: AbortSignal,
): Promise<void> {
  // 第1次（也是唯一一次）请求
  const timeBucket = toISOYearMonthUTC(timelineMonth.yearMonth);
  const bucketResponse = await getTimeBucket(
    {
      ...authManager.params,
      ...options,  // options = { albumId, order }
      timeBucket,
    },
    { signal },
  );

  // ❌ 不会触发二次请求：options.timelineAlbumId 为 undefined
  if (options.timelineAlbumId) {
    // 此分支不会执行
  }

  timelineMonth.addAssets(bucketResponse, true);
}
```

**二次请求判断依据**（代码第32行）：
```typescript
if (options.timelineAlbumId) {
  // 只有 timelineAlbumId 存在时才发起第二次请求
}
```

由于普通相册模式 `options` 只有 `{ albumId, order }`，**没有 `timelineAlbumId` 字段**，所以绝对不会触发二次请求。

#### 3.2.4 服务端 SQL 过滤

**文件**: `server/src/repositories/asset.repository.ts:388-392`

```typescript
.$if(!!options.albumId, (qb) =>
  qb
    .innerJoin('album_asset', 'asset.id', 'album_asset.assetId')
    .where('album_asset.albumId', '=', asUuid(options.albumId!)),
)
```

**返回资产集合边界**：
- ✅ **严格限定在相册内**
- 通过 `INNER JOIN album_asset` 确保只返回相册关联的资产
- 不存在任何相册外资产的可能

#### 3.2.5 关于"可能包含相册外资产"的说明修正

**旧版本说法**（错误）："所有资产（可能包含相册外资产）"

**正确适用前提**：这句话**只适用于选图模式**，不适用于普通相册浏览模式。

**代码证据**：
普通相册模式的第一次请求就携带了 `albumId` 参数，服务端通过 `INNER JOIN album_asset` 严格过滤，返回的 100% 是相册内资产，不存在任何相册外资产的可能。

---

### 3.3 路径二：选图模式时间线（timelineAlbumId 模式）

#### 3.3.1 触发条件
`viewMode === AlbumPageViewMode.SELECT_ASSETS`（添加照片到相册时）

#### 3.3.2 前端扩展字段定义

**文件**: `web/src/lib/managers/timeline-manager/types.ts:8-12`

```typescript
export type TimelineManagerOptions = Omit<AssetApiGetTimeBucketsRequest, 'size'> & {
  timelineAlbumId?: string;  // 前端内部标记字段，不会传给 API
  deferInit?: boolean;
  assetFilter?: Set<string>;
};
```

**关键**：`timelineAlbumId` 是前端的内部扩展字段，**不会作为 API 参数发送给后端**。

#### 3.3.3 第一次请求：主时间线（可能包含相册外资产）

**请求参数**（第19-26行）：
```typescript
const bucketResponse = await getTimeBucket(
  {
    ...authManager.params,
    ...options,
    // options = { visibility: AssetVisibility.Timeline, withPartners: true }
    // ❗ 注意：这里没有 albumId 参数！
    timeBucket,
  },
  { signal },
);
```

**返回资产集合边界**：
- 来源：用户自己的时间线 + 伴侣共享的时间线
- 过滤条件：`visibility = Timeline`（可见资产）
- ✅ **确实可能包含相册外资产**（这是设计意图：让用户可以选择添加更多照片到相册）

#### 3.3.4 第二次请求：获取相册内资产标记集

**触发条件**（第32行）：`options.timelineAlbumId` 存在时

```typescript
if (options.timelineAlbumId) {
  const albumAssets = await getTimeBucket(
    {
      ...authManager.params,
      albumId: options.timelineAlbumId,  // 这里才用 albumId 查询
      timeBucket,
    },
    { signal },
  );
  for (const id of albumAssets.id) {
    timelineManager.albumAssets.add(id);  // 仅收集 ID，用于标记
  }
}
```

**第二次请求的唯一目的**：
1. 获取该月份相册内所有资产的 ID 列表
2. 存入 `timelineManager.albumAssets` Set
3. **不渲染这些资产**，只做标记使用

#### 3.3.5 渲染时的资产分类逻辑

在 UI 渲染时，根据 `asset.id in timelineManager.albumAssets` 判断：
- ✅ `true`：相册内资产（可能显示"已添加"标记或禁用选择）
- ❌ `false`：相册外资产（可选择添加到相册）

**选图模式的设计意图**：
显示用户的完整时间线（自己+伴侣），让用户可以看到哪些照片已经在相册中，哪些还可以添加。

---

### 3.4 两条路径对比总结表

| 对比维度 | 路径一：普通相册时间线 (albumId 模式) | 路径二：选图模式时间线 (timelineAlbumId 模式) |
|---------|--------------------------------------|---------------------------------------------|
| **触发场景** | 浏览相册内容 | 向相册添加照片时的选图界面 |
| **viewMode** | `VIEW` 或 `SELECT_THUMBNAIL` | `SELECT_ASSETS` |
| **传给 API 参数** | `{ albumId, order }` | `{ visibility: Timeline, withPartners: true }` |
| **请求次数** | 1次 | 2次 |
| **触发二次请求条件** | ❌ 不会（无 timelineAlbumId 字段） | ✅ 会（options.timelineAlbumId 存在） |
| **第一次请求资产边界** | ✅ 严格相册内资产（通过 INNER JOIN） | ⚠️ 主时间线（自己+伴侣），可能包含相册外资产 |
| **第二次请求用途** | 无 | 收集相册内资产 ID 到 albumAssets Set，仅用于标记 |
| **"包含相册外资产"适用** | ❌ 不适用 | ✅ 适用（设计意图） |
| **权限校验** | AlbumRead | TimelineRead（需要 withPartners 权限） |
| **SQL过滤方式** | `INNER JOIN album_asset WHERE album_id = ?` | `WHERE owner_id IN (?, ?, ...)` |

---

### 3.5 完整数据流图对比

#### 路径一：普通相册浏览模式
```
用户打开相册（VIEW 模式）
      ↓
options = { albumId, order }
      ↓
getTimeBuckets({ albumId })
      ↓
服务端：INNER JOIN album_asset
      ↓
返回：仅相册内资产的月份分桶
      ↓
滚动加载月份
      ↓
getTimeBucket({ albumId, timeBucket })
      ↓
服务端：INNER JOIN album_asset
      ↓
返回：该月份相册内的所有资产
      ↓
渲染时间线（100% 相册内资产）
      ↓
✅ 无二次请求（timelineAlbumId 不存在）
```

#### 路径二：选图添加模式
```
用户点击"添加照片"（SELECT_ASSETS 模式）
      ↓
options = { visibility: Timeline, withPartners: true, timelineAlbumId: albumId }
      ↓
getTimeBuckets({ visibility: Timeline, withPartners: true })
      ↓
服务端：WHERE owner_id IN (myId, partnerId1, ...)
      ↓
返回：主时间线的月份分桶（可能包含相册外）
      ↓
滚动加载月份
      ↓
【第1次请求】getTimeBucket({ visibility: Timeline, withPartners: true, timeBucket })
      ↓
返回：该月份的主时间线资产（可能包含相册外）
      ↓
【第2次请求】因 timelineAlbumId 存在，触发 getTimeBucket({ albumId, timeBucket })
      ↓
返回：该月份相册内资产（仅收集 ID）
      ↓
存入 timelineManager.albumAssets Set
      ↓
渲染时判断：asset.id in albumAssets ?
      ↓
├─ true → "已在相册中"
└─ false → 可选择添加
```

---

### 3.6 设计意图分析

1. **普通相册浏览模式**的设计目标：
   - 性能优先：一次请求即可，不需要额外标记
   - 数据精确：严格限定在相册范围内，用户只关心已有的照片

2. **选图模式**的设计目标：
   - 用户体验优先：让用户在完整时间线中挑选
   - 视觉反馈：清晰标记哪些已经在相册中，避免重复添加
   - 代价：多一次 API 请求换取更好的交互体验

---

## 四、关键代码文件索引

| 功能 | 文件路径 |
|------|----------|
| 相册页面双模式参数逻辑 | `web/src/routes/(user)/albums/[albumId=id]/[[photos=photos]]\[[assetId=id]]/+page.svelte:221-230` |
| 时间线分桶加载器 | `web/src/lib/managers/timeline-manager/internal/load-support.svelte.ts` |
| TimelineManagerOptions 类型定义 | `web/src/lib/managers/timeline-manager/types.ts:8-12` |
| 资产仓储时间线查询 | `server/src/repositories/asset.repository.ts:388-392` |
| 时间线服务 | `server/src/services/timeline.service.ts` |
