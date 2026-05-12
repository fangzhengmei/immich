# Immich 时间线与 Memory 查询逻辑分析 (Round2)

---

## 一、Memory 前端完整展示链路

### 1.1 初始化与 `searchMemories($for)` 请求流程

#### 1.1.1 前端调用点 (memory-manager.svelte.ts)

**文件**: `web/src/lib/managers/memory-manager.svelte.ts:133-136`

```typescript
private async load() {
  const memories = await searchMemories({ $for: asLocalTimeISO(DateTime.now()) });
  this.memories = memories.filter((memory) => memory.assets.length > 0);
}
```

**关键特征**:
- 携带 `$for` 参数，值为当前本地时间
- 自动过滤掉无资产的空记忆

#### 1.1.2 服务端接收与权限校验

**文件**: `server/src/controllers/memory.controller.ts:23-33`

```typescript
@Get()
@Authenticated({ permission: Permission.MemoryRead })
searchMemories(@Auth() auth: AuthDto, @Query() dto: MemorySearchDto): Promise<MemoryResponseDto[]> {
  return this.service.search(auth, dto);
}
```

#### 1.1.3 `$for` 参数的 SQL 过滤逻辑

**文件**: `server/src/repositories/memory.repository.ts:37-43`

```typescript
searchBuilder(ownerId: string, dto: MemorySearchDto) {
  return this.db
    .selectFrom('memory')
    .$if(dto.for !== undefined, (qb) =>
      qb
        .where((where) => where.or([where('showAt', 'is', null), where('showAt', '<=', dto.for!)]))
        .where((where) => where.or([where('hideAt', 'is', null), where('hideAt', '>=', dto.for!)])),
    )
    .where('deletedAt', dto.isTrashed ? 'is not' : 'is', null)
    .where('ownerId', '=', ownerId);
}
```

**过滤逻辑解析**:

| 条件 | SQL 含义 | 业务逻辑 |
|------|----------|----------|
| `showAt IS NULL OR showAt <= $for` | 记忆无开始时间限制或当前时间已过开始时间 | 只有到了该显示的时间才展示给用户 |
| `hideAt IS NULL OR hideAt >= $for` | 记忆无结束时间限制或当前时间未过结束时间 | 该记忆的展示窗口还未结束 |
| `deletedAt IS [NOT] NULL` | 软删除过滤 | 根据 `isTrashed` 参数决定是否显示已删除记忆 |
| `ownerId = userId` | 只能看自己的记忆 | 数据隔离 |

### 1.2 整点刷新机制

**文件**: `web/src/lib/managers/memory-manager.svelte.ts:138-159`

```typescript
private scheduleHourlyRefresh() {
  const now = DateTime.utc();
  let nextEvent = now.set({ minute: 0, second: 5 });

  if (nextEvent <= now) {
    nextEvent = nextEvent.plus({ hours: 1 });
  }

  const initialDelay = nextEvent.diff(now).as('milliseconds');

  setTimeout(() => {
    this.#loading = this.load();

    // Schedule subsequent events hourly
    setInterval(
      () => {
        this.#loading = this.load();
      },
      60 * 60 * 1000,
    );
  }, initialDelay);
}
```

**刷新策略设计**:

| 时间点 | 行为 | 目的 |
|--------|------|------|
| 首次加载 | 立即调用 `searchMemories($for)` | 用户打开应用时立即看到当天记忆 |
| 下一个整点 (HH:00:05) | 首次定时刷新 | 确保日期变更时（00:00 左右）新一天的记忆能及时出现 |
| 之后每小时（60分钟间隔） | 周期性刷新 | 确保记忆状态（如保存、删除）同步 |

### 1.3 `years_ago` 标题计算逻辑

#### 1.3.1 标题格式化函数

**文件**: `web/src/lib/utils.ts:322-331`

