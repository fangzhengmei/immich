# Immich 后台作业队列优先级编排与失败重试逻辑分析

## 一、整体架构概述

Immich 使用 **BullMQ**（基于 `@nestjs/bullmq` 封装）作为后台作业队列引擎，Redis 作为后端存储。整个队列系统围绕三个核心角色展开：

| 角色 | 所在模块 | 核心职责 |
|------|---------|---------|
| `JobRepository` | `server/src/repositories/job.repository.ts` | 队列底层操作、作业调度、Worker 管理 |
| `QueueService` | `server/src/services/queue.service.ts` | 队列 API 层、并发配置、夜间作业编排 |
| `JobService` | `server/src/services/job.service.ts` | 作业执行生命周期管理、作业链编排 |

---

## 二、队列调度机制

### 2.1 队列定义

共定义了 **18 个命名队列**（`enum.ts:759-778`），按功能领域划分：

```typescript
enum QueueName {
  ThumbnailGeneration = 'thumbnailGeneration',      // 缩略图生成
  MetadataExtraction = 'metadataExtraction',        // 元数据提取
  VideoConversion = 'videoConversion',              // 视频转码
  FaceDetection = 'faceDetection',                  // 人脸检测
  FacialRecognition = 'facialRecognition',          // 人脸识别
  SmartSearch = 'smartSearch',                      // 智能搜索
  DuplicateDetection = 'duplicateDetection',        // 重复检测
  BackgroundTask = 'backgroundTask',                // 后台任务
  StorageTemplateMigration = 'storageTemplateMigration', // 存储模板迁移
  Migration = 'migration',                          // 文件迁移
  Search = 'search',                                // 搜索索引
  Sidecar = 'sidecar',                              // Sidecar 文件
  Library = 'library',                              // 媒体库扫描
  Notification = 'notifications',                   // 通知
  BackupDatabase = 'backupDatabase',                // 数据库备份
  Ocr = 'ocr',                                      // OCR 识别
  Workflow = 'workflow',                            // 工作流
  Editor = 'editor',                                // 编辑器
}
```

### 2.2 作业处理器注册机制

通过 `@OnJob` 装饰器（`decorators.ts:150-154`）声明作业与队列的映射关系：

```typescript
// decorators.ts:150-154
export type JobConfig = {
  name: JobName;
  queue: QueueName;
};
export const OnJob = (config: JobConfig) => SetMetadata(MetadataKey.JobConfig, config);
```

**注册流程**（`job.repository.ts:36-84`）：
1. `JobRepository.setup()` 在启动时扫描所有 Service 类
2. 通过 `Reflector` 提取 `@OnJob` 装饰器元数据
3. 建立 `JobName` → `handler` 的映射关系
4. 校验所有 `JobName` 必须有对应的处理器

**作业处理示例**：
```typescript
// person.service.ts:302
@OnJob({ name: JobName.AssetDetectFaces, queue: QueueName.FaceDetection })
async handleDetectFaces({ id }: JobOf<JobName.AssetDetectFaces>) {
  // 人脸检测逻辑
}
```

### 2.3 Worker 启动流程

Worker 仅在 **Microservices** 进程中启动（`job.repository.ts:86-96`）：

```typescript
startWorkers() {
  const { bull } = this.configRepository.getEnv();
  for (const queueName of Object.values(QueueName)) {
    this.workers[queueName] = new Worker(
      queueName,
      (job) => this.eventRepository.emit('JobRun', queueName, job as JobItem),
      { ...bull.config, concurrency: 1 }, // 初始并发为 1，后续动态调整
    );
  }
}
```

---

## 三、优先级编排实现

Immich 的优先级编排较为分散，主要通过以下几种机制实现：

### 3.1 队列级隔离（隐式优先级）

不同作业分发到不同队列，通过队列的并发配置间接实现优先级。这是最主要的优先级控制方式。

### 3.2 作业级优先级选项

在 `JobRepository.getJobOptions()`（`job.repository.ts:218-245`）中为特定作业设置特殊选项：

| 作业名称 | 优先级选项 | 说明 |
|---------|-----------|------|
| `NotifyAlbumUpdate` | `jobId`, `delay` | 按相册+收件人去重，延迟 300s 发送 |
| `StorageTemplateMigrationSingle` | `jobId` | 按资产 ID 去重，防止重复迁移 |
| `PersonGenerateThumbnail` | `priority: 1` | 高优先级（数字越小优先级越高） |
| `FacialRecognitionQueueAll` | `deduplication` | 防止重复触发全量人脸识别 |
| `VersionCheck` | `deduplication` | 防止重复版本检查 |
| `DatabaseBackup` | `deduplication` | 防止重复备份 |

**代码示例**：
```typescript
// job.repository.ts:218-245
private getJobOptions(item: JobItem): JobsOptions | null {
  switch (item.name) {
    case JobName.PersonGenerateThumbnail: {
      return { priority: 1 };
    }
    case JobName.FacialRecognitionQueueAll: {
      return { deduplication: { id: JobName.FacialRecognitionQueueAll } };
    }
    // ...
  }
}
```

