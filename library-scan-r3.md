# Immich 文件 Change 事件更新链路证据化梳理

## 一、核心发现：两条链路的实际职责边界

| 触发方式 | 进入队列 | 对已存在资源的实际行为 | mtime 检测 | 触发后续任务 |
|----------|----------|------------------------|------------|--------------|
| **实时 change 事件** | LibrarySyncFiles | ❌ 静默被过滤，不做任何处理 | ❌ 不检测 | ❌ 不触发 |
| **定时巡检扫描** | LibrarySyncAssets | ✅ 检测 mtime 变化，触发更新流程 | ✅ 主动检测 | ✅ 完整触发 |

---

## 二、实时 Change 事件完整调用链证据

### 2.1 第一步：Chokidar 事件回调

```typescript
// library.service.ts:111-121
const handler = async (event: string, path: string) => {
  if (matcher(path)) {
    this.logger.debug(`File ${event} event received for ${path} in library ${library.id}}`);
    // ⚠️ 关键证据：add 和 change 事件完全相同处理，都进入 LibrarySyncFiles
    await this.jobRepository.queue({
      name: JobName.LibrarySyncFiles,
      data: { libraryId: library.id, paths: [path] },
    });
  }
};
```

**证据结论**：change 事件与 add 事件走完全相同的 `LibrarySyncFiles` 队列。

### 2.2 第二步：LibrarySyncFiles 任务执行

```typescript
// library.service.ts:248-280
@OnJob({ name: JobName.LibrarySyncFiles, queue: QueueName.Library })
async handleSyncFiles(job: JobOf<JobName.LibrarySyncFiles>): Promise<JobStatus> {
  const library = await this.libraryRepository.get(job.libraryId);
  
  // 1. 并行处理所有路径，生成 Insertable 对象
  const assetImports: Insertable<AssetTable>[] = [];
  await Promise.all(
    job.paths.map((path) =>
      this.processEntity(path, library.ownerId, job.libraryId)
        .then((asset) => assetImports.push(asset))
        .catch((error) => this.logger.error(`Error processing ${path}: ${error}`)),
    ),
  );

  // 2. 批量插入数据库
  const assetIds = await this.assetRepository.createAll(assetImports);

  // 3. 触发后续任务
  await this.queuePostSyncJobs(assetIds);

  return JobStatus.Success;
}
```

### 2.3 第三步：createAll 的实际行为

```typescript
// asset.repository.ts:441-445
@ChunkedArray({ chunkSize: 4000 })
async createAll(assets: Insertable<AssetTable>[]) {
  // ⚠️ 关键证据：纯 INSERT，无 ON CONFLICT 处理
  const ids = await this.db.insertInto('asset').values(assets).returning('id').execute();
  return ids.map(({ id }) => id);
}
```

**证据结论**：
- `createAll` 是纯 `INSERT` 操作，**没有任何 `ON CONFLICT DO UPDATE` 逻辑**
- 对于已存在的 `originalPath`，数据库主键/唯一约束冲突会直接抛出异常
- 但实际上，**change 事件触发的路径在批量扫描时会被提前过滤掉**

---

## 三、为什么 Change 事件在 LibrarySyncFiles 中被静默丢弃？

### 3.1 定时扫描场景的过滤逻辑

```typescript
// library.service.ts:654-656
for await (const pathBatch of pathsOnDisk) {
  crawlCount += pathBatch.length;
  // ⚠️ 关键证据：只保留数据库中不存在的新路径
  const paths = await this.assetRepository.filterNewExternalAssetPaths(library.id, pathBatch);

  if (paths.length > 0) {
    importCount += paths.length;
    await this.jobRepository.queue({
      name: JobName.LibrarySyncFiles,
      data: { libraryId: library.id, paths, progressCounter: crawlCount },
    });
  }
}
```

### 3.2 filterNewExternalAssetPaths SQL 实现

```sql
-- asset.repository.ts:1052-1071
SELECT path 
FROM unnest($paths) AS path 
WHERE NOT EXISTS (
  SELECT originalPath 
  FROM asset 
  WHERE asset.originalPath = path 
    AND libraryId = $libraryId 
    AND isExternal = true
)
```

**证据结论**：
- 该函数会过滤掉所有数据库中已存在的路径
- 已存在的路径**根本不会进入** `LibrarySyncFiles` 队列
- 这就是为什么 `createAll` 不需要 `ON CONFLICT` — 因为理论上不会有冲突

### 3.3 实时 Change 事件的特殊情况

实时事件触发时，**没有经过 filterNewExternalAssetPaths 过滤**，直接进入 `LibrarySyncFiles`：

