# Immich 资源分组模型设计

## 概述

Immich 中的单个资源（照片/视频）可以同时归属于多个分组维度，包括：相册（Album）、堆叠（Stack）、Live Photo 关联、自定义标签（Tag）。这些分组机制采用独立的关系设计，互不冲突，共同支撑跨资源的操作与查询。

---

## 一、模型设计

### 1.1 核心关系总览

| 分组类型 | 关系类型 | 关联方式 | 关键约束 |
|---------|---------|---------|---------|
| 相册 (Album) | 多对多 (M:N) | `album_asset` 连接表 | 复合主键防重复；级联删除 |
| 堆叠 (Stack) | 一对多 (1:N) | `asset.stackId` 外键 | 单资源只能属于一个 Stack；`primaryAssetId` 全局唯一 |
| Live Photo | 一对一 (1:1) | `asset.livePhotoVideoId` 自引用 | 视频被隐藏；关联视频自动从所有相册移除 |
| 标签 (Tag) | 多对多 (M:N) + 树结构 | `tag_asset` 连接表 + `tag_closure` 闭包表 | 支持层级标签；树状查询优化 |

### 1.2 表结构详细设计

#### 1.2.1 Asset 主表（核心实体）

```sql
CREATE TABLE asset (
  id                    UUID PRIMARY KEY,
  ownerId               UUID NOT NULL REFERENCES "user"(id) ON DELETE CASCADE,
  type                  VARCHAR NOT NULL,  -- IMAGE / VIDEO / AUDIO / OTHER
  stackId               UUID REFERENCES stack(id) ON DELETE SET NULL,
  livePhotoVideoId      UUID REFERENCES asset(id) ON DELETE SET NULL,
  visibility            VARCHAR DEFAULT 'timeline',  -- timeline / hidden
  checksum              BYTEA NOT NULL,
  originalPath          VARCHAR NOT NULL,
  fileCreatedAt         TIMESTAMPTZ NOT NULL,
  localDateTime         TIMESTAMPTZ NOT NULL,
  isFavorite            BOOLEAN DEFAULT FALSE,
  deletedAt             TIMESTAMPTZ,
  status                VARCHAR DEFAULT 'active',
  -- ... 其他字段
);

CREATE UNIQUE INDEX idx_asset_owner_checksum ON asset(ownerId, checksum)
  WHERE "libraryId" IS NULL;
```

**关键设计点**：
- `stackId`: 可空外键，一个资源最多属于一个 Stack
- `livePhotoVideoId`: 自引用外键，指向关联的视频资源
- `visibility`: 控制资源在时间线的可见性，Live Photo 的视频部分设为 `hidden`
- `checksum` 唯一约束：同一用户同一文件不能重复上传

#### 1.2.2 Album 相册关系（多对多）

```sql
CREATE TABLE album (
  id                    UUID PRIMARY KEY,
  ownerId               UUID NOT NULL REFERENCES "user"(id) ON DELETE CASCADE,
  albumName             VARCHAR NOT NULL,
  description           TEXT,
  albumThumbnailAssetId UUID REFERENCES asset(id) ON DELETE SET NULL,
  createdAt             TIMESTAMPTZ DEFAULT NOW(),
  updatedAt             TIMESTAMPTZ DEFAULT NOW(),
  deletedAt             TIMESTAMPTZ,
  -- ... 其他字段
);

CREATE TABLE album_asset (
  albumId               UUID REFERENCES album(id) ON DELETE CASCADE ON UPDATE CASCADE,
  assetId               UUID REFERENCES asset(id) ON DELETE CASCADE ON UPDATE CASCADE,
  createdAt             TIMESTAMPTZ DEFAULT NOW(),
  updatedAt             TIMESTAMPTZ DEFAULT NOW(),
  PRIMARY KEY (albumId, assetId)
);

CREATE TABLE album_user (
  albumId               UUID REFERENCES album(id) ON DELETE CASCADE,
  userId                UUID REFERENCES "user"(id) ON DELETE CASCADE,
  role                  VARCHAR NOT NULL,  -- owner / editor / viewer
  PRIMARY KEY (albumId, userId)
);
```

