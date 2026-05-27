# Immich 照片堆栈（Stack）分组逻辑分析 v5

## 概述

Immich 的堆栈（Stack）功能用于将同一时间点拍摄的多张照片合并成一组，在时间线上只显示主照片，用户可以展开查看组内其他照片。本文档从代码层面梳理堆栈的判定信号、自动分组、手动覆盖的完整流程，以及状态迁移和事件通知机制。

**v5 版本修正点**：
- 精确区分软删除与硬删除的执行路径
- 澄清外键级联 `ON DELETE SET NULL` 的触发方向和条件
- 分析非主照片删除时的真实行为及其对事件通知的影响

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

| 字段 | 位置 | 用途 |
|------|------|------|
| `localDateTime` | `asset` 表 | 照片拍摄的本地时间（用于时间线分组） |
| `fileCreatedAt` | `asset` 表 | 文件创建时间 |
| `livePhotoCID` | `asset_exif` 表 | 实况照片内容 ID（用于关联照片和视频） |

---

## 二、autoStackId 的消费链路（Consumption Pipeline）

### 2.1 查询消费（Query Consumption）

`autoStackId` 字段在多个查询场景中被读取和返回，但**不会用于自动创建堆栈**。

#### 2.1.1 资产查询
**代码位置**：`server/src/repositories/asset.repository.ts:273`

```typescript
autoStackId: ref('autoStackId'),
```

在 `getById`、`getByIds`、`getUserAssets` 等查询中，`autoStackId` 会作为 EXIF 信息的一部分被返回给前端。

#### 2.1.2 工作流查询
**代码位置**：`server/src/repositories/workflow.repository.ts:169`

```typescript
'asset_exif.autoStackId',
```

工作流（Workflow）系统可以读取 `autoStackId` 字段，理论上可以基于该字段实现自动堆栈工作流，但当前版本未实现。

#### 2.1.3 共享链接查询
**代码位置**：`server/src/queries/shared.link.repository.sql:19, 104`

在共享链接查询中，`autoStackId` 也会被包含在 EXIF 信息中返回。

### 2.2 同步消费（Sync Consumption）

`autoStackId` 字段通过 `AssetExif` 同步实体在多设备间同步。

**同步类型**：`SyncRequestType.AssetExifsV1`、`SyncRequestType.AlbumAssetExifsV1`、`SyncRequestType.PartnerAssetExifsV1`

**同步列定义**：`server/src/database.ts:417-438`
```typescript
syncAssetExif: [
  'asset_exif.assetId',
  'asset_exif.description',
  ...
  'asset_exif.autoStackId',
  ...
],
```

**同步流程**：
1. 当 `asset_exif` 表更新时，数据库触发器记录变更到审计表
2. 同步服务（`sync.service.ts`）通过 `getUpserts` 读取变更
3. 客户端接收 `SyncEntityType.AssetExifV1` 类型的同步数据
4. 客户端更新本地 `autoStackId` 字段

### 2.3 工作流消费（Workflow Consumption）

**代码位置**：`packages/plugin-sdk/src/types.ts:116`

```typescript
autoStackId: string | null;
```

插件 SDK 暴露了 `autoStackId` 字段，允许第三方工作流插件：
1. 读取资产的 `autoStackId`
2. 基于 `autoStackId` 实现自定义分组逻辑
3. 通过 API 手动创建堆栈

**关键发现**：截至当前代码版本，**系统仅提取和存储 `autoStackId`，但没有任何内置的自动堆栈创建逻辑**。所有堆栈都需要用户手动创建或通过第三方工作流插件实现。

---

## 三、自动分组流程（Automatic Grouping Process）

### 3.1 元数据提取阶段

当照片上传到 Immich 后，系统会执行元数据提取任务：

**代码位置**：`server/src/services/metadata.service.ts:235-416`

**流程**：
1. 触发 `JobName.AssetExtractMetadata` 任务
2. 读取照片 EXIF 标签
3. 调用 `getAutoStackId()` 提取自动堆栈 ID
4. 将 `autoStackId` 写入 `asset_exif` 表
5. 触发 `AssetMetadataExtracted` 事件

### 3.2 时间线分组显示

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

### 3.3 时间桶（Time Bucket）分组

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