### 3.3 作业链编排（依赖优先级）

通过 `JobService.onDone()`（`job.service.ts:68-225`）实现作业间的依赖关系，前序作业完成后触发后续作业：

```
AssetGenerateThumbnails 完成后 →
  ├─→ SmartSearch
  ├─→ AssetDetectFaces
  ├─→ Ocr
  └─→ (若为视频) AssetEncodeVideo

SidecarCheck 完成后 → AssetExtractMetadata

SmartSearch(upload 来源) 完成后 → AssetDetectDuplicates
```

### 3.4 夜间作业批量编排

`QueueService.handleNightlyJobs()`（`queue.service.ts:261-293`）按配置在指定时间批量触发作业：

```typescript
async handleNightlyJobs() {
  const config = await this.getConfig({ withCache: false });
  const jobs: JobItem[] = [];

  if (config.nightlyTasks.databaseCleanup) {
    jobs.push(
      { name: JobName.AssetDeleteCheck },
      { name: JobName.UserDeleteCheck },
      { name: JobName.PersonCleanup },
      // ...
    );
  }
  // ... 其他夜间作业

  await this.jobRepository.queueAll(jobs);
}
```

---

## 四、并发上限控制

### 4.1 队列并发分类

**单例队列**（并发固定为 1，不可配置）：
- `FacialRecognition` - 人脸识别需要顺序执行
- `StorageTemplateMigration` - 存储模板迁移
- `DuplicateDetection` - 重复检测
- `BackupDatabase` - 数据库备份

**可配置并发队列**（`types.ts:199-205`）：
```typescript
export type ConcurrentQueueName = Exclude<
  QueueName,
  | QueueName.StorageTemplateMigration
  | QueueName.FacialRecognition
  | QueueName.DuplicateDetection
  | QueueName.BackupDatabase
>;
```

### 4.2 默认并发配置

默认值定义在 `config.ts:228-243`：

| 队列名称 | 默认并发 | 说明 |
|---------|---------|------|
| `BackgroundTask` | 5 | 后台清理类任务 |
| `SmartSearch` | 2 | 智能搜索向量化 |
| `MetadataExtraction` | 5 | EXIF 元数据提取 |
| `FaceDetection` | 2 | 人脸检测 |
| `Search` | 5 | 搜索索引 |
| `Sidecar` | 5 | XMP/侧记文件处理 |
| `Library` | 5 | 媒体库扫描 |
| `Migration` | 5 | 文件迁移 |
| `ThumbnailGeneration` | 3 | 缩略图生成（CPU 密集） |
| `VideoConversion` | 1 | 视频转码（资源密集） |
| `Notification` | 5 | 通知发送 |
| `Ocr` | 1 | OCR 识别（ML 调用） |
| `Workflow` | 5 | 工作流执行 |
| `Editor` | 2 | 编辑器任务 |

### 4.3 并发动态调整

`QueueService.updateConcurrency()`（`queue.service.ts:86-96`）在配置变更时动态调整：

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

触发时机：
- `ConfigInit` 事件（启动时）
- `ConfigUpdate` 事件（运行时配置变更）

### 4.4 BullMQ 全局默认配置

```typescript
// config.repository.ts:281-292
bull: {
  config: {
    prefix: 'immich_bull',
    connection: { ...redisConfig },
    defaultJobOptions: {
      attempts: 1,           // 无自动重试
      removeOnComplete: true, // 成功后删除
      removeOnFail: false,    // 失败后保留以便排查
    },
  },
  queues: Object.values(QueueName).map((name) => ({ name })),
}
```

---

## 五、失败重试与恢复路径

### 5.1 核心发现：无自动重试

**关键配置**（`config.repository.ts:286`）：
```typescript
defaultJobOptions: {
  attempts: 1,  // 仅尝试 1 次，无自动重试
  // ...
}
```

这是 Immich 队列系统最显著的设计特征——**所有作业默认不进行自动重试**。

### 5.2 作业执行生命周期

`JobService.onJobRun()`（`job.service.ts:49-63`）管理作业执行全流程：

```typescript
@OnEvent({ name: 'JobRun' })
async onJobRun(...[queueName, job]: ArgsOf<'JobRun'>) {
  try {
    await this.eventRepository.emit('JobStart', queueName, job);
    const response = await this.jobRepository.run(job);
    await this.eventRepository.emit('JobSuccess', { job, response });
    if (response && typeof response === 'string' && [JobStatus.Success, JobStatus.Skipped].includes(response)) {
      await this.onDone(job); // 触发后续作业链
    }
  } catch (error: Error | any) {
    await this.eventRepository.emit('JobError', { job, error }); // 发布失败事件
  } finally {
    await this.eventRepository.emit('JobComplete', queueName, job);
  }
}
```

