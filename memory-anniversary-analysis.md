# Immich Memory「往年今日」功能深度分析

## 一、整体架构概览

Immich 的 Memory「往年今日」功能是一个自动化的回忆生成系统，核心流程包括：

```
定时任务触发 → 素材筛选 → 日期匹配 → 按年分组 → Memory 创建 → 清理过期 → 前端展示
```

主要代码分布：
- **后端服务**：`server/src/services/memory.service.ts`
- **数据查询**：`server/src/repositories/asset.repository.ts` (getByDayOfYear)
- **Memory 仓储**：`server/src/repositories/memory.repository.ts`
- **前端管理**：`web/src/lib/managers/memory-manager.svelte.ts`
- **前端展示**：`web/src/routes/(user)/memory/`

---

## 二、生成触发机制

### 2.1 定时任务调度

Memory 生成由定时任务驱动，在 `memory.service.ts:16-45` 中定义：

```typescript
@OnJob({ name: JobName.MemoryGenerate, queue: QueueName.BackgroundTask })
async onMemoriesCreate() {
  const users = await this.userRepository.getList({ withDeleted: false });

  await this.databaseRepository.withLock(DatabaseLock.MemoryCreation, async () => {
    const state = await this.systemMetadataRepository.get(SystemMetadataKey.MemoriesState);
    const start = DateTime.utc().startOf('day').minus({ days: DAYS });
    const lastOnThisDayDate = state?.lastOnThisDayDate ? DateTime.fromISO(state.lastOnThisDayDate) : start;

    // generate a memory +/- X days from today
    for (let i = 0; i <= DAYS * 2; i++) {
      const target = start.plus({ days: i });
      if (lastOnThisDayDate >= target) {
        continue;
      }

      this.logger.log(`Creating memories for ${target.toISO()}`);
      try {
        await Promise.all(users.map((owner) => this.createOnThisDayMemories(owner.id, target)));
      } catch (error) {
        this.logger.error(`Failed to create memories for ${target.toISO()}: ${error}`);
      }
      // update system metadata even when there is an error to minimize the chance of duplicates
      await this.systemMetadataRepository.set(SystemMetadataKey.MemoriesState, {
        ...state,
        lastOnThisDayDate: target.toISO(),
      });
    }
  });
}
```

**关键参数**：
- 时间窗口：`DAYS = 3`（生成当天 ±3 天范围内的回忆，共 7 天）
- 幂等保证：通过 `SystemMetadataKey.MemoriesState` 记录最后处理日期
- 分布式锁：使用 `DatabaseLock.MemoryCreation` 防止并发重复执行

### 2.2 清理机制

```typescript
@OnJob({ name: JobName.MemoryCleanup, queue: QueueName.BackgroundTask })
async onMemoriesCleanup() {
  await this.memoryRepository.cleanup();
}
```

清理规则（`memory.repository.ts:17-30`）：

1. **清理关联资产**：删除 `asset.visibility != Timeline` 的 memory_asset 记录
   ```typescript
   .where('asset.visibility', '!=', AssetVisibility.Timeline)
   ```
   > 注意：这里判断的是资产可见性是否为 Timeline，而非判断资产是否被归档

2. **清理过期 Memory**：删除创建超过 30 天且未被保存（`isSaved = false`）的 memory

---

## 三、素材筛选与日期匹配逻辑

### 3.1 getByDayOfYear 精确日期匹配链路

核心筛选逻辑在 `asset.repository.ts:448-496` 的 `getByDayOfYear` 方法中实现。