```typescript
export const memoryLaneTitle = derived(t, ($t) => {
  return (memory: MemoryResponseDto) => {
    const now = new Date();
    if (memory.type === MemoryType.OnThisDay) {
      return $t('years_ago', { values: { years: now.getFullYear() - memory.data.year } });
    }

    return $t('unknown');
  };
});
```

#### 1.3.2 i18n 多语言支持

**文件**: `i18n/<lang>.json` (全语言统一模式)

示例（中文）:
```json
"years_ago": "{years, plural, one {#年} other {#年}}前"
```

**关键设计**:
- 使用 ICU MessageFormat 复数格式
- `{years}` 变量 = 当前年份 - 记忆年份 (`now.getFullYear() - memory.data.year`)
- 例如：2024 年查看 2022 年的记忆 → "2年前"

#### 1.3.3 `OnThisDay` 记忆创建时的 `data.year` 设置

**文件**: `server/src/services/memory.service.ts:47-66`

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
          data: { year },   // <-- 这里写入年份
          memoryAt: target.set({ year }).toISO()!,
          showAt,
          hideAt,
        },
        new Set(assets.map(({ id }) => id)),
      ),
    ),
  );
}
```

### 1.4 Memory 展示链路汇总

```
用户登录/页面加载
      ↓
MemoryManager 初始化
      ↓
searchMemories({ $for: 当前本地时间ISO })
      ↓
服务端权限校验 (MemoryRead)
      ↓
SQL过滤: showAt <= $for <= hideAt
      ↓
返回记忆列表（含资产信息）
      ↓
前端过滤掉空资产的记忆 (assets.length > 0)
      ↓
计算 years_ago 标题 (当前年份 - memory.data.year)
      ↓
渲染展示给用户
      ↓
【定时刷新】
      ├─ 下一个整点 (HH:00:05) → 重新加载
      └─ 之后每60分钟 → 重新加载
```

---

## 二、旅行场景功能未实现的闭环证据

### 2.1 枚举类型层面：仅定义了 `OnThisDay`

**文件**: `server/src/enum.ts:77-80`

```typescript
export enum MemoryType {
  /** pictures taken on this day X years ago */
  OnThisDay = 'on_this_day',
}
```

**结论**: `MemoryType` 枚举中没有任何与旅行/地理位置相关的类型定义。

### 2.2 i18n 文案层面：无旅行相关文案

```bash
# 在所有多语言文件中搜索
grep -r "years_ago" i18n/  # 存在：OnThisDay 标题
grep -r "travel\|trip\|vacation\|location" i18n/  # 无：旅行记忆标题
```

**搜索结果**:
- ✅ `years_ago` 存在于所有语言文件中
- ❌ 无任何 `trip_ago`、`travel_to`、`location_cluster` 等旅行相关 i18n key

### 2.3 定时任务层面：仅生成往年今日

**文件**: `server/src/services/memory.service.ts:16-44`

定时任务 `JobName.MemoryGenerate` 的完整逻辑:
1. 处理日期范围：今天 ± 3 天（共7天）
2. 对每天调用 `createOnThisDayMemories()`
3. **没有任何**按地理位置聚类、按城市分组、按旅行时间跨度识别的代码

### 2.4 数据库层面：无旅行相关字段

**Memory 表字段** (`server/src/schema/tables/memory.table.ts`):
- id, ownerId, createdAt, updatedAt, deletedAt
- type, data, isSaved, memoryAt, seenAt, showAt, hideAt

**没有**:
- ❌ `location` / `city` / `country` 字段
- ❌ `startDate` / `endDate` 旅行时间跨度字段
- ❌ `isTrip` / `isVacation` 标识字段
- ❌ `clusterId` 地理位置聚类ID

### 2.5 前端管理器层面：无旅行记忆的处理逻辑

**文件**: `web/src/lib/managers/memory-manager.svelte.ts`

搜索整个文件:
- 只有 `OnThisDay` 一种类型处理（通过 `memory.type === MemoryType.OnThisDay` 判断标题）
- 无任何旅行/地理位置相关的识别、分组、渲染逻辑

### 2.6 旅行场景未实现的最终结论

| 检查维度 | 结果 |
|----------|------|
| ✅ API 端点存在 | `POST /memories` 支持手动创建记忆 |
| ❌ MemoryType 枚举 | 仅有 `OnThisDay` |
| ❌ 自动生成逻辑 | 定时任务仅生成往年今日 |
| ❌ 数据库支持 | 无旅行/地理位置相关字段 |
| ❌ 前端 UI 支持 | 无旅行记忆渲染组件 |
| ❌ i18n 文案 | 无相关翻译键 |

**最终结论**: 旅行场景记忆功能在当前主线代码中完全未实现，只有 API 框架支持手动创建记忆，但没有自动识别和生成旅行记忆的逻辑。

---

## 三、共享时间线裁剪的端到端完整路径

### 3.1 路径总览

```
用户进入共享相册页面
      ↓
