# Immich 任务队列调度机制详解

## 1. 核心架构

### 1.1 进程分工与调度入口

Immich 采用多进程架构，通过独立的 Worker 进程实现任务的生产与消费分离：

| 进程类型 | 启动方式 | 职责 | 是否处理任务 | 是否入队 |
|---------|---------|------|-------------|---------|
| **API 进程** | `fork()` 子进程 | 处理 HTTP 请求、Web UI、用户上传 | ❌ 仅生产任务 | ✅ 可入队 |
| **Microservices 进程** | `Worker Threads` 线程 | 任务执行、队列消费 | ✅ 实际执行任务 | ✅ 可入队（任务链式调用） |
| **Maintenance 进程** | `Worker Threads` 线程 | 维护模式（数据库备份/恢复等） | ❌ 特殊维护操作 | ❌ |

**关键调度流程**：

1. **任务生产**：API 进程接收用户请求（如照片上传），通过 `JobRepository.queue()` 将任务推入 Redis 队列
2. **任务消费**：只有 Microservices 进程会调用 `JobRepository.startWorkers()` 启动 BullMQ Worker，监听并执行任务
3. **任务链式**：任务执行完成后，Microservices 进程可通过 `JobService.onDone()` 触发后续任务入队

### 1.2 关键组件

| 组件 | 文件 | 职责 |
|------|------|------|
| **JobRepository** | `repositories/job.repository.ts` | 核心队列管理：任务入队、Worker 启动、并发配置、队列状态查询 |
| **QueueService** | `services/queue.service.ts` | 定时任务调度、并发配置更新、批量任务触发 |
| **JobService** | `services/job.service.ts` | 任务生命周期管理、后续任务链式调度 |
| **@OnJob 装饰器** | `decorators.ts` | 任务处理器注册与队列绑定 |

## 2. 队列分类与并发配置

### 2.1 队列总数与分类

系统共有 **18 个队列**，分为两大类：

| 类别 | 数量 | 说明 |
|-----|-----|------|
| **并发队列** | 14 个 | 支持多任务并行执行，并发数可配置 |
| **串行队列** | 4 个 | 强制单线程执行，避免资源竞争或数据冲突 |

### 2.2 并发队列详细配置

14 个并发队列的默认并发数：

| 队列名称 | 默认并发数 | 主要任务类型 | 资源类型 |
|---------|-----------|------------|---------|
| `BackgroundTask` | 5 | 清理任务、删除检查 | 轻量 IO 密集 |
| `SmartSearch` | 2 | CLIP 特征提取、向量搜索 | ML 计算密集 |
| `MetadataExtraction` | 5 | EXIF 元数据提取、地理位置解析 | CPU + IO |
| `FaceDetection` | 2 | 人脸检测、特征提取 | ML 计算密集 |
| `Search` | 5 | 搜索索引相关任务 | 轻量 |
| `Sidecar` | 5 | XMP Sidecar 文件读写 | IO 密集 |
| `Library` | 5 | 库扫描、文件同步 | IO 密集 |
| `Migration` | 5 | 数据迁移、存储模板迁移 | IO 密集 |
| `ThumbnailGeneration` | 3 | 缩略图生成、图片处理 | CPU 密集 |
| `VideoConversion` | 1 | 视频转码、编码 | 重度 CPU/GPU 密集 |
| `Notification` | 5 | 邮件发送、相册通知 | IO 密集 |
| `Ocr` | 1 | 文字识别 | ML 计算密集 |
| `Workflow` | 5 | 工作流执行 | 通用 |
| `Editor` | 2 | 编辑器相关任务、预览生成 | CPU 密集 |

### 2.3 串行队列（强制单线程）

4 个串行队列，并发数固定为 1：

| 队列名称 | 串行原因 | 典型任务 |
|---------|---------|---------|
| `FacialRecognition` | 人脸聚类需要顺序处理，避免同一人物同时被多个任务修改导致数据不一致 | 人脸聚类、人物合并 |
| `StorageTemplateMigration` | 涉及文件系统移动操作，避免冲突 | **单资产存储模板迁移**、**全量存储模板迁移** |
| `DuplicateDetection` | 重复检测需要全局一致性，且资源消耗大 | 批量重复照片检测 |
| `BackupDatabase` | 数据库备份需要独占资源 | 全量数据库备份 |

### 2.4 StorageTemplateMigration vs Migration 队列职责对比

两个队列名称相似，但职责完全不同：

