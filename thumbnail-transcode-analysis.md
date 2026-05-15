# 缩略图生成与视频转码队列机制分析

## 1. 概述

本文档分析 Immich 项目中缩略图生成和视频转码功能的队列执行机制，包括任务调度、队列管理、并发控制、批量拆分、任务联动和错误处理等核心实现细节。

---

## 2. 队列系统基础架构

### 2.1 技术栈
- **队列引擎**: BullMQ (基于 Redis)
- **框架**: NestJS
- **核心服务**: `QueueService`, `JobService`, `JobRepository`, `MediaService`

### 2.2 队列配置
队列系统采用集中式管理，每个队列对应独立的 Worker 实例，支持动态调整并发数。

```typescript
// 启动 Worker (server/src/repositories/job.repository.ts:86-96)
startWorkers() {
  const { bull } = this.configRepository.getEnv();
  for (const queueName of Object.values(QueueName)) {
    this.workers[queueName] = new Worker(
      queueName,
      (job) => this.eventRepository.emit('JobRun', queueName, job as JobItem),
      { ...bull.config, concurrency: 1 },  // 默认并发数为 1
    );
  }
}
```

### 2.3 并发队列分类（已修正）

**必须串行执行的队列**（不支持并发）：
```typescript
// server/src/services/queue.service.ts:252-259
private isConcurrentQueue(name: QueueName): name is ConcurrentQueueName {
  return ![
    QueueName.FacialRecognition,        // 人脸识别
    QueueName.StorageTemplateMigration, // 存储模板迁移
    QueueName.DuplicateDetection,       // 重复检测
    QueueName.BackupDatabase,           // 数据库备份
  ].includes(name);
}
```

**支持并发的队列**（可配置并发数）：
- `ThumbnailGeneration` - 缩略图生成
- `VideoConversion` - 视频转码
- `MetadataExtraction` - 元数据提取
- `FaceDetection` - 人脸检测
- `SmartSearch` - 智能搜索
- `BackgroundTask` - 后台任务
- `Migration` - 文件迁移
- `Search` - 搜索
- `Sidecar` - Sidecar 文件处理
- `Library` - 库扫描
- `Notification` - 通知
- `Ocr` - 文字识别
- `Workflow` - 工作流
- `Editor` - 编辑器

---

## 3. 缩略图生成队列完整链路 (ThumbnailGeneration)

### 3.1 队列基本信息
- **队列名**: `QueueName.ThumbnailGeneration`
- **处理服务**: `MediaService`
- **并发控制**: 可配置，支持并发执行
- **批量分页大小**: `JOBS_ASSET_PAGINATION_SIZE = 1000`

### 3.2 任务完整链路图

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                           触发方式                                           │
├─────────────────────────────────────────────────────────────────────────────┤
│  1. 手动触发: API 调用 start(QueueName.ThumbnailGeneration)                 │
│  2. 夜间任务: NightlyJobs 自动触发 (missingThumbnails 配置开启)             │
│  3. 上传触发: StorageTemplateMigrationSingle 完成后触发 (source=upload/copy) │
│  4. 编辑触发: 资产编辑后直接调用 AssetEditThumbnailGeneration                │
└─────────────────────────────────────────────────────────────────────────────┘
                                      ↓