初始化 TimelineManager，携带 timelineAlbumId
      ↓
请求时间分桶 → /timeline/buckets?albumId=xxx
      ↓
权限校验 → requireAccess(AlbumRead)
      ↓
SQL 按月份分组计数，仅返回该相册内资产
      ↓
【懒加载】滚动到某个月份
      ↓
请求月份详情 (第1次) → /timeline/bucket?albumId=xxx&timeBucket=YYYY-MM
      ↓
【二次请求】如果有 timelineAlbumId (第2次) → /timeline/bucket?albumId=xxx&timeBucket=YYYY-MM
      ↓
收集相册内资产 ID 到 albumAssets Set
      ↓
SQL 过滤资产可见性
      ↓
渲染时间线
```

### 3.2 步骤1：权限检查层

**文件**: `server/src/services/timeline.service.ts:48-68`

```typescript
private async timeBucketChecks(auth: AuthDto, dto: TimeBucketDto) {
  if (dto.visibility === AssetVisibility.Locked) {
    requireElevatedPermission(auth);
  }

  if (dto.albumId) {
    // 相册时间线权限检查
    await this.requireAccess({ auth, permission: Permission.AlbumRead, ids: [dto.albumId] });
  } else {
    dto.userId = dto.userId || auth.user.id;
  }

  if (dto.userId) {
    // 用户时间线权限检查
    await this.requireAccess({ auth, permission: Permission.TimelineRead, ids: [dto.userId] });
    if (dto.visibility === AssetVisibility.Archive) {
      await this.requireAccess({ auth, permission: Permission.ArchiveRead, ids: [dto.userId] });
    }
  }

  if (dto.tagId) {
    await this.requireAccess({ auth, permission: Permission.TagRead, ids: [dto.tagId] });
  }

  if (dto.withPartners) {
    // withPartners 不能用于查询归档/收藏/回收站
    const requestedArchived = dto.visibility === AssetVisibility.Archive || dto.visibility === undefined;
    const requestedFavorite = dto.isFavorite === true || dto.isFavorite === false;
    const requestedTrash = dto.isTrashed === true;

    if (requestedArchived || requestedFavorite || requestedTrash) {
      throw new BadRequestException(
        'withPartners is only supported for non-archived, non-trashed, non-favorited assets',
      );
    }
  }
}
```

**权限矩阵**:

| 场景 | 所需权限 | 检查点 |
|------|----------|--------|
| 主时间线 | TimelineRead | userId 匹配 |
| 相册时间线 | AlbumRead | albumId 匹配 |
| 归档时间线 | ArchiveRead | userId + visibility=ARCHIVE |
| 锁定资产 | ElevatedPermission | visibility=LOCKED |
| 标签时间线 | TagRead | tagId 匹配 |
| 伴侣共享 | TimelineRead + 伴侣关系 | withPartners=true |

### 3.3 步骤2：参数组装与用户ID构建

**文件**: `server/src/services/timeline.service.ts:28-45`

```typescript
private async buildTimeBucketOptions(auth: AuthDto, dto: TimeBucketDto): Promise<TimeBucketOptions> {
  const { userId, ...options } = dto;
  let userIds: string[] | undefined = undefined;

  if (userId) {
    userIds = [userId];
    if (dto.withPartners) {
      // 融合伴侣用户ID列表
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

**userIds 组装规则**:
1. **相册时间线**: `userIds = undefined` → 不按用户过滤，按 `albumId` 关联
2. **个人时间线**: `userIds = [auth.user.id]` → 只看自己的
3. **伴侣时间线**: `userIds = [myId, partnerId1, partnerId2, ...]` → 自己 + 所有启用时间线的伴侣

### 3.4 步骤3：SQL 过滤层

**文件**: `server/src/repositories/asset.repository.ts:709-760`

```typescript
async getTimeBuckets(options: TimeBucketOptions): Promise<TimeBucketItem[]> {
  return this.db
    .with('asset', (qb) =>
      qb
        .selectFrom('asset')
        .select(truncatedDate<Date>().as('timeBucket'))
        .$if(!!options.isTrashed, (qb) => qb.where('asset.status', '!=', AssetStatus.Deleted))
        .where('asset.deletedAt', options.isTrashed ? 'is not' : 'is', null)
        .$if(options.visibility === undefined, withDefaultVisibility)
        .$if(!!options.visibility, (qb) => qb.where('asset.visibility', '=', options.visibility!))
        .$if(!!options.albumId, (qb) =>
          qb
            .innerJoin('album_asset', 'asset.id', 'album_asset.assetId')
            .where('album_asset.albumId', '=', asUuid(options.albumId!)),
        )
        .$if(!!options.withStacked, (qb) =>
          qb
            .leftJoin('stack', (join) =>
              join.onRef('stack.id', '=', 'asset.stackId').onRef('stack.primaryAssetId', '=', 'asset.id'),
            )
            .where((eb) => eb.or([eb('asset.stackId', 'is', null), eb(eb.table('stack'), 'is not', null)])),
        )
        .$if(!!options.userIds, (qb) => qb.where('asset.ownerId', '=', anyUuid(options.userIds!)))
        .$if(options.isFavorite !== undefined, (qb) => qb.where('asset.isFavorite', '=', options.isFavorite!))
        .$if(!!options.assetType, (qb) => qb.where('asset.type', '=', options.assetType!))
        .$if(options.isDuplicate !== undefined, (qb) =>
          qb.where('asset.duplicateId', options.isDuplicate ? 'is not' : 'is', null),
        )
        .$if(!!options.tagId, (qb) => withTagId(qb, options.tagId!))
    )
    .selectFrom('asset')
    .select(sql<string>`("timeBucket" AT TIME ZONE 'UTC')::date::text`.as('timeBucket'))
    .select((eb) => eb.fn.countAll<number>().as('count'))
    .groupBy('timeBucket')
    .orderBy('timeBucket', options.order ?? 'desc')
    .execute() as any as Promise<TimeBucketItem[]>;
}
```

**关键过滤条件梳理**:

| 参数 | SQL 条件 | 作用 |
|------|----------|------|
| `albumId` | `INNER JOIN album_asset ON asset.id = album_asset.assetId WHERE album_asset.albumId = ?` | 只返回相册内的资产 |
| `userIds` | `WHERE asset.ownerId IN (?, ?, ...)` | 只返回指定用户的资产 |
| `isFavorite` | `WHERE asset.isFavorite = true/false` | 收藏状态过滤 |
| `visibility` | `WHERE asset.visibility = ?` | TIMELINE/ARCHIVE/HIDDEN/LOCKED |
| `isTrashed` | `WHERE asset.deletedAt IS NOT NULL` | 回收站过滤 |
| `withStacked` | 仅保留堆叠主资产 | 避免堆叠资产重复计数 |
| `tagId` | `JOIN asset_tag WHERE tag_id = ?` | 标签过滤 |

### 3.5 步骤4：前端 timelineAlbumId 二次请求

**文件**: `web/src/lib/managers/timeline-manager/internal/load-support.svelte.ts:9-60`

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

  // 第1次请求：获取月份资产数据
  const timeBucket = toISOYearMonthUTC(timelineMonth.yearMonth);
  const bucketResponse = await getTimeBucket(
    {
      ...authManager.params,
      ...options,
      timeBucket,
    },
    { signal },
  );

  if (!bucketResponse || signal.aborted) {
    return;
  }

  // 【关键】如果有 timelineAlbumId，发起第2次相同请求
  if (options.timelineAlbumId) {
    const albumAssets = await getTimeBucket(
      {
        ...authManager.params,
        albumId: options.timelineAlbumId,
        timeBucket,
      },
      { signal },
    );
    if (!albumAssets || signal.aborted) {
      return;
    }
    for (const id of albumAssets.id) {
      timelineManager.albumAssets.add(id);
    }
  }

  const unprocessedAssets = timelineMonth.addAssets(bucketResponse, true);
  // ...
}
```

#### 3.5.1 二次请求的设计意图

| 请求 | 参数 | 目的 | 结果 |
|------|------|------|------|
| 第1次 | `options` 原参数 | 加载完整时间线资产 | 所有资产（可能包含相册外资产） |
| 第2次 | `albumId: timelineAlbumId` | 仅加载相册内资产 | 收集相册资产 ID 到 `albumAssets Set` |

**albumAssets 用途**:
- 标记哪些资产属于当前相册
- 在 UI 上区分"相册内资产"和"相册外资产"
- 支持选择操作（只能选择相册内的资产进行移除等操作）

### 3.6 共享时间线完整数据流图

```
客户端                                   服务端

【初始化相册页面】
    TimelineManager({ timelineAlbumId })
            ↓
【获取分桶列表】
    getTimeBuckets({ albumId })
            │
            ├─[HTTP]→ /timeline/buckets?albumId=xxx
            │              ↓
            │       权限检查 AlbumRead
            │              ↓
            │       INNER JOIN album_asset
            │              ↓
            │       GROUP BY 月份计数
            │              ↓
            └───── 返回月份分桶列表

【滚动懒加载月份】
    loadFromTimeBuckets(month, { timelineAlbumId })
            │
            ├─ 第1次请求
            │   getTimeBucket({ options, timeBucket })
            │          ↓
            │   返回该月所有资产
            │          ↓
            │   加入月份视图
            │
            └─ 第2次请求（仅当有 timelineAlbumId）
                getTimeBucket({ albumId, timeBucket })
                       ↓
                返回该相册该月的资产
                       ↓
                收集 asset.id 到 albumAssets Set
                       ↓
                【渲染】
                       ↓
                asset.id in albumAssets ?
                  ├─ true → 标记为"相册内资产"
                  └─ false → 标记为"相册外资产"
```

---

## 四、关键代码文件索引

| 功能 | 文件路径 |
|------|----------|
| Memory 前端管理器 | `web/src/lib/managers/memory-manager.svelte.ts` |
| Memory 标题格式化 | `web/src/lib/utils.ts:322-331` |
| Memory 服务层 | `server/src/services/memory.service.ts` |
| Memory 仓储层 | `server/src/repositories/memory.repository.ts` |
| Memory 控制器 | `server/src/controllers/memory.controller.ts` |
| 时间线服务 | `server/src/services/timeline.service.ts` |
| 资产仓储时间线查询 | `server/src/repositories/asset.repository.ts:709-903` |
| 时间线分桶加载器 | `web/src/lib/managers/timeline-manager/internal/load-support.svelte.ts` |
| Memory 类型枚举 | `server/src/enum.ts:77-80` |