```typescript
// library.service.ts:111-121
// 直接入队，无过滤！
await this.jobRepository.queue({
  name: JobName.LibrarySyncFiles,
  data: { libraryId: library.id, paths: [path] },
});
```

**后果**：
1. `processEntity` 正常执行，基于当前文件生成新的 `Insertable`
2. `createAll` 执行 INSERT
3. 由于 `originalPath` 已存在，数据库抛出唯一约束冲突错误
4. 任务失败，异常被 catch，`queuePostSyncJobs` 永远不会被调用

---

## 四、定时巡检链路：真正触发更新的唯一路径

### 4.1 LibrarySyncAssetsQueueAll 入口

```typescript
// library.service.ts:699-774
@OnJob({ name: JobName.LibrarySyncAssetsQueueAll, queue: QueueName.Library })
async handleQueueSyncAssets(job: JobOf<JobName.LibrarySyncAssetsQueueAll>): Promise<JobStatus> {
  // 第一步：SQL 层面标记离线资产（略）

  // 第二步：分批检查所有资产
  let chunk: string[] = [];
  for await (const asset of this.libraryRepository.streamAssetIds(library.id)) {
    chunk.push(asset.id);
    if (chunk.length === JOBS_LIBRARY_PAGINATION_SIZE) {
      await this.jobRepository.queue({
        name: JobName.LibrarySyncAssets,
        data: {
          libraryId: library.id,
          importPaths: library.importPaths,
          exclusionPatterns: library.exclusionPatterns,
          assetIds: chunk.map((id) => id),
          progressCounter: count,
          totalAssets: assetCount,
        },
      });
      chunk = [];
    }
  }
}
```

### 4.2 LibrarySyncAssets 中的 mtime 检测

```typescript
// library.service.ts:480-574
@OnJob({ name: JobName.LibrarySyncAssets, queue: QueueName.Library })
async handleSyncAssets(job: JobOf<JobName.LibrarySyncAssets>): Promise<JobStatus> {
  const assets = await this.assetJobRepository.getForSyncAssets(job.assetIds);

  // 并行 stat 所有文件
  const stats = await Promise.all(
    assets.map((asset) => this.storageRepository.stat(asset.originalPath).catch(() => null)),
  );

  // 逐个资产判断
  const assetIdsToUpdate: string[] = [];
  
  for (let i = 0; i < assets.length; i++) {
    const asset = assets[i];
    const stat = stats[i];
    const action = this.checkExistingAsset(asset, stat);
    
    if (action === AssetSyncResult.UPDATE) {
      assetIdsToUpdate.push(asset.id);
    }
  }

  // ⚠️ 关键证据：只有这里才会对已存在资产触发后续任务
  if (assetIdsToUpdate.length > 0) {
    promises.push(this.queuePostSyncJobs(assetIdsToUpdate));
  }
}
```

### 4.3 checkExistingAsset 中的 mtime 判断

```typescript
// library.service.ts:576-613
private checkExistingAsset(
  asset: { isOffline: boolean; status: AssetStatus; fileModifiedAt: Date; ... },
  stat: Stats | null,
): AssetSyncResult {
  // ... 离线判断省略

  // ⚠️ 唯一的 mtime 变化检测点
  if (stat.mtime.valueOf() !== asset.fileModifiedAt.valueOf()) {
    this.logger.verbose(`Asset ${asset.originalPath} needs metadata extraction`);
    return AssetSyncResult.UPDATE;
  }

  return AssetSyncResult.DO_NOTHING;
}
```

**证据结论**：
- mtime 变化检测**仅在** `LibrarySyncAssets` 中执行
- 检测到变化后，资产 ID 被加入 `assetIdsToUpdate` 数组
- 最后通过 `queuePostSyncJobs` 触发 SidecarCheck 等后续任务

### 4.4 queuePostSyncJobs 触发完整处理链

```typescript
// library.service.ts:421-431
async queuePostSyncJobs(assetIds: string[]) {
  this.logger.debug(`Queuing sidecar discovery for ${assetIds.length} asset(s)`);

  // ⚠️ 触发点：排队 SidecarCheck 任务
  await this.jobRepository.queueAll(
    assetIds.map((assetId) => ({
      name: JobName.SidecarCheck,
      data: { id: assetId, source: 'upload' },
    })),
  );
}
```