**设计特点**：
- 复合主键确保同一资源不能重复加入同一相册
- 级联删除：删除相册时自动移除所有关联关系
- 资源可同时属于多个相册，无数量限制
- 通过 `album_user` 表实现共享相册权限管理

#### 1.2.3 Stack 堆叠关系（一对多）

```sql
CREATE TABLE stack (
  id                    UUID PRIMARY KEY,
  ownerId               UUID NOT NULL REFERENCES "user"(id) ON DELETE CASCADE ON UPDATE CASCADE,
  primaryAssetId        UUID NOT NULL REFERENCES asset(id) UNIQUE,
  createdAt             TIMESTAMPTZ DEFAULT NOW(),
  updatedAt             TIMESTAMPTZ DEFAULT NOW(),
  updateId              UUID
);
```

**设计特点**：
- Stack 拥有 `primaryAssetId` 作为封面资源，**全局唯一约束**确保一个资源不能同时作为多个 Stack 的封面
- 资源通过 `asset.stackId` 外键关联到 Stack，**一个资源最多属于一个 Stack**
- 删除 Stack 时，`ON DELETE SET NULL` 策略保留关联资源但解除堆叠关系
- TODO 注释表明缺少数据库级约束：需要确保 `primaryAssetId` 存在于该 Stack 的资产数组中（业务层保证）

#### 1.2.4 Tag 标签关系（多对多 + 树结构）

```sql
CREATE TABLE tag (
  id                    UUID PRIMARY KEY,
  userId                UUID NOT NULL REFERENCES "user"(id) ON DELETE CASCADE,
  value                 VARCHAR NOT NULL,  -- 标签值，支持层级如 "旅行/2024"
  parentId              UUID REFERENCES tag(id) ON DELETE CASCADE,
  color                 VARCHAR,
  UNIQUE (userId, value)
);

-- 闭包表：用于高效查询标签树结构
CREATE TABLE tag_closure (
  id_ancestor           UUID REFERENCES tag(id) ON DELETE CASCADE,
  id_descendant         UUID REFERENCES tag(id) ON DELETE CASCADE,
  PRIMARY KEY (id_ancestor, id_descendant)
);

CREATE TABLE tag_asset (
  tagId                 UUID REFERENCES tag(id) ON DELETE CASCADE ON UPDATE CASCADE,
  assetId               UUID REFERENCES asset(id) ON DELETE CASCADE ON UPDATE CASCADE,
  PRIMARY KEY (tagId, assetId)
);

CREATE INDEX idx_tag_asset_asset_id ON tag_asset(assetId);
```

**设计特点**：
- 支持标签层级结构（`parentId` 自引用）
- `tag_closure` 闭包表用于高效的树状查询，存储所有祖先-后代关系
- `tag_asset` 复合主键防止重复打标
- 资源可同时拥有多个标签，标签可应用于多个资源
- 删除标签时自动级联删除所有资产关联和闭包表记录

---

## 二、归属判定逻辑

### 2.1 Album 相册归属判定

```typescript
// 真实实现：server/src/repositories/album.repository.ts:105-123
async getByAssetId(ownerId: string, assetId: string) {
  return this.db
    .selectFrom('album')
    .selectAll('album')
    .innerJoin('album_asset', 'album_asset.albumId', 'album.id')
    // 关键：通过 album_user 检查用户对相册的权限
    .where((eb) =>
      eb.exists(
        eb
          .selectFrom('album_user')
          .whereRef('album_user.albumId', '=', 'album.id')
          .where('album_user.userId', '=', ownerId),
      ),
    )
    .where('album_asset.assetId', '=', assetId)
    .where('album.deletedAt', 'is', null)
    .select(withAlbumUsers(ownerId))  // 同时加载相册用户
    .orderBy('album.createdAt', 'desc')
    .execute();
}
```

**判定规则**：
- 通过 `album_asset` 连接表存在性判定
- 必须同时检查 `album_user` 确保用户有权访问该相册
- 同一资源可在多个相册中同时存在
- 排除已删除的相册

### 2.2 Stack 堆叠归属判定