## 四、手动覆盖流程（Manual Override Process）

### 4.1 堆栈 API 接口

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

### 4.2 创建堆栈

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

### 4.3 堆栈合并逻辑（Stack Merge Logic）

**代码位置**：`server/src/repositories/stack.repository.ts:63-125`

```typescript
async create(entity: Omit<Insertable<StackTable>, 'primaryAssetId'>, assetIds: string[]) {
  return this.db.transaction().execute(async (tx) => {
    // 步骤1: 查找已有堆栈（以传入的 assetIds 作为主照片的堆栈）
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

    // 步骤2: 收集所有要合并的照片 ID（去重）
    const uniqueIds = new Set<string>(assetIds);
    for (const stack of stacks) {
      if (stack.assets && stack.assets.length > 0) {
        for (const asset of stack.assets) {
          uniqueIds.add(asset.id);
        }
      }
    }

    // 步骤3: 删除旧堆栈（被合并的堆栈）
    if (stacks.length > 0) {
      await tx
        .deleteFrom('stack')
        .where('id', 'in', stacks.map((stack) => stack.id))
        .execute();
    }

    // 步骤4: 创建新堆栈（第一张照片作为主照片）
    const newRecord = await tx
      .insertInto('stack')
      .values({ ...entity, primaryAssetId: assetIds[0] })
      .returning('id')
      .executeTakeFirstOrThrow();

    // 步骤5: 更新所有照片的 stackId
    await tx
      .updateTable('asset')
      .set({
        stackId: newRecord.id,
        updatedAt: new Date(),
      })
      .where('id', 'in', [...uniqueIds])
      .execute();

    // 步骤6: 返回新堆栈（包含所有成员照片）
    return tx
      .selectFrom('stack')
      .selectAll('stack')
      .select(withAssets)
      .where('id', '=', newRecord.id)
      .executeTakeFirstOrThrow();
  });
}
```

**合并触发条件**：当创建堆栈时，如果传入的 `assetIds` 中包含任何已有堆栈的 `primaryAssetId`，则会触发合并。

**合并结果**：
- 被合并的旧堆栈被删除
- 所有成员照片的 `stackId` 更新为新堆栈 ID
- 新堆栈以 `assetIds[0]` 作为主照片

### 4.4 主照片更新（Primary Asset Update）

**代码位置**：`server/src/services/stack.service.ts:36-48`

```typescript
async update(auth: AuthDto, id: string, dto: StackUpdateDto): Promise<StackResponseDto> {
  await this.requireAccess({ auth, permission: Permission.StackUpdate, ids: [id] });
  const stack = await this.findOrFail(id);
  if (dto.primaryAssetId && !stack.assets.some(({ id }) => id === dto.primaryAssetId)) {
    throw new BadRequestException('Primary asset must be in the stack');
  }

  const updatedStack = await this.stackRepository.update(id, { id, primaryAssetId: dto.primaryAssetId });

  await this.eventRepository.emit('StackUpdate', { stackId: id, userId: auth.user.id });

  return mapStack(updatedStack, { auth });
}
```

**约束**：新的主照片必须是堆栈的成员之一。

### 4.5 成员移除（Asset Removal）

**代码位置**：`server/src/services/stack.service.ts:62-78`

```typescript
async removeAsset(auth: AuthDto, dto: UUIDAssetIDParamDto): Promise<void> {
  const { id: stackId, assetId } = dto;
  await this.requireAccess({ auth, permission: Permission.StackUpdate, ids: [stackId] });

  const stack = await this.stackRepository.getForAssetRemoval(assetId);

  if (!stack?.id || stack.id !== stackId) {
    throw new BadRequestException('Asset not in stack');
  }

  if (stack.primaryAssetId === assetId) {
    throw new BadRequestException("Cannot remove stack's primary asset");
  }

  await this.assetRepository.update({ id: assetId, stackId: null });
  await this.eventRepository.emit('StackUpdate', { stackId, userId: auth.user.id });
}
```

**⚠️ 关键澄清**：
- 成员移除是**服务层主动更新**（`assetRepository.update({ id: assetId, stackId: null })`）
- 不是数据库级联删除
- 会显式触发 `StackUpdate` 事件

