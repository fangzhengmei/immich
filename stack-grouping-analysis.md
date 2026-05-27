# Immich 照片堆栈（Stack）分组逻辑分析

## 概述

Immich 的堆栈（Stack）功能用于将同一时间点拍摄的多张照片合并成一组，在时间线上只显示主照片，用户可以展开查看组内其他照片。本文档从代码层面梳理堆栈的判定信号、自动分组和手动覆盖的完整流程。

---

## 一、判定信号（Detection Signals）

### 1.1 自动堆栈 ID（autoStackId）提取

堆栈分组的核心判定信号是 `autoStackId`，该值从照片的 EXIF 元数据中提取。

**代码位置**：`server/src/services/metadata.service.ts:1040-1045`

```typescript
private getAutoStackId(tags: ImmichTags | null): string | null {
  if (!tags) {
    return null;
  }
  return tags.BurstID ?? tags.BurstUUID ?? tags.CameraBurstID ?? tags.MediaUniqueID ?? null;
}
```

**判定优先级**（按顺序）：
1. `BurstID` - 连拍 ID（苹果设备等）
2. `BurstUUID` - 连拍 UUID
3. `CameraBurstID` - 相机连拍 ID
4. `MediaUniqueID` - 媒体唯一 ID

**存储位置**：
- 字段：`asset_exif.autoStackId`
- 表结构：`server/src/schema/tables/asset-exif.table.ts:104-105`
- 数据库索引：`IDX_auto_stack_id`（用于快速查询相同 autoStackId 的照片）

### 1.2 其他辅助信号

虽然代码中主要依赖 `autoStackId` 作为分组信号，但从数据模型设计来看，以下字段也可能与分组相关：

| 字段 | 位置 | 用途 |
|------|------|------|
| `localDateTime` | `asset` 表 | 照片拍摄的本地时间（用于时间线分组） |
| `fileCreatedAt` | `asset` 表 | 文件创建时间 |
| `livePhotoCID` | `asset_exif` 表 | 实况照片内容 ID（用于关联照片和视频） |

---

## 二、自动分组流程（Automatic Grouping Process）

### 2.1 元数据提取阶段

当照片上传到 Immich 后，系统会执行元数据提取任务：

**代码位置**：`server/src/services/metadata.service.ts:235-416`

**流程**：
1. 触发 `JobName.AssetExtractMetadata` 任务
2. 读取照片 EXIF 标签
3. 调用 `getAutoStackId()` 提取自动堆栈 ID
4. 将 `autoStackId` 写入 `asset_exif` 表
5. 触发 `AssetMetadataExtracted` 事件

**注意**：截至当前代码版本，**系统仅提取和存储 `autoStackId`，但不会自动基于该字段创建堆栈**。自动创建堆栈的功能可能在后续版本中实现，或者通过工作流（Workflow）机制扩展。

### 2.2 时间线分组显示

虽然没有自动创建堆栈，但时间线显示时会对已有的堆栈进行特殊处理：

**代码位置**：`server/src/repositories/asset.repository.ts:834-858`

```typescript
.$if(!!options.withStacked, (qb) =>
  qb
    .where((eb) =>
      eb.not(
        eb.exists(
          eb
            .selectFrom('stack')
            .whereRef('stack.id', '=', 'asset.stackId')
            .whereRef('stack.primaryAssetId', '!=', 'asset.id'),
        ),
      ),
    )
    .leftJoinLateral(
      (eb) =>
        eb
          .selectFrom('asset as stacked')
          .select(sql`array[stacked."stackId"::text, count('stacked')::text]`.as('stack'))
          .whereRef('stacked.stackId', '=', 'asset.stackId')
          .where('stacked.deletedAt', 'is', null)
          .where('stacked.visibility', '=', AssetVisibility.Timeline)
          .groupBy('stacked.stackId')
          .as('stacked_assets'),
      (join) => join.onTrue(),
    )
    .select('stack'),
)
```

**显示逻辑**：
- 当 `withStacked=true` 时，只返回堆栈的主照片（`primaryAssetId`）
- 堆栈信息以 `[stackId, assetCount]` 元组形式返回
- 非堆栈照片和堆栈主照片会显示在时间线上

### 2.3 时间桶（Time Bucket）分组

时间线使用按天分组的方式组织照片：

**代码位置**：`server/src/repositories/asset.repository.ts:710-760`

```typescript
async getTimeBuckets(options: TimeBucketOptions): Promise<TimeBucketItem[]> {
  return this.db
    .with('asset', (qb) => qb.selectFrom('asset').select(truncatedDate<Date>(options.orderBy).as('timeBucket')))
    .selectFrom('asset')
    .select(sql<string>`("timeBucket" AT TIME ZONE 'UTC')::date::text`.as('timeBucket'))
    .select((eb) => eb.fn.countAll<number>().as('count'))
    .groupBy('timeBucket')
    .orderBy('timeBucket', options.order ?? 'desc')
    .execute() as any as Promise<TimeBucketItem[]>;
}
```

