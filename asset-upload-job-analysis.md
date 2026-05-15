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

### 1.2 短路分支一：前置 checksum 重复检测

这是上传流程中的**第一级快速失败优化路径**，在文件真正上传到服务器之前就完成重复检测：

```
客户端上传请求 (携带 x-immich-checksum header)
    ↓
AuthGuard → 认证用户
    ↓
AssetUploadInterceptor 拦截
    ├─ 从 header 中提取 SHA1 checksum
    └─ 调用 getUploadAssetIdByChecksum(userId, checksum) 查询 DB
    ↓
┌─ 命中？─┐
│  Yes    │  No ────> 继续执行 FileUploadInterceptor，进入正常上传流程
│    ↓    │
│  HTTP 200 OK 返回
│  {
│    status: "DUPLICATE",
│    id: <existing-asset-id>
│  }
│    ↓
│  流程结束（无文件写入、无作业排队）
└─────────┘
```

**设计意图**:
- 节省带宽：避免不必要的大文件传输
- 节省计算资源：避免重复的文件处理、缩略图生成等
- 快速响应：客户端可以立即知道文件已存在

### 1.3 短路分支二：数据库 checksum 约束冲突兜底返回

这是上传流程中的**第二级重复防护机制**，在文件已上传到服务器后、数据库层面的最后关口拦截重复。

#### 1.3.1 数据库约束定义

**文件路径**: `server/src/schema/migrations/1744910873969-InitialMigration.ts:441`

```sql
CREATE UNIQUE INDEX "UQ_assets_owner_checksum 
ON "assets" ("ownerId", "checksum") 
WHERE ("libraryId" IS NULL
```

**约束说明**:
- 联合唯一索引：`(ownerId, checksum)`
- 生效条件：仅对用户个人资产 (`libraryId IS NULL`)
- 设计目的：确保同一用户不会有相同 checksum 的资产

#### 1.3.2 触发时机与流程

**文件路径**: `server/src/services/asset-media.service.ts:288-318`

```
FileUploadInterceptor 完成文件接收
    ↓
uploadAsset() 调用 create()
    ├─ requireQuota() 检查配额
    ├─ 处理 Live Photo 关联
    └─ 调用 assetRepository.create()
        └─ 执行 INSERT 语句
        └─ 触发数据库唯一约束冲突 (UQ_assets_owner_checksum)
    ↓
异常抛出，进入 catch 异常
    ↓
handleUploadError() 捕获异常
    ├─ 识别 isAssetChecksumConstraint(error)
    │   └─ 判断 error.constraint_name === 'UQ_assets_owner_checksum'
    ├─ 排队 FileDelete 作业：删除已写入磁盘的文件 + sidecar 文件
    ├─ 调用 getUploadAssetIdByChecksum(userId, file.checksum) 查询已有资产 ID
    ├─ 如是共享链接上传，将已有资产加入共享链接/相册
    └─ 返回 { status: DUPLICATE, id: duplicateId }
    ↓
HTTP 200 OK 返回
    ↓
流程结束（无后续处理任务排队）
```

**关键代码** (`asset-media.service.ts:288-318`):
```typescript
private async handleUploadError(error, auth, file, sidecarFile) {
  // 第一步：清理已上传的文件
  await this.jobRepository.queue({
    name: JobName.FileDelete,
    data: { files: [file.originalPath, sidecarFile?.originalPath] },
  });

  // 第二步：识别为 checksum 约束冲突，走重复兜底逻辑
  if (isAssetChecksumConstraint(error)) {
    const duplicateId = await this.assetRepository.getUploadAssetIdByChecksum(
      auth.user.id, file.checksum);
    
    if (auth.sharedLink) {
      await this.addToSharedLink(auth.sharedLink, duplicateId);
    }
    
    return { status: AssetMediaStatus.DUPLICATE, id: duplicateId };
  }

  throw error;
}
```

#### 1.3.3 两条短路分支对比

| 对比维度 | 分支一：前置 checksum 短路 | 分支二：约束冲突兜底 |
|---------|----------------------|------------------|
| **触发时机** | 文件接收前（Interceptor 层） | 文件接收后（Service 层） |
| **触发条件** | 客户端携带 `x-immich-checksum` header | 客户端未传 header 或 header 不准确，但文件实际 checksum 重复 |
| **发生阶段** | 无任何文件写入 | 文件已写入磁盘但尚未创建数据库记录 |
| **数据库操作** | 1 次 SELECT 查询 | 1 次 INSERT 失败 + 1 次 SELECT 查询 |
| **磁盘写入** | 无 | 有（后删除） |
| **资源消耗** | 极低（仅网络 + 1 次 DB 查询） | 中等（文件 I/O + 数据库事务回滚） |
| **排队作业** | 无 | 仅 `FileDelete`（清理已上传文件） |
| **后续任务分发** | 不进入（在 interceptor 层就返回） | 不进入（异常在 `create()` 调用 `jobRepository.queue()` 之前抛出） |
| **典型场景** | 移动客户端批量上传，客户端本地已计算好 checksum | Web 端上传、客户端未传 header 或网络异常重试 |