**关键约束**：
- 不能移除主照片（必须先指定其他主照片）
- 只能移除属于该堆栈的照片

### 4.6 堆栈删除（Stack Deletion）

**代码位置**：`server/src/services/stack.service.ts:50-60`

```typescript
async delete(auth: AuthDto, id: string): Promise<void> {
  await this.requireAccess({ auth, permission: Permission.StackDelete, ids: [id] });
  await this.stackRepository.delete(id);
  await this.eventRepository.emit('StackDelete', { stackId: id, userId: auth.user.id });
}

async deleteAll(auth: AuthDto, dto: BulkIdsDto): Promise<void> {
  await this.requireAccess({ auth, permission: Permission.StackDelete, ids: dto.ids });
  await this.stackRepository.deleteAll(dto.ids);
  await this.eventRepository.emit('StackDeleteAll', { stackIds: dto.ids, userId: auth.user.id });
}
```

**⚠️ 外键级联行为（非常重要）**：
- 外键定义：`asset.stackId` → `stack.id`，`ON DELETE SET NULL`
- **当堆栈被删除时**（删除 `stack` 表中的行），触发级联
- 所有成员的 `asset.stackId` 被自动设置为 `NULL`
- 成员照片本身不会被删除

---

## 五、状态迁移链路（State Migration Pipeline）

### 5.1 资产删除的两种路径

#### 5.1.1 软删除（Soft Delete）- 进入回收站

**代码位置**：`server/src/services/asset.service.ts:370-382`

```typescript
async deleteAll(auth: AuthDto, dto: AssetBulkDeleteDto): Promise<void> {
  const { ids, force } = dto;

  await this.requireAccess({ auth, permission: Permission.AssetDelete, ids });
  await this.assetRepository.updateAll(ids, {
    deletedAt: new Date(),
    status: force ? AssetStatus.Deleted : AssetStatus.Trashed,
  });
  await this.eventRepository.emit(force ? 'AssetDeleteAll' : 'AssetTrashAll', {
    assetIds: ids,
    userId: auth.user.id,
  });
}
```

**行为**：
- 执行 `UPDATE asset SET deletedAt = NOW(), status = 'TRASHED'`
- 资产行仍然存在于数据库中
- **不会触发外键级联**（因为没有 DELETE 操作）
- `asset.stackId` 保持不变
- 堆栈查询时自动过滤 `deletedAt IS NOT NULL` 的资产

#### 5.1.2 硬删除（Hard Delete）- 永久删除

**代码位置**：`server/src/services/asset.service.ts:307-337`

```typescript
@OnJob({ name: JobName.AssetDelete, queue: QueueName.BackgroundTask })
async handleAssetDeletion(job: JobOf<JobName.AssetDelete>): Promise<JobStatus> {
  const { id, deleteOnDisk } = job;
  const asset = await this.assetJobRepository.getForAssetDeletion(id);

  if (!asset) {
    return JobStatus.Failed;
  }

  // 如果被删除的是堆栈的主照片
  if (asset.stack?.primaryAssetId === id) {
    const stackAssetIds = asset.stack.assets.map((a) => a.id);
    if (stackAssetIds.length >= 2) {
      const newPrimaryAssetId = stackAssetIds.find((a) => a !== id)!;
      await this.stackRepository.update(asset.stack.id, {
        id: asset.stack.id,
        primaryAssetId: newPrimaryAssetId,
      });
    } else {
      await this.stackRepository.delete(asset.stack.id);
    }
  }

  await this.assetRepository.remove(asset);  // 硬删除
  await this.eventRepository.emit('AssetDelete', { assetId: id, userId: asset.ownerId });
  // ... 后续文件清理逻辑
}
```

**代码位置**：`server/src/repositories/asset.repository.ts:639-641`

```typescript
async remove(asset: { id: string }): Promise<void> {
  await this.db.deleteFrom('asset').where('id', '=', asUuid(asset.id)).execute();
}
```

**行为**：
- 执行 `DELETE FROM asset WHERE id = ?`
- 资产行被永久删除
- 触发后续的文件删除任务

### 5.2 主照片删除时的状态迁移

**代码位置**：`server/src/services/asset.service.ts:317-330`

