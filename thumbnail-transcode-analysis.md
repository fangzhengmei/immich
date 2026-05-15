# 缩略图生成与视频转码队列机制分析

## 1. 概述

本文档分析 Immich 项目中缩略图生成和视频转码功能的队列执行机制，包括任务调度、队列管理、并发控制、批量拆分、任务联动和错误处理等核心实现细节，所有内容均与实际代码严格对应。

---

## 2. 队列系统基础架构

### 2.1 技术栈
- **队列引擎**: BullMQ (基于 Redis)
- **框架**: NestJS
- **核心服务**: `QueueService`, `JobService`, `JobRepository`, `MediaService`

### 2.2 并发队列分类（已修正）

**必须串行执行的队列（共4个，不支持并发）**：
```typescript
// server/src/services/queue.service.ts:252-259
private isConcurrentQueue(name: QueueName): name is ConcurrentQueueName {
  return ![
    QueueName.FacialRecognition,        // 人脸识别聚类（串行）
    QueueName.StorageTemplateMigration, // 存储模板迁移（串行）
    QueueName.DuplicateDetection,       // 重复检测（串行）
    QueueName.BackupDatabase,           // 数据库备份（串行）
  ].includes(name);
}
```

**支持并发的队列（可配置并发数）**：
- `ThumbnailGeneration` - 缩略图生成队列
- `VideoConversion` - 视频转码队列
- `MetadataExtraction` - 元数据提取队列
- `FaceDetection` - 人脸检测队列
- `SmartSearch` - 智能搜索队列
- `BackgroundTask` - 后台任务队列
- `Migration` - 文件迁移队列
- `Search` - 搜索队列
- `Sidecar` - Sidecar 文件处理队列
- `Library` - 库扫描队列
- `Notification` - 通知队列
- `Ocr` - 文字识别队列
- `Editor` - 编辑器队列（AssetEditThumbnailGeneration 所在队列）
- `Workflow` - 工作流队列

---

## 3. 统一调度链路总览

### 3.1 完整调度链路图