#### 1.3.4 为何两条路径都不会进入后续任务分发链路

**分支一（前置短路）不进入原因**：
- 发生在 NestJS Interceptor 层，在调用 `uploadAsset()` 服务方法之前
- `AssetUploadInterceptor.intercept()` 直接通过 `of()` 返回 Observable，不调用 `next.handle()`
- 完全不会执行 Service 层的任何逻辑，包括 `create()` 方法中的作业排队

**分支二（约束冲突兜底）不进入原因**：
- `create()` 方法中作业排队发生在**最后一步**（第 361 行）：`await this.jobRepository.queue({ name: JobName.AssetExtractMetadata, ... })`
- 而 `assetRepository.create()` 数据库插入发生在**第一步**（第 321 行）
- 约束冲突异常在插入时抛出，直接跳转到 catch 块，不会执行到排队语句
- 异常处理中唯一排队的是 `FileDelete` 清理作业，不属于资产处理任务链

```typescript
// create() 方法执行顺序
private async create(ownerId, dto, file, sidecarFile) {
  const asset = await this.assetRepository.create({...}); // ← 这里抛出异常
  // ...
  await this.jobRepository.queue({ name: JobName.AssetExtractMetadata, ... }); // ← 异常时不会执行到这里
}
```

### 1.4 文件上传处理

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

### 2.1 核心创建步骤（正常流程，非短路）

```
上传请求（已通过拦截器检查）
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

#### 4.2.4 智能搜索完成后触发重复检测

**文件路径**: `job.service.ts:218-222`

```typescript
case JobName.SmartSearch: {
  if (item.data.source === 'upload') {
    await this.jobRepository.queue({ name: JobName.AssetDetectDuplicates, data: item.data });
  }
  break;
}
```

这是一个**条件触发**的后续作业，只有当智能搜索是由上传流程触发时（`source === 'upload'`），才会排队重复检测任务。

---

## 5. 重复检测作业流程

**文件路径**: `server/src/services/duplicate.service.ts:301-409`

### 5.1 作业信息

| 属性 | 值 |
|------|-----|
| 作业名称 | `JobName.AssetDetectDuplicates` |
| 队列名称 | `QueueName.DuplicateDetection` |
| 触发时机 | SmartSearch 作业完成后，且 source === 'upload' |

### 5.2 批量队列触发作业

存在一个批量触发作业 `AssetDetectDuplicatesQueueAll`，用于全量扫描：

```typescript
@OnJob({ name: JobName.AssetDetectDuplicatesQueueAll, queue: QueueName.DuplicateDetection })
async handleQueueSearchDuplicates({ force }) {
  // 检查重复检测开关
  if (!isDuplicateDetectionEnabled(machineLearning)) {
    return JobStatus.Skipped;
  }

  // 流式遍历需要检测的资产，批量排队
  for await (const asset of this.assetJobRepository.streamForSearchDuplicates(force)) {
    jobs.push({ name: JobName.AssetDetectDuplicates, data: { id: asset.id } });
    // 批量提交...
  }
}
```

### 5.3 单个重复检测作业处理流程

```
DuplicateDetection 队列接收任务 (AssetDetectDuplicates)
    ↓
前置检查，任一条件满足则 Skipped:
  ├─ 重复检测配置未启用 → Skipped
  ├─ 资产不存在 → Failed
  ├─ 资产是堆栈成员 → Skipped
  ├─ 资产可见性为 Hidden → Skipped
  ├─ 资产可见性为 Locked → Skipped
  └─ 缺少 CLIP 特征 embedding → Failed
    ↓
调用 duplicateRepository.search() 进行相似度搜索:
  ├─ 参数: assetId, embedding, maxDistance, type, userIds
  └─ 返回: 相似资产列表 (按相似度排序)
    ↓