```typescript
// 真实实现：server/src/repositories/stack.repository.ts:157-164
@GenerateSql({ params: [DummyValue.UUID, DummyValue.UUID] })
getForAssetRemoval(assetId: string) {
  return this.db
    .selectFrom('asset')
    .leftJoin('stack', 'stack.id', 'asset.stackId')
    .select(['stackId as id', 'stack.primaryAssetId'])
    .where('asset.id', '=', assetId)
    .executeTakeFirst();
}

// Stack 搜索时加载关联资产：server/src/repositories/stack.repository.ts:16-46
const withAssets = (eb: ExpressionBuilder<DB, 'stack'>, withTags = false) => {
  return jsonArrayFrom(
    eb
      .selectFrom('asset')
      .selectAll('asset')
      .innerJoinLateral(/* ... exif 关联 ... */)
      .$if(withTags, /* ... 标签关联 ... */)
      .where('asset.deletedAt', 'is', null)
      .whereRef('asset.stackId', '=', 'stack.id')
      .$call(withDefaultVisibility),
  ).as('assets');
};
```

**判定规则**：
- 通过 `asset.stackId` 非空判定归属，直接从 asset 表左连接 stack 表查询
- **一个资源最多属于一个 Stack**（外键唯一性隐式约束）
- Stack 的主资源（`primaryAssetId`）不能被移除出 Stack（业务层约束）
- 查询 Stack 时通过 `withAssets` 子查询同时加载所有堆叠的资产

### 2.3 Live Photo 关联判定

```typescript
// 真实关联查询：server/src/repositories/asset.repository.ts:679-691
findLivePhotoMatch(options: LivePhotoSearchOptions) {
  const { ownerId, otherAssetId, livePhotoCID, type } = options;
  return this.db
    .selectFrom('asset')
    .select(['asset.id', 'asset.ownerId'])
    .innerJoin('asset_exif', 'asset.id', 'asset_exif.assetId')
    .where('id', '!=', asUuid(otherAssetId))  // 排除自身
    .where('ownerId', '=', asUuid(ownerId))
    .where('type', '=', type)  // 匹配目标类型（IMAGE 找 VIDEO 或反之）
    .where('asset_exif.livePhotoCID', '=', livePhotoCID)
    .limit(1)
    .executeTakeFirst();
}

// 链接前置检查：server/src/utils/asset.util.ts:145-164
export const onBeforeLink = async (
  { asset: assetRepository, event: eventRepository }: AssetHookRepositories,
  { userId, livePhotoVideoId }: { userId: string; livePhotoVideoId: string },
) => {
  const motionAsset = await assetRepository.getById(livePhotoVideoId);
  if (!motionAsset) throw new BadRequestException('Live photo video not found');
  if (motionAsset.type !== AssetType.Video) throw new BadRequestException('Live photo video must be a video');
  if (motionAsset.ownerId !== userId) throw new BadRequestException('Live photo video does not belong to the user');

  // 如果视频在时间线可见，隐藏它
  if (motionAsset && motionAsset.visibility === AssetVisibility.Timeline) {
    await assetRepository.update({ id: livePhotoVideoId, visibility: AssetVisibility.Hidden });
    await eventRepository.emit('AssetHide', { assetId: motionAsset.id, userId });
  }
};
```

**判定规则**：
- 通过 `asset.livePhotoVideoId` 字段判定静态照片是否有关联视频
- 通过 `asset_exif.livePhotoCID` 匹配 Live Photo 对（同 CID，不同类型）
- **关联视频的 `visibility` 设为 `hidden`**，不在主时间线显示
- 关联视频会被自动从所有相册中移除（metadata.service.ts 中执行）
- Android 运动照片不支持解除链接（`onBeforeUnlink` 中有检查）

### 2.4 Tag 标签归属判定