```
┌─────────────────────────────────────────────────────────────────────────────────────┐
│                          任务触发入口总览                                             │
├─────────────────────────────────────────────────────────────────────────────────────┤
│                                                                                     │
│  ┌─────────────────────────────────┐  ┌─────────────────────────────────┐         │
│  │  1. 手动 API 手动队列          │  │  2. 夜间任务自动触发         │         │
│  │     - QueueName.Thumbnail-     │  │     - NightlyJobs          │         │
│  │       Generation 队列         │  │       - missingThumbnails    │         │
│  │     - QueueName.Video-         │  │         → AssetGenerate-      │         │
│  │       Conversion 队列        │  │           ThumbnailsQueueAll │         │
│  └─────────────────────────────────┘  └─────────────────────────────────┘         │
│                                                                                     │
│  ┌─────────────────────────────────┐  ┌─────────────────────────────────┐         │
│  │  3. 上传流程自动触发         │  │  4. 资产操作触发          │         │
│  │     - AssetMetadata-     │  │     - AssetShow 事件        │         │
│  │       Extracted 事件      │  │       (显示隐藏后重新生成     │         │
│  │       → StorageTemplate-    │  │       → AssetGenerate-     │         │
│  │         MigrationSingle   │  │         Thumbnails         │         │
│  │         (source=upload)    │  │         (notify=true)       │         │
│  └─────────────────────────────────┘  └─────────────────────────────────┘         │
│                                                                                     │
│  ┌─────────────────────────────────┐  ┌─────────────────────────────────┐         │
│  │  5. 资产编辑触发           │  │  6. 单个资产任务          │         │
│  │     - updateAssetEdits        │  │     - REGENERATE_THUMBNAIL  │         │
│  │       → AssetEditThumbnail-    │  │       → AssetGenerate-      │         │
│  │         Generation           │  │         Thumbnails         │         │
│  │     - removeAssetEdits        │  │     - TRANSCODE_VIDEO        │         │
│  │       → AssetEditThumbnail-    │  │       → AssetEncodeVideo     │         │
│  │         Generation           │  │                               │         │
│  └─────────────────────────────────┘  └─────────────────────────────────┘         │
│                                                                                     │
│  ┌───────────────────────────────────────────────────────────────────────────┐         │
│  │  7. Motion Photo 视频提取直接触发 ⚠️                                        │         │
│  │     - metadata.service 检测 Motion Photo                                   │         │
│  │     - 提取内嵌视频 → 创建独立视频资产                                      │         │
│  │     - handleMetadataExtraction({ id: motionAsset.id })                   │         │
│  │     - 直接 queue({ name: AssetEncodeVideo, data: { id: motionAsset.id } })│         │
│  └───────────────────────────────────────────────────────────────────────────┘         │
└─────────────────────────────────────────────────────────────────────────────────────┘
                                    ↓
┌─────────────────────────────────────────────────────────────────────────────────────┐
│                           批量任务调度层                                               │
├─────────────────────────────────────────────────────────────────────────────────────┤
│                                                                                     │
│  ┌─────────────────────────────────┐  ┌─────────────────────────────────┐         │
│  │ AssetGenerateThumbnailsQueueAll │  │  AssetEncodeVideoQueueAll    │         │
│  │ 队列: ThumbnailGeneration       │  │  队列: VideoConversion       │         │
│  │ 流式读取资产                      │  │  流式读取视频资产           │         │
│  │   - force=true/false            │  │   - force=true/false         │         │
│  │   - 每 1000 个批量入队        │  │   - 每 1000 个批量入队    │         │
│  │     (JOBS_ASSET_PAGINATION_SIZE) │  │     (JOBS_ASSET_PAGINATION_SIZE) │         │
│  │                                   │  │                               │         │
│  │  子任务拆分:                      │  │  子任务拆分:                │         │
│  │  ┌─────────────────────────────┐│  │ ┌─────────────────────────┐│         │
│  │ │未编辑资产:                    ││  │ │视频资产:                 ││         │
│  │ │ AssetGenerateThumbnails      ││  │ │ AssetEncodeVideo         ││         │
│  │ │ 队列: ThumbnailGeneration   ││  │ │ 队列: VideoConversion   ││         │
│  │ └─────────────────────────────┘│  │ └─────────────────────────┘│         │
│  │  ┌─────────────────────────────┐│  │                             │         │
│  │ │已编辑资产:                    ││  │                             │         │
│  │ │ AssetEditThumbnailGeneration ││  │                             │         │
│  │ │ 队列: Editor ← ⚠️ 注意这里     ││  │                             │         │
│  │ └─────────────────────────────┘│  │                             │         │
│  │  ┌─────────────────────────────┐│  │                             │         │
│  │ │人物缩略图:                    ││  │                             │         │
│  │ │ PersonGenerateThumbnail    ││  │                             │         │
│  │ │ 队列: ThumbnailGeneration ││  │                             │         │
│  │ │ priority: 1 (高优先级)       ││  │                             │         │
│  │ └─────────────────────────────┘│  │                             │         │
│  └─────────────────────────────────┘  └─────────────────────────────────┘         │
└─────────────────────────────────────────────────────────────────────────────────────┘
                                    ↓
┌─────────────────────────────────────────────────────────────────────────────────────┐
│                         单个任务执行层 + 联动触发                                     │
├─────────────────────────────────────────────────────────────────────────────────────┤
│                                                                                     │
│  ┌─────────────────────────────────┐                                             │
│  │ AssetGenerateThumbnails       │                                             │
│  │ 队列: ThumbnailGeneration       │                                             │
│  │ 执行: 生成缩略图、预览图、fullsize│                                             │
│  │      计算 thumbhash               │                                             │
│  │      同步文件记录                │                                             │
│  └──────────┬──────────────────────┘                                             │
│             ↓ (条件触发后续任务)                                                       │
│  ┌─────────────────────────────────────────────────────────────────────┐         │
│  │  IF source='upload' OR notify=true → 触发后续任务链              │         │
│  │  ┌──────────────┐  ┌──────────────┐  ┌───────────────────┐ │         │
│  │  │ SmartSearch  │→│AssetDetect-│→│        OCR        │ │         │
│  │  │ 队列:        │ ││Faces         │ ││ 队列: OCR        │ │         │
│  │  │ SmartSearch   │ ││队列: Face-   │ ││                   │ │         │
│  │  └──────────────┘ ││Detection    │ │└───────────────────┘ │         │
│  │                     │                   │ │                         │         │
│  │                     ↓  (IF 资产类型是视频            │         │
│  │                ┌───────────────────────────────┐                    │         │
│  │                │ AssetEncodeVideo │                    │         │
│  │                │ 队列: Video-     │                    │         │
│  │                │ Conversion        │                    │         │
│  │                └───────────────────┘                    │         │
│  └─────────────────────────────────────────────────────────────────────┘         │
│                                                                                     │
│  ┌─────────────────────────────────┐  ┌─────────────────────────────────┐         │
│  │ AssetEditThumbnailGeneration │  │ PersonGenerateThumbnail    │         │
│  │ ⚠️ 队列: Editor           │  │ 队列: ThumbnailGeneration       │         │
│  │ 执行: 生成编辑后缩略图        │  │ 优先级: 1 (高优先级)         │         │
│  │      更新人脸/OCR可见性         │  │ 执行: 基于人脸检测裁剪     │         │
│  │      → 裁剪区域              │  │      生成人物缩略图         │         │
│  │ 后续: 仅发送 WebSocket 通知     │  │ 后续: 仅发送 WebSocket 通知 │         │
│  │       (AssetEditReadyV2)       │  │       (on_person_thumbnail)│         │
│  │       ❌ 不触发后续任务链     │  │       ❌ 不触发后续任务链 │         │
│  └─────────────────────────────────┘  └─────────────────────────────────┘         │
│                                                                                     │
│  ┌─────────────────────────────────┐                                             │
│  │ StorageTemplateMigrationSingle │                                             │
│  │ ⚠️ 队列: StorageTemplateMigration (串行)│                                             │
│  │ 执行: 迁移文件到模板位置        │                                             │
│  │ 后续: IF source='upload' OR 'copy'                                      │
│  │       → AssetGenerateThumbnails                                      │
│  └─────────────────────────────────┘                                             │
│                                                                                     │
│  ┌─────────────────────────────────┐                                             │
│  │ AssetEncodeVideo            │                                             │
│  │ 队列: VideoConversion       │                                             │
│  │ 执行: 判断转码必要性           │                                             │
│  │      三级降级重试机制         │                                             │
│  │        1. 完整硬件加速        │                                             │
│  │        2. 仅编码硬件加速       │                                             │
│  │        3. 纯软件编解码         │                                             │
│  │ 后续: ❌ 无后续任务         │                                             │
│  └─────────────────────────────────┘                                             │
└─────────────────────────────────────────────────────────────────────────────────────┘
                                    ↓
┌─────────────────────────────────────────────────────────────────────────────────────┐
│                        Motion Photo / Live Photo 特殊处理                              │
├─────────────────────────────────────────────────────────────────────────────────────┤
│  Motion Photo 检测 → 提取内嵌视频 → 创建独立视频资产 → 隐藏该视频资产                │
│                                                                                     │
│  注意: 提取后的视频资产通过正常流程处理                                           │
│        - 元数据提取 → 存储模板迁移 → 缩略图生成 → (如需)视频转码                   │
│        - 无特殊队列联动，与普通视频一致                                           │
└─────────────────────────────────────────────────────────────────────────────────────┘
```

