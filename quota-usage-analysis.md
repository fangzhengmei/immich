# Immich 配额与用量分析

本文以代码为准，把 "配额怎么算"、"什么时候加/减"、"什么时候报警" 这三个问题串起来说明。配额字段存元数据里（`user` 表），真正的文件占用属于存储层（磁盘上的原图、缩略图、编码视频等），但 Immich 只把**原图字节数**计入用户配额，统计口径在元数据侧（`asset_exif.fileSizeInByte`）与用户表的缓存字段（`user.quotaUsageInBytes`）之间来回维护。

## 1. 数据模型与字段来源

### 1.1 `user` 表上的两个核心字段

文件：`server/src/schema/tables/user.table.ts`

| 字段 | 类型 | 含义 |
| --- | --- | --- |
| `quotaSizeInBytes` | `bigint \| null` | 配额上限。`null` 表示不限额。 |
| `quotaUsageInBytes` | `bigint` | 已用字节数，运行时缓存值，默认 `0`。 |

两者都参与鉴权上下文：在 `server/src/database.ts:342` 把这两个字段加入 `authUser` 投影；任何通过 session / API key / shared link 解析出的 `auth.user` 都会带着这两个字段，供服务代码随时访问。

### 1.2 真实口径：`asset_exif.fileSizeInByte`

`quotaUsageInBytes` 只是一个缓存值。权威口径是所有 `asset` 记录（`deletedAt IS NULL`、`libraryId IS NULL`）对应的 `asset_exif.fileSizeInByte` 之和。
见 `server/src/repositories/user.repository.ts:310` 的 `syncUsage()`：

```sql
UPDATE "user"
SET "quotaUsageInBytes" = (
    SELECT COALESCE(SUM(asset_exif."fileSizeInByte"), 0)
    FROM asset
    LEFT JOIN asset_exif ON asset_exif."assetId" = asset.id
    WHERE asset."libraryId" IS NULL
      AND asset."ownerId" = "user".id
),
"updatedAt" = now()
WHERE "user"."deletedAt" IS NULL;
```

**口径要点**：
- 只统计 `asset.libraryId IS NULL` 的资产（外部挂载 Library 的文件不计入用户配额）。
- 直接使用 `asset_exif.fileSizeInByte`，因此不区分图片/视频/sidecar/缩略图/转码产物——缩略图、转码视频等派生文件的字节数一律不占用配额，只有原始上传文件计入。

## 2. 配额的设置与来源

### 2.1 管理员手动设置

`server/src/services/user-admin.service.ts:53` 的 `update()`：

```ts
if (dto.quotaSizeInBytes && user.quotaSizeInBytes !== dto.quotaSizeInBytes) {
  await this.userRepository.syncUsage(id);
}
```

**关键点**：修改配额上限时会先触发一次按资产重算的 `syncUsage(id)`，把缓存值与真实值对齐后再应用新的上限，避免缓存漂移导致的"刚改完配额就提示超限"问题。

### 2.2 OAuth 自动注册时从 Claim 注入

`server/src/services/auth.service.ts:371`：

```ts
quotaSizeInBytes: storageQuota === null ? null : storageQuota * HumanReadableSize.GiB,
```

从 OAuth 配置的 `storageQuotaClaim` 读取，单位按 GiB 解析；`null` 表示不限额。

## 3. 用量累加：三条路径

`user.quotaUsageInBytes` 的增量更新只走一个方法：`UserRepository.updateUsage(id, delta)`，实现是一条原子 SQL：

```sql
UPDATE "user"
SET "quotaUsageInBytes" = "quotaUsageInBytes" + $1,
    "updatedAt" = now()
WHERE id = $2 AND "deletedAt" IS NULL;
```

（见 `server/src/repositories/user.repository.ts:300`）

调用 `updateUsage` 的地方只有三处，覆盖了所有"改变用户实际占用量"的场景：

| 触发点 | 位置 | 场景 | delta |
| --- | --- | --- | --- |
| `AssetMediaService.uploadAsset` | `server/src/services/asset-media.service.ts:155` | 通过上传接口成功创建资产 | `+file.size` |
| `MetadataService`（运动照片视频提取） | `server/src/services/metadata.service.ts:750` | 从 HEIC/JPEG Motion Photo 中拆出独立的 mp4 资产（`!asset.isExternal` 时） | `+video.byteLength` |
| `AssetService.handleAssetDeletion` | `server/src/services/asset.service.ts:334` | 后台任务真正执行资产删除（仅 `asset.libraryId IS NULL` 时） | `-(asset.exifInfo?.fileSizeInByte \|\| 0)` |