```typescript
// 如果被删除的是堆栈的主照片
if (asset.stack?.primaryAssetId === id) {
  // 获取堆栈中其他成员（排除已删除的主照片）
  const stackAssetIds = asset.stack.assets.map((a) => a.id);
  
  if (stackAssetIds.length >= 2) {
    // 还有其他成员：自动选择新的主照片（第一个非主照片成员）
    const newPrimaryAssetId = stackAssetIds.find((a) => a !== id)!;
    await this.stackRepository.update(asset.stack.id, {
      id: asset.stack.id,
      primaryAssetId: newPrimaryAssetId,
    });
  } else {
    // 只有主照片一个成员：删除整个堆栈
    await this.stackRepository.delete(asset.stack.id);
  }
}
```

**⚠️ 关键澄清**：
- 主照片被删除时，调用的是 `this.stackRepository.update()` 和 `this.stackRepository.delete()`
- 这是**直接调用 repository 层**，不经过 service 层
- 因此**不会触发 `StackUpdate` 或 `StackDelete` 事件**！
- 只会触发 `AssetDelete` 事件

**状态迁移规则**：
| 堆栈状态 | 主照片删除后行为 | 触发事件 |
|---------|----------------|---------|
| 只有主照片 1 个成员 | 整个堆栈被删除 | `AssetDelete`（无 Stack 事件） |
| 有多个成员 | 自动选择第一个非主照片成员作为新主照片 | `AssetDelete`（无 Stack 事件） |

### 5.3 非主照片成员删除时的状态迁移

**⚠️ 核心修正 - 非主照片删除的真实行为**：

**外键关系图**：
```
stack 表（父表）
  └── id (主键)
       ↑
       │ FOREIGN KEY (asset.stackId) REFERENCES stack(id) ON DELETE SET NULL
       │
asset 表（子表）
  ├── id (主键)
  └── stackId (外键，可为空)
```

**外键级联触发条件**：
- ✅ **父表行被删除**（删除 `stack` 表中的行）→ 触发 `ON DELETE SET NULL` → 所有子表行的 `stackId` 置空
- ❌ **子表行被删除**（删除 `asset` 表中的行）→ **不触发任何级联** → 父表和其他子表行不受影响

**非主照片成员删除的真实流程**：

1. **软删除（进入回收站）**：
   - 执行 `UPDATE asset SET deletedAt = NOW() WHERE id = ?`
   - 资产行仍然存在
   - `asset.stackId` 保持不变
   - 堆栈查询时自动过滤 `deletedAt IS NOT NULL` 的资产
   - **不触发任何堆栈事件**

2. **硬删除（永久删除）**：
   - 执行 `DELETE FROM asset WHERE id = ?`
   - 资产行被永久删除
   - **删除的是子表行，不触发外键级联**
   - 堆栈的其他成员的 `stackId` 保持不变
   - 堆栈本身不受影响
   - **不触发任何堆栈事件**（只有 `AssetDelete` 事件）

**堆栈成员查询过滤**：`server/src/repositories/stack.repository.ts:16-46`

```typescript
const withAssets = (eb: ExpressionBuilder<DB, 'stack'>, withTags = false) => {
  return jsonArrayFrom(
    eb
      .selectFrom('asset')
      .selectAll('asset')
      // ...
      .where('asset.deletedAt', 'is', null)  // ← 自动过滤已软删除的资产
      .whereRef('asset.stackId', '=', 'stack.id')
      .$call(withDefaultVisibility),
  ).as('assets');
};
```

### 5.4 资产复制时的堆栈迁移

**代码位置**：`server/src/services/asset.service.ts:231-245`

```typescript
private async copyStack({
  sourceAsset,
  targetAsset,
}: {
  sourceAsset: { id: string; stackId: string | null };
  targetAsset: { id: string; stackId: string | null };
}) {
  if (!sourceAsset.stackId) {
    return;
  }

  if (targetAsset.stackId) {
    // 目标已有堆栈：合并两个堆栈
    await this.stackRepository.merge({ sourceId: sourceAsset.stackId, targetId: targetAsset.stackId });
    await this.stackRepository.delete(sourceAsset.stackId);
  } else {
    // 目标没有堆栈：加入源堆栈
    await this.assetRepository.update({ id: targetAsset.id, stackId: sourceAsset.stackId });
  }
}
```

