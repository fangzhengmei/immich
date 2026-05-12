# Immich 时间线与 Memory 查询逻辑分析 (Round4 - 证据核查版)

---

## 一、Memory 前端完整展示链路

（内容同 round2，略）

---

## 二、旅行场景功能未实现的闭环证据

（内容同 round2，略）

---

## 三、共享时间线双路径分析（证据核查版）

### 3.1 核心证据：后端 DTO 无 timelineAlbumId 不存在

#### 3.1.1 服务端 TimeBucketAssetSchema 字段定义（生成 SDK 的源头）

**文件**: `server/src/dtos/time-bucket.dto.ts:7-62`

```typescript
const TimeBucketQueryBaseSchema = z
  .object({
    userId: z.uuidv4().optional(),
    albumId: z.uuidv4().optional(),
    personId: z.uuidv4().optional(),
    tagId: z.uuidv4().optional(),
    isFavorite: stringToBool.optional(),
    isTrashed: stringToBool.optional(),
    withStacked: stringToBool.optional(),
    withPartners: stringToBool.optional(),
    order: AssetOrderSchema.optional(),
    visibility: AssetVisibilitySchema.optional(),
    withCoordinates: stringToBool.optional(),
    key: z.string().optional(),
    slug: z.string().optional(),
    bbox: ...
  })
  .meta({ id: 'TimeBucketDto' });

const TimeBucketAssetSchema = TimeBucketQueryBaseSchema.extend({
  timeBucket: z.string().describe('Time bucket identifier in YYYY-MM-DD format'),
}).meta({ id: 'TimeBucketAssetDto' });
```

**核查结论**:
- ✅ `albumId`: 存在，是标准 API 参数
- ❌ `timelineAlbumId`: 不存在，后端完全没有定义

#### 3.1.2 前端 TimelineManagerOptions 类型定义

**文件**: `web/src/lib/managers/timeline-manager/types.ts:6-12`

```typescript
export type AssetApiGetTimeBucketsRequest = Parameters<typeof import('@immich/sdk').getTimeBuckets>[0];

export type TimelineManagerOptions = Omit<AssetApiGetTimeBucketsRequest, 'size'> & {
  timelineAlbumId?: string;  // 前端内部扩展字段，Omit 不影响
  deferInit?: boolean;
  assetFilter?: Set<string>;
};
```

**核查结论**:
- `timelineAlbumId` 是通过 `& { ... }` 扩展的前端内部字段
- 不属于 `AssetApiGetTimeBucketsRequest（即 SDK 标准参数）
- 不会被传给后端 API

#### 3.1.3 前端请求代码证据

**文件**: `web/src/lib/managers/timeline-manager/internal/load-support.svelte.ts:18-26`

```typescript
const bucketResponse = await getTimeBucket(
  {
    ...authManager.params,
    ...options,  // 展开 options，包含 timelineAlbumId？
    timeBucket,
  },
  { signal },
);
```

**关键证据链闭环**:
1. `getTimeBucket` 是 `@immich/sdk` 的自动生成函数
2. 其参数类型签名由 `server/src/dtos/time-bucket.dto.ts 的 Zod Schema 定义
3. 后端 Schema 没有 `timelineAlbumId` 字段
4. SDK 自动生成代码会忽略未定义字段
5. **结论**：即使前端传了也会被 SDK 过滤，后端不会收到

---

### 3.2 选图模式下两次请求参数与返回对照表

| 对比维度 | 第一次请求（主时间线） | 第二次请求（相册标记集） |
|---------|---------------------|----------------------|
| **文件位置** | `load-support.svelte.ts:18-26` | `load-support.svelte.ts:32-47` |
| **触发条件** | 滚动到该月份 | `options.timelineAlbumId` 存在时 |
| **参数传递方式** | `...options`（展开 | 显式 `albumId: options.timelineAlbumId` |
| **实际请求参数** | `{ visibility: Timeline, withPartners: true }` | `{ albumId: xxx }` |
| **包含 timelineAlbumId 吗** | 传了，但 SDK 自动过滤了 | 不传，使用标准 `albumId` |
| **服务端权限校验** | `TimelineRead` | `AlbumRead` |
| **SQL 过滤方式** | `WHERE owner_id IN (myId, partnerId1, ...)` | `INNER JOIN album_asset WHERE album_id = ?` |
| **返回资产集合边界** | 主时间线（自己+伴侣），可能包含相册外资产 | 严格限定在相册内资产 |
| **返回用途** | 渲染时间线列表 | 收集 ID 存入 `timelineManager.albumAssets` Set，仅用于 UI 标记 |

