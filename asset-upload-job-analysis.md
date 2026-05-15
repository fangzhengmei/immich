# Immich 资产上传到作业分发完整链路分析

## 概述

本文档详细分析了 Immich 系统中从资产上传到后台作业分发的完整处理链路。整个流程涵盖了文件接收、存储、元数据提取、以及各类后续处理任务的排队和执行。

---

## 1. 上传请求处理流程

### 1.1 拦截器链

**文件路径**: `server/src/middleware/`

上传请求经过以下拦截器处理：

| 拦截器 | 职责 | 关键代码 |
|--------|------|---------|
| `AssetUploadInterceptor` | 预检查重复资产 | 检查 `x-immich-checksum` header，如已存在则直接返回 |
| `FileUploadInterceptor` | 处理文件上传 | 使用 `multer` 处理 multipart/form-data，生成 SHA1 校验和 |

**关键代码片段** (`asset-upload.interceptor.ts:14-26`):
```typescript
async intercept(context: ExecutionContext, next: CallHandler<any>) {
  const checksum = fromMaybeArray(req.headers[ImmichHeader.Checksum]);
  const response = await this.service.getUploadAssetIdByChecksum(req.user, checksum);
  if (response) {
    res.status(200);
    return of({ status: AssetMediaStatus.DUPLICATE, id: response.id });
  }
  return next.handle();
}
```

### 1.2 文件上传处理

**文件路径**: `server/src/middleware/file-upload.interceptor.ts:97-137`

文件上传处理流程：
1. 生成随机 UUID 作为文件标识
2. 计算 SHA1 校验和（仅对资产数据文件）
3. 流式写入到上传目录
4. 返回文件路径、大小、校验和等信息

**存储路径规则**:
- 上传临时目录: `upload/{userId}/{fileUuid}/`
- 最终存储: 根据存储模板决定

---

## 2. 资产创建流程

**文件路径**: `server/src/services/asset-media.service.ts:320-364`

### 2.1 核心创建步骤

```
上传请求
    ↓
检查配额 (requireQuota)
    ↓
处理 Live Photo 关联 (onBeforeLink)
    ↓
创建资产记录 (assetRepository.create)
    ├─ ownerId, libraryId
    ├─ checksum, originalPath
    ├─ fileCreatedAt, fileModifiedAt, localDateTime
    ├─ type, isFavorite, duration
    ├─ visibility, livePhotoVideoId
    └─ originalFileName
    ↓
如提供 metadata，upsert metadata
    ↓
如有 sidecar 文件，upsert sidecar 记录
    ↓
更新文件时间戳 (storageRepository.utimes)
    ↓
Upsert EXIF 信息 (fileSizeInByte)
    ↓
发出 AssetCreate 事件
    ↓
排队元数据提取作业 (AssetExtractMetadata)
    ↓
返回资产信息
```

### 2.2 关键方法

**`create()` 方法** (`asset-media.service.ts:320-364`):
- 初始化资产基本信息
- 处理 sidecar 文件
- 写入 EXIF 初始数据
- 触发事件和后续作业

**第一个排队的作业**:
```typescript
await this.jobRepository.queue({ 
  name: JobName.AssetExtractMetadata, 
  data: { id: asset.id, source: 'upload' } 
});
```

---

## 3. 元数据提取作业

**文件路径**: `server/src/services/metadata.service.ts:235-416`

### 3.1 作业信息

| 属性 | 值 |
|------|-----|
| 作业名称 | `JobName.AssetExtractMetadata` |
| 队列名称 | `QueueName.MetadataExtraction` |
| 触发时机 | 资产创建后立即排队 |

### 3.2 处理流程