### 3.1 上传路径的完整链路

`AssetMediaService.uploadAsset`（`server/src/services/asset-media.service.ts:127`）的顺序非常关键：

1. `requireAccess`：权限检查。
2. `requireQuota(auth, file.size)`：**先校验**，见第 4 节。
3. `create(...)`：写 `asset` 表、写 `asset_exif.fileSizeInByte`、落盘、触发 `AssetExtractMetadata` 任务。
4. `updateUsage(auth.user.id, file.size)`：写成功后再累加缓存。

这样即使第 4 步之前抛错（磁盘满、校验重复等），缓存也不会提前累加。`handleUploadError` 会在失败时清理磁盘文件，但不反向扣减缓存——因为缓存还没加过。

### 3.2 删除时为何是在 `handleAssetDeletion` 里扣

- 用户侧 `POST /trash` 只是把 `asset.deletedAt` 置为当前时间，并触发 `AssetTrash`/`AssetTrashAll` 事件，**不扣用量**。
- 真正扣减发生在后台任务 `JobName.AssetDelete`（由 `AssetDeleteCheck` 或管理员强制删除派发），对应 `AssetService.handleAssetDeletion`。
- 只有 `asset.libraryId IS NULL` 才扣，与 `syncUsage` 的口径一致。
- 使用 `asset.exifInfo?.fileSizeInByte || 0` 兜底：exif 若缺失就按 0 处理，不致引发崩溃。

### 3.3 运动视频的特殊处理

`metadata.service.ts:729` 分支里，如果拆出的 motion asset 是新创建的，才会 `updateUsage`；如果按 checksum 已存在（`isAssetChecksumConstraint` 捕获），则复用老资产，不再重复累加，避免同一个视频被不同 motion photo 多次计入。

## 4. 配额校验与"超限提示"下发

配额校验只在上传入口做，并且是**前置同步校验**：

`server/src/services/asset-media.service.ts:366`

```ts
private requireQuota(auth: AuthDto, size: number) {
  if (auth.user.quotaSizeInBytes !== null
      && auth.user.quotaSizeInBytes < auth.user.quotaUsageInBytes + size) {
    throw new BadRequestException('Quota has been exceeded!');
  }
}
```

调用时机：`uploadAsset` 第 2 步，在落盘 / 写元数据之前。

### 4.1 校验用的是哪个值

- `auth.user.quotaUsageInBytes` 来自 `session.repository` / `api.key.repository` / `shared.link.repository` 里 `authUser` 投影在**请求开始时**从 `user` 表读到的缓存值。
- 因此这是**乐观检查**：在请求开始到 `updateUsage` 落库之间若有其他请求同时上传，理论上会出现竞态（两个请求都看到相同的 `quotaUsageInBytes`，都通过校验，都成功），最终缓存值会短暂大于上限，但下次 `syncUsage` 会把缓存拉回到真实值，后续上传将被拒。Immich 选择接受这种短暂越界，以避免在每次上传里加数据库事务。

### 4.2 "提示"以什么形式下发

Immich 没有独立的"配额告警"事件/邮件/websocket 推送；所谓的"上限提示"就是上传接口抛出的 HTTP 400：

```
BadRequestException('Quota has been exceeded!')
```

客户端（web / 移动）按通用错误提示向用户呈现。代码里没有 `NotificationService` 监听 `AssetCreate` / `UserSyncUsage` 来发送配额邮件或推送，`NotificationService` 的 `onAssetTrash` / `onAssetDelete` 等 handler 也不涉及配额（`server/src/services/notification.service.ts:151` 起）。

因此"告警下发"的边界应理解为：**同步阻塞上传并返回 400**，没有异步通知通道。

## 5. 缓存对齐机制

`quotaUsageInBytes` 是缓存，真实口径在 `asset_exif.fileSizeInByte`。对齐手段有两类：

### 5.1 主动全量 / 单用户重算

- `UserRepository.syncUsage(id?)`（`server/src/repositories/user.repository.ts:310`）：
  - 传 `id` 时只重算该用户；
  - 不传时全量重算所有 `deletedAt IS NULL` 的用户。
