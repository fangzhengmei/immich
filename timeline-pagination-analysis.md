# Immich 时间线查询与分页分析

## 1. 核心架构概览

Immich 的时间线系统采用**分层分页**架构：后端按月份进行时间分桶（Time Bucketing），前端通过虚拟滚动（Virtual Scrolling）按需加载月份数据。

```
客户端请求 → TimelineService → AssetRepository → PostgreSQL
     ↑                                            ↓
     └──────── 虚拟滚动管理器 ←────── 月份分桶数据 ──┘
```

---

## 2. 时间分桶 (Time Bucketing)

### 2.1 分桶原理

时间分桶是 Immich 时间线的基础单元，实现位于 `server/src/repositories/asset.repository.ts:709-761`。

```sql
date_trunc('MONTH', "localDateTime" AT TIME ZONE 'UTC') AT TIME ZONE 'UTC'
```

**关键特性：**
- **分桶粒度**：固定按月分桶（`MONTH`），无动态调整粒度选项
- **分桶字段**：由 `orderBy` 参数决定
  - `AssetOrderBy.TakenAt` → 使用 `localDateTime`（拍摄时间）
  - `AssetOrderBy.CreatedAt` → 使用 `asset.createdAt`（上传时间）
- **时区处理**：所有时间统一转换为 UTC 后进行分桶计算
- **分桶标识**：格式为 `YYYY-MM-DD`，表示该月第一天（如 `2024-01-01`）

### 2.2 分桶 API

两个端点都使用 **查询参数（Query Parameter）**，而非路径参数。

#### GET /timeline/buckets — 获取时间桶列表

**用途**：获取符合筛选条件的所有时间桶元数据（不含资产详情）。

**参数**（均为可选）：
- `userId` / `albumId` / `personId` / `tagId` — 范围筛选
- `isFavorite` / `isTrashed` / `visibility` — 状态筛选
- `orderBy` / `order` — 分桶排序方式
- `withStacked` / `withPartners` / `withCoordinates` — 内容控制
- `bbox` — 地理边界筛选

**返回值**：
```json
[{ "timeBucket": "2024-01-01", "count": 42 }, { "timeBucket": "2024-02-01", "count": 17 }]
```

#### GET /timeline/bucket — 获取单个桶的资产详情

**用途**：获取指定时间桶内的所有资产详情。

**参数**：
- **`timeBucket`（必需）** — 格式 `YYYY-MM-DD`，如 `2024-01-01`
- 上述所有筛选、排序参数（与 buckets 端点相同）

**返回值**：资产属性列式数组（id、ratio、thumbhash、localOffsetHours 等）

**控制器实现**（`timeline.controller.ts:26-37`）：
```typescript
@Get('bucket')
getTimeBucket(@Auth() auth: AuthDto, @Query() dto: TimeBucketAssetDto) {
  return this.service.getTimeBucket(auth, dto);
}
```

### 2.3 TimeBucket 参数特殊处理

**前导正负号清理**是一个关键的隐式行为，仅影响 `GET /timeline/bucket` 端点，实现于 `asset.repository.ts:820`：

```typescript
.where(truncatedDate(options.orderBy), '=', timeBucket.replace(/^[+-]/, ''))
```

**处理规则**：
- 正则表达式 `/^[+-]/` 会移除 `timeBucket` 字符串开头的 `+` 或 `-` 符号
- 例如：`"+2024-01-01"` → `"2024-01-01"`，`"-0001-01-01"` → `"0001-01-01"`
- 仅移除开头的单个正负号，后续字符不受影响

### 2.4 单桶匹配一致性与异常输入容错

正负号清理行为对单桶查询的影响：

1. **单桶匹配一致性**：
   - `getTimeBuckets` 返回的 `timeBucket` 字段格式为 `YYYY-MM-DD`，不带正负号
   - 前端正常使用返回值调用 `getTimeBucket` 时，不会触发符号清理
   - 因此常规分页流程不受影响，分桶列表与单桶查询结果保持一致