┌─────────────────────────────────────────────────────────────────────────────┐
│                     批量任务入队: AssetGenerateThumbnailsQueueAll            │
│  - 参数: force (boolean)                                                     │
│  - 流式读取数据库，分批处理                                                  │
└─────────────────────────────────────────────────────────────────────────────┘
                                      ↓
                        ┌───────────────────────────────┐
                        │  流式遍历需要处理的资产        │
                        │  for await (const asset of    │
                        │    streamForThumbnailJob())   │
                        └───────────────┬───────────────┘
                                        ↓
                          ┌───────────────────────────┐
                          │  判断资产是否已编辑        │
                          └───────────┬───────────────┘
                                      ↓
                        ┌─────────────┴─────────────┐
                        ↓                           ↓
              ┌───────────────────┐       ┌───────────────────────────┐
              │  未编辑资产        │       │  已编辑资产                │
              │  AssetGenerate-    │       │  AssetEditThumbnail-      │
              │  Thumbnails        │       │  Generation                │
              └───────────┬───────┘       └───────────────┬───────────┘
                          ↓                                 ↓
                        ┌───────────────────────────────────────────────┐
                        │  累积任务，每批 = 1000 个后调用 queueAll()      │
                        │  jobs.length >= JOBS_ASSET_PAGINATION_SIZE     │
                        └───────────────────────┬───────────────────────┘
                                                        ↓
                                      ┌─────────────────────────────┐
                                      │  批量入队: queueAll()        │
                                      │  - 按队列名称分组            │
                                      │  - 有 jobId 的任务单独入队   │
                                      │    (去重)                   │
                                      │  - 无 jobId 的任务 addBulk   │
                                      └──────────────┬──────────────┘
                                                     ↓
                                      ┌─────────────────────────────┐
                                      │  Worker 从队列获取任务        │
                                      │  触发 JobRun 事件            │
                                      └──────────────┬──────────────┘
                                                     ↓
                                      ┌─────────────────────────────┐
                                      │  子任务执行                   │
                                      │  - 生成预览图、缩略图         │
                                      │  - 计算 thumbhash            │
                                      │  - 同步文件记录              │
                                      └──────────────┬──────────────┘
                                                     ↓
                                    ┌──────────────────────────────────┐
                                    │  执行成功?                       │
                                    └───────────┬──────────────────────┘
                                                ↓
                        ┌───────────────────────┴───────────────────────┐
                        ↓                                               ↓
              ┌─────────────────────┐                       ┌─────────────────────┐
              │  Success/Skipped    │                       │  Failed             │
              └───────────┬─────────┘                       └─────────────────────┘
                          ↓
              ┌─────────────────────────────┐
              │  触发后续联动任务            │
              │  (onDone 回调)               │
              └───────────────┬─────────────┘
                              ↓
                  ┌───────────┴───────────┐
                  ↓                       ↓
          ┌───────────────────┐   ┌───────────────────┐
          │ notify=true OR    │→  │  不触发后续任务   │
          │ source='upload'   │   └───────────────────┘
          └─────────┬─────────┘
                    ↓
          ┌─────────┴─────────────────────────────────────┐
          │  入队后续任务: queueAll([                      │
          │    { name: SmartSearch, data },               │
          │    { name: AssetDetectFaces, data },          │
          │    { name: Ocr, data },                       │
          │    (视频资产) { name: AssetEncodeVideo, data }│
          │  ])                                           │
          └───────────────────────────────────────────────┘