| 维度 | `StorageTemplateMigration` 队列 | `Migration` 队列 |
|-----|--------------------------------|-----------------|
| **并发模式** | 串行（固定 1） | 并发（默认 5） |
| **核心职责** | 按用户配置的模板重命名/移动**原始文件**在库目录的位置 | 迁移**衍生文件**（缩略图、预览图、编码视频、人脸图等）到新目录结构 |
| **触发时机** | 1. 新资产上传后自动触发<br>2. 用户修改存储模板后批量触发 | 系统升级、目录结构变更时批量触发 |
| **典型任务** | `StorageTemplateMigrationSingle`（单资产）<br>`StorageTemplateMigration`（全量） | `AssetFileMigration`（资产衍生文件）<br>`PersonFileMigration`（人脸缩略图）<br>`FileMigrationQueueAll`（批量触发） |
| **迁移对象** | 原始照片/视频文件（library 目录） | 缩略图、预览图、编码视频、人脸缩略图（thumbs、encoded-video 目录） |
| **依赖关系** | 依赖 EXIF 元数据提取完成（拍摄时间等信息生成路径） | 不依赖元数据，仅需文件存在 |

### 2.4 并发配置机制

在 `QueueService.updateConcurrency()` 中动态设置：

```typescript
private updateConcurrency(config: SystemConfig) {
  for (const queueName of Object.values(QueueName)) {
    let concurrency = 1;
    if (this.isConcurrentQueue(queueName)) {
      concurrency = config.job[queueName].concurrency;
    }
    this.jobRepository.setConcurrency(queueName, concurrency);
  }
}
```

## 3. 任务类型与优先级

### 3.1 任务类型概览

系统定义了 **60+ 种任务**（`JobName` 枚举），按功能分类：

| 分类 | 代表任务 | 所属队列 |
|-----|---------|---------|
| **缩略图** | `AssetGenerateThumbnails`, `PersonGenerateThumbnail`, `AssetEditThumbnailGeneration` | `ThumbnailGeneration` |
| **视频处理** | `AssetEncodeVideo`, `AssetEncodeVideoQueueAll` | `VideoConversion` |
| **元数据** | `AssetExtractMetadata`, `SidecarCheck`, `SidecarWrite` | `MetadataExtraction`, `Sidecar` |
| **人脸 ML** | `AssetDetectFaces`, `FacialRecognition` | `FaceDetection`, `FacialRecognition` |
| **智能搜索** | `SmartSearch`, `AssetDetectDuplicates` | `SmartSearch`, `DuplicateDetection` |
| **OCR** | `Ocr`, `OcrQueueAll` | `Ocr` |
| **存储迁移** | `StorageTemplateMigrationSingle`, `AssetFileMigration` | `StorageTemplateMigration`, `Migration` |
| **清理任务** | `AssetDeleteCheck`, `UserDeleteCheck`, `SessionCleanup`, `TagCleanup` | `BackgroundTask` |
| **通知** | `SendMail`, `NotifyAlbumInvite`, `NotifyAlbumUpdate` | `Notification` |
| **回忆** | `MemoryGenerate`, `MemoryCleanup` | `BackgroundTask` |
| **数据库** | `DatabaseBackup` | `BackupDatabase` |

### 3.2 任务优先级机制

BullMQ 中优先级数值越小，优先级越高。目前只有 **1 种任务**设置了高优先级：

| 任务名称 | 优先级值 | 说明 |
|---------|---------|------|
| `PersonGenerateThumbnail` | 1 | 人物缩略图生成，用户感知强，需要优先处理 |

**其他所有任务**：默认优先级（不设置，使用 BullMQ 默认值）

```typescript
// job.repository.ts
private getJobOptions(item: JobItem): JobsOptions | null {
  switch (item.name) {
    case JobName.PersonGenerateThumbnail: {
      return { priority: 1 };  // 高优先级
    }
    // ...
  }
}
```

### 3.3 任务去重与延迟

部分任务通过 `jobId` 实现去重，避免重复执行：

| 任务 | 去重 ID 模式 | 额外配置 |
|-----|-------------|---------|
| `NotifyAlbumUpdate` | `${id}/${recipientId}` | 支持 `delay` 延迟执行 |
| `StorageTemplateMigrationSingle` | `asset.id` | 按资产 ID 去重 |
| `FacialRecognitionQueueAll` | 固定 `JobName` | 全局单例，同时只跑一个 |
| `VersionCheck` | 固定 `JobName` | 全局单例 |

## 4. 任务入队来源详解

### 4.1 入队方式

#### 方式一：单个任务入队

