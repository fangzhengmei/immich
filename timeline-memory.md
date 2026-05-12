# Immich 时间线与 Memory 查询逻辑分析

## 一、时间线按月分桶懒加载查询逻辑

### 1.1 核心设计思路
时间线采用"分桶+懒加载"的二级查询策略，避免一次性加载大量数据：
- **第一级**：获取所有月份分桶的元数据（仅包含时间范围和资产数量）
- **第二级**：用户滚动到对应月份时，按需加载该月份的详细资产数据

### 1.2 服务端查询实现

#### 1.2.1 分桶列表查询 (`getTimeBuckets`)
**文件**：`server/src/services/timeline.service.ts:12-16`

**查询参数**：
- `userId`: 用户ID过滤
- `albumId`: 相册ID过滤
- `personId`: 人物ID过滤（人脸识别）
- `tagId`: 标签ID过滤
- `isFavorite`: 收藏状态过滤
- `isTrashed`: 回收站状态过滤
- `withStacked`: 是否包含堆叠资产
- `withPartners`: 是否包含伴侣共享资产
- `order`: 排序顺序（ASC/DESC）
- `visibility`: 可见性状态（ARCHIVE/TIMELINE/HIDDEN/LOCKED）
- `bbox`: 地理位置边界框过滤

**SQL查询逻辑**（`asset.repository.ts:709-760`）：
1. 按日期截断（`date_trunc('month', localDateTime)`）生成分桶
2. 按分桶分组统计每个月的资产数量
3. 支持多种过滤条件组合
4. 返回结果格式：`{ timeBucket: "YYYY-MM-DD", count: number }`

#### 1.2.2 单桶详细查询 (`getTimeBucket`)
**文件**：`server/src/services/timeline.service.ts:19-26`

**查询参数**：
- 所有分桶列表的参数
- `timeBucket`: 目标月份（格式：`YYYY-MM-DD`）

**返回数据结构**（`time-bucket.dto.ts:72-115`）：
```typescript
{
  id: string[],                    // 资产ID数组
  ownerId: string[],               // 所有者ID数组
  ratio: number[],                 // 宽高比数组
  isFavorite: boolean[],           // 收藏状态
  visibility: AssetVisibility[],   // 可见性状态
  isTrashed: boolean[],            // 回收站状态
  isImage: boolean[],              // 是否为图片
  thumbhash: string[],             // 缩略图hash
  fileCreatedAt: string[],         // 文件创建时间
  localOffsetHours: number[],      // 本地时区偏移
  duration: number[],              // 视频时长
  stack: [stackId, count][],       // 堆叠信息
  city: string[],                  // 城市
  country: string[],               // 国家
  latitude?: number[],             // 纬度
  longitude?: number[],            // 经度
}
```

**SQL优化策略**（`asset.repository.ts:765-903`）：
- 使用CTE（Common Table Expression）进行高效查询
- 对JOIN进行优化，仅在需要时关联表
- 按日期和创建时间双重排序
- 结果聚合为数组格式，减少网络传输

### 1.3 前端懒加载实现

#### 1.3.1 TimelineManager 管理器
**文件**：`web/src/lib/managers/timeline-manager/timeline-manager.svelte.ts`

**懒加载触发时机**：
- 初始化时仅加载分桶列表（`#initializeTimelineMonths`, 第241-258行）
- 滚动视口接近对应月份时触发加载（`updateViewportProximities`, 第204-229行）
- 加载函数：`loadTimelineMonth(yearMonth, options)`（第344-365行）

#### 1.3.2 分桶加载器
**文件**：`web/src/lib/managers/timeline-manager/internal/load-support.svelte.ts`

**加载流程**：
1. 检查月份是否已有资产，有则跳过
2. 调用API获取对应月份分桶的资产数据
3. 将资产添加到对应月份的时间线中
4. 支持AbortSignal取消请求，避免不必要的网络开销

**关键代码**（第8-60行）：
```typescript
export async function loadFromTimeBuckets(
  timelineManager: TimelineManager,
  timelineMonth: TimelineMonth,
  options: TimelineManagerOptions,
  signal: AbortSignal,
): Promise<void> {
  if (timelineMonth.getFirstAsset()) {
    return;
  }

  const timeBucket = toISOYearMonthUTC(timelineMonth.yearMonth);
  const bucketResponse = await getTimeBucket(
    {
      ...authManager.params,
      ...options,
      timeBucket,
    },
    { signal },
  );
  // ...
}
```

---

## 二、Memory 归类规则

### 2.1 Memory 类型定义
**文件**：`server/src/enum.ts:77-80`

目前系统仅实现了一种 Memory 类型：

```typescript
export enum MemoryType {
  /** pictures taken on this day X years ago */
  OnThisDay = 'on_this_day',
}
```

> **注**：用户提到的"旅行场景"归类在当前代码库中**尚未实现**，可能是未来规划功能或特定分支功能。

### 2.2 "往年今日" (OnThisDay) 生成规则

