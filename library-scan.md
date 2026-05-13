# Immich 外部库文件系统事件与资产模型贯通方案

## 一、定时全量扫描 (Scheduled Full Scan)

### 1.1 调度机制

定时扫描通过 **CronJob** 调度，在系统配置初始化时设置：

```typescript
// library.service.ts:56-62
this.cronRepository.create({
  name: CronJob.LibraryScan,
  expression: scan.cronExpression,
  onTick: () => handlePromiseError(
    this.jobRepository.queue({ name: JobName.LibraryScanQueueAll }), 
    this.logger
  ),
  start: scan.enabled,
});
```

**关键参数：**
- `cronExpression`: Cron 表达式，决定扫描频率
- `scan.enabled`: 开关控制是否启用定时扫描
- **分布式锁**: 使用 `DatabaseLock.Library` 确保只有一个 microservice 执行调度

### 1.2 扫描流程 (LibraryScanQueueAll)

`JobName.LibraryScanQueueAll` 是扫描入口点：

```typescript
// library.service.ts:452-478
@OnJob({ name: JobName.LibraryScanQueueAll, queue: QueueName.Library })
async handleQueueScanAll(): Promise<JobStatus> {
  // 1. 检查待删除的库
  await this.jobRepository.queue({ name: JobName.LibraryDeleteCheck, data: {} });

  // 2. 获取所有外部库
  const libraries = await this.libraryRepository.getAll(true);

  // 3. 并行排队文件同步 (发现新文件)
  await this.jobRepository.queueAll(
    libraries.map((library) => ({
      name: JobName.LibrarySyncFilesQueueAll,
      data: { id: library.id },
    })),
  );

  // 4. 并行排队资产同步 (检查现有文件状态)
  await this.jobRepository.queueAll(
    libraries.map((library) => ({
      name: JobName.LibrarySyncAssetsQueueAll,
      data: { id: library.id },
    })),
  );
}
```

### 1.3 文件发现 (LibrarySyncFilesQueueAll)

遍历文件系统，发现需要导入的新文件：

```typescript
// library.service.ts:615-683
@OnJob({ name: JobName.LibrarySyncFilesQueueAll, queue: QueueName.Library })
async handleQueueSyncFiles(job: JobOf<JobName.LibrarySyncFilesQueueAll>): Promise<JobStatus> {
  // 1. 验证导入路径
  const validImportPaths: string[] = [];
  for (const importPath of library.importPaths) {
    const validation = await this.validateImportPath(importPath);
    if (validation.isValid) {
      validImportPaths.push(path.normalize(importPath));
    }
  }

  // 2. 使用 fast-glob 流式遍历文件 (分批处理)
  const pathsOnDisk = this.storageRepository.walk({
    pathsToCrawl: validImportPaths,
    includeHidden: false,
    exclusionPatterns: library.exclusionPatterns,
    take: JOBS_LIBRARY_PAGINATION_SIZE, // 分批大小
  });

  // 3. 过滤数据库中已存在的路径
  for await (const pathBatch of pathsOnDisk) {
    const paths = await this.assetRepository.filterNewExternalAssetPaths(
      library.id, 
      pathBatch
    );

    // 4. 排队导入新文件
    if (paths.length > 0) {
      await this.jobRepository.queue({
        name: JobName.LibrarySyncFiles,
        data: { libraryId: library.id, paths, progressCounter: crawlCount },
      });
    }
  }
}
```

**存储层实现 (storage.repository.ts:218-247)：**
```typescript
async *walk(walkOptions: WalkOptionsDto): AsyncGenerator<string[]> {
  const globbedPaths = pathsToCrawl.map((path) => this.asGlob(path));
  const stream = globStream(globbedPaths, {
    absolute: true,
    caseSensitiveMatch: false,
    onlyFiles: true,
    dot: includeHidden,
    ignore: exclusionPatterns,
  });

  // 分批 yield
  let batch: string[] = [];
  for await (const value of stream) {
    batch.push(value.toString());
    if (batch.length === walkOptions.take) {
      yield batch;
      batch = [];
    }
  }
}
```