```

### 3.3 任务类型详解

#### 3.3.1 批量调度任务: AssetGenerateThumbnailsQueueAll

**核心实现**：
```typescript
// server/src/services/media.service.ts:69-116
@OnJob({ name: JobName.AssetGenerateThumbnailsQueueAll, queue: QueueName.ThumbnailGeneration })
async handleQueueGenerateThumbnails({ force }) {
  const queueAll = async () => {
    await this.jobRepository.queueAll(jobs);
    jobs = [];
  };

  // 第一步：处理资产缩略图
  for await (const asset of this.assetJobRepository.streamForThumbnailJob({ force, fullsizeEnabled })) {
    if (force || !asset.isEdited) {
      jobs.push({ name: JobName.AssetGenerateThumbnails, data: { id: asset.id } });
    }
    if (asset.isEdited) {
      jobs.push({ name: JobName.AssetEditThumbnailGeneration, data: { id: asset.id } });
    }
    
    // 每累积 1000 个任务批量入队
    if (jobs.length >= JOBS_ASSET_PAGINATION_SIZE) {
      await queueAll();
    }
  }
  await queueAll();  // 处理剩余任务

  // 第二步：处理人物缩略图
  for await (const person of this.personRepository.getAll(force ? undefined : { thumbnailPath: '' })) {
    jobs.push({ name: JobName.PersonGenerateThumbnail, data: { id: person.id } });
    if (jobs.length >= JOBS_ASSET_PAGINATION_SIZE) {
      await queueAll();
    }
  }
  await queueAll();
}
```

**关键特性**：
1. **流式处理**：使用 `for await...of` 流式读取数据库，避免内存溢出
2. **批量入队**：每 1000 个任务调用一次 `queueAll()`，减少 Redis 交互
3. **任务分流**：区分普通资产和已编辑资产，分别创建不同类型的子任务
4. **两阶段处理**：先处理资产缩略图，再处理人物缩略图

#### 3.3.2 单个资产缩略图生成: AssetGenerateThumbnails

**执行流程**：
```typescript
// server/src/services/media.service.ts:209-249
@OnJob({ name: JobName.AssetGenerateThumbnails, queue: QueueName.ThumbnailGeneration })
async handleGenerateThumbnails({ id }) {
  // 1. 获取资产信息
  const asset = await this.assetJobRepository.getForGenerateThumbnailJob(id);
  
  // 2. 隐藏资产跳过
  if (asset.visibility === AssetVisibility.Hidden) {
    return JobStatus.Skipped;
  }

  // 3. 根据资产类型选择生成方式
  if (asset.type === AssetType.Video || gif) {
    generated = await this.generateVideoThumbnails(asset, config);
  } else if (asset.type === AssetType.Image) {
    generated = await this.generateImageThumbnails(asset, config);
  }

  // 4. 同步文件记录
  await this.syncFiles(asset.files, generated.files);
  
  // 5. 更新 thumbhash
  await this.assetRepository.update({ id: asset.id, thumbhash: generated.thumbhash });
}
```

#### 3.3.3 编辑资产缩略图生成: AssetEditThumbnailGeneration

**特性**：
- 专门处理已编辑资产的缩略图生成
- 裁剪检测后会更新人脸和 OCR 的可见状态
- 通过 WebSocket 推送 `AssetEditReadyV2` 事件通知前端

#### 3.3.4 人物缩略图生成: PersonGenerateThumbnail

**特性**：
- 高优先级任务 (`priority: 1`)
- 基于人脸检测结果，裁剪生成人物缩略图
- 完成后通过 WebSocket 推送通知

---

## 4. 视频转码队列完整链路 (VideoConversion)

### 4.1 队列基本信息
- **队列名**: `QueueName.VideoConversion`
- **处理服务**: `MediaService`
- **并发控制**: 可配置，支持并发执行
- **批量分页大小**: `JOBS_ASSET_PAGINATION_SIZE = 1000`

### 4.2 任务完整链路图

```
┌─────────────────────────────────────────────────────────────────────────┐
│                           触发方式                                       │
├─────────────────────────────────────────────────────────────────────────┤
│  1. 手动触发: API 调用 start(QueueName.VideoConversion)                 │
│  2. 联动触发: AssetGenerateThumbnails 完成后，视频资产自动触发           │
└─────────────────────────────────────────────────────────────────────────┘
                                      ↓
