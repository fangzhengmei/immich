# Immich 任务队列调度机制详解

## 1. 概述

Immich 使用 BullMQ 作为任务队列的底层实现，通过模块化的任务调度系统处理照片入库后的各种后台任务，包括缩略图生成、视频转码、机器学习标签提取、人脸检测等。

## 2. 核心架构

### 2.1 关键组件

| 组件 | 文件 | 职责 |
|------|------|------|
| JobRepository | `repositories/job.repository.ts` | 任务管理、队列操作、Worker 管理 |
| QueueService | `services/queue.service.ts` | 队列服务、并发配置、定时任务、任务启动 |
| JobService | `services/job.service.ts` | 任务生命周期、后续任务调度 |
| @OnJob 装饰器 | `decorators.ts` | 任务处理器注册 |

### 2.2 Worker 分组

Immich 有三种 Worker 类型，由 `ImmichWorker` 枚举定义：

```typescript
enum ImmichWorker {
  Api = 'api',              // API 服务，处理 HTTP 请求
  Maintenance = 'maintenance',  // 维护任务
  Microservices = 'microservices', // 微服务 Worker，实际执行后台任务
}
```

**只有 `Microservices` Worker 会启动任务队列的 Worker 进程**，负责实际执行后台任务。

## 3. 队列分类与并发配置

### 3.1 队列列表

共有 14 个队列，由 `QueueName` 枚举定义，分为并发队列和串行队列两类。

### 3.2 并发队列（支持多任务并行）

以下队列支持并发执行，默认并发数配置在 `config.ts` 中：

| 队列名称 | 默认并发数 | 主要任务 |
|---------|-----------|---------|
| `BackgroundTask` | 5 | 各种后台清理任务 |
| `SmartSearch` | 2 | CLIP 智能搜索特征提取 |
| `MetadataExtraction` | 5 | EXIF 元数据提取 |
| `FaceDetection` | 2 | 人脸检测 |
| `Search` | 5 | 搜索相关任务 |
| `Sidecar` | 5 | XMP Sidecar 文件读写 |
| `Library` | 5 | 库扫描、同步 |
| `Migration` | 5 | 数据迁移 |
| `ThumbnailGeneration` | 3 | 缩略图生成 |
| `VideoConversion` | 1 | 视频转码（默认单线程，因为资源密集） |
| `Notification` | 5 | 邮件通知 |
| `Ocr` | 1 | 文字识别（资源密集） |
| `Workflow` | 5 | 工作流执行 |
| `Editor` | 2 | 编辑器相关任务 |

### 3.3 串行队列（单任务执行）

以下队列强制单线程执行，不支持并发：

| 队列名称 | 原因 |
|---------|------|
| `FacialRecognition` | 人脸聚类需要顺序处理，避免数据冲突 |
| `StorageTemplateMigration` | 存储模板迁移，涉及文件系统操作 |
| `DuplicateDetection` | 重复照片检测，资源密集型 |
| `BackupDatabase` | 数据库备份，需要独占资源 |

### 3.4 并发配置机制

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

## 4. 任务分类与优先级

### 4.1 任务类型（JobName）

系统定义了 60+ 种任务类型，主要分类：

| 类别 | 任务示例 |
|-----|---------|
| **缩略图** | `AssetGenerateThumbnails`, `PersonGenerateThumbnail`, `AssetEditThumbnailGeneration` |
| **视频** | `AssetEncodeVideo`, `AssetEncodeVideoQueueAll` |
| **元数据** | `AssetExtractMetadata`, `SidecarCheck`, `SidecarWrite` |
| **ML 人脸** | `AssetDetectFaces`, `FacialRecognition`, `PersonCleanup` |
| **ML 搜索** | `SmartSearch`, `AssetDetectDuplicates` |
| **OCR** | `Ocr` |
| **存储** | `StorageTemplateMigration`, `AssetFileMigration` |
| **清理** | `AssetDeleteCheck`, `UserDeleteCheck`, `SessionCleanup`, `TagCleanup` |
| **通知** | `SendMail`, `NotifyAlbumInvite`, `NotifyAlbumUpdate` |
| **记忆** | `MemoryGenerate`, `MemoryCleanup` |

### 4.2 任务优先级

目前只有 `PersonGenerateThumbnail`（人物缩略图）设置了高优先级：

```typescript
// job.repository.ts
private getJobOptions(item: JobItem): JobsOptions | null {
  switch (item.name) {
    case JobName.PersonGenerateThumbnail: {
      return { priority: 1 };  // 数值越小优先级越高
    }
    // ...
  }
}
```

### 4.3 任务去重与延迟

部分任务支持去重和延迟执行：

| 任务 | 配置 | 说明 |
|-----|------|------|
| `NotifyAlbumUpdate` | `jobId: ${id}/${recipientId}`, delay | 同一接收者的相册更新通知去重 + 延迟 |
| `StorageTemplateMigrationSingle` | `jobId: item.data.id` | 按资产 ID 去重 |
| `FacialRecognitionQueueAll` | `jobId: JobName` | 全局单例 |
| `VersionCheck` | `jobId: JobName` | 全局单例 |

## 5. 任务入队流程

### 5.1 任务入队方式

#### 方式一：单个任务入队

```typescript
await this.jobRepository.queue({
  name: JobName.AssetGenerateThumbnails,
  data: { id: assetId, source: 'upload' }
});
```