2. **异常输入容错**：
   - 意外带 `+` 号的参数（如 `"+2024-01-01"`）仍能正确匹配到 2024 年 1 月
   - 这提供了一定的输入容错性，但也隐藏了潜在的客户端格式错误

3. **公元前日期限制**：
   - 理论上 ISO 8601 格式中公元前年份以 `-` 开头（如 `-0001-01-01` 表示公元前 1 年）
   - 但由于前导 `-` 会被移除，`"-0001-01-01"` 会被当作 `"0001-01-01"`（公元 1 年）处理
   - **结论**：当前实现无法正确查询公元前的资产

### 2.5 分桶查询流程

```typescript
// server/src/services/timeline.service.ts:12-16
async getTimeBuckets(auth: AuthDto, dto: TimeBucketDto): Promise<TimeBucketsResponseDto[]> {
  await this.timeBucketChecks(auth, dto);
  const timeBucketOptions = await this.buildTimeBucketOptions(auth, dto);
  return await this.assetRepository.getTimeBuckets(timeBucketOptions);
}
```

---

## 3. 分页游标 (Pagination Cursor)

### 3.1 当前分页机制

**重要发现**：Immich 当前采用**基于时间桶的粗粒度分页**，而非传统的游标分页。

```typescript
// server/src/services/timeline.service.ts:23
// TODO: use id cursor for pagination
```

代码中明确标记了待实现项，说明 ID 游标分页是未来规划。当前实现：

1. **后端**：每次返回**完整月份**的所有资产数据，无服务器端游标
2. **前端**：通过 `VirtualScrollManager` 实现客户端虚拟滚动
3. **加载策略**：
   - 初始加载：获取所有时间桶列表（仅元数据，不含资产详情）
   - 按需加载：当月份进入视口附近时，调用 `getTimeBucket` 获取该月资产

### 3.2 前端虚拟滚动实现

位于 `web/src/lib/managers/timeline-manager/`：

```typescript
// timeline-manager.svelte.ts:242-261
async #initializeTimelineMonths() {
  const timebuckets = await getTimeBuckets({...authManager.params, ...this.#options});
  this.months = timebuckets.map((timeBucket) => {
    const date = new SvelteDate(timeBucket.timeBucket);
    return new TimelineMonth(this, { year, month }, timeBucket.count, ...);
  });
}
```

**视口邻近度加载**：
```typescript
// timeline-month.svelte.ts:87-98
set viewportProximity(newValue: ViewportProximity) {
  if (isInOrNearViewportUtil(newValue)) {
    void this.timelineManager.loadTimelineMonth(this.yearMonth);
  } else {
    this.cancel();
  }
}
```

---

## 4. 媒体排序 (Sorting)

### 4.1 排序维度

排序由两个参数共同控制：`orderBy`（排序字段）和 `order`（排序方向）。

#### 排序字段 (`orderBy`)

| 值 | 字段 | 说明 |
|----|------|------|
| `takenAt` | `localDateTime` | 按拍摄时间排序（默认） |
| `createdAt` | `asset.createdAt` | 按上传时间排序 |

#### 排序方向 (`order`)

| 值 | 说明 |
|----|------|
| `desc` | 降序，最新/最大在前（默认） |
| `asc` | 升序，最旧/最小在前 |

### 4.2 排序实现

后端 SQL 排序逻辑（`asset.repository.ts:866-872`）：

```typescript
.orderBy(
  options.orderBy == AssetOrderBy.CreatedAt
    ? sql`"createdAt"`
    : sql`(asset."localDateTime" AT TIME ZONE 'UTC')::date`,
  order,
)
.orderBy('asset.fileCreatedAt', order)
```

**两级排序策略**：
1. **第一级**：按日期（`orderBy` 指定的字段）
2. **第二级**：按文件创建时间（`fileCreatedAt`）打破日期相同的平局

### 4.3 前端排序