```
MetadataExtraction 队列接收任务
    ↓
并行执行:
  ├─ 读取 EXIF 标签 (metadataRepository.readTags)
  ├─ 读取 sidecar 文件标签 (如存在)
  ├─ 视频文件 probe (如为视频)
  └─ 获取文件 stat 信息
    ↓
合并标签优先级: sidecar > video probe > media tags
    ↓
提取日期时间信息 (firstDateTime)
    ↓
提取图像尺寸 (width, height)
    ↓
如有 GPS 信息，执行反向地理编码
    ↓
提取标签列表 (TagsList, HierarchicalSubject, Keywords)
    ↓
构建 EXIF 数据对象:
  ├─ 日期时间信息
  ├─ GPS/地理信息
  ├─ 图像/文件信息
  ├─ 相机信息 (make, model, iso, etc.)
  ├─ 评论/描述信息
  └─ 分组信息 (livePhotoCID, autoStackId)
    ↓
如为视频，提取音频、视频流信息
    ↓
Tasks 并行执行:
  ├─ 更新资产基本信息 (duration, dates, dimensions)
  ├─ upsert EXIF 到数据库
  ├─ 应用标签列表
  ├─ 如为 Motion Photo，提取视频
  └─ 如启用人脸导入，应用标签人脸
    ↓
如存在 livePhotoCID，关联 Live Photo 资产
    ↓
更新资产作业状态 (metadataExtractedAt)
    ↓
发出 AssetMetadataExtracted 事件
```

### 3.3 元数据提取后事件触发

**事件**: `AssetMetadataExtracted` (`metadata.service.ts:411-415`)

该事件触发后续作业链，在 `JobService` 中处理：

---

## 4. 作业分发与链式处理

**文件路径**: `server/src/services/job.service.ts:49-225`

### 4.1 作业执行流程

```
JobRun 事件
    ↓
JobService.onJobRun() 处理
    ├─ 发出 JobStart 事件
    ├─ 调用 jobRepository.run(job) 执行作业
    ├─ 作业成功后发出 JobSuccess 事件
    ├─ 如返回 Success/Skipped，调用 onDone() 处理后续作业
    ├─ 异常时发出 JobError 事件
    └─ 最终发出 JobComplete 事件
```

### 4.2 关键作业链 (onDone 方法)

#### 4.2.1 Sidecar 流程
```typescript
case JobName.SidecarCheck:
  → queue JobName.AssetExtractMetadata

case JobName.SidecarWrite:
  → queue JobName.AssetExtractMetadata (source: 'sidecar-write')
```

#### 4.2.2 存储模板迁移
```typescript
case JobName.StorageTemplateMigrationSingle:
  if (source === 'upload' || source === 'copy')
    → queue JobName.AssetGenerateThumbnails
```

#### 4.2.3 缩略图生成完成后的作业链

**文件路径**: `job.service.ts:134-215`

当 `AssetGenerateThumbnails` 作业完成且 `source === 'upload'` 时：

```
AssetGenerateThumbnails (source: 'upload') 完成
    ↓
并行排队以下作业:
  ├─ JobName.SmartSearch              # 智能搜索特征提取
  ├─ JobName.AssetDetectFaces         # 人脸识别
  ├─ JobName.Ocr                      # OCR 文字识别
  └─ (如为视频) JobName.AssetEncodeVideo  # 视频转码
    ↓
如资产可见 (Timeline/Archive)，发送 websocket 通知:
  ├─ on_upload_success (给用户)
  └─ AssetUploadReadyV2 (含 EXIF 详情)
```

---

## 5. 核心作业队列与任务

### 5.1 完整作业列表

| 作业名称 (JobName) | 队列 (QueueName) | 职责 |
|-------------------|-----------------|------|
| `AssetExtractMetadata` | `MetadataExtraction` | 提取 EXIF、地理编码、标签、人脸等 |
| `AssetGenerateThumbnails` | `ThumbnailGeneration` | 生成缩略图、预览图、WebP 格式 |
| `AssetEncodeVideo` | `VideoConversion` | 视频转码为兼容格式 |
| `AssetDetectFaces` | `FacialRecognition` | 人脸检测与识别 |
| `SmartSearch` | `SmartSearch` | CLIP 图像特征提取 |
| `Ocr` | `Ocr` | 图像文字识别 |
| `SidecarCheck` | `Sidecar` | 检查 sidecar 文件变化 |
| `SidecarWrite` | `Sidecar` | 写入元数据到 XMP sidecar |
| `AssetDelete` | `BackgroundTask` | 资产删除清理 |
| `PersonGenerateThumbnail` | `ThumbnailGeneration` | 生成人物头像 |