```typescript
// 真实实现：标签层级操作使用闭包表
// server/src/repositories/tag.repository.ts:35-68
async upsertValue({ userId, value, parentId: _parentId }: { userId: string; value: string; parentId?: string }) {
  const parentId = _parentId ?? null;
  return this.db.transaction().execute(async (tx) => {
    const tag = await this.db
      .insertInto('tag')
      .values({ userId, value, parentId })
      .onConflict((oc) => oc.columns(['userId', 'value']).doUpdateSet({ parentId }))
      .returning(columns.tag)
      .executeTakeFirstOrThrow();

    // 更新闭包表：插入自引用
    await tx
      .insertInto('tag_closure')
      .values({ id_ancestor: tag.id, id_descendant: tag.id })
      .onConflict((oc) => oc.doNothing())
      .execute();

    // 如果有父标签，插入所有祖先关系
    if (parentId) {
      await tx
        .insertInto('tag_closure')
        .columns(['id_ancestor', 'id_descendant'])
        .expression(
          this.db
            .selectFrom('tag_closure')
            .select(['id_ancestor', sql.raw<string>(`'${tag.id}'`).as('id_descendant')])
            .where('id_descendant', '=', parentId),
        )
        .onConflict((oc) => oc.doNothing())
        .execute();
    }

    return tag;
  });
}

// 空标签清理：server/src/repositories/tag.repository.ts:165-183
async deleteEmptyTags() {
  const result = await this.db
    .deleteFrom('tag')
    .where(({ not, exists, selectFrom }) =>
      not(
        exists(
          selectFrom('tag_closure')
            .whereRef('tag.id', '=', 'tag_closure.id_ancestor')
            .innerJoin('tag_asset', 'tag_closure.id_descendant', 'tag_asset.tagId'),
        ),
      ),
    )
    .executeTakeFirst();
  // ...
}
```

**判定规则**：
- 通过 `tag_asset` 连接表存在性判定直接标签关联
- 利用 `tag_closure` 闭包表支持**树状继承查询**：子标签的资产自动归属于所有祖先标签
- `userId + value` 唯一约束防止同一用户创建重复标签
- 清理空标签时通过闭包表检查标签及其所有后代是否有关联资产

---

## 三、跨资源操作设计

### 3.1 Stack 创建与合并

```typescript
// 真实实现：server/src/repositories/stack.repository.ts:63-124
async create(entity: Omit<Insertable<StackTable>, 'primaryAssetId'>, assetIds: string[]) {
  return this.db.transaction().execute(async (tx) => {
    // 步骤1：查找包含任一目标资产的现有 Stack
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

    // 步骤2：收集所有唯一资产 ID（包括现有 Stack 的所有资产）
    const uniqueIds = new Set<string>(assetIds);
    for (const stack of stacks) {
      if (stack.assets && stack.assets.length > 0) {
        for (const asset of stack.assets) {
          uniqueIds.add(asset.id);
        }
      }
    }

    // 步骤3：删除冲突的旧 Stack
    if (stacks.length > 0) {
      await tx
        .deleteFrom('stack')
        .where('id', 'in', stacks.map((stack) => stack.id))
        .execute();
    }

    // 步骤4：创建新 Stack（第一个资产作为 primaryAssetId）
    const newRecord = await tx
      .insertInto('stack')
      .values({ ...entity, primaryAssetId: assetIds[0] })
      .returning('id')
      .executeTakeFirstOrThrow();

    // 步骤5：批量更新所有资产的 stackId
    await tx
      .updateTable('asset')
      .set({ stackId: newRecord.id, updatedAt: new Date() })
      .where('id', 'in', [...uniqueIds])
      .execute();

    // 步骤6：返回完整的 Stack（含所有关联资产）
    return tx
      .selectFrom('stack')
      .selectAll('stack')
      .select(withAssets)
      .where('id', '=', newRecord.id)
      .executeTakeFirstOrThrow();
  });
}

// Stack 合并：server/src/repositories/stack.repository.ts:167-169
merge({ sourceId, targetId }: { sourceId: string; targetId: string }) {
  return this.db
    .updateTable('asset')
    .set({ stackId: targetId })
    .where('asset.stackId', '=', sourceId)
    .execute();
}
```

**关键约束边界**：
- 创建 Stack 时**自动合并包含目标资产的所有旧 Stack**，防止重复堆叠
- 第一个传入的 `assetIds[0]` 自动成为 `primaryAssetId`（封面）
- 整个操作在事务中执行，保证原子性
- 删除旧 Stack 后，所有关联资产的 `stackId` 被批量更新到新 Stack

### 3.2 相册批量添加资源

