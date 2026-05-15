# 缩略图生成与视频转码队列机制分析

## 1. 概述

本文档分析 Immich 项目中缩略图生成和视频转码功能的队列执行机制，包括任务调度、队列管理、并发控制和错误处理等核心实现细节。

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

---

## 3. 缩略图生成队列 (ThumbnailGeneration)

### 3.1 队列基本信息
- **队列名**: `QueueName.ThumbnailGeneration`
- **处理服务**: `MediaService`
- **并发控制**: 可配置

### 3.2 任务类型

#### 3.2.1 批量任务: AssetGenerateThumbnailsQueueAll

**触发方式**:
1. 手动触发队列
2. 夜间任务自动触发 (nightly tasks)

**执行流程**:
1. 从数据库流式获取需要生成缩略图的资产列表
2. 按分页大小 (`JOBS_ASSET_PAGINATION_SIZE`) 批量创建子任务
3. 区分普通资产和编辑过的资产，分别创建不同的任务
4. 处理人物缩略图生成任务

```typescript
// 核心实现 (server/src/services/media.service.ts:69-117)
@OnJob({ name: JobName.AssetGenerateThumbnailsQueueAll, queue: QueueName.ThumbnailGeneration })
async handleQueueGenerateThumbnails({ force }): Promise<JobStatus> {
  // 流式处理资产
  for await (const asset of this.assetJobRepository.streamForThumbnailJob({ force, fullsizeEnabled })) {
    if (force || !asset.isEdited) {
      jobs.push({ name: JobName.AssetGenerateThumbnails, data: { id: asset.id } });
    }
    if (asset.isEdited) {
      jobs.push({ name: JobName.AssetEditThumbnailGeneration, data: { id: asset.id } });
    }
  }

  // 处理人物缩略图
  for await (const person of people) {
    jobs.push({ name: JobName.PersonGenerateThumbnail, data: { id: person.id } });
  }
}
```

#### 3.2.2 单个资产任务: AssetGenerateThumbnails

**执行流程**:
1. 获取资产详细信息
2. 根据资产类型（视频/图片）选择不同的缩略图生成策略
3. 生成预览图和缩略图
4. 计算 thumbhash
5. 同步文件记录

```typescript
// 核心实现 (server/src/services/media.service.ts:209-249)
@OnJob({ name: JobName.AssetGenerateThumbnails, queue: QueueName.ThumbnailGeneration })
async handleGenerateThumbnails({ id }): Promise<JobStatus> {
  if (asset.type === AssetType.Video || asset.originalFileName.toLowerCase().endsWith('.gif')) {
    generated = await this.generateVideoThumbnails(asset, config);
  } else if (asset.type === AssetType.Image) {
    generated = await this.generateImageThumbnails(asset, config);
  }
}
```

#### 3.2.3 编辑资产任务: AssetEditThumbnailGeneration

专门处理已编辑资产的缩略图生成，确保编辑效果正确反映在缩略图中。

#### 3.2.4 人物缩略图任务: PersonGenerateThumbnail

基于人脸检测结果，为每个人物生成独立的缩略图。

---

## 4. 视频转码队列 (VideoConversion)

### 4.1 队列基本信息
- **队列名**: `QueueName.VideoConversion`
- **处理服务**: `MediaService`
- **并发控制**: 可配置

### 4.2 任务类型

#### 4.2.1 批量任务: AssetEncodeVideoQueueAll

**执行流程**:
1. 从数据库流式获取需要转码的视频资产
2. 按分页大小批量创建子任务

```typescript
// 核心实现 (server/src/services/media.service.ts:550-567)
@OnJob({ name: JobName.AssetEncodeVideoQueueAll, queue: QueueName.VideoConversion })
async handleQueueVideoConversion({ force }): Promise<JobStatus> {
  for await (const asset of this.assetJobRepository.streamForVideoConversion(force)) {
    queue.push({ name: JobName.AssetEncodeVideo, data: { id: asset.id } });
  }
}
```

#### 4.2.2 单个视频转码任务: AssetEncodeVideo

**转码策略判断**:
系统根据以下因素判断是否需要转码：
1. **转码策略配置**:
   - `Disabled`: 不转码
   - `All`: 全部转码
   - `Required`: 仅在必要时转码（非目标编码格式或像素格式不兼容）
   - `Optimal`: 必要时或分辨率大于目标时转码
   - `Bitrate`: 必要时或比特率过高时转码

2. **硬件加速**:
   - 支持 Nvenc、Qsv、Vaapi、Rkmpp 加速
   - 失败时自动降级：硬件加速 → 仅编码加速 → 纯软件

```typescript
// 核心实现 (server/src/services/media.service.ts:569-654)
@OnJob({ name: JobName.AssetEncodeVideo, queue: QueueName.VideoConversion })
async handleVideoConversion({ id }): Promise<JobStatus> {
  // 判断转码目标
  const target = this.getTranscodeTarget(ffmpeg, videoStream, audioStream);
  
  // 尝试硬件加速转码，失败时自动降级
  try {
    await this.mediaRepository.transcode(input, output, command);
  } catch (error) {
    // 降级重试：关闭解码加速
    if (ffmpeg.accelDecode) {
      await this.mediaRepository.transcode(input, output, command);
    }
    // 进一步降级：完全关闭硬件加速
    await this.mediaRepository.transcode(input, output, command);
  }
}
```

---

## 5. 队列调度与执行机制

### 5.1 任务注册与发现

系统通过装饰器 `@OnJob` 自动发现并注册任务处理器：