#### 方式二：批量任务入队

```typescript
await this.jobRepository.queueAll([
  { name: JobName.SmartSearch, data: { id: assetId } },
  { name: JobName.AssetDetectFaces, data: { id: assetId } },
  { name: JobName.Ocr, data: { id: assetId } },
]);
```

`queueAll` 内部优化：
- 按队列分组，使用 `addBulk` 批量入队提高性能
- 对需要 `jobId` 去重的任务，单独使用 `add` 入队

### 5.2 任务处理器注册

使用 `@OnJob` 装饰器注册任务处理器：

```typescript
@OnJob({ name: JobName.AssetGenerateThumbnails, queue: QueueName.ThumbnailGeneration })
async generateThumbnails({ id }: IEntityJob): Promise<JobStatus> {
  // 处理逻辑
  return JobStatus.Success;
}
```

`JobRepository.setup()` 在启动时自动扫描所有服务类，收集标记了 `@OnJob` 的方法作为任务处理器。

### 5.3 任务执行流程

```
任务入队 → BullMQ 队列 → Worker 池 → JobRun 事件 →
JobService.onJobRun() → 发射 JobStart → 调用处理器 →
发射 JobSuccess/JobError → 发射 JobComplete
```

## 6. 照片入库后的完整任务链路

### 6.1 上传后的任务触发点

在 `JobService.onDone()` 中，当一个任务完成后，会触发后续任务：

```typescript
// 缩略图生成完成后，触发后续 ML 任务
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

// SmartSearch 完成后触发重复检测
case JobName.SmartSearch: {
  if (item.data.source === 'upload') {
    await this.jobRepository.queue({
      name: JobName.AssetDetectDuplicates,
      data: item.data
    });
  }
  break;
}
```

### 6.2 完整任务链路图

```
照片上传
    ↓
存储模板迁移 (StorageTemplateMigrationSingle)
    ↓  [source: upload/copy 才继续]
缩略图生成 (AssetGenerateThumbnails)  ──  ThumbnailGeneration 队列 (并发: 3)
    ├─────────────────────────────────┐
    ↓                                 ↓
SmartSearch                     AssetDetectFaces          OCR
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

## 7. 定时任务与夜间任务

### 7.1 Cron 定时任务

| 任务 | 触发时间 | 说明 |
|-----|---------|------|
| 库扫描 | 每天午夜 | `LibraryScanQueueAll` |
| 夜间任务 | 可配置，默认 00:00 | 批量执行清理和维护任务 |
| 版本检查 | 内置 | `VersionCheck` |

### 7.2 夜间任务（Nightly Tasks）

在 `QueueService.handleNightlyJobs()` 中批量触发：

```typescript
async handleNightlyJobs() {
  const jobs: JobItem[] = [];

  if (config.nightlyTasks.databaseCleanup) {
    jobs.push(
      { name: JobName.AssetDeleteCheck },     // 资产删除检查
      { name: JobName.UserDeleteCheck },      // 用户删除检查
      { name: JobName.PersonCleanup },        // 人物数据清理
      { name: JobName.MemoryCleanup },        // 记忆清理
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

## 8. 任务队列管理 API

QueueService 提供以下队列管理功能：

| 操作 | 说明 |
|-----|------|
| `pause(queueName)` | 暂停队列 |
| `resume(queueName)` | 恢复队列 |
| `empty(queueName)` | 清空队列 |
| `clear(queueName, type)` | 清理失败任务 |
| `getJobCounts(queueName)` | 获取队列统计（active、completed、failed、delayed、waiting、paused） |
| `isActive(queueName)` | 检查队列是否有活跃任务 |

## 9. 关键设计特点

### 9.1 资源感知的并发策略

- CPU 密集型任务（视频转码、OCR）默认并发为 1
- IO 密集型任务（元数据提取、通知）并发数较高
- ML 相关任务并发适中（2-3）

### 9.2 任务链依赖管理

通过 `onDone` 回调实现任务间的依赖关系，确保：
- 缩略图生成完成后才开始 ML 处理
- 存储模板迁移完成后才生成缩略图
- 特征提取完成后才进行重复检测

### 9.3 幂等性保证

通过 `jobId` 去重机制，确保关键任务不会重复执行：
- 数据库备份全局单例
- 人脸聚类全局单例
- 同一资产的存储迁移不会重复执行
- 同一接收者的通知会合并

## 10. 任务数据结构

### 10.1 JobItem 类型

所有任务都遵循 `JobItem` 接口，主要数据类型：

```typescript
// 基础任务
interface IBaseJob {
  force?: boolean;
}

// 实体任务（最常用）
interface IEntityJob extends IBaseJob {
  id: string;
  source?: 'upload' | 'sidecar-write' | 'copy' | 'edit';  // 任务来源
  notify?: boolean;
}

// 延迟任务
interface IDelayedJob extends IBaseJob {
  delay?: number;  // 毫秒
}

// 批量实体任务
interface IBulkEntityJob {
  ids: string[];
}
```

### 10.2 任务来源（source）

`source` 字段用于区分任务触发来源，影响后续任务调度：

- `upload`: 用户上传触发
- `copy`: 复制操作触发
- `edit`: 编辑操作触发
- `sidecar-write`: Sidecar 文件写入触发

例如，只有 `upload` 来源的 `SmartSearch` 完成后才会触发重复检测。