---

## 4. 缩略图生成任务详细分析

### 4.1 队列与任务分类

| 任务名称 | 所属队列 | 优先级 | 触发后联动
|---|---|---|---|
| `AssetGenerateThumbnailsQueueAll` | `ThumbnailGeneration` | 普通 | 拆分子任务
| `AssetGenerateThumbnails` | `ThumbnailGeneration` | 普通 | ✅ 触发后续任务链（条件）
| `AssetEditThumbnailGeneration` | `Editor` ⚠️ | 普通 | ❌ 仅 WebSocket 通知
| `PersonGenerateThumbnail` | `ThumbnailGeneration` | 高 (priority: 1) | ❌ 仅 WebSocket 通知

### 4.2 所有触发入口详解

#### 入口 1: 手动队列 API 触发
```typescript
// server/src/services/queue.service.ts:222-224
case QueueName.ThumbnailGeneration: {
  return this.jobRepository.queue({ 
    name: JobName.AssetGenerateThumbnailsQueueAll, 
    data: { force } 
  });
}
```

#### 入口 2: 夜间任务自动触发
```typescript
// server/src/services/queue.service.ts:284-286
if (config.nightlyTasks.missingThumbnails) {
  jobs.push({ name: JobName.AssetGenerateThumbnailsQueueAll, data: { force: false } });
}
```