```typescript
// 任务注册 (server/src/repositories/job.repository.ts:36-84)
setup(services) {
  for (const Service of services) {
    const instance = this.moduleRef.get<any>(Service);
    for (const methodName of getMethodNames(instance)) {
      const config = reflector.get<JobConfig>(MetadataKey.JobConfig, handler);
      if (config) {
        this.handlers[jobName] = { label, jobName, queueName, handler: handler.bind(instance) };
      }
    }
  }
}
```

### 5.2 任务执行流程

```
事件触发流程:
┌─────────────────────────────────────────────────────────────────┐
│  1. JobRepository.queue() / queueAll()                           │
│     → 任务加入 BullMQ 队列                                       │
└─────────────────────────────────────────────────────────────────┘
                              ↓
┌─────────────────────────────────────────────────────────────────┐
│  2. Worker 监听并获取任务，触发 JobRun 事件                      │
└─────────────────────────────────────────────────────────────────┘
                              ↓
┌─────────────────────────────────────────────────────────────────┐
│  3. JobService.onJobRun() 处理执行                               │
│     → 发射 JobStart 事件                                         │
│     → 调用 JobRepository.run() 执行具体任务                      │
│     → 成功时发射 JobSuccess 事件，失败时发射 JobError 事件       │
│     → 最终发射 JobComplete 事件                                  │
└─────────────────────────────────────────────────────────────────┘
                              ↓
┌─────────────────────────────────────────────────────────────────┐
│  4. JobService.onDone() 处理后续任务                             │
│     → 缩略图生成完成后，排队人脸识别、OCR、智能搜索任务           │
│     → 视频资产额外排队视频转码任务                                │
└─────────────────────────────────────────────────────────────────┘
```

### 5.3 并发控制

并发数配置在系统配置中定义，可动态更新：

```typescript
// 动态更新并发 (server/src/services/queue.service.ts:86-96)
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

**支持并发的队列**:
- ThumbnailGeneration
- MetadataExtraction
- VideoConversion
- FaceDetection
- SmartSearch
- DuplicateDetection
- BackupDatabase

**不支持并发的队列**（串行执行）:
- FacialRecognition
- StorageTemplateMigration

### 5.4 任务优先级与去重

部分任务支持优先级和去重：

```typescript
// 任务选项配置 (server/src/repositories/job.repository.ts:218-242)
private getJobOptions(item: JobItem): JobsOptions | null {
  switch (item.name) {
    case JobName.PersonGenerateThumbnail: {
      return { priority: 1 };  // 高优先级
    }
    case JobName.StorageTemplateMigrationSingle: {
      return { jobId: item.data.id };  // 按资产ID去重
    }
  }
}
```

---

## 6. 任务状态管理

### 6.1 任务状态枚举
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

## 7. 错误处理与重试机制

### 7.1 视频转码降级重试
视频转码失败时采用三级降级策略：
1. **第一级**: 完整硬件加速（编码 + 解码）
2. **第二级**: 仅编码硬件加速，软件解码
3. **第三级**: 完全软件编解码

### 7.2 任务失败处理
- 失败任务保留在队列中，可通过 API 清理
- 通过 `JobError` 事件记录错误日志
- 支持手动重新触发队列

---

## 8. 数据流与依赖关系

### 8.1 缩略图生成后的任务链
```
AssetGenerateThumbnails 完成
        ↓
    ┌───┴───┬─────────────┬──────────┐
    ↓       ↓             ↓          ↓
SmartSearch FaceDetection  OCR    (视频资产)
                                    ↓
                          AssetEncodeVideo
```

### 8.2 夜间任务触发
夜间任务（Nightly Jobs）自动触发缩略图生成等维护任务：

```typescript
// 夜间任务配置 (server/src/services/queue.service.ts:261-293)
async handleNightlyJobs() {
  if (config.nightlyTasks.missingThumbnails) {
    jobs.push({ name: JobName.AssetGenerateThumbnailsQueueAll, data: { force: false } });
  }
}
```

---

## 9. 关键配置项

### 9.1 缩略图相关配置
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

### 9.2 视频转码相关配置
```typescript
ffmpeg: {
  transcode: TranscodePolicy,       // 转码策略
  accel: TranscodeHardwareAcceleration,  // 硬件加速类型
  accelDecode: boolean,              // 硬件解码加速
  targetResolution: string,          // 目标分辨率
  maxBitrate: string,                // 最大比特率
  crf: number,                       // 恒定质量因子
  preset: string,                    // 编码预设
  threads: number,                   // 线程数
  acceptedVideoCodecs: VideoCodec[], // 可接受视频编码
  acceptedAudioCodecs: AudioCodec[], // 可接受音频编码
  acceptedContainers: VideoContainer[], // 可接受容器格式
  targetVideoCodec: VideoCodec,      // 目标视频编码
  targetAudioCodec: AudioCodec,      // 目标音频编码
  targetResolution: string,          // 目标分辨率
}
```

---

## 10. 总结

### 10.1 架构特点
1. **基于 BullMQ 的可靠队列系统**，支持任务持久化、重试、优先级
2. **事件驱动的任务调度**，通过事件总线解耦
3. **流式批量处理**，避免内存压力
4. **灵活的并发控制**，按队列配置并发数
5. **多级容错机制**，特别是视频转码的硬件加速降级

### 10.2 队列性能优化点
1. 按资产类型分流任务（图片/视频/人脸）
2. 批量排队减少 Redis 操作
3. 流式处理避免一次性加载全部数据
4. 任务去重避免重复处理
5. 优先级调度确保重要任务先执行

### 10.3 关键文件位置
- 队列服务: `server/src/services/queue.service.ts`
- 任务服务: `server/src/services/job.service.ts`
- 任务仓库: `server/src/repositories/job.repository.ts`
- 媒体处理: `server/src/services/media.service.ts`
- 枚举定义: `server/src/enum.ts`