```typescript
// 真实实现：server/src/utils/asset.util.ts:33-71
export const addAssets = async (
  auth: AuthDto,
  repositories: { access: AccessRepository; bulk: IBulkAsset },
  dto: { parentId: string; assetIds: string[] },
) => {
  const { access, bulk } = repositories;

  // 步骤1：查询已存在的资源，避免重复添加
  const existingAssetIds = await bulk.getAssetIds(dto.parentId, dto.assetIds);
  const notPresentAssetIds = dto.assetIds.filter((id) => !existingAssetIds.has(id));

  // 步骤2：检查权限（AssetShare 权限）
  const allowedAssetIds = await checkAccess(access, {
    auth,
    permission: Permission.AssetShare,
    ids: notPresentAssetIds,
  });

  // 步骤3：逐个判定结果
  const results: BulkIdResponseDto[] = [];
  for (const assetId of dto.assetIds) {
    const hasAsset = existingAssetIds.has(assetId);
    if (hasAsset) {
      results.push({ id: assetId, success: false, error: BulkIdErrorReason.DUPLICATE });
      continue;
    }

    const hasAccess = allowedAssetIds.has(assetId);
    if (!hasAccess) {
      results.push({ id: assetId, success: false, error: BulkIdErrorReason.NO_PERMISSION });
      continue;
    }

    existingAssetIds.add(assetId);
    results.push({ id: assetId, success: true });
  }

  // 步骤4：批量添加成功的资源
  const newAssetIds = results.filter(({ success }) => success).map(({ id }) => id);
  if (newAssetIds.length > 0) {
    await bulk.addAssetIds(dto.parentId, newAssetIds);
  }

  return results;
};
```

**关键约束边界**：
- **幂等性保证**：已存在的资源返回 `DUPLICATE` 错误，不重复添加
- **权限校验**：需要 `AssetShare` 权限才能将资源加入相册
- **部分成功**：支持部分资源成功、部分失败的场景，每个资源独立判定
- 该通用函数同时用于相册、记忆等多个分组的资源添加

### 3.3 标签批量操作

```typescript
// 标签批量打标：server/src/services/tag.service.ts:79-98
async bulkTagAssets(auth: AuthDto, dto: TagBulkAssetsDto): Promise<TagBulkAssetsResponseDto> {
  // 步骤1：并行检查权限
  const [tagIds, assetIds] = await Promise.all([
    this.checkAccess({ auth, permission: Permission.TagAsset, ids: dto.tagIds }),
    this.checkAccess({ auth, permission: Permission.AssetUpdate, ids: dto.assetIds }),
  ]);

  // 步骤2：笛卡尔积生成所有关联项
  const items: Insertable<TagAssetTable>[] = [];
  for (const tagId of tagIds) {
    for (const assetId of assetIds) {
      items.push({ tagId, assetId });
    }
  }

  // 步骤3：批量 upsert（冲突时跳过）
  const results = await this.tagRepository.upsertAssetIds(items);

  // 步骤4：更新 exif 标签字段并触发事件
  for (const assetId of new Set(results.map((item) => item.assetId))) {
    await this.updateTags(assetId);  // 同步到 asset_exif.tags 用于搜索
    await this.eventRepository.emit('AssetTag', { assetId });
  }

  return { count: results.length };
}

// 资源标签替换（原子操作）：server/src/repositories/tag.repository.ts:148-163
@Chunked({ paramIndex: 1 })
replaceAssetTags(assetId: string, tagIds: string[]) {
  return this.db.transaction().execute(async (tx) => {
    // 先删除所有现有标签
    await tx.deleteFrom('tag_asset').where('assetId', '=', assetId).execute();

    if (tagIds.length === 0) return;

    // 再批量插入新标签
    return tx
      .insertInto('tag_asset')
      .values(tagIds.map((tagId) => ({ tagId, assetId })))
      .onConflict((oc) => oc.doNothing())
      .returningAll()
      .execute();
  });
}
```

**关键约束边界**：
- 需要同时具备 `TagAsset`（标签操作）和 `AssetUpdate`（资源操作）权限
- `upsertAssetIds` 使用 `ON CONFLICT DO NOTHING`，实现幂等打标
- `replaceAssetTags` 使用事务保证"先删后插"的原子性
- 标签变更同步更新到 `asset_exif.tags` 数组字段用于搜索

### 3.4 Live Photo 联动操作