#### 入口 3: 上传流程联动触发
```
上传流程链路:
1. 资产上传完成 → 元数据提取
2. 触发 AssetMetadataExtracted 事件
3. → StorageTemplateMigrationSingle (串行队列)
4. → IF source='upload' OR source='copy'
   → AssetGenerateThumbnails
```

```typescript
// server/src/services/job.service.ts:83-87
case JobName.StorageTemplateMigrationSingle: {
  if (item.data.source === 'upload' || item.data.source === 'copy') {
    await this.jobRepository.queue({ 
      name: JobName.AssetGenerateThumbnails, 
      data: item.data 
    });
  }
  break;
}
```

#### 入口 4: 资产显示触发
```typescript
// server/src/services/notification.service.ts:147-149
@OnEvent({ name: 'AssetShow' })
async onAssetShow({ assetId }: ArgOf<'AssetShow'>) {
  await this.jobRepository.queue({ 
    name: JobName.AssetGenerateThumbnails, 
    data: { id: assetId, notify: true } 
  });
}
```

#### 入口 5: 资产编辑触发
```typescript
// server/src/services/asset.service.ts:596-597
const newEdits = await this.assetEditRepository.replaceAll(id, edits);
await this.jobRepository.queue({ 
  name: JobName.AssetEditThumbnailGeneration, 
  data: { id } 
});

// server/src/services/asset.service.ts:614-616
await this.assetEditRepository.replaceAll(id, []);
await this.jobRepository.queue({ 
  name: JobName.AssetEditThumbnailGeneration, 
  data: { id } 
});
```

#### 入口 6: 单个资产任务触发
```typescript
// server/src/services/asset.service.ts:475-477
case AssetJobName.REGENERATE_THUMBNAIL: {
  jobs.push({ name: JobName.AssetGenerateThumbnails, data: { id } });
  break;
}
```

### 4.3 批量任务调度逻辑

```typescript
// server/src/services/media.service.ts:69-116
@OnJob({ name: JobName.AssetGenerateThumbnailsQueueAll, queue: QueueName.ThumbnailGeneration })
async handleQueueGenerateThumbnails({ force }) {
  const queueAll = async () => {
    await this.jobRepository.queueAll(jobs);
    jobs = [];
  };

  // 第一阶段: 资产缩略图处理
  for await (const asset of this.assetJobRepository.streamForThumbnailJob({ force, fullsizeEnabled })) {
    if (force || !asset.isEdited) {
      // 未编辑资产 → 普通缩略图任务
      jobs.push({ name: JobName.AssetGenerateThumbnails, data: { id: asset.id } });
    }
    if (asset.isEdited) {
      // ⚠️ 已编辑资产 → Editor 队列任务
      jobs.push({ name: JobName.AssetEditThumbnailGeneration, data: { id: asset.id } });
    }
    
    // 每累积 1000 个任务批量入队
    if (jobs.length >= JOBS_ASSET_PAGINATION_SIZE) {
      await queueAll();
    }
  }
  await queueAll();

  // 第二阶段: 人物缩略图处理
  for await (const person of this.personRepository.getAll(force ? undefined : { thumbnailPath: '' })) {
    jobs.push({ name: JobName.PersonGenerateThumbnail, data: { id: person.id } });
    if (jobs.length >= JOBS_ASSET_PAGINATION_SIZE) {
      await queueAll();
    }
  }
  await queueAll();
}
```

### 4.4 后续任务联动逻辑（条件触发）

```typescript
// server/src/services/job.service.ts:134-155
case JobName.AssetGenerateThumbnails: {
  // ⚠️ 关键条件: 仅在以下情况触发后续任务
  if (!item.data.notify && item.data.source !== 'upload') {
    break;
  }

  // 必选后续任务
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
```

### 4.5 AssetEditThumbnailGeneration 任务逻辑（⚠️ Editor 队列）