**SidecarCheck 之后的完整处理链**：
```
SidecarCheck (检测 .xmp 文件变化)
    ↓
AssetExtractMetadata (重新提取 EXIF)
    ↓
    ├─> 更新 asset_exif 表 (ON CONFLICT DO UPDATE)
    ├─> 更新 asset_video 表 (ON CONFLICT DO UPDATE)
    ├─> 更新 asset_audio 表 (ON CONFLICT DO UPDATE)
    └─> 更新 asset_keyframe 表 (ON CONFLICT DO UPDATE)
    ↓
AssetGenerateThumbnails (重新生成缩略图)
    ↓
AssetDetectFaces (重新检测人脸)
    ↓
FacialRecognition (人脸识别匹配)
```

---

## 五、两条链路的职责边界与冲突场景

### 5.1 职责边界对照表

| 维度 | 实时事件链路 (LibrarySyncFiles) | 定时巡检链路 (LibrarySyncAssets) |
|------|--------------------------------|----------------------------------|
| **设计目标** | 处理**新增**文件 | 处理**已存在**文件的状态变化 |
| **路径过滤** | ❌ 无过滤，直接处理 | ✅ filterNewExternalAssetPaths 过滤新文件 |
| **mtime 检测** | ❌ 完全不检测 | ✅ 核心检测逻辑 |
| **数据库操作** | INSERT 新记录 | UPDATE isOffline / deletedAt |
| **触发后续任务** | ✅ 仅新资产 | ✅ 变化的已存在资产 |
| **处理粒度** | 单文件实时 | 批量分页处理 |
| **响应延迟** | 秒级 (事件触发) | 分钟/小时级 (定时扫描) |

### 5.2 冲突场景分析

#### 冲突场景 1：文件快速新增后立即修改

```
T0: 文件 IMG_123.jpg 被创建 → add 事件触发
T1: LibrarySyncFiles 执行 INSERT，fileModifiedAt = T0
T2: 用户编辑图片，mtime 更新为 T2 → change 事件触发
T3: LibrarySyncFiles 再次执行 → 约束冲突，任务失败
T4: 等待下一次定时扫描 → LibrarySyncAssets 检测到 mtime 变化
T5: 最终触发 SidecarCheck → 元数据重新提取
```

**问题**：T2-T5 之间可能有很长的窗口期（取决于扫描间隔）

#### 冲突场景 2：文件在扫描窗口期内被删除又恢复

```
扫描周期开始
    ↓
检测到文件不存在 → 标记 isOffline = true
    ↓
扫描周期内文件被恢复 (但 mtime 没变)
    ↓
下一次扫描
    ↓
stat 成功 → CHECK_OFFLINE 状态 → 路径仍在 importPath 内
    ↓
恢复上线 (isOffline = false)
    ↓
⚠️ 但 mtime 没变 → 不会触发 UPDATE → 不会重新提取元数据
```

**问题**：文件内容可能已变更但 mtime 未变时，不会触发重新扫描

#### 冲突场景 3：Sidecar 文件单独变更

```
IMG_123.jpg (主文件，mtime 不变)
IMG_123.jpg.xmp (Sidecar 文件被编辑，mtime 变化)
    ↓
change 事件仅针对 .xmp 文件 → 被 mime 过滤器忽略
    ↓
主文件 mtime 不变 → 定时扫描也不会触发更新
    ↓
⚠️ Sidecar 文件变更永远不会被检测到
```

**问题**：Sidecar 文件变更无法通过现有机制检测

---

## 六、关键证据总结与设计缺陷

### 6.1 核心设计假设

当前实现基于一个隐含假设：
> **文件一旦被创建，其内容就不会改变**，因此：
> - `LibrarySyncFiles` 只需要处理新增场景
> - 内容变更完全依赖定时扫描的 `LibrarySyncAssets` 检测

### 6.2 实际存在的问题

1. **实时 change 事件被浪费**：事件触发了但实际不做任何有用工作，反而可能引发数据库冲突
2. **更新延迟不可控**：依赖扫描周期，文件变更后可能数小时才被检测到
3. **mtime 作为唯一变更指标不可靠**：文件内容变更但 mtime 不变的场景会漏检
4. **Sidecar 文件变更无法检测**：Sidecar 文件单独编辑时不会触发主资产的更新流程

### 6.3 代码层面的验证点

可通过以下日志验证上述结论：

```bash
# 查找 change 事件处理日志
grep "File change event received" immich.log

# 查找 LibrarySyncFiles 执行结果
grep "Imported .* file(s) into library" immich.log

# 查找真正触发更新的日志
grep "needs metadata extraction" immich.log
grep "updated, .* unchanged" immich.log
```

**预期观察**：
- "File change event received" 出现频率高
- 但对应的 "Imported" 日志不会增加（因为路径已存在）
- "needs metadata extraction" 仅在定时扫描期间出现