#### 2.2.1 定时任务触发
**文件**：`server/src/services/memory.service.ts:16-45`

**触发机制**：
- 由后台定时任务 `JobName.MemoryGenerate` 触发
- 生成范围：今天 ±3天（`DAYS = 3`），共7天
- 使用分布式锁 `DatabaseLock.MemoryCreation` 防止重复生成
- 通过 `lastOnThisDayDate` 记录已处理日期，避免重复创建

**生成循环逻辑**（第25-43行）：
```typescript
// generate a memory +/- X days from today
for (let i = 0; i <= DAYS * 2; i++) {
  const target = start.plus({ days: i });
  if (lastOnThisDayDate >= target) {
    continue;
  }
  // 创建该日期的往年今日记忆
  await Promise.all(users.map((owner) => this.createOnThisDayMemories(owner.id, target)));
}
```

#### 2.2.2 记忆创建逻辑
**文件**：`server/src/services/memory.service.ts:47-66`

**按年分桶规则**：
1. 查询指定日期（月+日）的所有历史年份照片
2. **按年份分组**，每一年独立创建一个 Memory
3. 每个 Memory 包含该年当天的所有照片（最多20张）
4. 设置显示时间窗口：当天 00:00 到 23:59

**关键代码**（第47-66行）：
```typescript
private async createOnThisDayMemories(ownerId: string, target: DateTime) {
  const showAt = target.startOf('day').toISO();
  const hideAt = target.endOf('day').toISO();
  const memories = await this.assetRepository.getByDayOfYear([ownerId], target);
  
  await Promise.all(
    memories.map(({ year, assets }) =>
      this.memoryRepository.create(
        {
          ownerId,
          type: MemoryType.OnThisDay,
          data: { year },
          memoryAt: target.set({ year }).toISO()!,
          showAt,
          hideAt,
        },
        new Set(assets.map(({ id }) => id)),
      ),
    ),
  });
}
```

#### 2.2.3 按日查询SQL实现
**文件**：`server/src/repositories/asset.repository.ts:446-495`

**查询策略**：
1. 使用 `generate_series` 生成年份序列（从最早照片年份到去年）
2. 对每个年份，构造完整日期（月日与今天相同，年份为历史年份）
3. 查询该日期的所有照片
4. 每个年份创建一条记忆记录

### 2.3 Memory 数据结构

#### 2.3.1 数据库表结构
**文件**：`server/src/schema/tables/memory.table.ts`

| 字段 | 类型 | 说明 |
|------|------|------|
| id | uuid | Memory唯一ID |
| ownerId | uuid | 所有者用户ID |
| createdAt | timestamp | 创建时间 |
| updatedAt | timestamp | 更新时间 |
| deletedAt | timestamp | 删除时间（软删除） |
| type | varchar | Memory类型（OnThisDay） |
| data | jsonb | 附加数据（如年份） |
| isSaved | boolean | 是否被用户保存 |
| memoryAt | timestamp | 记忆发生的日期 |
| seenAt | timestamp | 用户查看时间 |
| showAt | timestamp | 开始显示时间 |

#### 2.3.2 关联表：memory_asset
- 多对多关系表，关联 Memory 和 Asset
- 支持批量添加/移除照片

### 2.4 Memory 生命周期管理

#### 2.4.1 保存机制
- 用户可手动保存记忆（`isSaved = true`）
- 未保存的记忆：30天后自动清理
- 已保存的记忆：永久保留，直到用户手动删除

#### 2.4.2 清理任务
**文件**：`server/src/services/memory.service.ts:68-71`

- 由 `JobName.MemoryCleanup` 定时任务触发
- 清理30天前未保存的记忆记录
- 同时清理关联的 memory_asset 记录

#### 2.4.3 Memory 搜索/查询
**文件**：`server/src/repositories/memory.repository.ts:32-94`

支持的过滤条件：
- `isSaved`: 是否保存
- `type`: Memory类型
- `for`: 日期范围过滤（只显示当前应该看到的记忆）
- 排序：按 memoryAt 降序或随机排序
- 分页：支持 limit 限制返回数量

---

## 三、共享时间线裁剪逻辑

### 3.1 伴侣共享时间线

#### 3.1.1 withPartners 参数
**文件**：`server/src/services/timeline.service.ts:32-41`

**使用限制**（第74-79行）：
```typescript
if (requestedArchived || requestedFavorite || requestedTrash) {
  throw new BadRequestException(
    'withPartners is only supported for non-archived, non-trashed, non-favorited assets',
  );
}
```

**说明**：
- `withPartners = true` 时，不能同时查询归档、收藏或回收站资产
- 仅适用于主时间线（TIMELINE可见性）的查询

#### 3.1.2 用户ID列表构建
**文件**：`server/src/services/timeline.service.ts:28-45`