前端在接收数据后也会进行二次排序：

```typescript
// timeline-month.svelte.ts:124-130
sortTimelineDays() {
  if (this.#sortOrder === AssetOrder.Asc) {
    return this.timelineDays.sort((a, b) => a.day - b.day);
  }
  return this.timelineDays.sort((a, b) => b.day - a.day);
}
```

---

## 5. 筛选条件 (Filtering)

### 5.1 可用筛选参数

定义于 `server/src/dtos/time-bucket.dto.ts:7-64`

| 参数 | 类型 | 说明 |
|------|------|------|
| `userId` | UUID | 按用户筛选 |
| `albumId` | UUID | 按相册筛选 |
| `personId` | UUID | 按人物（人脸识别）筛选 |
| `tagId` | UUID | 按标签筛选 |
| `isFavorite` | boolean | 收藏状态筛选 |
| `isTrashed` | boolean | 回收站状态筛选 |
| `visibility` | enum | 可见性：`ARCHIVE`/`TIMELINE`/`HIDDEN`/`LOCKED` |
| `withStacked` | boolean | 是否包含堆叠资产 |
| `withPartners` | boolean | 是否包含合作伙伴资产 |
| `withCoordinates` | boolean | 是否返回坐标数据 |
| `bbox` | string | 边界框：`west,south,east,north` |

### 5.2 筛选条件组合逻辑

所有筛选条件使用 **AND** 逻辑组合，构建于 `asset.repository.ts:710-761`：

```typescript
qb
  .where('asset.deletedAt', options.isTrashed ? 'is not' : 'is', null)
  .$if(!!options.visibility, qb => qb.where('asset.visibility', '=', options.visibility!))
  .$if(!!options.albumId, qb => qb.innerJoin('album_asset', ...))
  .$if(!!options.personId, qb => hasPeople(qb, [options.personId!]))
  .$if(!!options.userIds, qb => qb.where('asset.ownerId', '=', anyUuid(options.userIds!)))
  .$if(options.isFavorite !== undefined, qb => qb.where('asset.isFavorite', '=', options.isFavorite!))
  .$if(!!options.tagId, qb => withTagId(qb, options.tagId!))
```

### 5.3 特殊筛选约束

**合作伙伴资产限制**（`timeline.service.ts:69-79`）：
- `withPartners` 不能与 `isFavorite`、`isTrashed`、`visibility=ARCHIVE` 同时使用
- 否则抛出 `BadRequestException`

**权限检查**：
- 访问 `LOCKED` 资产需要 elevated permission
- 访问归档资产需要 `ArchiveRead` 权限
- 访问相册需要 `AlbumRead` 权限

---

## 6. 各因素协同作用分析

### 6.1 完整查询流程

```
用户浏览时间线
    ↓
1. 调用 getTimeBuckets(筛选条件 + 排序参数)
    ↓
   后端：
   - 应用所有 WHERE 筛选
   - 按 orderBy 字段进行月份分桶
   - 按 order 方向排序分桶
   - 返回分桶列表（含每个桶的资产数）
    ↓
2. 前端创建 TimelineMonth 对象（仅元数据）
    ↓
3. 用户滚动，月份进入视口邻近区域
    ↓
4. 调用 getTimeBucket(具体月份 + 相同筛选排序参数)
    ↓
   后端：
   - 应用相同的 WHERE 筛选
   - 按 timeBucket 过滤特定月份
   - 按 (日期, fileCreatedAt) 两级排序
   - 聚合为列式数组返回
    ↓
5. 前端解析资产，按天分组，计算布局
    ↓
6. 渲染到虚拟滚动容器
```

### 6.2 因素交互矩阵