---

## 三、手动覆盖流程（Manual Override Process）

### 3.1 堆栈 API 接口

**代码位置**：`server/src/controllers/stack.controller.ts`

| 方法 | 端点 | 权限 | 说明 |
|------|------|------|------|
| GET | `/stacks` | `StackRead` | 查询堆栈列表 |
| POST | `/stacks` | `StackCreate` | 创建堆栈 |
| GET | `/stacks/:id` | `StackRead` | 获取单个堆栈详情 |
| PUT | `/stacks/:id` | `StackUpdate` | 更新堆栈（设置主照片等） |
| DELETE | `/stacks/:id` | `StackDelete` | 删除单个堆栈 |
| DELETE | `/stacks` | `StackDelete` | 批量删除堆栈 |
| DELETE | `/stacks/:id/assets/:assetId` | `StackUpdate` | 从堆栈中移除照片 |

### 3.2 创建堆栈

**代码位置**：`server/src/services/stack.service.ts:20-28`

```typescript
async create(auth: AuthDto, dto: StackCreateDto): Promise<StackResponseDto> {
  await this.requireAccess({ auth, permission: Permission.AssetUpdate, ids: dto.assetIds });

  const stack = await this.stackRepository.create({ ownerId: auth.user.id }, dto.assetIds);

  await this.eventRepository.emit('StackCreate', { stackId: stack.id, userId: auth.user.id });

  return mapStack(stack, { auth });
}
```

**堆栈创建逻辑**：
1. 验证用户对所有照片的更新权限
2. 调用 `stackRepository.create()` 创建堆栈
3. 如果传入的照片 ID 中包含已有堆栈的主照片 ID，会自动合并这些堆栈
4. 触发 `StackCreate` 事件，通知前端更新

### 3.3 堆栈合并逻辑

**代码位置**：`server/src/repositories/stack.repository.ts:63-125`

```typescript
async create(entity: Omit<Insertable<StackTable>, 'primaryAssetId'>, assetIds: string[]) {
  return this.db.transaction().execute(async (tx) => {
    // 查找已有堆栈（以传入的 assetIds 作为主照片的堆栈）
    const stacks = await tx
      .selectFrom('stack')
      .where('stack.ownerId', '=', entity.ownerId)
      .where('stack.primaryAssetId', 'in', assetIds)
      .select('stack.id')
      .select((eb) =>
        jsonArrayFrom(
          eb
            .selectFrom('asset')
            .select('asset.id')
            .whereRef('asset.stackId', '=', 'stack.id')
            .where('asset.deletedAt', 'is', null),
        ).as('assets'),
      )
      .execute();

    // 收集所有要合并的照片 ID
    const uniqueIds = new Set<string>(assetIds);
    for (const stack of stacks) {
      if (stack.assets && stack.assets.length > 0) {
        for (const asset of stack.assets) {
          uniqueIds.add(asset.id);
        }
      }
    }

    // 删除旧堆栈
    if (stacks.length > 0) {
      await tx.deleteFrom('stack').where('id', 'in', stacks.map((stack) => stack.id)).execute();
    }

    // 创建新堆栈（第一张照片作为主照片）
    const newRecord = await tx
      .insertInto('stack')
      .values({ ...entity, primaryAssetId: assetIds[0] })
      .returning('id')
      .executeTakeFirstOrThrow();

    // 更新所有照片的 stackId
    await tx
      .updateTable('asset')
      .set({ stackId: newRecord.id, updatedAt: new Date() })
      .where('id', 'in', [...uniqueIds])
      .execute();

    // 返回新堆栈
    return tx.selectFrom('stack').selectAll('stack').select(withAssets).where('id', '=', newRecord.id).executeTakeFirstOrThrow();
  });
}
```

### 3.4 前端操作入口

#### 3.4.1 重复照片页面堆叠

**代码位置**：`web/src/routes/(user)/utilities/duplicates/[[photos=photos]]/[[assetId=id]]/+page.svelte:126-130`

```typescript
const handleStack = async (duplicateId: string, assets: AssetResponseDto[]) => {
  const assetIds = assets.map((asset) => asset.id);
  await createStack({ stackCreateDto: { assetIds } });
  await updateAssets({ assetBulkUpdateDto: { ids: assetIds, duplicateId: null } });
  duplicates = duplicates.filter((duplicate) => duplicate.duplicateId !== duplicateId);
  await navigateToIndex(duplicatesIndex);
};
```

#### 3.4.2 批量选择堆叠

**代码位置**：`web/src/lib/utils/asset-utils.ts:315-342`