```typescript
private async buildTimeBucketOptions(auth: AuthDto, dto: TimeBucketDto): Promise<TimeBucketOptions> {
  const { userId, ...options } = dto;
  let userIds: string[] | undefined = undefined;

  if (userId) {
    userIds = [userId];
    if (dto.withPartners) {
      const partnerIds = await getMyPartnerIds({
        userId: auth.user.id,
        repository: this.partnerRepository,
        timelineEnabled: true,
      });
      userIds.push(...partnerIds);
    }
  }

  return { ...options, userIds };
}
```

### 3.2 相册共享时间线

#### 3.2.1 timelineAlbumId 参数
**文件**：`web/src/lib/managers/timeline-manager/types.ts`

**前端加载逻辑**（`load-support.svelte.ts:32-47`）：
```typescript
if (options.timelineAlbumId) {
  const albumAssets = await getTimeBucket(
    {
      ...authManager.params,
      albumId: options.timelineAlbumId,
      timeBucket,
    },
    { signal },
  );
  for (const id of albumAssets.id) {
    timelineManager.albumAssets.add(id);
  }
}
```

#### 3.2.2 共享链接访问权限
**文件**：`server/src/controllers/timeline.controller.ts:15-37`

时间线API支持共享链接访问：
```typescript
@Authenticated({ permission: Permission.AssetRead, sharedLink: true })
```

这意味着：
- 拥有相册共享链接的用户可以查看该相册的时间线
- 权限检查由中间件自动处理
- 只能访问共享范围内的资产

### 3.3 权限裁剪机制

#### 3.3.1 服务端权限检查
**文件**：`server/src/services/timeline.service.ts:47-80`

每个时间线查询都会经过以下权限检查：
1. **用户可见性权限**：如果指定 userId，检查是否有 TimelineRead 权限
2. **归档权限**：如果查询归档，检查是否有 ArchiveRead 权限
3. **相册权限**：如果查询相册，检查是否有 AlbumRead 权限
4. **锁定相册权限**：如果查询锁定资产，需要 elevated permission

#### 3.3.2 资产可见性过滤
在 assetRepository 的 SQL 查询中会自动应用：
- `visibility = TIMELINE`（默认）
- 根据请求参数调整过滤条件
- 软删除的资产自动过滤

---

## 四、核心 API 接口汇总

### 4.1 时间线 API

| 接口 | 方法 | 路径 | 说明 |
|------|------|------|------|
| 获取分桶列表 | GET | `/timeline/buckets` | 获取所有月份分桶的元数据 |
| 获取单桶详情 | GET | `/timeline/bucket` | 获取指定月份的所有资产数据 |

### 4.2 Memory API

| 接口 | 方法 | 路径 | 说明 |
|------|------|------|------|
| 搜索记忆 | GET | `/memory` | 按条件搜索记忆列表 |
| 获取统计 | GET | `/memory/statistics` | 获取记忆数量统计 |
| 获取详情 | GET | `/memory/:id` | 获取单个记忆详情 |
| 创建记忆 | POST | `/memory` | 手动创建记忆 |
| 更新记忆 | PATCH | `/memory/:id` | 更新记忆状态（保存/已看） |
| 删除记忆 | DELETE | `/memory/:id` | 删除记忆 |
| 添加资产 | PUT | `/memory/:id/assets` | 向记忆添加照片 |
| 移除资产 | DELETE | `/memory/:id/assets` | 从记忆移除照片 |

---

## 五、关键文件索引

| 功能模块 | 主要文件路径 |
|----------|--------------|
| 时间线服务 | `server/src/services/timeline.service.ts` |
| 时间线控制器 | `server/src/controllers/timeline.controller.ts` |
| 资产仓储 | `server/src/repositories/asset.repository.ts` |
| Memory服务 | `server/src/services/memory.service.ts` |
| Memory仓储 | `server/src/repositories/memory.repository.ts` |
| 时间线管理器 | `web/src/lib/managers/timeline-manager/timeline-manager.svelte.ts` |
| 分桶加载器 | `web/src/lib/managers/timeline-manager/internal/load-support.svelte.ts` |
| DTO定义 | `server/src/dtos/time-bucket.dto.ts` |
| Memory DTO | `server/src/dtos/memory.dto.ts` |
| 枚举定义 | `server/src/enum.ts` |

---

## 六、补充说明

### 6.1 关于"旅行场景"归类
根据当前代码库的全面分析，**旅行场景（按地理位置聚类的记忆）功能尚未在主线代码中实现**。代码中：
- MemoryType 枚举仅定义了 `OnThisDay` 一种类型
- 没有找到按城市/地理位置自动生成记忆的逻辑
- 没有旅行相关的定时任务或服务

这可能是：
1. 未来版本的规划功能
2. 特定分支中的实验性功能
3. 用户期望的功能而非现有功能

### 6.2 性能优化要点
1. **分桶查询**：按月分桶大大减少了单次查询的数据量
2. **懒加载**：按需加载月份数据，降低初始加载时间
3. **数组聚合**：SQL层面直接聚合为数组格式，减少ORM开销
4. **缓存策略**：前端记忆管理器可能对记忆列表进行缓存
5. **AbortSignal**：滚动时自动取消未完成的请求