**数据库过滤 (asset.repository.ts:1052-1071)：**
```sql
-- 仅返回数据库中不存在的路径
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

### 1.4 资产同步 (LibrarySyncAssetsQueueAll)

检查数据库中已存在的资产是否在文件系统中仍然有效：

```typescript
// library.service.ts:699-775
@OnJob({ name: JobName.LibrarySyncAssetsQueueAll, queue: QueueName.Library })
async handleQueueSyncAssets(job: JobOf<JobName.LibrarySyncAssetsQueueAll>): Promise<JobStatus> {
  // 1. 初步离线检测 (SQL层面)
  const offlineResult = await this.assetRepository.detectOfflineExternalAssets(
    library.id,
    library.importPaths,
    library.exclusionPatterns,
  );

  // 2. 分批检查剩余资产 (文件系统层面)
  let chunk: string[] = [];
  for await (const asset of this.libraryRepository.streamAssetIds(library.id)) {
    chunk.push(asset.id);
    if (chunk.length === JOBS_LIBRARY_PAGINATION_SIZE) {
      await this.jobRepository.queue({
        name: JobName.LibrarySyncAssets,
        data: { libraryId: library.id, assetIds: chunk, ... },
      });
      chunk = [];
    }
  }
}
```

**离线检测 SQL (asset.repository.ts:1025-1050)：**
```sql
UPDATE asset 
SET isOffline = true, deletedAt = NOW() 
WHERE isOffline = false 
  AND isExternal = true 
  AND libraryId = $libraryId 
  AND (
    -- 不在任何导入路径下
    originalPath NOT LIKE ANY($importPaths)
    OR 
    -- 匹配排除模式
    originalPath LIKE ANY($exclusionPatterns)
  )
```

---

## 二、实时事件监听 (Real-time Event Watching)

### 2.1 初始化与启动

在系统配置初始化时启动文件监听：

```typescript
// library.service.ts:45-68
@OnEvent({ name: 'ConfigInit', workers: [ImmichWorker.Microservices] })
async onConfigInit({ newConfig: { library: { watch, scan } } }: ArgOf<'ConfigInit'>) {
  // 获取分布式锁，确保只有一个 microservice 进行监听
  this.lock = await this.databaseRepository.tryLock(DatabaseLock.Library);

  this.watchLibraries = this.lock && watch.enabled;

  if (this.watchLibraries) {
    await this.watchAll();
  }
}
```

### 2.2 Chokidar 监听器配置

使用 **chokidar** 库实现跨平台文件系统监听：

```typescript
// library.service.ts:89-162
private async watch(id: string): Promise<boolean> {
  const library = await this.findOrFail(id);
  
  // 文件类型匹配器
  const matcher = picomatch(
    `**/*{${mimeTypes.getSupportedFileExtensions().join(',')}}`, 
    { nocase: true, ignore: library.exclusionPatterns }
  );

  // storage.repository.ts:249-259
  this.watchers[id] = this.storageRepository.watch(
    library.importPaths,
    {
      usePolling: false,           // 使用系统原生事件 (非轮询)
      ignoreInitial: true,          // 忽略初始添加事件
      awaitWriteFinish: {           // 等待文件写入完成
        stabilityThreshold: 5000,   // 5秒无变化视为完成
        pollInterval: 1000,         // 每秒检查一次
      },
    },
    {
      onReady: () => _resolve(),
      onAdd: (path) => handler('add', path),
      onChange: (path) => handler('change', path),
      onUnlink: (path) => deletionHandler(path),
      onError: (error) => this.logger.error(`Watcher error: ${error}`),
    },
  );
}
```

### 2.3 事件处理流程

**添加/修改事件处理：**
```typescript
const handler = async (event: string, path: string) => {
  if (matcher(path)) {
    this.logger.debug(`File ${event} event received for ${path}`);
    // 直接排队导入任务
    await this.jobRepository.queue({
      name: JobName.LibrarySyncFiles,
      data: { libraryId: library.id, paths: [path] },
    });
  }
};
```

**删除事件处理：**
```typescript
const deletionHandler = async (path: string) => {
  this.logger.debug(`File unlink event received for ${path}`);
  // 排队资产删除任务
  await this.jobRepository.queue({
    name: JobName.LibraryRemoveAsset,
    data: { libraryId: library.id, paths: [path] },
  });
};
```

### 2.4 监听器生命周期管理

```typescript
// 停止单个库监听
async unwatch(id: string) {
  if (this.watchers[id]) {
    await this.watchers[id](); // 调用 close 函数
    delete this.watchers[id];
  }
}