**状态流转**：
```
JobRun → JobStart → 执行 handler → JobSuccess → onDone(后续作业)
                    ↓ 异常
                  JobError → (仅 DatabaseBackup 发送通知)
                    ↓
                  JobComplete
```

### 5.3 失败事件监听

目前有两个服务监听 `JobError` 事件：

#### 5.3.1 通知服务（NotificationService）

`notification.service.ts:81-109`：
```typescript
@OnEvent({ name: 'JobError' })
async onJobError({ job, error }: ArgOf<'JobError'>) {
  const admin = await this.userRepository.getAdmin();
  if (!admin) return;

  this.logger.error(`Unable to run job handler (${job.name}): ${error}`, ...);

  switch (job.name) {
    case JobName.DatabaseBackup: {
      // 仅数据库备份失败时发送 UI 通知
      const item = await this.notificationRepository.create({
        userId: admin.id,
        type: NotificationType.JobFailed,
        level: NotificationLevel.Error,
        title: 'Job Failed',
        description: `Job ${[job.name]} failed with error: ${errorMessage}`,
      });
      this.websocketRepository.clientSend('on_notification', admin.id, mapNotification(item));
      break;
    }
    default:
      return; // 其他作业失败仅记录日志，不发送通知
  }
}
```

#### 5.3.2 遥测服务（TelemetryService）

`telemetry.service.ts:43-47`：
```typescript
@OnEvent({ name: 'JobError' })
onJobError({ job }: ArgOf<'JobError'>) {
  const jobMetric = `immich.jobs.${snakeCase(job.name)}.${JobStatus.Failed}`;
  this.telemetryRepository.jobs.addToCounter(jobMetric, 1);
}
```

### 5.4 手动恢复路径

由于无自动重试，失败恢复完全依赖手动操作。需要特别注意**新旧两套接口的功能差异**。

---

#### 5.4.1 新旧接口映射关系

Immich 存在两套队列运维接口，**启动队列的功能仅存在于旧接口**：

| 控制器 | 路由前缀 | 接口版本 | 核心功能 | 关键限制 |
|--------|---------|---------|---------|---------|
| `JobController` | `/jobs` | **旧接口** (v1/v2) | ✅ 启动队列 (start)<br>✅ 暂停/恢复队列<br>✅ 清空队列<br>✅ 清除失败作业<br>✅ 创建手动作业 | `PUT /jobs/:name` 是**唯一**能启动队列的入口 |
| `QueueController` | `/queues` | **新接口** (v2.4.0 alpha) | ✅ 查询队列列表/详情<br>✅ 暂停/恢复队列<br>✅ 清空队列<br>✅ 查询作业列表 | ❌ **无启动队列功能** |

> ⚠️ **重要提醒**：手动重跑操作（重新生成缩略图、重新识别等）必须通过 `PUT /api/jobs/:name` 旧接口进行，`/api/queues` 新接口不支持启动操作。

---

#### 5.4.2 接口详细说明

##### 旧接口（/jobs 前缀）

| HTTP 方法 | 路径 | 功能 | 说明 |
|----------|------|------|------|
| `GET` | `/api/jobs` | 查询队列状态（已废弃） | 返回所有队列的作业计数和状态，v2.4.0 起废弃 |
| `POST` | `/api/jobs` | 创建手动作业 | 触发清理类任务：人员清理、标签清理、用户清理、记忆清理、记忆生成、数据库备份 |
| `PUT` | `/api/jobs/:name` | 执行队列命令 | **唯一支持 start 命令的接口**，支持：`start`、`pause`、`resume`、`empty`、`clear-failed` |

##### 新接口（/queues 前缀）

| HTTP 方法 | 路径 | 功能 | 说明 |
|----------|------|------|------|
| `GET` | `/api/queues` | 查询所有队列 | 返回队列名称、暂停状态、统计信息 |
| `GET` | `/api/queues/:name` | 查询单个队列 | 返回指定队列的详细信息 |
| `PUT` | `/api/queues/:name` | 更新队列状态 | 仅支持 `isPaused` 参数，用于暂停/恢复 |
| `GET` | `/api/queues/:name/jobs` | 查询队列作业 | 可按状态筛选（active/failed/completed/delayed/waiting/paused） |
| `DELETE` | `/api/queues/:name/jobs` | 清空队列 | 清空等待中的作业，可选择同时清理失败作业 |

---

#### 5.4.3 队列可操作权限清单

共 18 个队列，按操作权限分类：

