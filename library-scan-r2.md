# Immich Library 删除与离线链路深度解析

## 一、两条核心链路概览

| 链路 | 触发源 | 处理方式 | 数据影响 | 可恢复性 |
|------|--------|----------|----------|----------|
| **实时删除链路** | 文件系统 unlink 事件 | 硬删除资产记录 | 数据从数据库永久删除 | ❌ 不可恢复 |
| **离线标记链路** | 定时扫描检测 | 软标记 isOffline 状态 | 记录保留，标记离线 | ✅ 可恢复 |

---

## 二、实时删除事件处理链路 (LibraryRemoveAsset)

### 2.1 触发源：文件系统监听

```typescript
// library.service.ts:123-129
const deletionHandler = async (path: string) => {
  this.logger.debug(`File unlink event received for ${path} in library ${library.id}}`);
  await this.jobRepository.queue({
    name: JobName.LibraryRemoveAsset,
    data: { libraryId: library.id, paths: [path] },
  });
};
```

**关键点**：
- 由 **Chokidar** 监听到 `unlink` 事件直接触发
- **仅针对单个文件路径**，不做批量处理
- 路径经过 `picomatch` 过滤，仅处理受支持的媒体文件类型

### 2.2 任务执行：资产硬删除

```typescript
// library.service.ts:685-697
@OnJob({ name: JobName.LibraryRemoveAsset, queue: QueueName.Library })
async handleAssetRemoval(job: JobOf<JobName.LibraryRemoveAsset>): Promise<JobStatus> {
  for (const assetPath of job.paths) {
    // 1. 根据 libraryId + originalPath 精确匹配资产
    const asset = await this.assetRepository.getByLibraryIdAndOriginalPath(
      job.libraryId, 
      assetPath
    );
    
    if (asset) {
      // 2. 执行硬删除 - DELETE FROM asset WHERE id = ?
      await this.assetRepository.remove(asset);
    }
  }
  return JobStatus.Success;
}
```

**底层删除实现**：
```typescript
// asset.repository.ts:639-641
async remove(asset: { id: string }): Promise<void> {
  await this.db.deleteFrom('asset').where('id', '=', asUuid(asset.id)).execute();
}
```

### 2.3 删除的级联影响

> ⚠️ **重要说明**：数据库外键约束决定了级联行为
- `asset_exif`、`asset_file`、`asset_face`、`asset_metadata` 等关联表
- 通常通过 **ON DELETE CASCADE** 外键约束，主表记录删除时关联数据同步删除
- **文件本身不删除**：`deleteOnDisk = false`，仅删除数据库记录

---

## 三、定时扫描离线标记链路 (LibrarySyncAssets)

### 3.1 第一层：SQL 批量离线检测

在进入文件系统检查前，先做 SQL 层面的快速过滤：

```typescript
// asset.repository.ts:1025-1050
async detectOfflineExternalAssets(
  libraryId: string,
  importPaths: string[],
  exclusionPatterns: string[],
): Promise<UpdateResult> {
  const paths = importPaths.map((importPath) => `${importPath}%`);  // LIKE 前缀匹配
  const exclusions = exclusionPatterns.map((pattern) => globToSqlPattern(pattern));

  return this.db
    .updateTable('asset')
    .set({
      isOffline: true,
      deletedAt: new Date(),  // 同时设置 deletedAt，在前端表现为隐藏
    })
    .where('isOffline', '=', false)          // 仅处理在线资产
    .where('isExternal', '=', true)          // 仅处理外部库资产
    .where('libraryId', '=', asUuid(libraryId))
    .where((eb) =>
      eb.or([
        // 条件1：不在任何导入路径下
        eb.not(eb.or(paths.map((path) => eb('originalPath', 'like', path)))),
        // 条件2：匹配排除模式
        eb.or(exclusions.map((path) => eb('originalPath', 'like', path))),
      ]),
    )
    .execute();
}
```

**离线标记字段组合**：
```sql
SET isOffline = true, deletedAt = NOW()
```
- `isOffline`: 技术标记，标识文件不可访问
- `deletedAt`: 业务标记，前端依此判断是否显示

### 3.2 第二层：文件系统逐批校验

SQL 过滤后，对剩余资产做文件系统存在性检查：