┌─ 找到重复资产？─┐
│  Yes            │  No (但 asset.duplicateId 存在)
│    ↓            │    ↓
│  updateDuplicates()│  清除 duplicateId 关联
│  ├─ 合并现有重复组 │
│  └─ 更新所有相关资产的 duplicateId
│    ↓            │
│  更新所有关联资产的 duplicatesDetectedAt 时间戳
│    ↓
│  任务完成 → JobStatus.Success
└─────────────────┘
```

### 5.4 重复组合并逻辑

```typescript
private async updateDuplicates(asset, duplicateAssets): Promise<string[]> {
  // 1. 收集所有命中的 duplicateId
  const duplicateIds = [...new Set(duplicateAssets.filter(a => a.duplicateId).map(a => a.duplicateId))];

  // 2. 确定目标重复组 ID
  const targetDuplicateId = asset.duplicateId ?? duplicateIds.shift() ?? randomUUID();

  // 3. 收集需要更新的资产 ID（排除已在目标组中的）
  const assetIdsToUpdate = duplicateAssets
    .filter(a => a.duplicateId !== targetDuplicateId)
    .map(a => a.assetId);
  assetIdsToUpdate.push(asset.id);

  // 4. 合并重复组：将 sourceIds 合并到 targetId，关联所有 assetIdsToUpdate
  await this.duplicateRepository.merge({
    targetId: targetDuplicateId,
    assetIds: assetIdsToUpdate,
    sourceIds: duplicateIds,
  });

  return assetIdsToUpdate;
}
```

---

## 6. 三级重复检测机制总览

| 检测层级 | 触发位置 | 检测方式 | 失败后行为 |
|---------|--------|---------|---------|
| **第一级**：前置短路 | Interceptor 层 | HTTP Header checksum 预查 | 直接返回，无文件写入 |
| **第二级**：约束兜底 | Service 层 | 数据库唯一索引 | 回滚事务、删除已上传文件、返回重复 ID |
| **第三级**：相似度检测 | 后台作业层 | CLIP embedding 向量相似度 | 标记重复组、支持后续去重操作 |

---

## 7. 核心作业队列与任务

### 7.1 完整作业列表（更新版）

| 作业名称 (JobName) | 队列 (QueueName) | 职责 | 触发源 |
|-------------------|-----------------|------|--------|
| `AssetExtractMetadata` | `MetadataExtraction` | 提取 EXIF、地理编码、标签、人脸等 | 资产创建 |
| `AssetGenerateThumbnails` | `ThumbnailGeneration` | 生成缩略图、预览图、WebP 格式 | 元数据提取后 / 模板迁移 |
| `SmartSearch` | `SmartSearch` | CLIP 图像特征提取 | 缩略图生成完成后 |
| `AssetDetectDuplicates` | `DuplicateDetection` | 基于 embedding 的相似度重复检测 | SmartSearch 完成后（source=upload） |
| `AssetDetectFaces` | `FacialRecognition` | 人脸检测与识别 | 缩略图生成完成后 |
| `Ocr` | `Ocr` | 图像文字识别 | 缩略图生成完成后 |
| `AssetEncodeVideo` | `VideoConversion` | 视频转码为兼容格式 | 缩略图生成完成后（仅视频） |
| `FileDelete` | `BackgroundTask` | 文件清理（上传失败/重复时删除临时文件） | handleUploadError 异常处理 |
| `SidecarCheck` | `Sidecar` | 检查 sidecar 文件变化 | 手动触发 / 定时 |
| `SidecarWrite` | `Sidecar` | 写入元数据到 XMP sidecar | 标签更新 / 重复解决后 |
| `AssetDelete` | `BackgroundTask` | 资产删除清理 | 手动删除 / 自动清理 |
| `PersonGenerateThumbnail` | `ThumbnailGeneration` | 生成人物头像 | 人脸检测完成后 |

### 7.2 队列并发控制

**文件路径**: `job.repository.ts:108-116`

```typescript
setConcurrency(queueName: QueueName, concurrency: number) {
  const worker = this.workers[queueName];
  worker.concurrency = concurrency;
}
```

并发配置通过系统配置管理，初始化于 `ConfigInit` 事件。

---

## 8. 作业仓库 (JobRepository) 详解

**文件路径**: `server/src/repositories/job.repository.ts`

### 8.1 核心功能

| 方法 | 功能 |
|------|------|
| `setup()` | 发现所有 `@OnJob()` 装饰的处理器，建立作业映射 |
| `startWorkers()` | 为每个 QueueName 启动 BullMQ Worker |
| `queue()` / `queueAll()` | 单个/批量排队作业 |
| `run()` | 执行具体作业处理器 |
| `pause()` / `resume()` | 暂停/恢复队列 |
| `getJobCounts()` | 获取队列统计 |
| `waitForQueueCompletion()` | 等待队列处理完成 (测试用) |

### 8.2 作业发现机制

使用 `@OnJob()` 装饰器标记作业处理器：
```typescript
// decorator 定义 (src/decorators/index.ts)
export const OnJob = (config: JobConfig) => 
  SetMetadata(MetadataKey.JobConfig, config);