| 队列名称 | 可启动 (start) | 可暂停 (pause) | 可清空 (empty) | 触发的批量作业 |
|---------|:--------------:|:--------------:|:--------------:|---------------|
| `ThumbnailGeneration` | ✅ | ✅ | ✅ | `AssetGenerateThumbnailsQueueAll` |
| `MetadataExtraction` | ✅ | ✅ | ✅ | `AssetExtractMetadataQueueAll` |
| `VideoConversion` | ✅ | ✅ | ✅ | `AssetEncodeVideoQueueAll` |
| `FaceDetection` | ✅ | ✅ | ✅ | `AssetDetectFacesQueueAll` |
| `FacialRecognition` | ✅ | ✅ | ✅ | `FacialRecognitionQueueAll` |
| `SmartSearch` | ✅ | ✅ | ✅ | `SmartSearchQueueAll` |
| `DuplicateDetection` | ✅ | ✅ | ✅ | `AssetDetectDuplicatesQueueAll` |
| `Sidecar` | ✅ | ✅ | ✅ | `SidecarQueueAll` |
| `Library` | ✅ | ✅ | ✅ | `LibraryScanQueueAll` |
| `BackupDatabase` | ✅ | ✅ | ✅ | `DatabaseBackup` |
| `Ocr` | ✅ | ✅ | ✅ | `OcrQueueAll` |
| `StorageTemplateMigration` | ✅ | ✅ | ✅ | `StorageTemplateMigration` |
| `Migration` | ✅ | ✅ | ✅ | `FileMigrationQueueAll` |
| `Search` | ❌ | ✅ | ✅ | - |
| `Notification` | ❌ | ✅ | ✅ | - |
| `Workflow` | ❌ | ✅ | ✅ | - |
| `Editor` | ❌ | ✅ | ✅ | - |
| `BackgroundTask` | ❌ | ❌ | ✅ | - |

> **说明**：
> - ✅ **可启动的队列**（13个）：均有对应的 `*QueueAll` 批量作业
> - ❌ **不可启动的队列**（5个）：均为事件驱动，无批量作业入口
> - ❌ **不可暂停的队列**（1个）：`BackgroundTask` 队列包含系统关键任务，禁止暂停

---

#### 5.4.4 `force` 参数深度解析

> ⚠️ **重要修正**：之前文档中"force=true=跳过已处理"的描述是错误的。实际含义完全相反。

##### `force` 参数的核心含义

| 参数值 | 含义 | 处理范围 |
|--------|------|---------|
| `force: false`（默认） | **增量处理** | 只处理**从未处理过**的资产，跳过已有处理结果的资产 |
| `force: true` | **全量重处理** | 处理**所有**资产，包括已处理过的；会先清空已有结果再重新处理 |

---

##### `force` 参数的完整传递链路

```
API 请求层
    ↓
PUT /api/jobs/:name { command: "start", force: boolean }
    ↓
Controller 层 (job.controller.ts:45-58)
    ↓
Service 层 (queue.service.ts:185-250)
  start(name, { force })
    ↓ 封装为批量作业
  { name: JobName.AssetGenerateThumbnailsQueueAll, data: { force } }
    ↓
批量作业 Handler 层 (各 service.handleQueue*)
  ├─ 预处理阶段：force=true 时先删除已有结果
  └─ 查询阶段：streamFor*(force) → 传递给 SQL
    ↓
Repository 层 (asset-job.repository.ts)
  $if(!force, (qb) => qb.where(/* 过滤已处理资产 */))
```

---

##### 各队列对 `force=true` 的预处理行为

当 `force=true` 时，不同队列在批量处理前会执行清空操作：

| 队列 | 预处理操作（force=true 时） | 代码位置 |
|------|----------------------------|---------|
| **FaceDetection** 人脸检测 | 删除所有 ML 人脸 → 人员清理 → 重建向量索引 | `person.service.ts:276-279` |
| **FacialRecognition** 人脸识别 | 取消所有 ML 人脸分配 → 人员清理 → 重建向量索引 | `person.service.ts:426-429` |
| **SmartSearch** 智能搜索 | 删除所有 CLIP embedding → 更新模型维度 | `smart-info.service.ts:74-78` |
| **Ocr** 文字识别 | 删除所有 OCR 记录 | `ocr.service.ts:20-22` |
| **ThumbnailGeneration** 缩略图 | 无预处理，直接重处理所有 | `media.service.ts:70-117` |
| **MetadataExtraction** 元数据 | 无预处理，直接重处理所有 | `metadata.service.ts:218-233` |
| **VideoConversion** 视频转码 | 无预处理，直接重处理所有 | `media.service.ts:551-*` |
| **DuplicateDetection** 重复检测 | 无预处理，直接重处理所有 | `duplicate.service.ts:302-325` |
| **Sidecar** 侧记文件 | 无预处理，直接重处理所有 | `metadata.service.ts:419-437` |

---

##### SQL 查询层的 `force` 逻辑

所有 `streamFor*` 方法都采用相同的条件构建模式：

```typescript
// asset-job.repository.ts:64-104 (缩略图示例)
streamForThumbnailJob(options: { force: boolean | undefined; ... }) {
  return this.db
    .selectFrom('asset')
    .select(['asset.id', 'asset.isEdited'])
    .where('asset.deletedAt', 'is', null)
    .$if(!options.force, (qb) =>
      qb
        .innerJoin('asset_job_status', ...)
        .where(({ and, eb, exists, not, or, selectFrom }) => {
          // force=false 时，只查询缺少缩略图/预览图的资产
          const conditions = [
            not(exists(file(AssetFileType.Thumbnail))),  // 无缩略图
            not(exists(file(AssetFileType.Preview))),    // 无预览图
            eb('asset.thumbhash', 'is', null),           // 无 thumbhash
          ];
          return or(conditions);
        }),
    )
    .stream();
}
```