```typescript
// server/src/services/media.service.ts:170-206
@OnJob({ name: JobName.AssetEditThumbnailGeneration, queue: QueueName.Editor })
async handleAssetEditThumbnailGeneration({ id }) {
  const generated = await this.generateEditedThumbnails(asset, config);
  await this.syncFiles(
    asset.files.filter((file) => file.isEdited),
    generated?.files ?? [],
  );
  
  // 更新裁剪后人脸和 OCR 的可见状态
  const faceStatuses = checkFaceVisibility(assetFaces, originalDimensions, cropBox);
  await this.personRepository.updateVisibility(faceStatuses.visible, faceStatuses.hidden);

  const ocrStatuses = checkOcrVisibility(ocrData, originalDimensions, cropBox);
  await this.ocrRepository.updateOcrVisibilities(asset.id, ocrStatuses.visible, ocrStatuses.hidden);
}

// server/src/services/job.service.ts:99-131
case JobName.AssetEditThumbnailGeneration: {
  // ⚠️ 仅发送 WebSocket 通知，不触发后续任务链
  if (asset) {
    this.websocketRepository.clientSend('AssetEditReadyV2', asset.ownerId, { asset, edit: edits });
  }
  break;
}
```

---

## 5. 视频转码任务详细分析

### 5.1 队列与任务分类

| 任务名称 | 所属队列 | 优先级 | 触发后联动
|---|---|---|---|
| `AssetEncodeVideoQueueAll` | `VideoConversion` | 普通 | 拆分子任务
| `AssetEncodeVideo` | `VideoConversion` | 普通 | ❌ 无后续任务

### 5.2 所有触发入口详解

#### 入口 1: 手动队列 API 触发
```typescript
// server/src/services/queue.service.ts:193-196
case QueueName.VideoConversion: {
  return this.jobRepository.queue({ 
    name: JobName.AssetEncodeVideoQueueAll, 
    data: { force } 
  });
}
```

#### 入口 2: 缩略图生成联动触发
```typescript
// server/src/services/job.service.ts:151-153
// 仅在 source='upload' 或 notify=true 时，且资产是视频
if (asset.type === AssetType.Video) {
  jobs.push({ name: JobName.AssetEncodeVideo, data: item.data });
}
```

#### 入口 3: 单个资产任务触发
```typescript
// server/src/services/asset.service.ts:480-482
case AssetJobName.TRANSCODE_VIDEO: {
  jobs.push({ name: JobName.AssetEncodeVideo, data: { id } });
  break;
}
```

