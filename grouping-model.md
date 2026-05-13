# Immich 资源分组模型设计

## 概述

Immich 中的单个资源（照片/视频）可以同时归属于多个分组维度，包括：相册（Album）、堆叠（Stack）、Live Photo 关联、自定义标签（Tag）。这些分组机制采用独立的关系设计，互不冲突，共同支撑跨资源的操作与查询。

---

## 一、模型设计

### 1.1 核心关系总览

| 分组类型 | 关系类型 | 关联方式 | 冲突处理 |
|---------|---------|---------|---------|
| 相册 (Album) | 多对多 (M:N) | `album_asset` 连接表 | 独立外键，无冲突 |
| 堆叠 (Stack) | 一对多 (1:N) | `asset.stackId` 外键 | 单资源只能属于一个 Stack |
| Live Photo | 一对一 (1:1) | `asset.livePhotoVideoId` 自引用 | 视频被隐藏，不显示在时间线 |
| 标签 (Tag) | 多对多 (M:N) | `tag_asset` 连接表 | 独立外键，无冲突 |

### 1.2 表结构详细设计

#### 1.2.1 Asset 主表（核心实体）

```sql
CREATE TABLE asset (
  id                    UUID PRIMARY KEY,
  ownerId               UUID NOT NULL REFERENCES "user"(id) ON DELETE CASCADE,
  type                  VARCHAR NOT NULL,  -- IMAGE / VIDEO
  stackId               UUID REFERENCES stack(id) ON DELETE SET NULL,
  livePhotoVideoId      UUID REFERENCES asset(id) ON DELETE SET NULL,
  visibility            VARCHAR DEFAULT 'timeline',  -- timeline / hidden
  -- ... 其他字段
);
```

**关键设计点：**
- `stackId`: 可空外键，一个资源最多属于一个 Stack
- `livePhotoVideoId`: 自引用外键，指向关联的视频资源
- `visibility`: 控制资源在时间线的可见性，Live Photo 的视频部分设为 `hidden`

#### 1.2.2 Album 相册关系（多对多）

```sql
CREATE TABLE album (
  id                    UUID PRIMARY KEY,
  ownerId               UUID NOT NULL REFERENCES "user"(id) ON DELETE CASCADE,
  albumName             VARCHAR NOT NULL,
  -- ... 其他字段
);

CREATE TABLE album_asset (
  albumId               UUID REFERENCES album(id) ON DELETE CASCADE,
  assetId               UUID REFERENCES asset(id) ON DELETE CASCADE,
  PRIMARY KEY (albumId, assetId)
);
```

**设计特点：**
- 复合主键确保同一资源不能重复加入同一相册
- 级联删除：删除相册时自动移除所有关联关系
- 资源可同时属于多个相册，无数量限制

#### 1.2.3 Stack 堆叠关系（一对多）

```sql
CREATE TABLE stack (
  id                    UUID PRIMARY KEY,
  ownerId               UUID NOT NULL REFERENCES "user"(id) ON DELETE CASCADE,
  primaryAssetId        UUID NOT NULL REFERENCES asset(id) UNIQUE,
  createdAt             TIMESTAMP WITH TIME ZONE DEFAULT NOW()
);
```

**设计特点：**
- Stack 拥有 `primaryAssetId` 作为封面资源
- 资源通过 `asset.stackId` 外键关联到 Stack
- `primaryAssetId` 唯一约束确保一个资源不能同时作为多个 Stack 的封面
- 删除 Stack 时，`SET NULL` 策略保留关联资源

#### 1.2.4 Tag 标签关系（多对多）

```sql
CREATE TABLE tag (
  id                    UUID PRIMARY KEY,
  userId                UUID NOT NULL REFERENCES "user"(id) ON DELETE CASCADE,
  value                 VARCHAR NOT NULL,  -- 标签值，支持层级如 "旅行/2024"
  parentId              UUID REFERENCES tag(id) ON DELETE CASCADE,
  color                 VARCHAR
);

CREATE TABLE tag_asset (
  tagId                 UUID REFERENCES tag(id) ON DELETE CASCADE,
  assetId               UUID REFERENCES asset(id) ON DELETE CASCADE,
  PRIMARY KEY (tagId, assetId)
);
```