┌─────────────────────────────────────────────────────────────────────────┐
│                   批量任务入队: AssetEncodeVideoQueueAll                 │
│  - 参数: force (boolean)                                                 │
│  - 流式读取数据库，分批处理                                              │
└─────────────────────────────────────────────────────────────────────────┘
                                      ↓
                        ┌───────────────────────────────┐
                        │  流式遍历需要转码的视频资产     │
                        │  for await (const asset of    │
                        │    streamForVideoConversion())│
                        └───────────────┬───────────────┘
                                        ↓
                          ┌───────────────────────────┐
                          │  创建子任务数据对象        │
                          │  { id: asset.id }         │
                          └───────────┬───────────────┘
                                      ↓
                        ┌───────────────────────────────────────┐
                        │  累积任务，每批 = 1000 个后批量入队    │
                        │  jobs.length >= JOBS_ASSET_PAGINATION │
                        └───────────────────────┬───────────────┘
                                                        ↓
                                      ┌─────────────────────────────┐
                                      │  批量入队: queueAll()        │
                                      │  - 按队列名称分组            │
                                      │  - addBulk 批量添加          │
                                      └──────────────┬──────────────┘
                                                     ↓
                                      ┌─────────────────────────────┐
                                      │  Worker 从队列获取任务        │
                                      │  触发 JobRun 事件            │
                                      └──────────────┬──────────────┘
                                                     ↓
                                      ┌─────────────────────────────┐
                                      │  单个视频转码任务执行         │
                                      │  AssetEncodeVideo           │
                                      └──────────────┬──────────────┘
                                                     ↓
                                    ┌──────────────────────────────────┐
                                    │  元数据检查                      │
                                    │  - videoStream?                 │
                                    │  - format?                      │
                                    │  - width/height?                │
                                    └───────────┬──────────────────────┘
                                                ↓
                                    ┌──────────────────────────────────┐
                                    │  判断转码必要性                   │
                                    │  getTranscodeTarget()            │
                                    │  isRemuxRequired()               │
                                    └───────────┬──────────────────────┘
                                                ↓
                        ┌───────────────────────┴───────────────────────┐
                        ↓                                               ↓
              ┌─────────────────────┐                       ┌─────────────────────┐
              │  需要转码           │                       │  无需转码           │
              │  target != None OR  │                       │  Skipped            │
              │  remux required     │                       └─────────────────────┘
              └───────────┬─────────┘
                          ↓
              ┌─────────────────────────────┐
              │  三级降级重试机制            │
              │  ┌─────────────────────────┐│
              │  │ 1. 完整硬件加速         ││
              │  │    (编码+解码加速)      ││
              │  └────────────┬────────────┘│
              │               ↓ 失败        │
              │  ┌─────────────────────────┐│
              │  │ 2. 仅编码硬件加速       ││
              │  │    (软件解码)           ││
              │  └────────────┬────────────┘│
              │               ↓ 失败        │
              │  ┌─────────────────────────┐│
              │  │ 3. 纯软件编解码         ││
              │  └─────────────────────────┘│
              └───────────────┬─────────────┘
                              ↓
                  ┌───────────┴───────────┐
                  ↓                       ↓
          ┌───────────────────┐   ┌───────────────────┐
          │  转码成功         │   │  全部降级仍失败   │
          │  - 更新文件记录   │   │  - 返回 Failed    │
          │  - Success        │   └───────────────────┘
          └───────────────────┘
```

### 4.3 任务类型详解

#### 4.3.1 批量调度任务: AssetEncodeVideoQueueAll

**核心实现**：
```typescript
// server/src/services/media.service.ts:550-566
@OnJob({ name: JobName.AssetEncodeVideoQueueAll, queue: QueueName.VideoConversion })
async handleQueueVideoConversion({ force }) {
  let queue = [];
  for await (const asset of this.assetJobRepository.streamForVideoConversion(force)) {
    queue.push({ name: JobName.AssetEncodeVideo, data: { id: asset.id } });
    
    // 每累积 1000 个任务批量入队
    if (queue.length >= JOBS_ASSET_PAGINATION_SIZE) {
      await this.jobRepository.queueAll(queue);
      queue = [];
    }
  }
  await this.jobRepository.queueAll(queue);  // 处理剩余任务
}
```

#### 4.3.2 单个视频转码任务: AssetEncodeVideo

**转码策略判断**：
```typescript
// server/src/services/media.service.ts:569-654
private getTranscodeTarget(config, videoStream, audioStream) {
  // 根据转码策略和编码格式判断
  // TranscodePolicy: All / Optimal / Bitrate / Required / Disabled
}

private isVideoTranscodeRequired(ffmpegConfig, stream) {
  // 判断因素：
  // 1. 目标分辨率 (targetResolution)
  // 2. 最大比特率 (maxBitrate)
  // 3. 目标视频编码格式 (acceptedVideoCodecs)
  // 4. 像素格式 (必须是 420p)
}