| 因素 | 影响分桶 | 影响排序 | 影响筛选 | 说明 |
|------|:--------:|:--------:|:--------:|------|
| `orderBy` | ✅ | ✅ | ❌ | 决定分桶字段和排序的第一关键字 |
| `order` | ✅ | ✅ | ❌ | 决定分桶列表和桶内资产的排序方向 |
| `userId` | ✅ | ❌ | ✅ | 限定分桶的资产范围 |
| `albumId` | ✅ | ❌ | ✅ | 仅返回相册内资产的分桶 |
| `personId` | ✅ | ❌ | ✅ | 仅返回含指定人物的分桶 |
| `tagId` | ✅ | ❌ | ✅ | 仅返回含指定标签的分桶 |
| `isFavorite` | ✅ | ❌ | ✅ | 仅返回收藏/非收藏资产 |
| `isTrashed` | ✅ | ❌ | ✅ | 仅返回回收站/非回收站资产 |
| `visibility` | ✅ | ❌ | ✅ | 按可见性过滤 |
| `withStacked` | ✅ | ❌ | ✅ | 控制堆叠资产的显示 |
| `bbox` | ✅ | ❌ | ✅ | 按地理边界过滤 |

### 6.3 典型场景示例

#### 场景 1：默认时间线（最新在前）
```
orderBy = takenAt, order = desc
→ 分桶按拍摄时间倒序：2024-05, 2024-04, 2024-03...
→ 桶内按 (拍摄日期 desc, fileCreatedAt desc) 排序
```

#### 场景 2：按上传时间升序查看
```
orderBy = createdAt, order = asc
→ 分桶按上传时间升序：2023-01, 2023-02, 2023-03...
→ 桶内按 (上传日期 asc, fileCreatedAt asc) 排序
```

#### 场景 3：仅查看收藏的特定人物照片
```
personId = xxx, isFavorite = true
→ 仅返回包含该人物且被收藏的资产
→ 分桶可能稀疏（某些月份可能为 0）
```

---

## 7. 性能与限制分析

### 7.1 优势
1. **分桶元数据轻量**：`getTimeBuckets` 仅返回 `(timeBucket, count)`，数据量小
2. **按需加载**：仅加载视口附近的月份数据，内存占用可控
3. **缓存友好**：月份数据一旦加载可长期缓存

### 7.2 局限性
1. **粗粒度分页**：大月份可能返回数千条资产，单次响应体积大
2. **无游标分页**：无法实现细粒度的"加载更多"，TODO 标记说明了这一缺陷
3. **排序开销**：每次查询都需要对全量匹配结果进行排序和分桶
4. **筛选组合爆炸**：复杂筛选条件下，SQL 查询性能可能下降

### 7.3 优化建议（基于代码 TODO）

**实现 ID 游标分页**：
```typescript
// 潜在实现方向
interface Cursor {
  lastId: string;
  lastDate: string;
  bucket: string;
}

async getTimeBucket(timeBucket: string, options: TimeBucketOptions, cursor?: Cursor) {
  // 使用 cursor 进行分页查询
  // SELECT ... WHERE (date, id) < (cursor.lastDate, cursor.lastId) LIMIT pageSize
}
```

---

## 8. 关键代码位置速查表

| 功能 | 文件 | 行号 |
|------|------|------|
| 时间分桶查询 | `server/src/repositories/asset.repository.ts` | 709-761 |
| 单桶资产查询 | `server/src/repositories/asset.repository.ts` | 766-911 |
| 时间分桶 DTO | `server/src/dtos/time-bucket.dto.ts` | 1-136 |
| 时间线控制器 | `server/src/controllers/timeline.controller.ts` | 1-38 |
| 时间线服务 | `server/src/services/timeline.service.ts` | 1-81 |
| timeBucket 符号清理 | `server/src/repositories/asset.repository.ts` | 820 |
| 前端时间线管理器 | `web/src/lib/managers/timeline-manager/timeline-manager.svelte.ts` | 1-644 |
| 月份分桶类 | `web/src/lib/managers/timeline-manager/timeline-month.svelte.ts` | 1-394 |
| 分桶日期截断 | `server/src/utils/database.ts` | 301-303 |