**各队列的过滤条件（force=false 时）**：

| 队列 | 过滤条件（只处理未处理的资产） |
|------|-------------------------------|
| **ThumbnailGeneration** | 缺少缩略图 / 缺少预览图 / 缺少 thumbhash / 已编辑但缺少全尺寸图 |
| **MetadataExtraction** | `metadataExtractedAt IS NULL` |
| **FaceDetection** | `facesRecognizedAt IS NULL` |
| **SmartSearch** | `smart_search` 表中无记录 |
| **DuplicateDetection** | `duplicatesDetectedAt IS NULL` |
| **VideoConversion** | 无 `EncodedVideo` 类型的 asset_file |
| **Ocr** | `ocrAt IS NULL` |
| **Sidecar** | 无 `Sidecar` 类型的 asset_file |

**force=true 时**：所有上述 `$if(!force, ...)` 条件被跳过，查询范围扩大到**所有未删除、非隐藏的资产**。

---

##### 对重跑任务范围的影响总结

| 场景 | force 参数 | 处理范围 | 预计处理量 | 适用场景 |
|------|-----------|---------|-----------|---------|
| 初次上传后 | `false` | 仅未处理的新资产 | 小 | 日常增量处理 |
| 夜间任务 | `false` | 仅未处理的资产 | 中 | 定时补漏 |
| 部分作业失败后 | `false` | 仅失败的未处理资产 | 小 | 故障恢复（推荐） |
| 模型升级后 | `true` | 所有资产，先清空旧结果 | 大 | ML 模型版本变更 |
| 数据损坏后 | `true` | 所有资产，先清空旧结果 | 大 | 数据修复 |
| 功能开启后 | `false` → 逐步 `true` | 先增量，再全量 | 中→大 | 新功能上线 |

> **最佳实践**：
> 1. 常规恢复优先使用 `force=false`，只处理未处理的资产，避免浪费资源
> 2. 只有在模型升级、数据损坏等特殊场景下才使用 `force=true`
> 3. `force=true` 会先删除已有结果，期间相关功能可能暂时不可用

---

#### 5.4.5 按接口分组的恢复操作清单

##### 📋 通过旧接口 `PUT /api/jobs/:name` 的恢复操作

**启动队列（重新处理所有资产）**：
```bash
# 重新生成所有缩略图
curl -X PUT /api/jobs/thumbnailGeneration \
  -H "Content-Type: application/json" \
  -d '{"command": "start", "force": false}'

# 强制重新生成所有缩略图（包括已处理过的）
curl -X PUT /api/jobs/thumbnailGeneration \
  -H "Content-Type: application/json" \
  -d '{"command": "start", "force": true}'

# 重新提取所有元数据
curl -X PUT /api/jobs/metadataExtraction \
  -H "Content-Type: application/json" \
  -d '{"command": "start", "force": false}'

# 重新识别人脸
curl -X PUT /api/jobs/faceDetection \
  -H "Content-Type: application/json" \
  -d '{"command": "start", "force": false}'

# 重新进行人脸识别聚类
curl -X PUT /api/jobs/facialRecognition \
  -H "Content-Type: application/json" \
  -d '{"command": "start", "force": false}'

# 重新向量化（智能搜索）
curl -X PUT /api/jobs/smartSearch \
  -H "Content-Type: application/json" \
  -d '{"command": "start", "force": false}'

# 重新进行重复检测
curl -X PUT /api/jobs/duplicateDetection \
  -H "Content-Type: application/json" \
  -d '{"command": "start", "force": false}'

# 重新进行 OCR 识别
curl -X PUT /api/jobs/ocr \
  -H "Content-Type: application/json" \
  -d '{"command": "start", "force": false}'
```

**其他旧接口命令**：
```bash
# 暂停队列（已废弃，建议用新接口）
curl -X PUT /api/jobs/thumbnailGeneration \
  -H "Content-Type: application/json" \
  -d '{"command": "pause"}'

# 恢复队列（已废弃，建议用新接口）
curl -X PUT /api/jobs/thumbnailGeneration \
  -H "Content-Type: application/json" \
  -d '{"command": "resume"}'

# 清空队列（已废弃，建议用新接口）
curl -X PUT /api/jobs/thumbnailGeneration \
  -H "Content-Type: application/json" \
  -d '{"command": "empty"}'

# 清除失败作业（已废弃，建议用新接口）
curl -X PUT /api/jobs/thumbnailGeneration \
  -H "Content-Type: application/json" \
  -d '{"command": "clear-failed"}'
```

##### 📋 通过新接口 `/api/queues` 的恢复操作