```typescript
await this.jobRepository.queue({
  name: JobName.AssetGenerateThumbnails,
  data: { id: assetId, source: 'upload' }
});
```

#### 方式二：批量任务入队（性能优化）

```typescript
await this.jobRepository.queueAll([
  { name: JobName.SmartSearch, data: { id: assetId } },
  { name: JobName.AssetDetectFaces, data: { id: assetId } },
  { name: JobName.Ocr, data: { id: assetId } },
]);
```

`queueAll` 内部优化：
- 按队列分组，使用 `addBulk` 批量入队减少 Redis 交互
- 对需要 `jobId` 去重的任务，单独使用 `add` 入队

### 4.2 入队来源分布

| 来源进程 | 典型触发场景 | 示例文件 |
|---------|------------|---------|
| **API 进程** | 用户上传照片、删除资产、用户操作、管理后台触发 | `asset-media.service.ts`, `user.service.ts`, `asset.service.ts` |
| **Microservices 进程** | 任务链式调度（`onDone`）、定时任务、扫描任务 | `job.service.ts`, `queue.service.ts`, `library.service.ts` |

### 4.3 任务处理器注册

使用 `@OnJob` 装饰器将任务绑定到指定队列：

```typescript
@OnJob({ name: JobName.AssetGenerateThumbnails, queue: QueueName.ThumbnailGeneration })
async generateThumbnails({ id }: IEntityJob): Promise<JobStatus> {
  // 处理逻辑
  return JobStatus.Success;
}
```

`JobRepository.setup()` 在启动时自动扫描所有 Service 类，收集标记了 `@OnJob` 的方法作为任务处理器。

## 5. 照片入库后的完整任务链路

### 5.1 上传触发流程

```
用户上传照片
    ↓  [API 进程]
POST /api/assets
    ↓
AssetMediaService.handleUpload()
    ↓
StorageTemplateMigrationSingle 入队
    └── 队列: StorageTemplateMigration（串行）
    ↓
任务写入 Redis
───────────────────────────────────────── 进程边界
    ↓  [Microservices 进程]
Worker 消费任务，执行存储迁移
    ↓  [source: upload/copy 才继续]
JobService.onDone() 触发后续任务
    ↓
AssetGenerateThumbnails 入队
    └── 队列: ThumbnailGeneration（并发: 3）
```

### 5.2 缩略图完成后的链式调度

缩略图生成完成后，在 `JobService.onDone()` 中并行触发多个 ML 任务：

```typescript
case JobName.AssetGenerateThumbnails: {
  const jobs: JobItem[] = [
    { name: JobName.SmartSearch, data: item.data },      // CLIP 特征提取
    { name: JobName.AssetDetectFaces, data: item.data }, // 人脸检测
    { name: JobName.Ocr, data: item.data },              // OCR 文字识别
  ];

  // 视频额外触发转码
  if (asset.type === AssetType.Video) {
    jobs.push({ name: JobName.AssetEncodeVideo, data: item.data });
  }

  await this.jobRepository.queueAll(jobs);
  break;
}
```

### 5.3 完整任务链路图

```
照片上传 (API 进程)
    ↓
存储模板迁移 (StorageTemplateMigrationSingle)
    ↓  [source: upload/copy 才继续]
缩略图生成 (AssetGenerateThumbnails)  ──  ThumbnailGeneration 队列 (并发: 3)
    ├─────────────────────────────────┬─────────────────────────┐
    ↓                                 ↓                         ↓
SmartSearch                     AssetDetectFaces          Ocr
 (CLIP 特征)                     (人脸检测)            (文字识别)
 SmartSearch (2)               FaceDetection (2)        Ocr (1)
    ↓
 [upload 来源才触发]
    ↓
AssetDetectDuplicates  ──  DuplicateDetection 队列 (串行)
(重复照片检测)

视频额外链路:
AssetGenerateThumbnails
    ↓  [如果是视频]
AssetEncodeVideo  ──  VideoConversion 队列 (并发: 1)
(视频转码)
```

### 5.4 任务来源字段（source）

`source` 字段用于区分任务触发来源，影响后续调度逻辑：

| source 值 | 触发场景 | 后续行为差异 |
|----------|---------|------------|
| `upload` | 用户上传 | SmartSearch 完成后触发重复检测 |
| `copy` | 复制资产 | 仅触发缩略图生成 |
| `edit` | 编辑操作 | 触发编辑器缩略图重新生成 |
| `sidecar-write` | Sidecar 文件写入 | 触发元数据重新提取 |