### 5.5 堆栈合并的状态迁移

**代码位置**：`server/src/repositories/stack.repository.ts:166-169`

```typescript
merge({ sourceId, targetId }: { sourceId: string; targetId: string }) {
  return this.db
    .updateTable('asset')
    .set({ stackId: targetId })
    .where('asset.stackId', '=', sourceId)
    .execute();
}
```

**合并行为**：
- 源堆栈的所有成员移动到目标堆栈
- 源堆栈需要单独调用 `delete` 方法删除
- 目标堆栈的主照片保持不变

---

## 六、事件通知链路（Event Notification Pipeline）

### 6.1 WebSocket 事件传递机制详解

#### 6.1.1 clientSend 方法签名

**代码位置**：`server/src/repositories/websocket.repository.ts:96-98`

```typescript
clientSend<T extends keyof ClientEventMap>(event: T, room: string, ...data: ClientEventMap[T]) {
  this.server?.to(room).emit(event, ...data);
}
```

**参数说明**：
- `event`：WebSocket 事件名称
- `room`：发送目标房间（通常是用户 ID）
- `...data`：Rest 参数，展开后传递给 socket.io 的 `emit`

#### 6.1.2 ClientEventMap 完整类型定义

**代码位置**：`server/src/repositories/websocket.repository.ts:22-41`

```typescript
export interface ClientEventMap {
  on_upload_success: [AssetResponseDto];
  on_user_delete: [string];
  on_asset_delete: [string];
  on_asset_trash: [string[]];
  on_asset_update: [AssetResponseDto];
  on_asset_hidden: [string];
  on_asset_restore: [string[]];
  on_asset_stack_update: string[];       // ⚠️ 唯一使用 string[] 而非 [string[]] 的事件
  on_person_thumbnail: [string];
  on_server_version: [ServerVersionResponseDto];
  on_config_update: [];
  on_new_release: [ReleaseNotification];
  on_notification: [NotificationDto];
  on_session_delete: [string];

  AssetUploadReadyV2: [{ asset: SyncAssetV2; exif: SyncAssetExifV1 }];
  AppRestartV1: [AppRestartEvent];
  AssetEditReadyV2: [{ asset: SyncAssetV2; edit: SyncAssetEditV1[] }];
}
```

#### 6.1.3 on_asset_stack_update 精确分析

**实际调用代码**（4 处调用全部相同）：
```typescript
// notification.service.ts:189, 194, 199, 204
this.websocketRepository.clientSend('on_asset_stack_update', userId);
```

**参数展开过程**：
1. `event = 'on_asset_stack_update'`
2. `room = userId`
3. `...data = []`（没有额外参数）
4. 最终执行：`this.server?.to(userId).emit('on_asset_stack_update')` — **没有传递任何 payload**

**socket.io 行为**：
- `emit('on_asset_stack_update')` 只发送事件名称，没有数据
- 前端回调被调用时**完全没有参数**
- 回调参数 `assetIds` 的值是 `undefined`（不是空数组 `[]`）

### 6.2 操作与事件映射表（最终版）

| 操作 | 调用路径 | 触发事件 | WebSocket 通知 | 前端感知方式 |
|------|---------|---------|---------------|-------------|
| 创建堆栈 | `StackService.create()` | `StackCreate` | ✅ `on_asset_stack_update` | API 调用后本地更新 |
| 更新主照片 | `StackService.update()` | `StackUpdate` | ✅ `on_asset_stack_update` | API 调用后本地更新 |
| 移除成员 | `StackService.removeAsset()` | `StackUpdate` | ✅ `on_asset_stack_update` | API 调用后本地更新 |
| 删除单个堆栈 | `StackService.delete()` | `StackDelete` | ✅ `on_asset_stack_update` | API 调用后本地更新 |
| 批量删除堆栈 | `StackService.deleteAll()` | `StackDeleteAll` | ✅ `on_asset_stack_update` | API 调用后本地更新 |
| **主照片被删除（自动选新主照片）** | `AssetService.handleAssetDeletion()` → `stackRepository.update()` | ❌ 无 Stack 事件 | ❌ 无 WebSocket 通知 | `on_asset_delete` 事件 |
| **主照片被删除（堆栈只剩 1 人）** | `AssetService.handleAssetDeletion()` → `stackRepository.delete()` | ❌ 无 Stack 事件 | ❌ 无 WebSocket 通知 | `on_asset_delete` 事件 |
| **非主照片软删除（进入回收站）** | `AssetService.deleteAll()` → `UPDATE asset SET deletedAt` | ❌ 无 Stack 事件 | ❌ 无 WebSocket 通知 | `on_asset_trash` 事件 |
| **非主照片硬删除（永久删除）** | `AssetService.handleAssetDeletion()` → `DELETE FROM asset` | ❌ 无 Stack 事件 | ❌ 无 WebSocket 通知 | `on_asset_delete` 事件 |