**查询失败作业**：
```bash
# 查询缩略图队列的失败作业
curl /api/queues/thumbnailGeneration/jobs?status=failed

# 查询所有状态的作业
curl /api/queues/thumbnailGeneration/jobs
```

**暂停/恢复队列**：
```bash
# 暂停队列
curl -X PUT /api/queues/thumbnailGeneration \
  -H "Content-Type: application/json" \
  -d '{"isPaused": true}'

# 恢复队列
curl -X PUT /api/queues/thumbnailGeneration \
  -H "Content-Type: application/json" \
  -d '{"isPaused": false}'
```

**清空队列**：
```bash
# 仅清空等待中的作业
curl -X DELETE /api/queues/thumbnailGeneration/jobs \
  -H "Content-Type: application/json" \
  -d '{}'

# 清空等待中的作业 + 清除失败作业
curl -X DELETE /api/queues/thumbnailGeneration/jobs \
  -H "Content-Type: application/json" \
  -d '{"failed": true}'
```

##### 📋 通过 `POST /api/jobs` 的手动作业

```bash
# 人员清理
curl -X POST /api/jobs \
  -H "Content-Type: application/json" \
  -d '{"name": "person-cleanup"}'

# 标签清理
curl -X POST /api/jobs \
  -H "Content-Type: application/json" \
  -d '{"name": "tag-cleanup"}'

# 用户清理
curl -X POST /api/jobs \
  -H "Content-Type: application/json" \
  -d '{"name": "user-cleanup"}'

# 记忆清理
curl -X POST /api/jobs \
  -H "Content-Type: application/json" \
  -d '{"name": "memory-cleanup"}'

# 生成记忆
curl -X POST /api/jobs \
  -H "Content-Type: application/json" \
  -d '{"name": "memory-create"}'

# 数据库备份
curl -X POST /api/jobs \
  -H "Content-Type: application/json" \
  -d '{"name": "backup-database"}'
```

---

#### 5.4.5 标准恢复流程建议

**典型故障恢复步骤**：

1. **查询失败作业**：
   ```bash
   curl /api/queues/thumbnailGeneration/jobs?status=failed
   ```

2. **清除失败作业**（如需）：
   ```bash
   curl -X DELETE /api/queues/thumbnailGeneration/jobs \
     -H "Content-Type: application/json" \
     -d '{"failed": true}'
   ```

3. **暂停队列**（如需）：
   ```bash
   curl -X PUT /api/queues/thumbnailGeneration \
     -H "Content-Type: application/json" \
     -d '{"isPaused": true}'
   ```

4. **重新触发处理**（**必须使用旧接口**）：
   ```bash
   curl -X PUT /api/jobs/thumbnailGeneration \
     -H "Content-Type: application/json" \
     -d '{"command": "start", "force": false}'
   ```

5. **恢复队列**（如果之前暂停了）：
   ```bash
   curl -X PUT /api/queues/thumbnailGeneration \
     -H "Content-Type: application/json" \
     -d '{"isPaused": false}'
   ```

---

#### 5.4.7 清除失败作业（代码实现）

```typescript
// queue.service.ts:126-130
case QueueCommand.ClearFailed: {
  const failedJobs = await this.jobRepository.clear(name, QueueCleanType.Failed);
  this.logger.debug(`Cleared failed jobs: ${failedJobs}`);
  break;
}
```

---

#### 5.4.7 重新触发队列（代码实现）

```typescript
// queue.service.ts:185-250
private async start(name: QueueName, { force }: QueueCommandDto): Promise<void> {
  const isActive = await this.jobRepository.isActive(name);
  if (isActive) throw new BadRequestException(`Job is already running`);

  await this.eventRepository.emit('QueueStart', { name });

  switch (name) {
    case QueueName.ThumbnailGeneration:
      return this.jobRepository.queue({ name: JobName.AssetGenerateThumbnailsQueueAll, data: { force } });
    case QueueName.MetadataExtraction:
      return this.jobRepository.queue({ name: JobName.AssetExtractMetadataQueueAll, data: { force } });
    case QueueName.FaceDetection:
      return this.jobRepository.queue({ name: JobName.AssetDetectFacesQueueAll, data: { force } });
    case QueueName.FacialRecognition:
      return this.jobRepository.queue({ name: JobName.FacialRecognitionQueueAll, data: { force } });
    case QueueName.SmartSearch:
      return this.jobRepository.queue({ name: JobName.SmartSearchQueueAll, data: { force } });
    case QueueName.DuplicateDetection:
      return this.jobRepository.queue({ name: JobName.AssetDetectDuplicatesQueueAll, data: { force } });
    case QueueName.Ocr:
      return this.jobRepository.queue({ name: JobName.OcrQueueAll, data: { force } });
    case QueueName.VideoConversion:
      return this.jobRepository.queue({ name: JobName.AssetEncodeVideoQueueAll, data: { force } });
    case QueueName.Sidecar:
      return this.jobRepository.queue({ name: JobName.SidecarQueueAll, data: { force } });
    case QueueName.Library:
      return this.jobRepository.queue({ name: JobName.LibraryScanQueueAll, data: { force } });
    case QueueName.BackupDatabase:
      return this.jobRepository.queue({ name: JobName.DatabaseBackup, data: { force } });
    case QueueName.StorageTemplateMigration:
      return this.jobRepository.queue({ name: JobName.StorageTemplateMigration });
    case QueueName.Migration:
      return this.jobRepository.queue({ name: JobName.FileMigrationQueueAll });
    default:
      throw new BadRequestException(`Invalid job name: ${name}`);
  }
}
```