```typescript
// library.service.ts:480-574
@OnJob({ name: JobName.LibrarySyncAssets, queue: QueueName.Library })
async handleSyncAssets(job: JobOf<JobName.LibrarySyncAssets>): Promise<JobStatus> {
  const assets = await this.assetJobRepository.getForSyncAssets(job.assetIds);

  // 并行获取所有文件状态 (stat 失败返回 null)
  const stats = await Promise.all(
    assets.map((asset) => this.storageRepository.stat(asset.originalPath).catch(() => null)),
  );

  // 逐个资产判断状态
  for (let i = 0; i < assets.length; i++) {
    const asset = assets[i];
    const stat = stats[i];
    const action = this.checkExistingAsset(asset, stat);
    
    switch (action) {
      case AssetSyncResult.OFFLINE:
        assetIdsToOffline.push(asset.id);
        break;
      case AssetSyncResult.UPDATE:
        assetIdsToUpdate.push(asset.id);
        break;
      case AssetSyncResult.CHECK_OFFLINE:
        // 离线资产需额外检查
        break;
    }
  }

  // 批量执行更新
  const promises = [];
  if (assetIdsToOffline.length > 0) {
    promises.push(this.assetRepository.updateAll(assetIdsToOffline, { 
      isOffline: true, 
      deletedAt: new Date() 
    }));
  }
  // ... 其他更新
}
```

### 3.3 四态决策逻辑

```typescript
// library.service.ts:576-613
private checkExistingAsset(
  asset: { isOffline: boolean; status: AssetStatus; fileModifiedAt: Date; ... },
  stat: Stats | null,
): AssetSyncResult {

  // 情况1：文件不存在
  if (!stat) {
    if (asset.isOffline) {
      // 已经是离线状态 → 保持不变
      return AssetSyncResult.DO_NOTHING;
    }
    // 在线资产 → 文件消失 → 标记离线
    return AssetSyncResult.OFFLINE;
  }

  // 情况2：资产已离线，但文件重新出现
  if (asset.isOffline && asset.status !== AssetStatus.Deleted) {
    // 需要额外检查路径是否仍在导入范围内
    return AssetSyncResult.CHECK_OFFLINE;
  }

  // 情况3：文件修改时间变化 → 需要重新提取元数据
  if (stat.mtime.valueOf() !== asset.fileModifiedAt.valueOf()) {
    return AssetSyncResult.UPDATE;
  }

  // 情况4：一切正常 → 无操作
  return AssetSyncResult.DO_NOTHING;
}
```

### 3.4 离线恢复机制

CHECK_OFFLINE 状态下的恢复逻辑：

```typescript
// library.service.ts:513-539
case AssetSyncResult.CHECK_OFFLINE: {
  // 检查1：路径是否仍在任何导入路径下
  const isInImportPath = job.importPaths.find((path) => asset.originalPath.startsWith(path));

  if (!isInImportPath) {
    // 不在导入路径 → 保持离线
    break;
  }

  // 检查2：是否被排除模式覆盖
  const isExcluded = job.exclusionPatterns.some((pattern) => 
    picomatch.isMatch(asset.originalPath, pattern)
  );

  if (!isExcluded) {
    // 文件存在 + 在导入路径内 + 不被排除 → 恢复上线
    if (asset.status === AssetStatus.Trashed) {
      trashedAssetIdsToOnline.push(asset.id);
    } else {
      assetIdsToOnline.push(asset.id);
    }
    break;
  }

  // 被排除模式覆盖 → 保持离线
  break;
}
```

**恢复执行**：
```typescript
// 普通资产恢复：清除离线标记和删除时间
this.assetRepository.updateAll(assetIdsToOnline, { isOffline: false, deletedAt: null });

// 回收站资产恢复：仅清除离线标记（保留 Trashed 状态）
this.assetRepository.updateAll(trashedAssetIdsToOnline, { isOffline: false });
```

---

## 四、文件 Change 事件与资产更新链路

### 4.1 链路全景

```
文件系统 change 事件
        ↓
Chokidar 触发 onChange 回调
        ↓
picomatch 路径匹配过滤
        ↓
LibrarySyncFiles 任务入队
        ↓
processEntity 处理 (但路径已存在)
        ↓
检测 mtime 变化 → AssetSyncResult.UPDATE
        ↓
queuePostSyncJobs 触发后续处理
        ↓
SidecarCheck (检测 .xmp 等附属文件)
        ↓
MetadataExtraction (重新提取 EXIF)
        ↓
ThumbnailGeneration (重新生成缩略图)
        ↓
FaceDetection (重新检测人脸)
```

### 4.2 Change 事件的特殊处理

```typescript
// library.service.ts:111-121
const handler = async (event: string, path: string) => {
  if (matcher(path)) {
    this.logger.debug(`File ${event} event received for ${path} in library ${library.id}}`);
    // ⚠️ add 和 change 事件走的是同一个处理流程！
    await this.jobRepository.queue({
      name: JobName.LibrarySyncFiles,
      data: { libraryId: library.id, paths: [path] },
    });
  }
};
```