private isRemuxRequired(ffmpegConfig, format) {
  // 判断是否需要重新封装
  // 检查容器格式是否在 acceptedContainers 中
}
```

**三级降级重试机制**：
```typescript
try {
  // 第一级：完整硬件加速
  await this.mediaRepository.transcode(input, output, command);
} catch (error) {
  if (ffmpeg.accel === TranscodeHardwareAcceleration.Disabled) {
    return JobStatus.Failed;
  }
  
  // 第二级：仅编码硬件加速，软件解码
  if (ffmpeg.accelDecode) {
    ffmpeg = { ...ffmpeg, accelDecode: false };
    await this.mediaRepository.transcode(input, output, command);
  }
  
  // 第三级：完全关闭硬件加速
  ffmpeg = { ...ffmpeg, accel: TranscodeHardwareAcceleration.Disabled };
  await this.mediaRepository.transcode(input, output, command);
}
```

---

## 5. 队列调度与执行机制

### 5.1 批量入队优化实现

```typescript
// server/src/repositories/job.repository.ts:159-189
async queueAll(items: JobItem[]): Promise<void> {
  const promises = [];
  const itemsByQueue = {};  // 按队列名称分组
  
  for (const item of items) {
    const queueName = this.getQueueName(item.name);
    const job = { name: item.name, data: item.data || {}, options: ... };
    
    if (job.options?.jobId) {
      // 有 jobId 的任务单独入队（去重）
      promises.push(this.getQueue(queueName).add(item.name, item.data, job.options));
    } else {
      // 无 jobId 的任务按队列分组
      itemsByQueue[queueName] = itemsByQueue[queueName] || [];
      itemsByQueue[queueName].push(job);
    }
  }
  
  // 每队列批量入队 (addBulk)
  for (const [queueName, jobs] of Object.entries(itemsByQueue)) {
    const queue = this.getQueue(queueName as QueueName);
    promises.push(queue.addBulk(jobs));
  }
  
  await Promise.all(promises);
}
```

### 5.2 任务执行与事件流

```typescript
// server/src/services/job.service.ts:49-63
@OnEvent({ name: 'JobRun' })
async onJobRun(queueName, job) {
  try {
    await this.eventRepository.emit('JobStart', queueName, job);
    const response = await this.jobRepository.run(job);
    await this.eventRepository.emit('JobSuccess', { job, response });
    
    // 只有成功或跳过才触发后续任务
    if ([JobStatus.Success, JobStatus.Skipped].includes(response)) {
      await this.onDone(job);
    }
  } catch (error) {
    await this.eventRepository.emit('JobError', { job, error });
  } finally {
    await this.eventRepository.emit('JobComplete', queueName, job);
  }
}
```

### 5.3 事件驱动的任务联动

```typescript
// server/src/services/job.service.ts:68-216
private async onDone(item: JobItem) {
  switch (item.name) {
    // 缩略图生成完成后触发后续任务
    case JobName.AssetGenerateThumbnails: {
      // 仅在 notify=true 或 source='upload' 时触发
      if (!item.data.notify && item.data.source !== 'upload') {
        break;
      }
      
      // 触发 3 个必选后续任务
      const jobs: JobItem[] = [
        { name: JobName.SmartSearch, data: item.data },
        { name: JobName.AssetDetectFaces, data: item.data },
        { name: JobName.Ocr, data: item.data },
      ];
      
      // 视频资产额外触发转码
      if (asset.type === AssetType.Video) {
        jobs.push({ name: JobName.AssetEncodeVideo, data: item.data });
      }
      
      await this.jobRepository.queueAll(jobs);
      break;
    }
    
    // 存储模板迁移完成后，触发缩略图生成
    case JobName.StorageTemplateMigrationSingle: {
      if (item.data.source === 'upload' || item.data.source === 'copy') {
        await this.jobRepository.queue({ 
          name: JobName.AssetGenerateThumbnails, 
          data: item.data 
        });
      }
      break;
    }
    
    // Sidecar 检查完成后，触发元数据提取
    case JobName.SidecarCheck: {
      await this.jobRepository.queue({ 
        name: JobName.AssetExtractMetadata, 
        data: item.data 
      });
      break;
    }
    
    // Sidecar 写入完成后，触发元数据重新提取
    case JobName.SidecarWrite: {
      await this.jobRepository.queue({
        name: JobName.AssetExtractMetadata,
        data: { id: item.data.id, source: 'sidecar-write' },
      });
      break;
    }
  }
}
```

---

## 6. 任务状态管理

### 6.1 队列任务状态
- **Active**: 执行中
- **Completed**: 已完成
- **Failed**: 失败
- **Delayed**: 延迟执行
- **Waiting**: 等待中
- **Paused**: 已暂停

### 6.2 任务执行结果状态
```typescript
enum JobStatus {
  Success = 'success',   // 成功
  Failed = 'failed',     // 失败
  Skipped = 'skipped',   // 跳过（无需处理）
}
```

---

## 7. 关键配置项

### 7.1 缩略图相关配置
```typescript
image: {
  thumbnail: {
    size: number,           // 缩略图尺寸
    format: ImageFormat,    // 输出格式 (Jpeg/Webp)
    quality: number,        // 质量 0-100
    progressive: boolean,   // 是否渐进式加载
  },
  preview: {
    size: number,           // 预览图尺寸
    format: ImageFormat,
    quality: number,
    progressive: boolean,
  },
  fullsize: {
    enabled: boolean,       // 是否启用全尺寸图
    format: ImageFormat,
    quality: number,
    progressive: boolean,
  },
  colorspace: Colorspace,   // 色彩空间 (Srgb/P3)
  extractEmbedded: boolean, // 是否提取 RAW 内嵌预览
}
```

### 7.2 视频转码相关配置
```typescript
ffmpeg: {
  transcode: TranscodePolicy,       // 转码策略: All/Optimal/Bitrate/Required/Disabled
  accel: TranscodeHardwareAcceleration,  // 硬件加速类型: nvenc/qsv/vaapi/rkmpp/disabled
  accelDecode: boolean,              // 硬件解码加速
  targetResolution: string,          // 目标分辨率 (original/720p/1080p 等)
  maxBitrate: string,                // 最大比特率
  crf: number,                       // 恒定质量因子
  preset: string,                    // 编码预设
  threads: number,                   // 线程数
  acceptedVideoCodecs: VideoCodec[], // 可接受视频编码
  acceptedAudioCodecs: AudioCodec[], // 可接受音频编码
  acceptedContainers: VideoContainer[], // 可接受容器格式
  targetVideoCodec: VideoCodec,      // 目标视频编码
  targetAudioCodec: AudioCodec,      // 目标音频编码
}
```

### 7.3 夜间任务配置
```typescript
nightlyTasks: {
  start: string,                     // 开始时间 HH:MM
  databaseCleanup: boolean,          // 数据库清理
  generateMemories: boolean,         // 生成回忆
  syncQuotaUsage: boolean,           // 同步配额使用
  missingThumbnails: boolean,        // 生成缺失缩略图
  clusterNewFaces: boolean,          // 聚类新面孔
}
```

---

## 8. 总结

### 8.1 架构特点
1. **基于 BullMQ 的可靠队列系统**，支持任务持久化、重试、优先级
2. **事件驱动的任务调度**，通过事件总线解耦，支持任务链式触发
3. **流式批量处理**，每 1000 个任务批量入队，避免内存压力
4. **灵活的并发控制**，4 个队列必须串行，其余可配置并发数
5. **多级容错机制**，视频转码支持三级硬件加速降级重试
6. **智能任务去重**，通过 jobId 机制避免重复任务

### 8.2 性能优化点
1. **按队列分组批量入队**：减少 Redis 网络交互
2. **流式数据库读取**：`for await...of` 避免一次性加载全部数据
3. **任务分流处理**：按资产类型、编辑状态创建不同任务
4. **条件触发后续任务**：仅上传或 notify=true 时触发后续处理链
5. **优先级调度**：人物缩略图等高优先级任务优先执行

### 8.3 关键文件位置
- 队列服务: `server/src/services/queue.service.ts`
- 任务服务: `server/src/services/job.service.ts`
- 任务仓库: `server/src/repositories/job.repository.ts`
- 媒体处理: `server/src/services/media.service.ts`
- 枚举定义: `server/src/enum.ts`
- 常量定义: `server/src/constants.ts`