### 5.3 批量任务调度逻辑

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
  await this.jobRepository.queueAll(queue);
}
```

### 5.4 三级降级重试机制

```typescript
// server/src/services/media.service.ts:569-653
@OnJob({ name: JobName.AssetEncodeVideo, queue: QueueName.VideoConversion })
async handleVideoConversion({ id }) {
  // 第一级: 完整硬件加速
  try {
    await this.mediaRepository.transcode(input, output, command);
  } catch (error) {
    // 第二级: 仅编码硬件加速，软件解码
    if (ffmpeg.accelDecode) {
      try {
        ffmpeg = { ...ffmpeg, accelDecode: false };
        await this.mediaRepository.transcode(input, output, command);
      } catch { /* 继续降级 */ }
    }
    
    // 第三级: 完全关闭硬件加速，纯软件编解码
    ffmpeg = { ...ffmpeg, accel: TranscodeHardwareAcceleration.Disabled };
    await this.mediaRepository.transcode(input, output, command);
  }
}
```

---

## 6. Motion Photo / Live Photo 处理流程

### 6.1 检测与提取

```typescript
// server/src/services/metadata.service.ts:670-808
private async applyMotionPhotos(asset, tags, dates, stats) {
  // 1. 检测 Motion Photo 标记
  const isMotionPhoto = tags.MotionPhoto;
  const hasMotionPhotoVideo = tags.MotionPhotoVideo;
  const hasEmbeddedVideoFile = tags.EmbeddedVideoType === 'MotionPhoto_Data' && tags.EmbeddedVideoFile;
  
  // 2. 从 XMP 目录或 EXIF 二进制字段提取视频
  if (isMotionPhoto && directory) {
    // 从目录条目提取
  } else if (hasMotionPhotoVideo) {
    video = await this.metadataRepository.extractBinaryTag(asset.originalPath, 'MotionPhotoVideo');
  } else if (hasEmbeddedVideoFile) {
    video = await this.metadataRepository.extractBinaryTag(asset.originalPath, 'EmbeddedVideoFile');
  }
  
  // 3. 创建独立的视频资产
  const motionAsset = await this.assetRepository.create({ ... });
  
  // 4. 隐藏视频资产
  await this.assetRepository.update({ id: motionAsset.id, visibility: AssetVisibility.Hidden });
  
  // 5. 关联到原图片资产
  await this.assetRepository.update({ id: asset.id, livePhotoVideoId: motionAsset.id });
}
```

### 6.2 后续任务处理

提取后的视频资产通过**正常流程**处理，无特殊队列联动：

```
Motion Photo 视频资产流程:
1. 创建视频资产 (Hidden 状态)
2. 元数据提取 (AssetExtractMetadata)
3. → StorageTemplateMigrationSingle
4. → AssetGenerateThumbnails (source=upload)
5. → (如需) AssetEncodeVideo
```

---

## 7. 批量入队优化机制

```typescript
// server/src/repositories/job.repository.ts:159-189
async queueAll(items: JobItem[]): Promise<void> {
  const promises = [];
  const itemsByQueue = {};  // 按队列名称分组优化 Redis 操作
  
  for (const item of items) {
    const queueName = this.getQueueName(item.name);
    const job = { name: item.name, data: item.data || {}, options: ... };
    
    if (job.options?.jobId) {
      // 有 jobId 的任务单独入队（去重，避免重复）
      promises.push(this.getQueue(queueName).add(item.name, item.data, job.options));
    } else {
      // 无 jobId 的任务按队列分组，批量 addBulk
      itemsByQueue[queueName] = itemsByQueue[queueName] || [];
      itemsByQueue[queueName].push(job);
    }
  }
  
  // 每队列批量入队，减少 Redis 网络交互
  for (const [queueName, jobs] of Object.entries(itemsByQueue)) {
    const queue = this.getQueue(queueName as QueueName);
    promises.push(queue.addBulk(jobs));
  }
  
  await Promise.all(promises);
}
```

---

## 8. 关键配置项

### 8.1 缩略图相关配置
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

### 8.2 视频转码相关配置
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

### 8.3 夜间任务配置
```typescript
nightlyTasks: {
  startTime: string,              // 开始时间 HH:MM
  missingThumbnails: boolean,        // 生成缺失缩略图
}
```

---

## 9. 总结

### 9.1 架构特点
1. **基于 BullMQ 的可靠队列系统**，支持任务持久化、重试、优先级
2. **事件驱动的任务调度**，通过事件总线解耦，支持任务链式触发
3. **流式批量处理**，每 1000 个任务批量入队，避免内存压力
4. **灵活的并发控制**，4 个队列必须串行，其余可配置并发数
5. **多级容错机制**，视频转码支持三级硬件加速降级重试
6. **智能任务去重**，通过 jobId 机制避免重复任务
7. **条件触发后续任务**，仅上传或通知场景触发完整处理链

### 9.2 关键修正说明
- ✅ `AssetEditThumbnailGeneration` 实际所属队列为 `QueueName.Editor`，**不是** `ThumbnailGeneration`
- ✅ `AssetEditThumbnailGeneration` 完成后**不触发**后续任务链（SmartSearch、人脸检测、OCR、视频转码），仅发送 WebSocket 通知
- ✅ `StorageTemplateMigrationSingle` 所属队列为 `QueueName.StorageTemplateMigration`，是**串行执行队列**
- ✅ `PersonGenerateThumbnail` 具有高优先级 (`priority: 1`)，完成后仅发送 WebSocket 通知，不触发后续任务
- ✅ Motion Photo/Live Photo 提取的视频资产通过正常流程处理，无特殊队列联动

### 9.3 关键文件位置
- 队列服务: `server/src/services/queue.service.ts`
- 任务服务: `server/src/services/job.service.ts`
- 任务仓库: `server/src/repositories/job.repository.ts`
- 媒体处理: `server/src/services/media.service.ts`
- 资产服务: `server/src/services/asset.service.ts`
- 存储模板服务: `server/src/services/storage-template.service.ts`
- 通知服务: `server/src/services/notification.service.ts`
- 枚举定义: `server/src/enum.ts`
- 常量定义: `server/src/constants.ts`