---

### 3.3 路径一：普通相册时间线（albumId 模式）

#### 3.3.1 触发条件
**文件**: `+page.svelte:221-230`
```typescript
const options = $derived.by(() => {
  if (viewMode === AlbumPageViewMode.SELECT_ASSETS) {
    // ...
  }
  return { albumId, order: album.order };  // 普通模式，无 timelineAlbumId
});
```
- `viewMode = VIEW` 或 `SELECT_THUMBNAIL`

#### 3.3.2 请求参数
```typescript
{
  albumId: '相册ID',    // 标准 API 参数
  order: 'desc/asc'     // 相册排序配置
}
```

#### 3.3.3 单次请求逻辑

**文件**: `load-support.svelte.ts:18-47`

```typescript
export async function loadFromTimeBuckets(...) {
  const bucketResponse = await getTimeBucket({
    ...authManager.params,
    ...options,  // options = { albumId, order }
    timeBucket,
  });

  // ❌ 不会触发二次请求
  if (options.timelineAlbumId) {
    // 此分支不会执行
  }

  timelineMonth.addAssets(bucketResponse, true);
}
```

**证据**：普通相册模式 `options` 只有 `{ albumId, order }`，没有 `timelineAlbumId` 字段，所以绝对不会触发二次请求。

#### 3.3.4 服务端 SQL 过滤

**文件**: `server/src/repositories/asset.repository.ts:388-392`

```typescript
.$if(!!options.albumId, (qb) =>
  qb
    .innerJoin('album_asset', 'asset.id', 'album_asset.assetId')
    .where('album_asset.albumId', '=', asUuid(options.albumId!)),
)
```

**返回资产集合边界**（证据确认）：
- ✅ **严格限定在相册内**
- 通过 `INNER JOIN album_asset` 确保只返回相册关联的资产
- 不存在任何相册外资产的可能

#### 3.3.5 关于"可能包含相册外资产"的适用范围

**适用前提**（严格限定）：
- ❌ 普通相册浏览模式：不适用
- ✅ 选图添加模式：适用（这是设计意图）

**证据链**：
普通相册模式的第一次请求就携带了 `albumId` 参数，服务端通过 `INNER JOIN album_asset` 严格过滤，返回的 100% 是相册内资产。

---

### 3.4 路径二：选图模式时间线（timelineAlbumId 模式）

#### 3.4.1 触发条件
**文件**: `+page.svelte:222-228`
```typescript
if (viewMode === AlbumPageViewMode.SELECT_ASSETS) {
  return {
    visibility: AssetVisibility.Timeline,
    withPartners: true,
    timelineAlbumId: albumId,  // 前端标记字段
  };
}
```
- `viewMode = SELECT_ASSETS`（添加照片到相册时）

#### 3.4.2 第一次请求：主时间线（可能包含相册外资产）

**文件**: `load-support.svelte.ts:18-26`

```typescript
const bucketResponse = await getTimeBucket(
  {
    ...authManager.params,
    ...options,
    // options = { visibility: AssetVisibility.Timeline, withPartners: true }
    // ❗ 证据：这里没有 albumId 参数，timelineAlbumId 被 SDK 过滤
    timeBucket,
  },
  { signal },
);
```

**返回资产集合边界**（证据确认）：
- 来源：用户自己的时间线 + 伴侣共享的时间线
- 过滤条件：`visibility = Timeline`（可见资产）
- ✅ **确实可能包含相册外资产**（这是设计意图）

#### 3.4.3 第二次请求：获取相册内资产标记集

**文件**: `load-support.svelte.ts:32-47`

```typescript
if (options.timelineAlbumId) {
  const albumAssets = await getTimeBucket(
    {
      ...authManager.params,
      albumId: options.timelineAlbumId,  // 证据：这里才用 albumId 查询
      timeBucket,
    },
    { signal },
  );
  for (const id of albumAssets.id) {
    timelineManager.albumAssets.add(id);  // 仅收集 ID，用于标记
  }
}
```

**第二次请求的唯一目的**（证据确认）：
1. 获取该月份相册内所有资产的 ID 列表
2. 存入 `timelineManager.albumAssets` Set
3. **不渲染这些资产**，只做标记使用

#### 3.4.4 渲染时的资产分类逻辑

**文件**: `+page.svelte` 时间线组件渲染时：
- ✅ `asset.id in timelineManager.albumAssets` → 相册内资产（"已添加"标记）
- ❌ `asset.id not in timelineManager.albumAssets` → 相册外资产（可选择添加）