// 使用示例
@OnJob({ name: JobName.AssetExtractMetadata, queue: QueueName.MetadataExtraction })
async handleMetadataExtraction(data: JobOf<JobName.AssetExtractMetadata>) { ... }
```

### 8.3 作业选项 (JobOptions)

部分作业有特殊排队选项：

| 作业 | 选项 | 说明 |
|------|------|------|
| `NotifyAlbumUpdate` | `jobId: {id}/{recipientId}`, `delay` | 去重 + 延迟通知 |
| `StorageTemplateMigrationSingle` | `jobId: asset.id` | 防止重复迁移 |
| `PersonGenerateThumbnail` | `priority: 1` | 高优先级 |
| `FacialRecognitionQueueAll` | `jobId: JobName.XXX` | 单例作业 |

---

## 9. 完整上传到作业分发链路图（更新版）

```
客户端上传请求 (可选携带 x-immich-checksum header)
    │
    ▼
[HTTP 层]
    ├─ AuthGuard → 认证用户
    │
    ├─ AssetUploadInterceptor
    │   └─ 预检查 checksum 是否存在
    │       ├─ █ 存在 ──> 【短路分支一】
    │       │           直接返回 { status: DUPLICATE, id }
    │       │           流程结束（无文件写入、无作业排队）
    │       └─ 不存在 ──> 继续执行
    │
    └─ FileUploadInterceptor → 接收文件，计算 SHA1
    │
    ▼