```typescript
getByDayOfYear(ownerIds: string[], { year, day, month }: YearMonthDay) {
  return this.db
    .with('res', (qb) =>
      qb
        .with('today', (qb) =>
          qb
            .selectFrom((eb) =>
              eb
                .fn('generate_series', [
                  sql`(select date_part('year', min(("localDateTime" at time zone 'UTC')::date))::int from asset)`,
                  sql`${year - 1}`,
                ])
                .as('year'),
            )
            .select((eb) => eb.fn('make_date', [sql`year::int`, sql`${month}::int`, sql`${day}::int`]).as('date')),
        )
        .selectFrom('today')
        .innerJoinLateral(
          (qb) =>
            qb
              .selectFrom('asset')
              .select(['asset.id', 'asset.localDateTime'])
              .innerJoin('asset_job_status', 'asset.id', 'asset_job_status.assetId')
              .where(sql`(asset."localDateTime" at time zone 'UTC')::date`, '=', sql`today.date`)
              .where('asset.ownerId', '=', anyUuid(ownerIds))
              .where('asset.visibility', '=', AssetVisibility.Timeline)
              .where((eb) =>
                eb.exists((qb) =>
                  qb
                    .selectFrom('asset_file')
                    .whereRef('assetId', '=', 'asset.id')
                    .where('asset_file.type', '=', AssetFileType.Preview),
                ),
              )
              .where('asset.deletedAt', 'is', null)
              .orderBy(sql`(asset."localDateTime" at time zone 'UTC')::date`, 'desc')
              .limit(20)
              .as('a'),
          (join) => join.onTrue(),
        )
        .selectAll('a'),
    )
    .selectFrom('res')
    .select(sql<number>`date_part('year', ("localDateTime" at time zone 'UTC')::date)::int`.as('year'))
    .select((eb) => eb.fn.jsonAgg(eb.table('res')).as('assets'))
    .groupBy(sql`("localDateTime" at time zone 'UTC')::date`)
    .orderBy(sql`("localDateTime" at time zone 'UTC')::date`, 'desc')
    .execute();
}
```

### 3.2 筛选条件详解

| 条件 | 说明 | 代码位置 |
|------|------|----------|
| 精确日期匹配 | `(asset."localDateTime" at time zone 'UTC')::date = today.date` | L471 |
| 所有权 | `asset.ownerId = anyUuid(ownerIds)` | L472 |
| 可见性 | `asset.visibility = AssetVisibility.Timeline` | L473 |
| 预览文件 | 必须存在 `AssetFileType.Preview` 类型的文件 | L474-L480 |
| 未删除 | `asset.deletedAt is null` | L482 |
| 任务状态记录 | 必须存在 `asset_job_status` 记录（仅表示状态记录存在） | L470 |

> **重要说明**：
> - `innerJoin('asset_job_status', 'asset.id', 'asset_job_status.assetId')` 仅表示该资产存在任务状态记录，**不代表转码已完成**。即使转码失败也可能存在该记录。

### 3.3 为什么「同一照片在±3天窗口跨 Memory 重复」不成立

**关键链路分析**：

1. **精确日期匹配**：`getByDayOfYear` 的核心匹配条件是：
   ```sql
   (asset."localDateTime" at time zone 'UTC')::date = today.date
   ```
   这是一个严格的**等于**匹配，而非范围匹配。

2. **每张照片有唯一拍摄日期**：资产的 `localDateTime` 是固定的，转换为日期后也是唯一的。

3. **循环处理独立日期**：虽然外层循环处理 ±3 天共 7 个 target 日期，但每个 target 调用 `getByDayOfYear` 时传入的是不同的 `{year, month, day}` 参数。

4. **匹配结果互斥**：
   - 假设照片A拍摄于 2023-05-15
   - 当 target=2024-05-14 时，`today.date` 生成 2023-05-14，不匹配
   - 当 target=2024-05-15 时，`today.date` 生成 2023-05-15，**唯一匹配**
   - 当 target=2024-05-16 时，`today.date` 生成 2023-05-16，不匹配

**结论**：每张照片只会在一个 target 日期的调用中被匹配到，不会跨多个 Memory 重复出现。±3 天窗口的设计目的是覆盖时区差异和任务重试，而非模糊匹配日期范围。

### 3.4 隐藏人物过滤

在 `memory.repository.ts:71-82` 的查询中，额外过滤了包含隐藏人物的资产：

```typescript
.where((eb) =>
  eb.not(
    eb.exists(
      eb
        .selectFrom('asset_face')
        .innerJoin('person', 'person.id', 'asset_face.personId')
        .select((eb) => eb.val(1).as('one'))
        .whereRef('asset_face.assetId', '=', 'asset.id')
        .where('person.isHidden', '=', true),
    ),
  ),
)
```

---