### 5.2 队列并发控制

**文件路径**: `job.repository.ts:108-116`

```typescript
setConcurrency(queueName: QueueName, concurrency: number) {
  const worker = this.workers[queueName];
  worker.concurrency = concurrency;
}
```

并发配置通过系统配置管理，初始化于 `ConfigInit` 事件。

---

## 6. 作业仓库 (JobRepository) 详解

**文件路径**: `server/src/repositories/job.repository.ts`

### 6.1 核心功能

| 方法 | 功能 |
|------|------|
| `setup()` | 发现所有 `@OnJob()` 装饰的处理器，建立作业映射 |
| `startWorkers()` | 为每个 QueueName 启动 BullMQ Worker |
| `queue()` / `queueAll()` | 单个/批量排队作业 |
| `run()` | 执行具体作业处理器 |
| `pause()` / `resume()` | 暂停/恢复队列 |
| `getJobCounts()` | 获取队列统计 |
| `waitForQueueCompletion()` | 等待队列处理完成 (测试用) |

### 6.2 作业发现机制

使用 `@OnJob()` 装饰器标记作业处理器：
```typescript
// decorator 定义 (src/decorators/index.ts)
export const OnJob = (config: JobConfig) => 
  SetMetadata(MetadataKey.JobConfig, config);

// 使用示例
@OnJob({ name: JobName.AssetExtractMetadata, queue: QueueName.MetadataExtraction })
async handleMetadataExtraction(data: JobOf<JobName.AssetExtractMetadata>) { ... }
```

### 6.3 作业选项 (JobOptions)

部分作业有特殊排队选项：

| 作业 | 选项 | 说明 |
|------|------|------|
| `NotifyAlbumUpdate` | `jobId: {id}/{recipientId}`, `delay` | 去重 + 延迟通知 |
| `StorageTemplateMigrationSingle` | `jobId: asset.id` | 防止重复迁移 |
| `PersonGenerateThumbnail` | `priority: 1` | 高优先级 |
| `FacialRecognitionQueueAll` | `jobId: JobName.XXX` | 单例作业 |

---

## 7. 完整上传到作业分发链路图

```
客户端上传请求
    │
    ▼
[HTTP 层]
    ├─ AuthGuard → 认证用户
    ├─ AssetUploadInterceptor → 预检查重复 (按 checksum)
    └─ FileUploadInterceptor → 接收文件，计算 SHA1
    │
    ▼
[Service 层: AssetMediaService]
    ├─ uploadAsset()
    │   ├─ 检查用户配额
    │   ├─ 处理 Live Photo 关联
    │   ├─ 创建资产 DB 记录
    │   ├─ 处理 metadata/sidecar
    │   ├─ 更新文件时间戳
    │   ├─ 初始化 EXIF (fileSizeInByte)
    │   ├─ 发出 AssetCreate 事件
    │   └─ ✅ 排队: AssetExtractMetadata (source: 'upload')
    │
    ▼
[队列层: BullMQ]
    ├─ QueueName.MetadataExtraction
    │   └─ AssetExtractMetadata
    │       ├─ 读取 EXIF/sidecar 标签
    │       ├─ 视频 probe (如需要)
    │       ├─ 反向地理编码 (GPS)
    │       ├─ 提取标签/人脸 (EXIF)
    │       ├─ Motion Photo 视频提取
    │       ├─ 更新 DB (Asset + AssetExif)
    │       ├─ 发出 AssetMetadataExtracted 事件
    │       └─ 更新 metadataExtractedAt
    │
    ▼  注: 下一阶段由 thumbnail generation 触发 (非 metadata)
    │
    ├─ (StorageTemplateMigrationSingle)  ← 可选，如果启用了模板
    │   └─ source === 'upload' → 排队 AssetGenerateThumbnails
    │
    ▼
    ├─ QueueName.ThumbnailGeneration
    │   └─ AssetGenerateThumbnails (source: 'upload')
    │       ├─ 生成缩略图 (WEBP)
    │       ├─ 生成预览图 (JPEG/WEBP)
    │       ├─ 提取 thumbhash
    │       ├─ 更新 asset.thumbhash
    │       └─ ✅ JobSuccess 触发 onDone()
    │           │
    │           ├─ 并行排队:
    │           │   ├─ SmartSearch
    │           │   ├─ AssetDetectFaces
    │           │   ├─ Ocr
    │           │   └─ (视频) AssetEncodeVideo
    │           │
    │           └─ WebSocket 通知 (如可见)
    │               ├─ on_upload_success
    │               └─ AssetUploadReadyV2
    │
    ├─ QueueName.SmartSearch
    │   └─ 提取 CLIP 特征，用于语义搜索
    │
    ├─ QueueName.FacialRecognition
    │   └─ 检测人脸，聚类，识别人物
    │
    ├─ QueueName.Ocr
    │   └─ 识别图像中的文字
    │
    └─ QueueName.VideoConversion
        └─ 视频转码 (H.264/H.265/AAC)
```