// 停止所有监听 (应用关闭时)
@OnEvent({ name: 'AppShutdown' })
async onShutdown() {
  await this.unwatchAll();
}
```

---

## 三、新增与删除处理 (Add & Remove Processing)

### 3.1 新增资产流程 (LibrarySyncFiles)

```typescript
// library.service.ts:248-281
@OnJob({ name: JobName.LibrarySyncFiles, queue: QueueName.Library })
async handleSyncFiles(job: JobOf<JobName.LibrarySyncFiles>): Promise<JobStatus> {
  const library = await this.libraryRepository.get(job.libraryId);

  // 1. 并行处理文件
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

  // 3. 排队后续处理任务
  await this.queuePostSyncJobs(assetIds);

  return JobStatus.Success;
}
```

**实体处理 (processEntity)：**
```typescript
// library.service.ts:400-419
private async processEntity(filePath: string, ownerId: string, libraryId: string) {
  const assetPath = path.normalize(filePath);
  const stat = await this.storageRepository.stat(assetPath);

  return {
    ownerId,
    libraryId,
    // 基于路径的校验和 (外部库专用)
    checksum: this.cryptoRepository.hashSha1(`path:${assetPath}`),
    checksumAlgorithm: ChecksumAlgorithm.sha1Path,
    originalPath: assetPath,
    fileCreatedAt: stat.mtime,
    fileModifiedAt: stat.mtime,
    localDateTime: stat.mtime,
    type: mimeTypes.isVideo(assetPath) ? AssetType.Video : AssetType.Image,
    originalFileName: parse(assetPath).base,
    isExternal: true,  // 标记为外部资产
    livePhotoVideoId: null,
  };
}
```

**后续任务队列 (queuePostSyncJobs)：**
```typescript
// library.service.ts:421-431
async queuePostSyncJobs(assetIds: string[]) {
  // 排队 Sidecar 发现 → 元数据提取 → 缩略图生成 → 人脸识别
  await this.jobRepository.queueAll(
    assetIds.map((assetId) => ({
      name: JobName.SidecarCheck,
      data: { id: assetId, source: 'upload' },
    })),
  );
}
```

### 3.2 删除资产流程 (LibraryRemoveAsset)

**文件系统删除事件触发：**
```typescript
// library.service.ts:685-697
@OnJob({ name: JobName.LibraryRemoveAsset, queue: QueueName.Library })
async handleAssetRemoval(job: JobOf<JobName.LibraryRemoveAsset>): Promise<JobStatus> {
  for (const assetPath of job.paths) {
    // 根据路径查找资产
    const asset = await this.assetRepository.getByLibraryIdAndOriginalPath(
      job.libraryId, 
      assetPath
    );
    if (asset) {
      // 软删除 (级联清理关联数据)
      await this.assetRepository.remove(asset);
    }
  }
  return JobStatus.Success;
}
```

### 3.3 现有资产状态检查 (LibrarySyncAssets)

检查数据库中资产与文件系统的一致性：

```typescript
// library.service.ts:480-574
@OnJob({ name: JobName.LibrarySyncAssets, queue: QueueName.Library })
async handleSyncAssets(job: JobOf<JobName.LibrarySyncAssets>): Promise<JobStatus> {
  const assets = await this.assetJobRepository.getForSyncAssets(job.assetIds);

  // 1. 并行获取文件状态
  const stats = await Promise.all(
    assets.map((asset) => this.storageRepository.stat(asset.originalPath).catch(() => null)),
  );

  // 2. 逐个检查资产状态
  for (let i = 0; i < assets.length; i++) {
    const asset = assets[i];
    const stat = stats[i];
    const action = this.checkExistingAsset(asset, stat);

    switch (action) {
      case AssetSyncResult.OFFLINE:
        // 标记为离线 (软删除)
        assetIdsToOffline.push(asset.id);
        break;
      case AssetSyncResult.UPDATE:
        // 文件修改，重新提取元数据
        assetIdsToUpdate.push(asset.id);
        break;
      case AssetSyncResult.CHECK_OFFLINE:
        // 离线资产需检查是否重新上线
        break;
    }
  }

  // 3. 批量更新状态
  if (assetIdsToOffline.length > 0) {
    await this.assetRepository.updateAll(assetIdsToOffline, { 
      isOffline: true, 
      deletedAt: new Date() 
    });
  }
}
```

**状态检查逻辑 (checkExistingAsset)：**
```typescript
// library.service.ts:576-613
private checkExistingAsset(asset: {...}, stat: Stats | null): AssetSyncResult {
  // 文件不存在 → 离线
  if (!stat) {
    if (asset.isOffline) {
      return AssetSyncResult.DO_NOTHING;
    }
    return AssetSyncResult.OFFLINE;
  }

  // 离线资产 → 检查是否重新上线
  if (asset.isOffline && asset.status !== AssetStatus.Deleted) {
    return AssetSyncResult.CHECK_OFFLINE;
  }

  // 修改时间变化 → 更新元数据
  if (stat.mtime.valueOf() !== asset.fileModifiedAt.valueOf()) {
    return AssetSyncResult.UPDATE;
  }

  return AssetSyncResult.DO_NOTHING;
}
```

---

## 四、架构总结

### 4.1 核心组件协作

```
┌─────────────────────────────────────────────────────────┐
│                    Configuration                        │
├─────────────────────────────────────────────────────────┤
│  scan.enabled: boolean                                  │
│  scan.cronExpression: string                            │
│  watch.enabled: boolean                                 │
└───────────────────┬─────────────────────────────────────┘
                    │