```typescript
export const stackAssets = async (assets: { id: string }[], showNotification = true): Promise<StackResponse> => {
  if (assets.length < 2) {
    return { stack: undefined, toDeleteIds: [] };
  }

  try {
    const stack = await createStack({ stackCreateDto: { assetIds: assets.map(({ id }) => id) } });
    // 显示成功通知
    return { stack, toDeleteIds: assets.slice(1).map((asset) => asset.id) };
  } catch (error) {
    handleError(error, $t('errors.failed_to_stack_assets'));
    return { stack: undefined, toDeleteIds: [] };
  }
};
```

#### 3.4.3 照片查看器操作

在照片查看器中，用户可以进行以下堆栈相关操作：

| 操作 | 代码位置 | 说明 |
|------|----------|------|
| 添加照片到堆栈 | `AddToStackAction.svelte` | 上传新照片并添加到当前堆栈 |
| 设为主照片 | `SetStackPrimaryAsset.svelte` | 将当前照片设为堆栈主照片 |
| 从堆栈移除 | `RemoveAssetFromStack.svelte` | 将当前照片从堆栈中移除 |
| 取消堆栈 | `UnstackAction.svelte` | 解散整个堆栈 |
| 保留此张删除其他 | `KeepThisDeleteOthers.svelte` | 保留当前照片，删除堆栈中其他照片 |

### 3.5 事件通知

堆栈操作会通过 WebSocket 实时通知前端：

**代码位置**：`server/src/services/notification.service.ts:187-205`

```typescript
@OnEvent({ name: 'StackCreate' })
onStackCreate({ userId }: ArgOf<'StackCreate'>) {
  this.websocketRepository.clientSend('on_asset_stack_update', userId);
}

@OnEvent({ name: 'StackUpdate' })
onStackUpdate({ userId }: ArgOf<'StackUpdate'>) {
  this.websocketRepository.clientSend('on_asset_stack_update', userId);
}

@OnEvent({ name: 'StackDelete' })
onStackDelete({ userId }: ArgOf<'StackDelete'>) {
  this.websocketRepository.clientSend('on_asset_stack_update', userId);
}
```

---

## 四、数据模型

### 4.1 Stack 表结构

**代码位置**：`server/src/schema/tables/stack.table.ts`

| 字段 | 类型 | 说明 |
|------|------|------|
| `id` | UUID | 堆栈 ID（主键） |
| `ownerId` | UUID | 所有者用户 ID |
| `primaryAssetId` | UUID | 主照片 ID |
| `createdAt` | Timestamp | 创建时间 |
| `updatedAt` | Timestamp | 更新时间 |
| `updateId` | UUID | 更新 ID（用于同步） |

### 4.2 Asset 表关联

**代码位置**：`server/src/schema/tables/asset.table.ts:125-126`

```typescript
@ForeignKeyColumn(() => StackTable, { nullable: true, onDelete: 'SET NULL', onUpdate: 'CASCADE' })
stackId!: string | null;
```

---

## 五、关键发现与设计分析

### 5.1 自动分组功能现状

- `autoStackId` 字段已完整实现提取和存储
- **但自动基于 `autoStackId` 创建堆栈的逻辑尚未实现**
- 目前所有堆栈都需要用户手动创建

### 5.2 设计亮点

1. **可扩展的信号系统**：`autoStackId` 支持多种 EXIF 标签，兼容不同品牌相机的连拍格式
2. **合并机制**：创建堆栈时自动合并已有堆栈，避免碎片化
3. **实时通知**：通过 WebSocket 实时推送堆栈变更
4. **权限控制**：细粒度的堆栈权限（StackRead/StackCreate/StackUpdate/StackDelete）

### 5.3 可能的改进方向

1. 实现基于 `autoStackId` 的自动堆栈创建任务
2. 添加基于时间窗口（如 1 秒内）的自动分组作为补充信号
3. 支持基于视觉相似度的智能分组（可结合已有的重复照片检测）
4. 提供自动分组的开关和敏感度设置

---

## 六、相关文件索引

| 功能模块 | 文件路径 |
|----------|----------|
| 堆栈服务 | `server/src/services/stack.service.ts` |
| 堆栈仓库 | `server/src/repositories/stack.repository.ts` |
| 堆栈控制器 | `server/src/controllers/stack.controller.ts` |
| 元数据服务（autoStackId 提取） | `server/src/services/metadata.service.ts` |
| 资产仓库（时间线查询） | `server/src/repositories/asset.repository.ts` |
| 通知服务（WebSocket 推送） | `server/src/services/notification.service.ts` |
| 堆栈表定义 | `server/src/schema/tables/stack.table.ts` |
| 资产表定义 | `server/src/schema/tables/asset.table.ts` |
| EXIF 表定义 | `server/src/schema/tables/asset-exif.table.ts` |
| 前端工具函数 | `web/src/lib/utils/asset-utils.ts` |
| 前端堆栈操作组件 | `web/src/lib/components/asset-viewer/actions/` |