**设计特点：**
- 支持标签层级结构（`parentId` 自引用）
- 复合主键防止重复打标
- 资源可同时拥有多个标签，标签可应用于多个资源

---

## 二、归属判定逻辑

### 2.1 Album 相册归属判定

```typescript
// 查询资源所属的所有相册
async getByAssetId(ownerId: string, assetId: string) {
  return this.db
    .selectFrom('album')
    .innerJoin('album_asset', 'album.id', 'album_asset.albumId')
    .where('album_asset.assetId', '=', assetId)
    .where('album.ownerId', '=', ownerId)
    .selectAll('album')
    .execute();
}
```

**判定规则：**
- 通过 `album_asset` 连接表存在性判定
- 同一资源可在多个相册中同时存在
- 共享相册的归属判定需额外检查 `album_user` 权限表

### 2.2 Stack 堆叠归属判定

```typescript
// 查询资源所属的 Stack
async getForAssetRemoval(assetId: string) {
  return this.db
    .selectFrom('asset')
    .where('asset.id', '=', assetId)
    .innerJoin('stack', 'asset.stackId', 'stack.id')
    .select([
      'stack.id',
      'stack.ownerId',
      'stack.primaryAssetId',
      sql<number[]>`array(select id from asset where "stackId" = stack.id)`.as('assetIds'),
    ])
    .executeTakeFirst();
}
```

**判定规则：**
- 通过 `asset.stackId` 非空判定归属
- 一个资源最多属于一个 Stack（外键唯一性隐式约束）
- Stack 的主资源（`primaryAssetId`）不能被移除出 Stack

### 2.3 Live Photo 关联判定

```typescript
// 判定资源是否为 Live Photo 的一部分
isLivePhotoStill(asset: Asset): boolean {
  return asset.livePhotoVideoId !== null;
}

isLivePhotoMotion(asset: Asset, allAssets: Asset[]): boolean {
  return allAssets.some(a => a.livePhotoVideoId === asset.id);
}
```

**判定规则：**
- 静态照片通过 `livePhotoVideoId` 指向关联视频
- 关联视频的 `visibility` 设为 `hidden`，不在主时间线显示
- 关联视频会被自动从所有相册中移除

### 2.4 Tag 标签归属判定

```typescript
// 查询资源的所有标签
async getForUpdateTags(assetId: string) {
  return this.db
    .selectFrom('asset')
    .where('id', '=', assetId)
    .select(({ fn }) => [
      fn.coalesce(fn.jsonAgg('tag'), sql`'[]'::json`).as('tags'),
    ])
    .leftJoin('tag_asset', 'asset.id', 'tag_asset.assetId')
    .leftJoin('tag', 'tag_asset.tagId', 'tag.id')
    .groupBy('asset.id')
    .executeTakeFirstOrThrow();
}
```

**判定规则：**
- 通过 `tag_asset` 连接表存在性判定
- 支持多级标签树结构
- 标签变更同步更新到 `asset_exif` 的 `tags` 字段用于搜索

---

## 三、跨资源操作设计

### 3.1 资源删除时的级联处理

当删除一个资源时，各分组的处理策略：

```typescript
// 数据库级联策略总结
asset 删除时:
  → album_asset: ON DELETE CASCADE (自动移除相册关联)
  → tag_asset: ON DELETE CASCADE (自动移除标签关联)
  → stack: 不直接级联，asset.stackId 变为 NULL
  → livePhotoVideoId: 自引用，变为 NULL

// 额外业务逻辑（metadata.service.ts）
当关联 Live Photo 时:
  1. 设置 photo.livePhotoVideoId = motionAsset.id
  2. 设置 motionAsset.visibility = HIDDEN
  3. 从所有相册中移除 motionAsset
```

### 3.2 批量操作的原子性

#### 3.2.1 批量添加资源到相册