### 6.3 事件缺失的影响分析

#### 6.3.1 对前端的影响

**当前前端更新策略**：
- 用户主动操作（创建/删除/更新堆栈）→ API 调用后立即本地更新
- 用户主动操作（删除资产）→ API 调用后立即本地更新
- 其他设备操作 → 依赖 WebSocket 事件或页面刷新

**问题场景**：

1. **设备 A 删除主照片，设备 B 浏览同一堆栈**：
   - 设备 A：调用删除 API → 本地移除该照片 → 触发 `AssetDelete` 事件
   - 设备 B：收到 `on_asset_delete` 事件 → 移除该照片
   - 但设备 B 不知道堆栈的主照片已变更 → 可能显示 stale 的主照片

2. **设备 A 删除非主照片，设备 B 浏览同一堆栈**：
   - 设备 A：调用删除 API → 本地移除该照片 → 触发 `AssetDelete` 事件
   - 设备 B：收到 `on_asset_delete` 事件 → 移除该照片
   - 堆栈计数在设备 B 上可能不准确，直到下次刷新

3. **非主照片软删除**：
   - 堆栈查询自动过滤 `deletedAt IS NOT NULL` 的资产
   - 但前端没有收到堆栈变更通知 → 计数可能显示错误

#### 6.3.2 对同步的影响

堆栈的同步通过以下机制：
1. 数据库触发器写入 `stack_audit` 表
2. 同步服务读取审计表
3. 客户端接收 `SyncEntityType.StackV1` 或 `StackDeleteV1` 事件

**潜在问题**：
- 主照片被删除时，`stackRepository.update()` 更新了 `primaryAssetId`
- 这会更新 `stack.updatedAt` 和 `stack.updateId`
- 同步服务会检测到变更并同步
- 但 WebSocket 通知缺失，实时性受影响

### 6.4 前端订阅分析

**前端类型定义**：`web/src/lib/stores/websocket.ts:33`
```typescript
export interface Events {
  // ...
  on_asset_stack_update: (assetIds: string[]) => void;
  // ...
}
```

**⚠️ 关键发现 - 前端未实际订阅**：
- 虽然定义了 `on_asset_stack_update` 事件类型
- 但前端代码中**没有任何地方实际订阅该事件**
- 前端的堆栈更新主要通过以下方式：
  1. API 调用后的本地状态更新（如 `updateStackedAssetInTimeline`）
  2. 页面刷新时重新获取数据
  3. 依赖 `on_asset_delete` 事件间接感知

### 6.5 统一修正建议

#### 6.5.1 修复主照片删除时的事件通知

```typescript
// server/src/services/asset.service.ts:317-330
if (asset.stack?.primaryAssetId === id) {
  const stackAssetIds = asset.stack.assets.map((a) => a.id);
  if (stackAssetIds.length >= 2) {
    const newPrimaryAssetId = stackAssetIds.find((a) => a !== id)!;
    await this.stackRepository.update(asset.stack.id, {
      id: asset.stack.id,
      primaryAssetId: newPrimaryAssetId,
    });
    // 新增：触发 StackUpdate 事件
    await this.eventRepository.emit('StackUpdate', {
      stackId: asset.stack.id,
      userId: asset.ownerId,
    });
  } else {
    await this.stackRepository.delete(asset.stack.id);
    // 新增：触发 StackDelete 事件
    await this.eventRepository.emit('StackDelete', {
      stackId: asset.stack.id,
      userId: asset.ownerId,
    });
  }
}
```

#### 6.5.2 修复 WebSocket payload 不一致