[Service 层: AssetMediaService.uploadAsset()
    │
    ├─ 检查用户配额
    ├─ 处理 Live Photo 关联
    │
    └─ 调用 create() 创建资产
    │   ├─ assetRepository.create() 执行 INSERT
    │   │   └─ █ checksum 约束冲突？
    │   │       ├─ Yes ──> 【短路分支二】
    │   │       │       抛出异常 → handleUploadError()
    │   │       │           ├─ 排队 FileDelete 清理文件
    │   │       │           ├─ 查询已有资产 ID
    │   │       │           ├─ 加入共享链接（如需要）
    │   │       │           └─ 返回 { status: DUPLICATE, id }
    │   │       │           流程结束（无后续处理任务排队）
    │   │       └─ No ──> 数据库插入成功，继续执行
    │   │
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
    ▼  注: 下一阶段由 thumbnail generation 触发 (非 metadata 事件直接触发)
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
    │           │   ├─ SmartSearch (source: 'upload')
    │           │   ├─ AssetDetectFaces
    │           │   ├─ Ocr
    │           │   └─ (视频) AssetEncodeVideo
    │           │
    │           └─ WebSocket 通知 (如可见)
    │               ├─ on_upload_success
    │               └─ AssetUploadReadyV2
    │
    ├─ QueueName.SmartSearch
    │   └─ 提取 CLIP 图像特征 embedding
    │       └─ ✅ onDone() 条件触发: source === 'upload'
    │           └─ 排队: AssetDetectDuplicates
    │
    ├─ QueueName.DuplicateDetection
    │   └─ AssetDetectDuplicates
    │       ├─ 前置检查 (可见性、是否在堆栈中、embedding 存在)
    │       ├─ 相似度搜索 (基于 CLIP embedding)
    │       ├─ 如找到重复，合并到重复组或创建新组
    │       ├─ 更新 assets.duplicateId 关联
    │       └─ 更新 duplicatesDetectedAt 时间戳
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

## 10. 关键设计模式

### 10.1 事件驱动架构

- 使用 `EventRepository` 作为事件总线
- `@OnEvent()` 装饰器订阅事件
- 作业完成后通过事件触发后续作业链

### 10.2 装饰器驱动的作业注册

通过 `@OnJob()` 装饰器自动发现和注册作业处理器，无需手动注册。

### 10.3 批量处理优化

- `queueAll()` 支持批量排队，使用 `addBulk()` 提高性能
- 作业按队列分组，统一批量提交

### 10.4 幂等性设计

- 关键作业使用 `jobId` 去重（如相册通知、模板迁移）
- 重复检测基于 SHA1 校验和（上传前）+ CLIP embedding（上传后）双重保障
- 数据库唯一约束作为最后防线

### 10.5 短路优化模式

在上传流程入口处通过 checksum 快速检测重复，避免不必要的文件传输和处理，这是典型的**失败快速 (Fail Fast)** 设计模式。

- **双重短路防护**：
  1. 网络层面：Interceptor 预检查，零成本快速失败
  2. 数据层面：数据库唯一约束兜底，确保数据一致性

---

## 11. 错误处理与重试

### 11.1 作业执行异常

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

### 11.2 视频转码降级策略

**文件路径**: `media.service.ts:616-642`

视频转码失败时的降级流程：
1. 尝试硬件加速编码 + 软件解码
2. 尝试纯软件编码
3. 仍失败则作业失败，由 BullMQ 重试机制处理

### 11.3 重复检测降级处理

**文件路径**: `duplicate.service.ts:328-383`

重复检测作业在以下情况会优雅降级：
- 重复检测功能未启用 → `Skipped`
- 资产在堆栈中（避免重复检测同一组图像）→ `Skipped`
- 资产可见性为 Hidden/Locked（隐私保护）→ `Skipped`
- 缺少 embedding（SmartSearch 尚未完成）→ `Failed`（会重试）

---

## 12. 性能考虑点

### 12.1 并发控制

- 每个队列独立配置并发数
- 元数据提取、缩略图生成、人脸识别等高 CPU 作业独立队列

### 12.2 批量操作

- 大规模作业（如库扫描）使用流式处理 + 批量排队
- 分页大小: `JOBS_ASSET_PAGINATION_SIZE`

### 12.3 优先级

- 人物缩略图生成使用高优先级 (`priority: 1`)
- 上传流程中的作业按自然顺序执行

### 12.4 重复检测性能优化

- 相似度搜索使用向量数据库索引加速
- 跳过 Hidden/Locked 资产，减少无效计算
- 只比较同一用户、同一类型（图片/视频）的资产
- 两级短路检测优先，避免 99% 以上重复文件到达第三级计算

---

## 13. 涉及的主要文件（更新版）

| 文件 | 主要职责 |
|------|---------|
| `server/src/controllers/asset-media.controller.ts` | 上传 API 端点 |
| `server/src/controllers/asset.controller.ts` | 资产管理 API |
| `server/src/services/asset-media.service.ts` | 资产上传核心逻辑、错误处理、短路兜底 |
| `server/src/services/metadata.service.ts` | 元数据提取作业 |
| `server/src/services/media.service.ts` | 缩略图、视频处理作业 |
| `server/src/services/job.service.ts` | 作业生命周期管理、链式触发 |
| `server/src/services/duplicate.service.ts` | 重复检测作业与重复组管理 |
| `server/src/repositories/job.repository.ts` | 队列与作业管理 |
| `server/src/middleware/asset-upload.interceptor.ts` | 上传前重复检测拦截器（第一级短路） |
| `server/src/middleware/file-upload.interceptor.ts` | 文件上传处理 |
| `server/src/utils/database.ts` | 数据库约束定义与检测函数 |

---

## 总结

整个上传到作业分发流程采用了**事件驱动 + 队列链式处理**的架构设计，包含**三层重复防护机制**：

### 两大核心流程分支 + 三级重复防护

#### 1. 两大短路分支（快速路径）

| 分支 | 触发层 | 触发条件 | 资源消耗 | 是否进入任务链 |
|-----|--------|---------|---------|---------------|
| **前置 checksum 短路** | Interceptor 层 | 客户端传 header 且 DB 命中 | 极低（仅 1 次 SELECT） | ❌ 不进入（在 Service 调用前返回） |
| **约束冲突兜底** | Service 层 | 数据库唯一索引冲突 | 中等（文件 I/O + 事务回滚） | ❌ 不进入（异常在排队前抛出） |

#### 2. 完整作业链分支（正常路径）

仅当两级短路都未命中时，进入完整处理链：
```
AssetExtractMetadata → AssetGenerateThumbnails → SmartSearch → AssetDetectDuplicates
并行分支：人脸识别、OCR、视频转码
```

### 关键设计要点

1. **分层处理**: HTTP 层 → Service 层 → 队列层
2. **条件链式触发**: 作业间触发关系带有条件（如 `source === 'upload'`）
3. **三重重复检测**: 上传前（Header checksum）→ 上传后（DB 约束）→ 后台（CLIP 相似度）
4. **可扩展性**: 新增处理步骤只需添加作业处理器和 `onDone()` 中的链接逻辑
5. **可靠性**: BullMQ 提供持久化、重试、失败管理
6. **失败快速**: 两级短路优化，在流程最前端拦截 99% 重复上传

该设计确保了上传流程的高效性和可扩展性，能够支持大规模媒体文件处理，同时通过多层次短路优化大幅提升了重复上传场景的性能。