- `UserAdminService.update`（`user-admin.service.ts:61`）：管理员改配额时对该用户重算一次。
- `UserService.handleUserSyncUsage`（`server/src/services/user.service.ts:233`）：处理 `JobName.UserSyncUsage` 任务，全量重算。

### 5.2 定时触发

`server/src/services/queue.service.ts:280` 的 `handleNightlyJobs`：

```ts
if (config.nightlyTasks.syncQuotaUsage) {
  jobs.push({ name: JobName.UserSyncUsage });
}
```

由 system config 的 `nightlyTasks.syncQuotaUsage` 开关控制；开启后，每个维护夜（`NightlyJobs`）队列会派发一次 `UserSyncUsage`，把所有用户的缓存值拉回到和 `asset_exif` 一致。

所以整体策略是：

- 热路径上传 / 删除用 `updateUsage` 做增量维护，保证用户当下看到的"已用空间"基本准确；
- 冷路径用夜间 `syncUsage` 做兜底重算，修复因崩溃、外部库删除、人工 SQL 等造成的漂移；
- 管理员改配额时立即重算一次，避免缓存滞后影响判断。

## 6. 统计展示（给管理后台用）

管理员端展示的"每个用户用量"不直接读 `quotaUsageInBytes`，而是独立聚合：

`server/src/repositories/user.repository.ts:232` 的 `getUserStats()` 直接 `SUM(asset_exif.fileSizeInByte)`（`asset.libraryId IS NULL`），并拆出 `usagePhotos` / `usageVideos`。`ServerService.getStatistics`（`server/src/services/server.service.ts:133`）把这个结果填到 `UsageByUserDto`，但 `quotaSizeInBytes` 直接取 `user.quotaSizeInBytes`——也就是说**展示层的"已用"是实时聚合计，"配额"是 user 表值**，两者的口径并不完全同步，这在夜间 `syncUsage` 之前会有差异。

## 7. 小结：一张链路图

```
                ┌──────────────────────┐
                │   POST /assets      │  (AssetMediaController)
                └─────────┬────────────┘
                          │
                          ▼
                ┌──────────────────────┐
                │ AssetMediaService    │
                │  .uploadAsset        │
                └──┬───────────┬───────┘
                   │           │
     ① requireQuota            ④ updateUsage(+size)
     （用 auth.user 的          （原子 SQL 累加
      缓存字段判断）             user.quotaUsageInBytes）
                   │
                   ▼
        ②③ create() 写 asset + asset_exif.fileSizeInByte
                   │
                   ▼
        文件落盘 + JobName.AssetExtractMetadata
```

删除方向的对称链路：

```
  用户 / 管理员 删除资产
        │
        ▼
  asset.deletedAt = now()   （扣用量不在这步）
        │
        ▼
  JobName.AssetDeleteCheck  →  JobName.AssetDelete
        │
        ▼
  AssetService.handleAssetDeletion
        │
        └─ asset.libraryId IS NULL 时：
            updateUsage(ownerId, -exif.fileSizeInByte)
```

校准：

```
  夜间 / 管理员改配额  ──►  UserRepository.syncUsage()
                              以 asset_exif SUM 回写 user.quotaUsageInBytes
```

## 8. 关键代码索引

| 主题 | 文件 | 行号 |
| --- | --- | --- |
| 配额字段定义 | `server/src/schema/tables/user.table.ts` | 72, 75 |
| 鉴权上下文携带配额 | `server/src/database.ts` | 342 |
| 上传入口 & 校验 | `server/src/services/asset-media.service.ts` | 127, 141, 155, 366 |
| 运动视频累加 | `server/src/services/metadata.service.ts` | 729, 750 |
| 删除时扣减 | `server/src/services/asset.service.ts` | 307, 334 |
| 原子累加 SQL | `server/src/repositories/user.repository.ts` | 300 |
| 重算缓存 | `server/src/repositories/user.repository.ts` | 310 |
| 管理员改配额 → 重算 | `server/src/services/user-admin.service.ts` | 60 |
| 夜间 sync 任务 | `server/src/services/user.service.ts` | 233 |
| 夜间任务派发 | `server/src/services/queue.service.ts` | 280 |
| 管理端用量聚合 | `server/src/repositories/user.repository.ts` | 232 |
| OAuth 配额注入 | `server/src/services/auth.service.ts` | 371 |