**方案 A**（推荐）：保持无 payload，修改类型定义

```typescript
// server/src/repositories/websocket.repository.ts
export interface ClientEventMap {
  // ...
  on_asset_stack_update: [];  // 改为空元组
  // ...
}

// web/src/lib/stores/websocket.ts
export interface Events {
  // ...
  on_asset_stack_update: () => void;  // 改为无参数回调
  // ...
}
```

**方案 B**：传递堆栈 ID 数组，修改调用方式

```typescript
// server/src/services/notification.service.ts
@OnEvent({ name: 'StackCreate' })
onStackCreate({ userId, stackId }: ArgOf<'StackCreate'>) {
  this.websocketRepository.clientSend('on_asset_stack_update', userId, [stackId]);
}
```

### 6.6 同步审计机制（Sync Audit）

**数据库触发器**：`server/src/schema/migrations/1752267649968-StandardizeNames.ts:148-158`

```sql
CREATE OR REPLACE FUNCTION stack_delete_audit()
RETURNS TRIGGER
LANGUAGE PLPGSQL
AS $$
  BEGIN
    INSERT INTO stack_audit ("stackId", "userId")
    SELECT "id", "ownerId"
    FROM OLD;
    RETURN NULL;
  END
$$;
```

**同步流程**：
1. 堆栈变更时，数据库触发器写入 `stack_audit` 表
2. 同步服务（`sync.service.ts`）通过 `StackSync.getDeletes()` 和 `getUpserts()` 读取变更
3. 客户端接收 `SyncEntityType.StackV1` 或 `StackDeleteV1` 同步事件
4. 客户端更新本地堆栈状态

---

## 七、前端操作入口

### 7.1 重复照片页面堆叠

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

### 7.2 批量选择堆叠

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

### 7.3 照片查看器操作

在照片查看器中，用户可以进行以下堆栈相关操作：

| 操作 | 代码位置 | 说明 |
|------|----------|------|
| 添加照片到堆栈 | `AddToStackAction.svelte` | 上传新照片并添加到当前堆栈 |
| 设为主照片 | `SetStackPrimaryAsset.svelte` | 将当前照片设为堆栈主照片 |
| 从堆栈移除 | `RemoveAssetFromStack.svelte` | 将当前照片从堆栈中移除 |
| 取消堆栈 | `UnstackAction.svelte` | 解散整个堆栈 |
| 保留此张删除其他 | `KeepThisDeleteOthers.svelte` | 保留当前照片，删除堆栈中其他照片 |

---

## 八、数据模型

### 8.1 Stack 表结构

**代码位置**：`server/src/schema/tables/stack.table.ts`

| 字段 | 类型 | 说明 |
|------|------|------|
| `id` | UUID | 堆栈 ID（主键） |
| `ownerId` | UUID | 所有者用户 ID |
| `primaryAssetId` | UUID | 主照片 ID |
| `createdAt` | Timestamp | 创建时间 |
| `updatedAt` | Timestamp | 更新时间 |
| `updateId` | UUID | 更新 ID（用于同步） |

### 8.2 Asset 表关联

**代码位置**：`server/src/schema/tables/asset.table.ts:125-126`

```typescript
@ForeignKeyColumn(() => StackTable, { nullable: true, onDelete: 'SET NULL', onUpdate: 'CASCADE' })
stackId!: string | null;
```

**⚠️ 外键级联方向澄清**：
- 父表：`stack`（被引用的表）
- 子表：`asset`（引用的表）
- 级联触发：只有当 `stack` 表中的行被删除时，才会触发 `ON DELETE SET NULL`
- 当 `asset` 表中的行被删除时，不触发任何级联

### 8.3 数据库约束与索引

| 约束/索引 | 作用 |
|-----------|------|
| `REL_91704e101438fd0653f582426d` (UNIQUE) | 一张照片只能作为一个堆栈的主照片 |
| `FK_f15d48fa3ea5e4bda05ca8ab207` (FOREIGN KEY) | `asset.stackId` 外键，删除堆栈时自动置空 |
| `IDX_f15d48fa3ea5e4bda05ca8ab207` | 加速按 `stackId` 查询成员照片 |
| `IDX_auto_stack_id` | 加速按 `autoStackId` 查询照片 |