---

#### 5.4.9 关键代码索引

| 文件 | 行号 | 说明 |
|------|------|------|
| `server/src/controllers/job.controller.ts` | 45-58 | 旧接口 `PUT /jobs/:name` - 唯一支持 start 命令 |
| `server/src/controllers/queue.controller.ts` | 44-57 | 新接口 `PUT /queues/:name` - 仅支持暂停/恢复 |
| `server/src/controllers/queue.controller.ts` | 74-84 | 新接口 `DELETE /queues/:name/jobs` - 清空队列 |
| `server/src/services/queue.service.ts` | 102-136 | `runCommandLegacy()` - 旧接口命令分发 |
| `server/src/services/queue.service.ts` | 185-250 | `start()` - 启动队列的核心实现 |
| `server/src/repositories/asset-job.repository.ts` | 63-466 | `streamFor*()` 系列方法 - `force` 参数 SQL 查询逻辑 |
| `server/src/enum.ts` | 874-884 | `QueueCommand` 枚举 - 5 种命令类型 |

### 5.5 特殊场景的重试机制

虽然框架层面无自动重试，但个别业务逻辑实现了自己的重试：

- `storage.repository.ts:174`：文件删除操作 `maxRetries: 5, retryDelay: 100`
- `machine-learning.repository.ts:170-191`：ML 服务调用时遍历多个 URL 重试

---

## 六、代码分散性问题分析

当前实现中，队列逻辑分散在多个位置，不利于统一管理：

| 逻辑类型 | 所在位置 |
|---------|---------|
| 队列配置与并发 | `config.ts`, `system-config.dto.ts` |
| 作业调度选项 | `job.repository.ts:getJobOptions()` |
| 作业链依赖 | `job.service.ts:onDone()` |
| 夜间作业编排 | `queue.service.ts:handleNightlyJobs()` |
| 失败处理（通知） | `notification.service.ts:onJobError()` |
| 失败处理（指标） | `telemetry.service.ts:onJobError()` |
| 元数据处理并发 | `metadata.service.ts`（独立于队列并发） |

### 建议的优化方向

1. **集中化作业配置**：将 `getJobOptions()` 中的优先级、去重配置迁移到 `@OnJob` 装饰器元数据中
2. **统一重试策略**：为关键作业（如 ML 调用、文件操作）添加可配置的重试策略
3. **失败通知扩展**：对更多关键作业（如缩略图生成、人脸识别）失败发送通知
4. **死信队列**：为持续失败的作业提供死信队列机制
5. **作业链 DSL**：将 `onDone()` 中的硬编码依赖改为声明式配置

---

## 七、关键代码索引

### 7.1 核心架构

| 文件 | 行号 | 说明 |
|------|------|------|
| `server/src/repositories/job.repository.ts` | 36-84 | 作业处理器发现与注册 |
| `server/src/repositories/job.repository.ts` | 86-96 | Worker 启动 |
| `server/src/repositories/job.repository.ts` | 108-116 | 并发设置 |
| `server/src/repositories/job.repository.ts` | 218-245 | 作业选项（优先级、去重） |
| `server/src/services/queue.service.ts` | 86-96 | 并发更新 |
| `server/src/services/queue.service.ts` | 261-293 | 夜间作业编排 |
| `server/src/services/job.service.ts` | 49-63 | 作业执行生命周期 |
| `server/src/services/job.service.ts` | 68-225 | 作业链编排 |
| `server/src/services/notification.service.ts` | 81-109 | 失败通知 |
| `server/src/repositories/config.repository.ts` | 281-292 | BullMQ 全局配置 |
| `server/src/config.ts` | 228-243 | 默认并发配置 |
| `server/src/decorators.ts` | 150-154 | @OnJob 装饰器 |

### 7.2 控制器与接口