```typescript
// album.service.ts
async addAssets(auth: AuthDto, id: string, dto: BulkIdsDto) {
  // 1. 权限检查
  await this.requireAccess({ auth, permission: Permission.AlbumAssetCreate, ids: [id] });
  
  // 2. 批量添加（内部事务保证原子性）
  const results = await addAssets(
    auth,
    { access: this.accessRepository, bulk: this.albumRepository },
    { parentId: id, assetIds: dto.ids },
  );

  // 3. 触发事件
  for (const { id: assetId, success } of results) {
    if (success) {
      await this.eventRepository.emit('AlbumAddAssets', { albumId: id, assetIds: [assetId] });
    }
  }

  return results;
}
```

#### 3.2.2 批量打标签

```typescript
// tag.service.ts
async bulkTagAssets(auth: AuthDto, dto: TagBulkAssetsDto) {
  // 1. 并行权限检查
  const [tagIds, assetIds] = await Promise.all([
    this.checkAccess({ auth, permission: Permission.TagAsset, ids: dto.tagIds }),
    this.checkAccess({ auth, permission: Permission.AssetUpdate, ids: dto.assetIds }),
  ]);

  // 2. 笛卡尔积生成关联项
  const items: Insertable<TagAssetTable>[] = [];
  for (const tagId of tagIds) {
    for (const assetId of assetIds) {
      items.push({ tagId, assetId });
    }
  }

  // 3. 批量 upsert
  const results = await this.tagRepository.upsertAssetIds(items);

  // 4. 更新 exif 标签字段并触发事件
  for (const assetId of new Set(results.map((item) => item.assetId))) {
    await this.updateTags(assetId);
    await this.eventRepository.emit('AssetTag', { assetId });
  }

  return { count: results.length };
}
```

### 3.3 Stack 操作的特殊约束

```typescript
// stack.service.ts
async removeAsset(auth: AuthDto, dto: UUIDAssetIDParamDto) {
  const { id: stackId, assetId } = dto;
  
  // 1. 检查资源确实在 Stack 中
  const stack = await this.stackRepository.getForAssetRemoval(assetId);
  if (!stack?.id || stack.id !== stackId) {
    throw new BadRequestException('Asset not in stack');
  }

  // 2. 主资源不能被移除
  if (stack.primaryAssetId === assetId) {
    throw new BadRequestException("Cannot remove stack's primary asset");
  }

  // 3. 移除关联（设置 stackId 为 NULL）
  await this.assetRepository.update({ id: assetId, stackId: null });
  await this.eventRepository.emit('StackUpdate', { stackId, userId: auth.user.id });
}
```

### 3.4 Live Photo 关联的联动操作

```typescript
// metadata.service.ts
private async linkLivePhotos(
  asset: { id: string; type: AssetType; ownerId: string; libraryId: string | null },
  exifInfo: Insertable<AssetExifTable>,
): Promise<void> {
  if (!exifInfo.livePhotoCID) return;

  // 1. 查找配对的资源（同 CID，不同类型）
  const otherType = asset.type === AssetType.Video ? AssetType.Image : AssetType.Video;
  const match = await this.assetRepository.findLivePhotoMatch({
    livePhotoCID: exifInfo.livePhotoCID,
    ownerId: asset.ownerId,
    libraryId: asset.libraryId,
    otherAssetId: asset.id,
    type: otherType,
  });

  if (!match) return;

  // 2. 区分静态照片和动态视频
  const [photoAsset, motionAsset] = asset.type === AssetType.Image ? [asset, match] : [match, asset];

  // 3. 执行联动操作（并行原子化）
  await Promise.all([
    this.assetRepository.update({ id: photoAsset.id, livePhotoVideoId: motionAsset.id }),
    this.assetRepository.update({ id: motionAsset.id, visibility: AssetVisibility.Hidden }),
    this.albumRepository.removeAssetsFromAll([motionAsset.id]),  // 从所有相册移除
  ]);

  // 4. 发送隐藏事件
  await this.eventRepository.emit('AssetHide', { assetId: motionAsset.id, userId: motionAsset.ownerId });
}
```

---

## 四、冲突解决机制

### 4.1 多重归属的无冲突设计