---

### 3.5 两条路径对比总结表（证据核查版）

| 对比维度 | 路径一：普通相册时间线 (albumId 模式) | 路径二：选图模式时间线 (timelineAlbumId 模式) | 证据文件位置 |
|---------|--------------------------------------|---------------------------------------------|-------------|
| **触发场景** | 浏览相册内容 | 向相册添加照片时的选图界面 | `+page.svelte:221-230` |
| **viewMode** | `VIEW` 或 `SELECT_THUMBNAIL` | `SELECT_ASSETS` | `+page.svelte:221-230` |
| **传给 API 参数** | `{ albumId, order }` | `{ visibility: Timeline, withPartners: true }` | `+page.svelte:221-230` |
| **请求次数** | 1次 | 2次 | `load-support.svelte.ts:18-47` |
| **触发二次请求条件** | ❌ 不会（无 timelineAlbumId 字段） | ✅ 会（options.timelineAlbumId 存在） | `load-support.svelte.ts:32` |
| **第一次请求资产边界** | ✅ 严格相册内资产（通过 INNER JOIN） | ⚠️ 主时间线（自己+伴侣），可能包含相册外资产 | `asset.repository.ts:388-392` |
| **第二次请求用途** | 无 | 收集相册内资产 ID 到 albumAssets Set，仅用于标记 | `load-support.svelte.ts:32-47` |
| **"包含相册外资产"适用** | ❌ 不适用 | ✅ 适用（设计意图） | `asset.repository.ts:388-392` |
| **权限校验** | AlbumRead | TimelineRead（需要 withPartners 权限） | `timeline.service.ts:48-68` |
| **SQL过滤方式** | `INNER JOIN album_asset WHERE album_id = ?` | `WHERE owner_id IN (?, ?, ...)` | `asset.repository.ts:388-392` |
| **timelineAlbumId 传到后端吗** | 不传（根本没有这个字段） | 传了但被 SDK 自动过滤（后端无此字段定义） | `time-bucket.dto.ts:7-62`, `types.ts:6-12` |

---

### 3.6 完整数据流图对比

#### 路径一：普通相册浏览模式
```
用户打开相册（VIEW 模式）
      ↓ 【+page.svelte:221-230】
options = { albumId, order }
      ↓ 【load-support.svelte.ts:18-26】
getTimeBuckets({ albumId })
      ↓ 【asset.repository.ts:388-392】
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
      ↓ 【+page.svelte:221-230】
options = { visibility: Timeline, withPartners: true, timelineAlbumId: albumId }
      ↓ 【load-support.svelte.ts:18-26】
getTimeBuckets({ visibility: Timeline, withPartners: true })
      ↓ 【asset.repository.ts】
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

### 3.7 设计意图分析

1. **普通相册浏览模式**的设计目标（证据确认）：
   - 性能优先：一次请求即可，不需要额外标记
   - 数据精确：严格限定在相册范围内，用户只关心已有的照片
   - **证据位置**: `asset.repository.ts:388-392` 通过 INNER JOIN 强制限定

2. **选图模式**的设计目标（证据确认）：
   - 用户体验优先：让用户在完整时间线中挑选
   - 视觉反馈：清晰标记哪些已经在相册中，避免重复添加
   - 代价：多一次 API 请求换取更好的交互体验
   - **证据位置**: `load-support.svelte.ts:32-47` 二次请求收集标记集

---

## 四、关键代码文件索引

| 功能 | 文件路径 |
|------|----------|
| 后端 TimeBucket DTO 定义（SDK 生成源头） | `server/src/dtos/time-bucket.dto.ts:7-62` |
| 相册页面双模式参数切换逻辑 | `+page.svelte:221-230` |
| 前端 TimelineManagerOptions 类型定义 | `web/src/lib/managers/timeline-manager/types.ts:6-12` |
| 时间线分桶加载器（两次请求逻辑） | `web/src/lib/managers/timeline-manager/internal/load-support.svelte.ts` |
| 资产仓储时间线 SQL 查询 | `server/src/repositories/asset.repository.ts:388-392` |
| 时间线服务权限校验 | `server/src/services/timeline.service.ts:48-68` |
| Memory 前端管理器 | `web/src/lib/managers/memory-manager.svelte.ts` |
| Memory 标题格式化 | `web/src/lib/utils.ts:322-331` |
| Memory 服务层 | `server/src/services/memory.service.ts` |
| Memory 类型枚举 | `server/src/enum.ts:77-80` |