---

## 九、关键发现与设计分析

### 9.1 自动分组功能现状

- ✅ `autoStackId` 字段已完整实现提取和存储
- ✅ `autoStackId` 通过 EXIF 同步在多设备间同步
- ✅ `autoStackId` 可通过工作流插件访问
- ❌ **没有内置的自动堆栈创建逻辑**
- ❌ 没有基于时间窗口的自动分组
- ❌ 没有基于视觉相似度的自动分组

### 9.2 设计亮点

1. **可扩展的信号系统**：`autoStackId` 支持多种 EXIF 标签，兼容不同品牌相机的连拍格式
2. **合并机制**：创建堆栈时自动合并已有堆栈，避免碎片化
3. **实时通知**：通过 WebSocket 实时推送堆栈变更（虽然 payload 有问题）
4. **权限控制**：细粒度的堆栈权限（StackRead/StackCreate/StackUpdate/StackDelete）
5. **主照片故障转移**：主照片被删除时自动选择新主照片，保证堆栈完整性
6. **同步审计**：完整的审计表和同步机制，确保多设备堆栈状态一致
7. **软删除过滤**：堆栈查询自动过滤已软删除的资产，保持数据一致性

### 9.3 潜在问题与 Bug

1. **⚠️ 主照片删除不通知**：主照片被删除时，直接调用 repository 层更新，不触发 Stack 事件
2. **⚠️ 非主照片删除不通知**：无论软删除还是硬删除，都不触发 Stack 事件
3. **⚠️ WebSocket payload 不一致**：`on_asset_stack_update` 类型定义与实际实现不符
4. **⚠️ 前端未订阅 WebSocket 事件**：虽然定义了事件类型，但前端没有实际订阅
5. **合并后主照片不可控**：合并堆栈时，新堆栈的主照片固定为 `assetIds[0]`

### 9.4 可能的改进方向

1. 实现基于 `autoStackId` 的自动堆栈创建任务
2. 添加基于时间窗口（如 1 秒内）的自动分组作为补充信号
3. 支持基于视觉相似度的智能分组（可结合已有的重复照片检测）
4. 提供自动分组的开关和敏感度设置
5. **修复主照片删除时的事件通知**：在 `asset.service.ts` 中显式触发 Stack 事件
6. **修复 WebSocket payload**：统一类型定义与实际实现
7. 合并堆栈时允许用户选择保留哪个主照片

---

## 十、相关文件索引

| 功能模块 | 文件路径 |
|----------|----------|
| 堆栈服务 | `server/src/services/stack.service.ts` |
| 堆栈仓库 | `server/src/repositories/stack.repository.ts` |
| 堆栈控制器 | `server/src/controllers/stack.controller.ts` |
| 元数据服务（autoStackId 提取） | `server/src/services/metadata.service.ts` |
| 资产服务（删除处理） | `server/src/services/asset.service.ts` |
| 资产仓库（硬删除实现） | `server/src/repositories/asset.repository.ts` |
| 同步服务 | `server/src/services/sync.service.ts` |
| 同步仓库 | `server/src/repositories/sync.repository.ts` |
| 通知服务（WebSocket 推送） | `server/src/services/notification.service.ts` |
| WebSocket 仓库 | `server/src/repositories/websocket.repository.ts` |
| 堆栈表定义 | `server/src/schema/tables/stack.table.ts` |
| 资产表定义（外键约束） | `server/src/schema/tables/asset.table.ts` |
| EXIF 表定义 | `server/src/schema/tables/asset-exif.table.ts` |
| 数据库列定义 | `server/src/database.ts` |
| 数据库迁移（外键创建） | `server/src/schema/migrations/1744910873969-InitialMigration.ts` |
| 插件 SDK 类型 | `packages/plugin-sdk/src/types.ts` |
| 前端工具函数 | `web/src/lib/utils/asset-utils.ts` |
| 前端操作工具 | `web/src/lib/utils/actions.ts` |
| 前端堆栈操作组件 | `web/src/lib/components/asset-viewer/actions/` |
| 前端 WebSocket 类型 | `web/src/lib/stores/websocket.ts` |
| 前端事件管理器 | `web/src/lib/managers/event-manager.svelte.ts` |