```typescript
// 真实实现：server/src/services/metadata.service.ts:177-206
private async linkLivePhotos(
  asset: { id: string; type: AssetType; ownerId: string; libraryId: string | null },
  exifInfo: Insertable<AssetExifTable>,
): Promise<void> {
  if (!exifInfo.livePhotoCID) return;

  // 步骤1：查找配对的资源（同 CID，不同类型）
  const otherType = asset.type === AssetType.Video ? AssetType.Image : AssetType.Video;
  const match = await this.assetRepository.findLivePhotoMatch({
    livePhotoCID: exifInfo.livePhotoCID,
    ownerId: asset.ownerId,
    libraryId: asset.libraryId,
    otherAssetId: asset.id,
    type: otherType,
  });

  if (!match) return;

  // 步骤2：区分静态照片和动态视频
  const [photoAsset, motionAsset] = asset.type === AssetType.Image ? [asset, match] : [match, asset];

  // 步骤3：执行联动操作（并行原子化）
  await Promise.all([
    this.assetRepository.update({ id: photoAsset.id, livePhotoVideoId: motionAsset.id }),
    this.assetRepository.update({ id: motionAsset.id, visibility: AssetVisibility.Hidden }),
    this.albumRepository.removeAssetsFromAll([motionAsset.id]),  // 从所有相册移除
  ]);

  // 步骤4：发送隐藏事件
  await this.eventRepository.emit('AssetHide', { assetId: motionAsset.id, userId: motionAsset.ownerId });
}

// 解除链接钩子：server/src/utils/asset.util.ts:166-188
export const onBeforeUnlink = async (
  { asset: assetRepository }: AssetHookRepositories,
  { livePhotoVideoId }: { livePhotoVideoId: string },
) => {
  const motion = await assetRepository.getById(livePhotoVideoId);
  if (!motion) return null;

  // Android 运动照片特殊处理：不能解除链接
  if (StorageCore.isAndroidMotionPath(motion.originalPath)) {
    throw new BadRequestException('Cannot unlink Android motion photos');
  }

  return motion;
};
```

**关键约束边界**：
- 通过 `livePhotoCID`（Content ID）自动匹配 Live Photo 对
- 关联视频被**自动隐藏**（`visibility = Hidden`）
- 关联视频被**自动从所有相册移除**
- Android 运动照片**不支持解除链接**
- 解除链接后视频恢复可见性（`onAfterUnlink` 中设置 `visibility = Timeline`）

### 3.5 资源删除时的级联处理

**数据库级联策略**：
```
asset 删除时:
  → album_asset: ON DELETE CASCADE (自动移除相册关联)
  → tag_asset: ON DELETE CASCADE (自动移除标签关联)
  → stack: 不直接级联，asset.stackId 变为 NULL (SET NULL)
  → livePhotoVideoId: 自引用，变为 NULL (SET NULL)
  → tag_closure: 通过 tag 的 ON DELETE CASCADE 间接级联

tag 删除时:
  → tag_asset: ON DELETE CASCADE
  → tag_closure: ON DELETE CASCADE (两个方向)

album 删除时:
  → album_asset: ON DELETE CASCADE
  → album_user: ON DELETE CASCADE
```

**额外业务逻辑**：
- Live Photo 的静态照片被删除时，关联视频不会被自动删除，但 `livePhotoVideoId` 变为 NULL
- Stack 被删除时，所有关联资产的 `stackId` 变为 NULL，但资产本身保留
- 标签删除时通过闭包表自动清理所有后代标签的关联

---

## 四、冲突解决机制

### 4.1 多重归属的无冲突设计

| 场景 | 处理策略 | 边界条件 |
|-----|---------|---------|
| 资源同时在多个相册 | 通过连接表独立存储，互不影响 | 最多受 `album_user` 权限限制 |
| 有标签的资源加入 Stack | 标签关系保留，Stack 关系独立 | 树状标签继承不受堆叠影响 |
| Live Photo 被打标签 | 标签应用于静态照片，视频隐藏但关系保留 | 搜索时视频的标签不会单独出现 |
| Stack 中的资源加入相册 | 仅添加指定资源，其他 Stack 成员不自动加入 | Stack 主资源可单独加入相册 |
| 共享相册资源堆叠 | 堆叠是用户私有操作，不影响其他共享用户 | 每个用户有独立的 Stack 空间 |