## 四、日期匹配边界与闰日问题

### 4.1 正常日期匹配流程

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
  );
}
```

### 4.2 闰日（2月29日）的边界问题

**核心问题**：`generate_series` + `make_date` 在处理闰日时存在边界条件。

#### 场景分析

假设当前日期是 2024年2月29日（闰年），目标日期 `target` 是 2月29日：

```sql
generate_series(2020, 2023) as "year"  -- 从最早年份到去年
```

然后对每个年份执行：
```sql
make_date(year::int, 2::int, 29::int)
```

**问题路径**：

| 年份 | `make_date(year, 2, 29)` 结果 | 说明 |
|------|--------------------------------|------|
| 2020 | 2020-02-29 | 成功，2020是闰年 |
| 2021 | **ERROR** | 2021不是闰年，2月没有29天 |
| 2022 | **ERROR** | 2022不是闰年 |
| 2023 | **ERROR** | 2023不是闰年 |

**PostgreSQL 中 `make_date(2021, 2, 29)` 会抛出错误**：
```
ERROR:  date field value out of range: 2021-02-29
```

#### 潜在失败路径

1. **整个查询失败**：如果 `generate_series` 生成的年份中包含非闰年，`make_date` 会抛出异常，导致整个 `getByDayOfYear` 查询完全失败，该用户当天无法生成任何回忆。

2. **部分年份丢失**：即使查询部分成功（如果有错误处理），非闰年的2月29日的回忆也会永久丢失。

3. **±3天窗口的连锁影响**：
   - 2月28日生成的回忆会包含2月28日往年照片
   - 3月1日生成的回忆会包含3月1日往年照片
   - **但2月29日的照片在非闰年时无法被匹配到**

#### 代码层面的验证

查看 `getByDayOfYear` 的 SQL 构造：
```typescript
.selectFrom((eb) =>
  eb
    .fn('generate_series', [
      sql`(select date_part('year', min(("localDateTime" at time zone 'UTC')::date))::int from asset)`,
      sql`${year - 1}`,
    ])
    .as('year'),
)
.select((eb) => eb.fn('make_date', [sql`year::int`, sql`${month}::int`, sql`${day}::int`]).as('date'))
```

这里没有对闰日做任何特殊处理，直接将 `month` 和 `day` 传入 `make_date`。

#### 可能的修复方向

- 在调用 `getByDayOfYear` 前检查是否为闰日，如果是闰日且目标年份不是闰年时：
  - 方案A：降级匹配2月28日
  - 方案B：跳过该年份
  - 方案C：使用 `make_date` 前先验证日期有效性

---

## 五、去重机制与幂等边界

### 5.1 现有防重措施

#### 5.1.1 系统级幂等（日期级）

通过 `SystemMetadataKey.MemoriesState` 记录最后处理日期：

```typescript
const state = await this.systemMetadataRepository.get(SystemMetadataKey.MemoriesState);
const lastOnThisDayDate = state?.lastOnThisDayDate ? DateTime.fromISO(state.lastOnThisDayDate) : start;