**关键点**：`add` 和 `change` 事件共用 `LibrarySyncFiles` 任务，区别在于：
- 新增文件：数据库不存在 → `processEntity` → 插入新记录
- 修改文件：数据库已存在 → 依赖后续 `LibrarySyncAssets` 检测 mtime 变化

### 4.3 更新触发：mtime 变化检测

```typescript
// library.service.ts:606-610
if (stat.mtime.valueOf() !== asset.fileModifiedAt.valueOf()) {
  this.logger.verbose(`Asset ${asset.originalPath} needs metadata extraction in library ${asset.libraryId}`);
  return AssetSyncResult.UPDATE;
}
```

**mtime 变化触发的后续任务链**：
```typescript
// library.service.ts:561-563
if (assetIdsToUpdate.length > 0) {
  promises.push(this.queuePostSyncJobs(assetIdsToUpdate));
}

// library.service.ts:421-430
async queuePostSyncJobs(assetIds: string[]) {
  await this.jobRepository.queueAll(
    assetIds.map((assetId) => ({
      name: JobName.SidecarCheck,
      data: { id: assetId, source: 'upload' },
    })),
  );
}
```

### 4.4 SidecarCheck 后的完整处理链

Sidecar 检查成功后，会继续触发：

```typescript
// SidecarCheck → MetadataExtraction 入队
// metadata.service.ts 中 handleSidecarCheck 完成后：
await this.jobRepository.queue({ 
  name: JobName.AssetExtractMetadata, 
  data: { id: asset.id } 
});

// MetadataExtraction 完成后继续触发：
// → ThumbnailGeneration (缩略图重生成)
// → FaceDetection (人脸重检测)
// → FacialRecognition (人脸识别更新)
```

---

## 五、链路对比与决策表

### 5.1 删除 vs 离线 行为对照表

| 维度 | 实时删除 (LibraryRemoveAsset) | 离线标记 (LibrarySyncAssets) |
|------|-------------------------------|-------------------------------|
| **触发时机** | 文件 unlink 事件实时触发 | 定时扫描周期性检测 |
| **匹配方式** | originalPath 精确匹配 | 路径前缀 + 模式匹配 + 文件存在性 |
| **数据库操作** | DELETE FROM asset | UPDATE asset SET isOffline=true |
| **关联数据** | 级联删除所有关联表 | 保留所有关联数据 |
| **磁盘文件** | 不删除原文件 | 不删除原文件 |
| **可恢复性** | 数据库记录消失，无法恢复 | 记录保留，文件恢复可自动上线 |
| **前端表现** | 资产彻底消失 | 资产不显示（因 deletedAt 存在） |
| **处理粒度** | 单文件处理 | 批量分页处理 |

### 5.2 离线资产恢复条件

✅ **恢复成功需同时满足**：
1. 文件在磁盘上重新存在 (`stat` 成功)
2. 路径仍在某个 `importPath` 前缀下
3. 路径不被任何 `exclusionPatterns` 匹配
4. 资产状态不是 `AssetStatus.Deleted` (已彻底删除)

❌ **保持离线的场景**：
- 文件仍不存在 / 权限不足
- 文件被移出所有导入路径
- 文件被新增的排除模式覆盖
- 资产已被标记为 Deleted 状态

### 5.3 AssetStatus 三态说明

```typescript
enum AssetStatus {
  Active = 'active',    // 正常状态，可见可用
  Trashed = 'trashed',  // 在回收站，30天后自动删除
  Deleted = 'deleted',  // 已删除，等待清理任务物理删除
}
```

**注意**：`isOffline` 是独立字段，不与 `status` 直接关联，但会配合 `deletedAt` 共同决定资产可见性。

---

## 六、常见问题 Q&A

**Q1: 为什么删除链路用硬删除，而扫描链路用离线标记？**
- 删除事件是**确定性行为**：文件已从文件系统移除，对应的数据库记录理应清除
- 扫描检测是**概率性行为**：可能因临时挂载问题、权限波动导致误判，保留记录可自动恢复

**Q2: 文件 change 后为什么不直接更新 mtime，要走完整的元数据提取？**
- 文件内容可能已变更（如重新编辑的照片）
- EXIF 信息可能已修改（如调整了拍摄时间、GPS）
- 缩略图、人脸检测等衍生数据都需要同步刷新

**Q3: 离线资产在回收站中，文件恢复后会怎样？**
- 仅清除 `isOffline` 标记
- 仍保留 `Trashed` 状态，继续留在回收站中
- 需用户手动从回收站恢复才会重新显示