┌───────────────────▼─────────────────────────────────────┐
│                    LibraryService                       │
├─────────────────────────────────────────────────────────┤
│  ┌─────────────────┐     ┌──────────────────────────┐  │
│  │  Cron Scheduler │────▶│  LibraryScanQueueAll     │  │
│  └─────────────────┘     └────────────┬─────────────┘  │
│                                        │                │
│  ┌─────────────────┐                  ▼                │
│  │  Chokidar Watch │───────┬────────────────────────┐  │
│  │    (onAdd)      │       │  LibrarySyncFilesQueue │  │
│  │    (onChange)   │       │                        │  │
│  │    (onUnlink)   │       │  LibrarySyncAssetsQueue│  │
│  └─────────────────┘       └──────────┬─────────────┘  │
│                                        │                │
└────────────────────────────────────────┼────────────────┘
                                         │
┌────────────────────────────────────────▼────────────────┐
│                    Job Queue (BullMQ)                   │
├─────────────────────────────────────────────────────────┤
│  LibrarySyncFiles → 批量创建资产 → SidecarCheck       │
│  LibrarySyncAssets → 检查文件存在性 → 状态更新        │
│  LibraryRemoveAsset → 软删除资产                       │
└─────────────────────────────────────────────────────────┘
```

### 4.2 关键设计决策

| 决策点 | 方案选择 | 理由 |
|--------|----------|------|
| **文件监听库** | Chokidar | 跨平台支持、性能稳定、Node.js 生态标准 |
| **校验和算法** | SHA1(path:) | 外部库文件可能很大，避免读取整个文件计算哈希 |
| **删除策略** | 软删除 (isOffline) | 避免误删导致的数据丢失，用户可手动恢复 |
| **批量处理** | 分页 (JOBS_LIBRARY_PAGINATION_SIZE) | 防止内存溢出、提高并行度 |
| **分布式锁** | DatabaseLock.Library | 确保单实例调度，避免重复扫描/监听 |
| **事件防抖** | awaitWriteFinish (5s) | 处理大文件复制期间的中间状态 |

### 4.3 状态流转图

```
    新增文件              文件修改             文件恢复
       │                     │                    │
       ▼                     ▼                    ▼
  ┌──────────┐         ┌──────────┐         ┌──────────┐
  │  新增    │────────▶│  在线    │────────▶│  在线    │
  │ (创建)   │         │ (正常)   │         │ (恢复)   │
  └──────────┘         └──────────┘         └──────────┘
                                                  ▲
                                                  │
                              路径重回导入路径下 / 排除模式移除
                                                  │
文件删除 ────────────────────────────────────► ┌──────┐
文件移出导入路径 ────────────────────────────► │离线  │
匹配排除模式 ─────────────────────────────────► │      │
                                               └──────┘
```