## 6. 定时任务与夜间任务

### 6.1 Cron 定时任务

| 任务 | 触发时机 | 调度位置 |
|-----|---------|---------|
| 库扫描 | 可配置，默认午夜 | `LibraryService` |
| 夜间任务 | 可配置，默认 00:00 | `QueueService.handleNightlyJobs()` |
| 数据库备份 | 可配置，默认每天 2 点 | `DatabaseBackupService` |
| 版本检查 | 内置定时 | `VersionService` |

### 6.2 夜间任务批量触发

在 `QueueService.handleNightlyJobs()` 中批量入队：

```typescript
async handleNightlyJobs() {
  const jobs: JobItem[] = [];

  if (config.nightlyTasks.databaseCleanup) {
    jobs.push(
      { name: JobName.AssetDeleteCheck },     // 资产删除检查
      { name: JobName.UserDeleteCheck },      // 用户删除检查
      { name: JobName.PersonCleanup },        // 人物数据清理
      { name: JobName.MemoryCleanup },        // 回忆清理
      { name: JobName.SessionCleanup },       // 会话清理
      { name: JobName.AuditTableCleanup },    // 审计表清理
    );
  }

  if (config.nightlyTasks.generateMemories) {
    jobs.push({ name: JobName.MemoryGenerate });  // 生成回忆
  }

  if (config.nightlyTasks.syncQuotaUsage) {
    jobs.push({ name: JobName.UserSyncUsage });   // 同步配额使用
  }

  if (config.nightlyTasks.missingThumbnails) {
    jobs.push({ name: JobName.AssetGenerateThumbnailsQueueAll });  // 补全缺失缩略图
  }

  if (config.nightlyTasks.clusterNewFaces) {
    jobs.push({ name: JobName.FacialRecognitionQueueAll });  // 人脸聚类
  }

  await this.jobRepository.queueAll(jobs);
}
```

## 7. 队列管理 API

| 操作 | 方法 | 说明 |
|-----|------|------|
| 暂停队列 | `pause(queueName)` | 暂停消费新任务 |
| 恢复队列 | `resume(queueName)` | 恢复队列消费 |
| 清空队列 | `empty(queueName)` | 移除所有等待中的任务 |
| 清理失败任务 | `clear(queueName, type)` | 移除失败的任务 |
| 获取统计 | `getJobCounts(queueName)` | 获取 active/completed/failed/delayed/waiting/paused 计数 |
| 检查活跃 | `isActive(queueName)` | 是否有正在运行的任务 |

## 8. 任务数据结构

### 8.1 主要 Task Data 类型

```typescript
// 基础任务
interface IBaseJob {
  force?: boolean;  // 强制执行，跳过某些检查
}

// 实体任务（最常用，占 80% 以上任务）
interface IEntityJob extends IBaseJob {
  id: string;                    // 资产/用户 ID
  source?: JobSource;            // 任务来源：upload/copy/edit/sidecar-write
  notify?: boolean;              // 是否发送 WebSocket 通知
}

// 延迟任务
interface IDelayedJob extends IBaseJob {
  delay?: number;  // 毫秒
}

// 批量实体任务
interface IBulkEntityJob {
  ids: string[];
}

// 文件删除任务
interface IDeleteFilesJob extends IBaseJob {
  files: Array<string | null | undefined>;
}

// 邮件任务
interface IEmailJob {
  to: string;
  subject: string;
  html: string;
  text: string;
  imageAttachments?: EmailImageAttachment[];
}
```

## 9. 设计特点总结

### 9.1 资源感知的并发策略

| 任务类型 | 并发数策略 | 原因 |
|---------|-----------|------|
| 视频转码、OCR | 1 | 重度 CPU/GPU 消耗，避免系统过载 |
| ML 特征提取（人脸、CLIP） | 2 | 平衡性能与资源占用 |
| 缩略图生成、编辑器 | 2-3 | CPU 密集，适度并发 |
| IO 密集（元数据、通知） | 5 | 等待时间长，高并发提升吞吐量 |

### 9.2 任务链依赖管理

通过 `onDone` 回调实现优雅的任务间依赖：
- 存储迁移完成 → 缩略图生成
- 缩略图生成完成 → 并行触发多个 ML 任务
- SmartSearch 完成（upload 来源）→ 重复检测

### 9.3 幂等性保证

通过 `jobId` 去重机制确保关键任务不会重复执行：
- 全局单例任务（备份、聚类）
- 同一资产的幂等操作（存储迁移）
- 同一接收者的通知合并（相册更新）