---

## 8. 关键设计模式

### 8.1 事件驱动架构

- 使用 `EventRepository` 作为事件总线
- `@OnEvent()` 装饰器订阅事件
- 作业完成后通过事件触发后续作业链

### 8.2 装饰器驱动的作业注册

通过 `@OnJob()` 装饰器自动发现和注册作业处理器，无需手动注册。

### 8.3 批量处理优化

- `queueAll()` 支持批量排队，使用 `addBulk()` 提高性能
- 作业按队列分组，统一批量提交

### 8.4 幂等性设计

- 关键作业使用 `jobId` 去重（如相册通知、模板迁移）
- 重复检测基于 SHA1 校验和

---

## 9. 错误处理与重试

### 9.1 作业执行异常

```typescript
// job.service.ts:58-61
try {
  await this.eventRepository.emit('JobStart', queueName, job);
  const response = await this.jobRepository.run(job);
  await this.eventRepository.emit('JobSuccess', { job, response });
} catch (error) {
  await this.eventRepository.emit('JobError', { job, error });
} finally {
  await this.eventRepository.emit('JobComplete', queueName, job);
}
```

### 9.2 视频转码降级策略

**文件路径**: `media.service.ts:616-642`

视频转码失败时的降级流程：
1. 尝试硬件加速编码 + 软件解码
2. 尝试纯软件编码
3. 仍失败则作业失败，由 BullMQ 重试机制处理

---

## 10. 性能考虑点

### 10.1 并发控制

- 每个队列独立配置并发数
- 元数据提取、缩略图生成、人脸识别等高 CPU 作业独立队列

### 10.2 批量操作

- 大规模作业（如库扫描）使用流式处理 + 批量排队
- 分页大小: `JOBS_ASSET_PAGINATION_SIZE`

### 10.3 优先级

- 人物缩略图生成使用高优先级 (`priority: 1`)
- 上传流程中的作业按自然顺序执行

---

## 11. 涉及的主要文件

| 文件 | 主要职责 |
|------|---------|
| `server/src/controllers/asset-media.controller.ts` | 上传 API 端点 |
| `server/src/controllers/asset.controller.ts` | 资产管理 API |
| `server/src/services/asset-media.service.ts` | 资产上传核心逻辑 |
| `server/src/services/metadata.service.ts` | 元数据提取作业 |
| `server/src/services/media.service.ts` | 缩略图、视频处理作业 |
| `server/src/services/job.service.ts` | 作业生命周期管理 |
| `server/src/repositories/job.repository.ts` | 队列与作业管理 |
| `server/src/middleware/asset-upload.interceptor.ts` | 重复检测拦截器 |
| `server/src/middleware/file-upload.interceptor.ts` | 文件上传处理 |

---

## 总结

整个上传到作业分发流程采用了**事件驱动 + 队列链式处理**的架构设计：

1. **分层处理**: HTTP 层 → Service 层 → 队列层
2. **作业链**: 元数据提取 → 缩略图生成 → (智能搜索/人脸识别/OCR/视频转码)
3. **可扩展性**: 新增处理步骤只需添加作业处理器和 `onDone()` 中的链接逻辑
4. **可靠性**: BullMQ 提供持久化、重试、失败管理

该设计确保了上传流程的高效性和可扩展性，能够支持大规模媒体文件处理。