| 文件 | 行号 | 说明 |
|------|------|------|
| `server/src/controllers/job.controller.ts` | 1-59 | 旧接口控制器（/jobs 前缀） |
| `server/src/controllers/job.controller.ts` | 21-43 | `GET /jobs` 和 `POST /jobs` 接口 |
| `server/src/controllers/job.controller.ts` | 45-58 | `PUT /jobs/:name` - **唯一支持 start 命令的接口** |
| `server/src/controllers/queue.controller.ts` | 1-85 | 新接口控制器（/queues 前缀） |
| `server/src/controllers/queue.controller.ts` | 22-31 | `GET /queues` 查询所有队列 |
| `server/src/controllers/queue.controller.ts` | 33-42 | `GET /queues/:name` 查询单个队列 |
| `server/src/controllers/queue.controller.ts` | 44-57 | `PUT /queues/:name` 暂停/恢复队列（不支持 start） |
| `server/src/controllers/queue.controller.ts` | 59-72 | `GET /queues/:name/jobs` 查询队列作业 |
| `server/src/controllers/queue.controller.ts` | 74-84 | `DELETE /queues/:name/jobs` 清空队列 |

### 7.3 业务逻辑层

#### 7.3.1 队列控制逻辑

| 文件 | 行号 | 说明 |
|------|------|------|
| `server/src/services/queue.service.ts` | 102-136 | `runCommandLegacy()` - 旧接口命令分发（start/pause/resume/empty/clear-failed） |
| `server/src/services/queue.service.ts` | 151-164 | `update()` - 新接口暂停/恢复逻辑 |
| `server/src/services/queue.service.ts` | 170-175 | `emptyQueue()` - 新接口清空队列逻辑 |
| `server/src/services/queue.service.ts` | 185-250 | `start()` - **启动队列的核心实现**，13 个可启动队列的 switch case |

#### 7.3.2 `force` 参数处理逻辑

| 文件 | 行号 | 说明 |
|------|------|------|
| `server/src/services/media.service.ts` | 70-117 | `handleQueueGenerateThumbnails()` - 缩略图批量处理，`force` 参数使用 |
| `server/src/services/media.service.ts` | 80 | `streamForThumbnailJob({ force, ... })` - 缩略图查询调用 |
| `server/src/services/smart-info.service.ts` | 68-93 | `handleQueueEncodeClip()` - 智能搜索批量处理 |
| `server/src/services/smart-info.service.ts` | 74-78 | `force=true` 时删除所有 CLIP embeddings |
| `server/src/services/person.service.ts` | 270-300 | `handleQueueDetectFaces()` - 人脸检测批量处理 |
| `server/src/services/person.service.ts` | 276-279 | `force=true` 时删除所有 ML 人脸并重建索引 |
| `server/src/services/person.service.ts` | 404-459 | `handleQueueRecognizeFaces()` - 人脸识别批量处理 |
| `server/src/services/person.service.ts` | 426-429 | `force=true` 时取消所有 ML 人脸分配 |
| `server/src/services/ocr.service.ts` | 14-38 | `handleQueueOcr()` - OCR 批量处理 |
| `server/src/services/ocr.service.ts` | 20-22 | `force=true` 时删除所有 OCR 记录 |
| `server/src/services/duplicate.service.ts` | 302-325 | `handleQueueSearchDuplicates()` - 重复检测批量处理 |
| `server/src/services/metadata.service.ts` | 218-233 | `handleQueueMetadataExtraction()` - 元数据批量处理 |
| `server/src/services/metadata.service.ts` | 419-437 | `handleQueueSidecar()` - Sidecar 文件批量处理 |

#### 7.3.3 Repository 层 `force` 参数 SQL 逻辑

| 文件 | 行号 | 说明 |
|------|------|------|
| `server/src/repositories/asset-job.repository.ts` | 63-104 | `streamForThumbnailJob()` - 缩略图查询，`$if(!force, ...)` 条件 |
| `server/src/repositories/asset-job.repository.ts` | 194-208 | `streamForSearchDuplicates()` - 重复检测查询 |
| `server/src/repositories/asset-job.repository.ts` | 210-218 | `streamForEncodeClip()` - 智能搜索查询 |
| `server/src/repositories/asset-job.repository.ts` | 313-336 | `streamForVideoConversion()` - 视频转码查询 |
| `server/src/repositories/asset-job.repository.ts` | 355-369 | `streamForMetadataExtraction()` - 元数据查询 |
| `server/src/repositories/asset-job.repository.ts` | 418-437 | `streamForSidecar()` - Sidecar 文件查询 |
| `server/src/repositories/asset-job.repository.ts` | 439-446 | `streamForDetectFacesJob()` - 人脸检测查询 |
| `server/src/repositories/asset-job.repository.ts` | 448-461 | `streamForOcrJob()` - OCR 查询 |

### 7.4 DTO 与枚举

| 文件 | 行号 | 说明 |
|------|------|------|
| `server/src/enum.ts` | 759-778 | `QueueName` 枚举 - 18 个队列名称 |
| `server/src/enum.ts` | 874-884 | `QueueCommand` 枚举 - 5 种命令类型（start/pause/resume/empty/clear-failed） |
| `server/src/dtos/queue.dto.ts` | 1-76 | 新接口 DTO 定义 |
| `server/src/dtos/queue-legacy.dto.ts` | 1-64 | 旧接口 DTO 与转换函数 |
| `server/src/dtos/job.dto.ts` | 1-11 | 手动作业 DTO（POST /jobs） |