### 4.2 权限边界隔离

各分组操作有独立的权限控制点（server/src/enum.ts）：

```typescript
enum Permission {
  // 相册权限
  AlbumRead, AlbumCreate, AlbumUpdate, AlbumDelete,
  AlbumAssetCreate, AlbumAssetRemove,

  // Stack 权限
  StackRead, StackCreate, StackUpdate, StackDelete,

  // 标签权限
  TagRead, TagCreate, TagUpdate, TagDelete, TagAsset,

  // 资源基础权限
  AssetRead, AssetUpdate, AssetDelete, AssetShare,
}
```

**权限检查模式**（server/src/utils/access.ts）：
```typescript
// checkAccess 函数被 addAssets/removeAssets 等通用操作调用
// 支持批量权限检查，返回有权限的 ID 集合
```

### 4.3 事件驱动的一致性

通过事件总线确保跨分组操作的最终一致性（server/src/repositories/event.repository.ts）：

```typescript
// 关键事件类型
'AssetCreate'       // 资源创建
'AssetHide'         // 资源隐藏（如 Live Photo 视频）
'AssetShow'         // 资源显示
'AssetTag'          // 资源打标签
'AssetUntag'        // 资源移除标签
'AssetTrash'        // 资源移入回收站
'AlbumAddAssets'    // 相册添加资源
'AlbumRemoveAssets' // 相册移除资源
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
CREATE INDEX idx_asset_owner_id ON asset(ownerId);
CREATE INDEX idx_asset_local_date_time ON asset((localDateTime AT TIME ZONE 'UTC')::date);

-- Album 关联索引
-- album_asset: (albumId, assetId) 复合主键天然支持按 albumId 查询
-- 需要额外索引支持按 assetId 反向查询（隐式通过主键第二个字段）

-- Tag 关联索引
-- tag_asset: (tagId, assetId) 复合主键
CREATE INDEX idx_tag_asset_asset_id ON tag_asset(assetId);  -- 反向查询优化

-- 闭包表索引
-- tag_closure: (id_ancestor, id_descendant) 复合主键
-- 支持树结构的向上/向下遍历
```

### 5.2 时间线查询中的 Stack 过滤

```typescript
// server/src/repositories/asset.repository.ts:740-746
// withStacked = false 时：只显示 Stack 主资源，隐藏其他堆叠成员
.$if(!!options.withStacked, (qb) =>
  qb
    .leftJoin('stack', (join) =>
      join.onRef('stack.id', '=', 'asset.stackId').onRef('stack.primaryAssetId', '=', 'asset.id'),
    )
    .where((eb) => eb.or([eb('asset.stackId', 'is', null), eb(eb.table('stack'), 'is not', null)])),
)
```

**优化逻辑**：
- 默认时间线只显示每个 Stack 的主资源（封面）
- 其他堆叠成员通过 `withStacked` 参数控制是否显示
- 通过左连接 stack 表并检查 `primaryAssetId = asset.id` 过滤非封面资源

---

## 总结

Immich 的分组模型采用"关系正交"设计原则：

### 核心设计原则
1. **独立存储**：每种分组关系使用独立的外键或连接表，互不干扰
2. **约束分层**：数据库级约束（主键、唯一键）+ 业务层约束（Stack 主资源不可移除）
3. **事务原子性**：批量操作和状态变更在事务中执行，保证一致性
4. **事件驱动**：操作通过事件总线传播，确保跨分组的最终一致性
5. **权限隔离**：每种分组操作有独立权限控制点，支持精细化共享管理

### 关键约束边界汇总
- **Stack**：一个资源最多属于一个 Stack；主资源不可移除；创建时自动合并冲突 Stack
- **Live Photo**：关联视频自动隐藏、自动从相册移除；Android 运动照片不可解除链接
- **Album**：资源可多相册归属；批量添加有重复检查和权限校验
- **Tag**：标签树通过闭包表维护；删除标签时自动清理所有后代关联

这种设计使得一张照片可以同时：
- 属于 N 个不同的相册
- 参与一个相似照片堆叠
- 作为 Live Photo 关联一个隐藏视频
- 带有 M 个自定义标签（支持层级）

所有这些状态同时存在且互不冲突，支撑了丰富的用户体验场景。