| 场景 | 处理策略 | 示例 |
|-----|---------|------|
| 资源同时在多个相册 | 通过连接表独立存储，互不影响 | 一张照片同时在"旅行"和"家庭"相册 |
| 有标签的资源加入 Stack | 标签关系保留，Stack 关系独立 | 标记为"风景"的照片参与堆叠 |
| Live Photo 被打标签 | 标签应用于静态照片，视频隐藏但关系保留 | Live Photo 的静态部分有标签 |
| Stack 中的资源加入相册 | 仅主资源显示，其他 Stack 成员不自动加入 | Stack 作为整体浏览，但相册独立管理 |

### 4.2 权限边界隔离

各分组操作有独立的权限控制点：

```typescript
// Permission 枚举
enum Permission {
  AlbumRead, AlbumUpdate, AlbumDelete, AlbumAssetCreate,  // 相册权限
  StackRead, StackUpdate, StackDelete,                    // Stack 权限
  TagRead, TagUpdate, TagDelete, TagAsset,                // 标签权限
  AssetRead, AssetUpdate, AssetDelete,                    // 资源基础权限
}
```

### 4.3 事件驱动的一致性

通过事件总线确保跨分组操作的最终一致性：

```typescript
// 关键事件类型
'AssetCreate'       // 资源创建
'AssetHide'         // 资源隐藏（如 Live Photo 视频）
'AssetTag'          // 资源打标签
'AssetUntag'        // 资源移除标签
'AlbumAddAssets'    // 相册添加资源
'StackCreate'       // Stack 创建
'StackUpdate'       // Stack 更新
'StackDelete'       // Stack 删除
```

---

## 五、查询优化设计

### 5.1 索引策略

```sql
-- Asset 表索引
CREATE INDEX idx_asset_stack_id ON asset(stackId);
CREATE INDEX idx_asset_live_photo_video_id ON asset(livePhotoVideoId);
CREATE UNIQUE INDEX idx_asset_owner_checksum ON asset(ownerId, checksum);

-- 连接表索引（利用主键隐式索引）
-- album_asset (albumId, assetId) - 复合主键
-- tag_asset (tagId, assetId) - 复合主键，额外索引 assetId 方向
CREATE INDEX idx_tag_asset_asset_id ON tag_asset(assetId);
```

### 5.2 关联查询模式

```typescript
// 一次性加载资源的所有分组信息
async getAssetWithAllGroupings(assetId: string) {
  return this.db
    .selectFrom('asset')
    .where('asset.id', '=', assetId)
    .selectAll('asset')
    // Stack 信息
    .leftJoin('stack', 'asset.stackId', 'stack.id')
    .select(['stack.id as stackId', 'stack.primaryAssetId'])
    // 相册列表（聚合）
    .select(({ fn }) => [
      fn.coalesce(fn.jsonAgg('album'), sql`'[]'::json`).as('albums'),
    ])
    .leftJoin('album_asset', 'asset.id', 'album_asset.assetId')
    .leftJoin('album', 'album_asset.albumId', 'album.id')
    // 标签列表（聚合）
    .select(({ fn }) => [
      fn.coalesce(fn.jsonAgg('tag'), sql`'[]'::json`).as('tags'),
    ])
    .leftJoin('tag_asset', 'asset.id', 'tag_asset.assetId')
    .leftJoin('tag', 'tag_asset.tagId', 'tag.id')
    .groupBy('asset.id', 'stack.id')
    .executeTakeFirst();
}
```

---

## 总结

Immich 的分组模型采用"关系正交"设计原则：

1. **独立存储**：每种分组关系使用独立的外键或连接表，互不干扰
2. **约束分层**：通过数据库级约束（主键、唯一键）和业务级约束（如 Stack 主资源不可移除）确保数据一致性
3. **事件驱动**：操作通过事件总线传播，确保跨分组的最终一致性
4. **权限隔离**：每种分组操作有独立权限控制点，支持精细化的共享管理

这种设计使得一张照片可以同时：
- 属于 5 个不同的相册
- 参与一个相似照片堆叠
- 作为 Live Photo 关联一个隐藏视频
- 带有 10 个自定义标签

所有这些状态同时存在且互不冲突，支撑了丰富的用户体验场景。