for (let i = 0; i <= DAYS * 2; i++) {
  const target = start.plus({ days: i });
  if (lastOnThisDayDate >= target) {
    continue;  // 已处理过，跳过
  }
  // ... 生成回忆
  await this.systemMetadataRepository.set(SystemMetadataKey.MemoriesState, {
    ...state,
    lastOnThisDayDate: target.toISO(),
  });
}
```

**特点**：
- 日期级别的幂等保证，已处理过的日期不会重复处理
- 使用分布式锁 `DatabaseLock.MemoryCreation` 防止并发重复执行

#### 5.1.2 资产级去重

```typescript
this.memoryRepository.create(
  { /* memory data */ },
  new Set(assets.map(({ id }) => id)),  // 使用 Set 天然去重
);
```

- 使用 `Set<string>` 存储 assetIds，单个 Memory 内部天然去重
- `memory_asset` 关联表保证资产与 Memory 的多对多关系

#### 5.1.3 展示级过滤

前端 `memory-manager.svelte.ts:134-136`：

```typescript
private async load() {
  const memories = await searchMemories({ $for: asLocalTimeISO(DateTime.now()) });
  this.memories = memories.filter((memory) => memory.assets.length > 0);
}
```

- 只展示当前日期范围内的 Memory（通过 `$for` 参数过滤）
- 过滤掉资产为空的 Memory

#### 5.1.4 后端搜索过滤

`memory.repository.ts:37-41`：

```typescript
.$if(dto.for !== undefined, (qb) =>
  qb
    .where((where) => where.or([where('showAt', 'is', null), where('showAt', '<=', dto.for!)]))
    .where((where) => where.or([where('hideAt', 'is', null), where('hideAt', '>=', dto.for!)]),
)
```

---

### 5.2 漏生成风险

#### 5.2.1 Promise.all 部分失败导致的漏生成

**核心问题**：`Promise.all` 任一用户失败，但 `lastOnThisDayDate` 仍前移。

```typescript
try {
  await Promise.all(users.map((owner) => this.createOnThisDayMemories(owner.id, target)));
} catch (error) {
  this.logger.error(`Failed to create memories for ${target.toISO()}: ${error}`);
}
// 无论成功失败，都前移 lastOnThisDayDate
await this.systemMetadataRepository.set(SystemMetadataKey.MemoriesState, {
  ...state,
  lastOnThisDayDate: target.toISO(),
});
```

**风险场景**：

| 时间点 | 操作 | 结果 |
|--------|------|------|
| T1 | 获取 users = [userA, userB, userC] | 共3个用户 |
| T2 | Promise.all 并发执行 createOnThisDayMemories | userA 成功，userB 成功，userC 失败（如 DB 超时） |
| T3 | catch 捕获异常，记录 error log | 仅记录日志，不重试 |
| T4 | 更新 lastOnThisDayDate = target | 该日期标记为已处理 |
| T5 | 下次任务执行 | lastOnThisDayDate >= target，跳过该日期 |

**后果**：
- userC 在该日期的回忆**永久丢失**
- 除非手动清理 `system_metadata` 中的 `MemoriesState`，否则永远不会重试
- 日志中仅有一条 error 记录，难以发现和排查

#### 5.2.2 缺少唯一约束导致的重复生成风险

**当前表结构**（`memory.table.ts`）：

Memory 表的字段包括：
- `id` (uuid, 主键)
- `ownerId` (uuid, 外键)
- `type` (MemoryType enum)
- `memoryAt` (timestamp)
- `showAt`, `hideAt` 等

**缺失的约束**：

表中没有定义 `(ownerId, type, memoryAt)` 或 `(ownerId, type, data->>'year', showAt)` 的唯一约束。

**重复生成条件**：

虽然 `lastOnThisDayDate` 机制在正常情况下可以防止重复，但以下情况可能导致重复：

1. **SystemMetadata 状态丢失**：如果 `system_metadata` 表中的 `MemoriesState` 记录被删除或损坏，`lastOnThisDayDate` 会重置为 `start`（3天前），导致重新生成这7天的回忆。

2. **分布式锁失效**：如果 `DatabaseLock.MemoryCreation` 锁机制失效，多个实例可能同时执行生成任务。

3. **任务并发执行**：如果定时任务调度器重复触发，且锁获取失败或释放过早。

在以上情况下，由于没有数据库级唯一约束，同一用户的同一日期可能生成多条重复的 Memory 记录。

---

### 5.3 可验证改进

#### 5.3.1 Promise.all 失败处理改进

**方案A：用户级独立错误处理**

```typescript
// 改为逐个处理用户，单个用户失败不影响其他用户
for (const owner of users) {
  try {
    await this.createOnThisDayMemories(owner.id, target);
  } catch (error) {
    this.logger.error(`Failed to create memories for user ${owner.id} on ${target.toISO()}: ${error}`);
    // 可考虑记录失败用户列表，后续重试
  }
}
```

**方案B：记录失败用户，单独重试**

```typescript
const failedUsers: string[] = [];
await Promise.all(users.map(async (owner) => {
  try {
    await this.createOnThisDayMemories(owner.id, target);
  } catch (error) {
    failedUsers.push(owner.id);
    this.logger.error(`Failed to create memories for user ${owner.id} on ${target.toISO()}: ${error}`);
  }
}));

// 对失败用户进行重试
for (const owner of failedUsers) {
  try {
    await this.createOnThisDayMemories(owner.id, target);
  } catch (error) {
    this.logger.error(`Retry failed for user ${owner.id} on ${target.toISO()}: ${error}`);
  }
}
```

#### 5.3.2 增加数据库唯一约束

在 `memory.table.ts` 中添加唯一约束：

```typescript
@Unique(['ownerId', 'type', 'memoryAt'])
```

或更精确的：

```typescript
@Unique(['ownerId', 'type', 'showAt', 'hideAt'])
```

这样即使应用层幂等机制失效，数据库也会拒绝重复插入。

#### 5.3.3 引入按用户的幂等标记

除了全局的 `lastOnThisDayDate`，可以考虑：
- 增加 `user_memory_state` 表，记录每个用户的最后处理日期
- 或在生成前先查询该用户该日期是否已有 Memory

---

## 六、数据模型与存储结构

### 6.1 Memory 表结构（`memory.table.ts`）

| 字段 | 类型 | 说明 |
|------|------|------|
| `id` | uuid | 主键 |
| `ownerId` | uuid | 所属用户 |
| `type` | enum | Memory 类型，目前只有 `on_this_day` |
| `data` | jsonb | 附加数据，存储 `{ year: number }` |
| `isSaved` | boolean | 用户是否保存，保存后不会被自动清理 |
| `memoryAt` | timestamp | 回忆日期（原始拍摄日期） |
| `seenAt` | timestamp | 用户上次查看时间 |
| `showAt` | timestamp | 开始显示时间 |
| `hideAt` | timestamp | 结束显示时间 |
| `createdAt` | timestamp | 创建时间 |
| `updatedAt` | timestamp | 更新时间 |
| `deletedAt` | timestamp | 删除时间（软删除） |

### 6.2 关联表 `memory_asset`

- `memoriesId`：关联 memory.id
- `assetId`：关联 asset.id
- 多对多关系，支持一个资产属于多个 Memory

---

## 七、展示组织方式

### 7.1 前端数据组织（`memory-manager.svelte.ts`）

前端将 Memory 扁平化为 `MemoryAsset` 列表，便于浏览：

```typescript
private memoryAssets = $derived.by(() => {
  const memoryAssets: MemoryAsset[] = [];
  let previous: MemoryAsset | undefined;
  for (const [memoryIndex, memory] of this.memories.entries()) {
    for (const [assetIndex, asset] of memory.assets.entries()) {
      const current = {
        memory,
        memoryIndex,
        previousMemory: this.memories[memoryIndex - 1],
        nextMemory: this.memories[memoryIndex + 1],
        asset: toTimelineAsset(asset),
        assetIndex,
        previous,
      };
      memoryAssets.push(current);
      if (previous) {
        previous.next = current;
      }
      previous = current;
    }
  }
  return memoryAssets;
});
```

**设计特点**：
- 双向链表结构（`previous`/`next`）
- 跨 Memory 边界的资产导航（`previousMemory`/`nextMemory`）
- 支持无缝滑动浏览不同年份的回忆

### 7.2 排序规则

后端默认排序（`memory.repository.ts:88-91`）：

```typescript
.$call((qb) =>
  dto.order === AssetOrderWithRandom.Random
    ? qb.orderBy(sql`RANDOM()`)
    : qb.orderBy('memoryAt', (dto.order?.toLowerCase() || 'desc') as OrderByDirection),
)
```

- 默认：按 `memoryAt` 降序（最近的年份在前）
- 支持随机排序
- 每个 Memory 内部资产按 `fileCreatedAt` 升序排列

### 7.3 自动刷新机制

`memory-manager.svelte.ts:138-159` 实现了每小时自动刷新：

```typescript
private scheduleHourlyRefresh() {
  const now = DateTime.utc();
  let nextEvent = now.set({ minute: 0, second: 5 });
  if (nextEvent <= now) {
    nextEvent = nextEvent.plus({ hours: 1 });
  }
  // ... 定时刷新
}
```

---

## 八、用户交互与生命周期

### 8.1 Memory 状态流转

```
生成 → 显示（当天） → 用户查看（更新 seenAt） → 用户保存（isSaved=true）
                      ↓
                    过期（hideAt）→ 30天后自动删除（未保存）
                                      ↓
                                    永久保留（已保存）
```

### 8.2 用户可执行操作

| 操作 | API 端点 | 说明 |
|------|----------|------|
| 查看 | `GET /memories` | 获取当前可用回忆 |
| 保存 | `PUT /memories/:id` | 设置 `isSaved=true` |
| 删除 | `DELETE /memories/:id` | 彻底删除 |
| 移除资产 | `DELETE /memories/:id/assets` | 从回忆中移除特定照片 |
| 添加资产 | `PUT /memories/:id/assets` | 向回忆中添加照片 |

### 8.3 展示级别的资产隐藏

`memory-manager.svelte.ts:77-84`：

```typescript
hideAssetsFromMemory(ids: string[]) {
  const idSet = new Set<string>(ids);
  for (const memory of this.memories) {
    memory.assets = memory.assets.filter((asset) => !idSet.has(asset.id));
  }
  this.memories = this.memories.filter((memory) => memory.assets.length > 0);
}
```

---

## 九、关键设计决策分析

### 9.1 为什么使用 ±3 天窗口？

- 覆盖时区差异，确保用户在任何时区都能看到当天的回忆
- 处理夏令时切换等边界情况
- 给定时任务失败留出重试窗口

### 9.2 为什么每年生成独立 Memory 而不是合并？

- 用户体验：按年份分组浏览更有仪式感
- 保存粒度：可以单独保存/删除某一年的回忆
- 展示灵活：前端可以自由组织展示方式

### 9.3 为什么需要 showAt/hideAt？

- 精确控制回忆的显示时间窗口
- 支持未来扩展（如生日回忆、特定节日回忆）
- 简化前端查询逻辑，只需传入当前日期

### 9.4 去重设计的权衡

| 层级 | 实现方式 | 优点 | 缺点 |
|------|----------|------|------|
| 系统级 | SystemMetadata 记录 | 简单高效 | 极端情况下可能丢失状态 |
| 资产级 | Set 去重 | 天然防重 | 无法跨 Memory 去重 |
| 展示级 | 前端过滤 | 用户感知好 | 增加一次 API 调用 |

---

## 十、已知问题与潜在优化

### 10.1 已知问题

1. **闰日处理缺失**：2月29日在非闰年时 `make_date` 会失败，导致回忆生成失败
2. **asset_job_status 含义不精确**：仅检查记录存在，不保证转码成功
3. **Promise.all 部分失败风险**：单个用户失败导致该用户该日期回忆永久丢失
4. **缺少数据库唯一约束**：极端情况下可能生成重复 Memory

### 10.2 潜在优化

1. **闰日兼容处理**：在 `getByDayOfYear` 中增加闰日判断，非闰年时匹配2月28日
2. **智能排序**：基于照片质量、内容重要性排序，而非仅按时间
3. **记忆摘要**：为每个 Memory 生成简短的文字描述
4. **批量管理**：支持批量保存/删除多个 Memory
5. **预览生成优化**：当前要求必须有 Preview 文件，可考虑降级使用缩略图
6. **转码状态校验**：增加对 asset_job_status 中具体字段的检查，确保转码成功
7. **用户级错误处理**：改进 Promise.all 失败处理，避免单个用户失败影响整体
8. **增加唯一约束**：在 memory 表增加 (ownerId, type, memoryAt) 唯一约束

---

## 十一、核心代码路径速查

| 功能 | 文件 | 行号 |
|------|------|------|
| 定时任务入口 | `memory.service.ts` | 16-45 |
| 创建 OnThisDay 回忆 | `memory.service.ts` | 47-66 |
| 按日期查询资产 | `asset.repository.ts` | 448-496 |
| Memory 搜索 | `memory.repository.ts` | 60-94 |
| 清理过期回忆 | `memory.repository.ts` | 17-30 |
| 前端 Memory 管理 | `memory-manager.svelte.ts` | 1-162 |
| 前端查看器 | `MemoryViewer.svelte` | 1- |
