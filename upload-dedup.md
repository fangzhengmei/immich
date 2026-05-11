# Immich 上传去重机制技术报告

## 概述

Immich 实现了一套完整的上传去重机制，通过**分块指纹计算**、**上传前预检**、**数据库唯一约束**和**断点续传**四层保障，确保大量照片上传时不会重复存储相同文件。

---

## 一、分块切片 (Chunk Processing)

### 1.1 服务端流式分块处理

**文件位置**: `server/src/middleware/file-upload.interceptor.ts`

```typescript
private handleFile(request: AuthRequest, file: Express.Multer.File, callback: Callback<Partial<ImmichFile>>) {
  (file as ImmichMulterFile).uuid = randomUUID();

  const uploadRequest = asUploadRequest(request, file);
  const path = join(
    this.assetService.getUploadFolder(uploadRequest),
    this.assetService.getUploadFilename(uploadRequest),
  );

  const writeStream = this.storageRepository.createWriteStream(path);
  const hash = file.fieldname === UploadFieldName.ASSET_DATA ? createHash('sha1') : null;

  let size = 0;

  // 流式分块处理
  file.stream.on('data', (chunk) => {
    hash?.update(chunk);  // 实时更新指纹
    size += chunk.length; // 累计文件大小
  });

  pipeline(file.stream, writeStream, (error) => {
    if (error) {
      hash?.destroy();
      return callback(error);
    }
    callback(null, {
      path,
      size,
      checksum: hash?.digest(), // 最终指纹
    });
  });
}
```

**核心特点**:
- **流式处理**: 不将整个文件加载到内存，而是通过 `stream` 分块处理
- **实时指纹计算**: 在 `data` 事件中逐块更新 SHA1 哈希
- **内存高效**: 大文件上传时内存占用稳定
- **错误处理**: 上传中断时及时销毁哈希对象释放资源

### 1.2 Web 端分块计算

**文件位置**: `web/src/lib/utils/file-uploader.ts`

```typescript
function hashFile(file: File): Promise<string> {
  return new Promise<string>((resolve, reject) => {
    const worker = new Worker(new URL('$lib/workers/hash-file.ts', import.meta.url), { type: 'module' });

    worker.addEventListener('message', ({ data }: MessageEvent<{ result?: string; error?: string }>) => {
      worker.terminate();
      data.error ? reject(new Error(data.error)) : resolve(data.result!);
    });

    worker.postMessage(file);
  });
}
```

**设计说明**:
- 使用 Web Worker 在后台线程计算文件指纹，避免阻塞 UI
- 支持大文件的分块流式计算
- 通过消息传递返回计算结果

---

## 二、指纹计算 (Fingerprint Calculation)

### 2.1 算法选择

Immich 使用 **SHA1** 作为文件指纹算法：

| 层面 | 实现位置 | 实现方式 |
|------|---------|---------|
| 服务端 | `crypto.repository.ts` | Node.js `crypto.createHash('sha1')` |
| Web端 | `hash-file.ts` Worker | Web Crypto API |
| 移动端 | 本地存储 | 原生平台哈希计算 |

### 2.2 服务端指纹实现

**文件位置**: `server/src/repositories/crypto.repository.ts`

```typescript
@Injectable()
export class CryptoRepository {
  // 内存数据哈希
  hashSha1(value: string | Buffer): Buffer {
    return createHash('sha1').update(value).digest();
  }

  // 流式文件哈希（支持大文件）
  hashFile(filepath: string | Buffer): Promise<Buffer> {
    return new Promise<Buffer>((resolve, reject) => {
      const hash = createHash('sha1');
      const stream = createReadStream(filepath);
      stream.on('error', (error) => reject(error));
      stream.on('data', (chunk) => hash.update(chunk));
      stream.on('end', () => resolve(hash.digest()));
    });
  }
}
```

### 2.3 指纹格式转换

**文件位置**: `server/src/utils/request.ts`

```typescript
export const fromChecksum = (checksum: string): Buffer => {
  // 支持两种格式：base64 (28字符) 或 hex (40字符)
  return Buffer.from(checksum, checksum.length === 28 ? 'base64' : 'hex');
};
```

**格式说明**:
- **Base64 格式**: 28 字符，用于 HTTP Header 传输
- **Hex 格式**: 40 字符，用于数据库存储和 API 响应
- 服务端自动识别并转换两种格式

### 2.4 移动端本地指纹存储

**文件位置**: `mobile/lib/infrastructure/repositories/backup.repository.dart`

```dart
Future<List<LocalAsset>> getCandidates(String userId, {bool onlyHashed = true}) async {
  final query = _db.localAssetEntity.select()
    ..where(
      (lae) =>
          // 通过 checksum 关联远程资产表判断是否已上传
          notExistsQuery(
            _db.remoteAssetEntity.selectOnly()
              ..addColumns([_db.remoteAssetEntity.checksum])
              ..where(
                _db.remoteAssetEntity.checksum.equalsExp(lae.checksum) &
                _db.remoteAssetEntity.ownerId.equals(userId),
              ),
          ),
    );

  if (onlyHashed) {
    query.where((lae) => lae.checksum.isNotNull()); // 只选择已计算指纹的文件
  }

  return query.map((localAsset) => localAsset.toDto()).get();
}
```

**设计优势**:
- 本地 SQLite 数据库缓存文件指纹
- 避免重复计算相同文件的指纹
- 上传前本地即可判断是否重复，节省网络请求

---

## 三、去重判定流程 (Duplicate Detection)

### 3.1 四层去重机制

```
┌─────────────────────────────────────────────────────────────┐
│                     上传去重四层保障                           │
├─────────────────────────────────────────────────────────────┤
│  1. 移动端本地预检                                           │
│     └─ 本地数据库对比 checksum，不上传已存在文件                │
├─────────────────────────────────────────────────────────────┤
│  2. HTTP Header 预检拦截 (AssetUploadInterceptor)           │
│     └─ 上传前在拦截器中检查是否已存在，直接返回重复状态          │
├─────────────────────────────────────────────────────────────┤
│  3. 批量上传预检 (bulkUploadCheck)                          │
│     └─ Web端批量上传前先发送指纹列表，服务端返回重复结果        │
├─────────────────────────────────────────────────────────────┤
│  4. 数据库唯一约束                                           │
│     └─ (ownerId, libraryId, checksum) 联合唯一索引            │
└─────────────────────────────────────────────────────────────┘
```

### 3.2 流程一：HTTP Header 预检拦截

**文件位置**: `server/src/middleware/asset-upload.interceptor.ts`

```typescript
@Injectable()
export class AssetUploadInterceptor implements NestInterceptor {
  async intercept(context: ExecutionContext, next: CallHandler<any>) {
    const req = context.switchToHttp().getRequest<AuthenticatedRequest>();
    const res = context.switchToHttp().getResponse<Response<AssetMediaResponseDto>>();

    // 从请求头获取客户端预计算的指纹
    const checksum = fromMaybeArray(req.headers[ImmichHeader.Checksum]);
    const response = await this.service.getUploadAssetIdByChecksum(req.user, checksum);

    if (response) {
      res.status(200);
      // 文件已存在，直接返回重复状态，不执行后续上传
      return of({ status: AssetMediaStatus.DUPLICATE, id: response.id });
    }

    return next.handle();
  }
}
```

**工作原理**:
1. 客户端在上传前计算文件 SHA1 指纹
2. 通过 HTTP Header `x-immich-checksum` 发送到服务端
3. 拦截器在 Multer 处理文件前先查询数据库
4. 如已存在则直接返回 `DUPLICATE` 状态，跳过文件上传

### 3.3 流程二：批量上传预检

**文件位置**: `server/src/services/asset-media.service.ts`

```typescript
async bulkUploadCheck(auth: AuthDto, dto: AssetBulkUploadCheckDto): Promise<AssetBulkUploadCheckResponseDto> {
  const checksums: Buffer[] = dto.assets.map((asset) => fromChecksum(asset.checksum));
  const results = await this.assetRepository.getByChecksums(auth.user.id, checksums);

  // 构建 checksum -> 资产信息 的映射
  const checksumMap: Record<string, { id: string; isTrashed: boolean }> = {};
  for (const { id, deletedAt, checksum } of results) {
    checksumMap[checksum.toString('hex')] = { id, isTrashed: !!deletedAt };
  }

  return {
    results: dto.assets.map(({ id, checksum }) => {
      const duplicate = checksumMap[fromChecksum(checksum).toString('hex')];
      if (duplicate) {
        return {
          id,
          action: AssetUploadAction.REJECT, // 标记为拒绝上传
          reason: AssetRejectReason.DUPLICATE,
          assetId: duplicate.id,
          isTrashed: duplicate.isTrashed,
        };
      }
      return {
        id,
        action: AssetUploadAction.ACCEPT, // 标记为允许上传
      };
    }),
  };
}
```

**Web端调用示例**:
```typescript
// file-uploader.ts
const checksum = await hashFile(assetFile);
const { results: [checkUploadResult] } = await checkBulkUpload({
  assetBulkUploadCheckDto: { assets: [{ id: assetFile.name, checksum }] }
});

if (checkUploadResult.action === AssetUploadAction.Reject) {
  // 文件重复，直接标记为重复状态，不执行上传
  responseData = {
    status: AssetMediaStatus.Duplicate,
    id: checkUploadResult.assetId,
    isTrashed: checkUploadResult.isTrashed,
  };
}
```

### 3.4 流程三：数据库唯一约束兜底

**文件位置**: `server/src/schema/tables/asset.table.ts`

```typescript
@Table('asset')
// 第一层唯一索引：无库时的用户级唯一约束
@Index({
  name: 'IDX_asset_owner_checksum_unique',
  columns: ['ownerId', 'checksum'],
  unique: true,
  where: '"libraryId" IS NULL',
})
// 第二层唯一索引：有库时的库级唯一约束
@Index({
  columns: ['ownerId', 'libraryId', 'checksum'],
  unique: true,
  where: '"libraryId" IS NOT NULL',
})
export class AssetTable {
  @Column({ type: 'bytea', index: true })
  checksum!: Buffer; // SHA1 二进制存储

  @Column({ enum: asset_checksum_algorithm_enum })
  checksumAlgorithm!: ChecksumAlgorithm;
}
```

**错误处理与重试**:
```typescript
private async handleUploadError(error: any, auth: AuthDto, file: UploadFile) {
  // 捕获唯一约束违反异常
  if (isAssetChecksumConstraint(error)) {
    const duplicateId = await this.assetRepository.getUploadAssetIdByChecksum(auth.user.id, file.checksum);
    if (duplicateId) {
      return { status: AssetMediaStatus.DUPLICATE, id: duplicateId };
    }
  }
  throw error;
}
```

---

## 四、断点续传与上传恢复 (Resume Upload)

### 4.1 移动端后台上传架构

Immich 移动端使用 **background_downloader** 库实现 iOS 后台上传和 Android 前台上传。

**文件位置**: `mobile/lib/services/background_upload.service.dart`

```dart
class BackgroundUploadService {
  BackgroundUploadService(this._uploadRepository, ...) {
    // 注册回调监听上传状态变化
    _uploadRepository.onUploadStatus = _onUploadCallback;
    _uploadRepository.onTaskProgress = _onTaskProgressCallback;
  }

  void _onUploadCallback(TaskStatusUpdate update) {
    switch (update.status) {
      case TaskStatus.complete:
        // 上传完成，处理 Live Photo 关联
        unawaited(_handleLivePhoto(update));
        break;
      case TaskStatus.failed:
        // 上传失败，记录日志，可重试
        _logger.warning('Upload failed: ${update.exception}');
        break;
      case TaskStatus.paused:
        // 上传暂停，可后续恢复
        break;
      default:
        break;
    }
  }

  // 构建可恢复的上传任务
  Future<UploadTask> buildUploadTask(File file, {
    required String group,
    required DateTime createdAt,
    required DateTime modifiedAt,
    String? deviceAssetId,
    // ... 其他参数
  }) async {
    return UploadTask(
      taskId: deviceAssetId,
      displayName: originalFileName ?? filename,
      httpRequestMethod: 'POST',
      url: url,
      headers: headers,
      filename: filename,
      fields: fieldsMap,
      baseDirectory: baseDirectory,
      directory: directory,
      fileField: 'assetData',
      group: group,
      requiresWiFi: requiresWiFi,
      priority: priority ?? 5,
      updates: Updates.statusAndProgress, // 启用状态和进度更新
      retries: 3, // 自动重试 3 次
    );
  }
}
```

### 4.2 并发上传与 Worker Pool

**文件位置**: `mobile/lib/services/foreground_upload.service.dart`

```dart
// 泛型 Worker Pool 实现，支持并发上传
Future<void> _executeWithWorkerPool<T>({
  required List<T> items,
  required Completer<void>? cancelToken,
  required Future<void> Function(T item) processItem,
  bool Function(T item)? shouldSkip,
  int concurrentWorkers = 2, // 默认 2 个并发
}) async {
  await _storageRepository.clearCache();
  shouldAbortUpload = false;

  int currentIndex = 0;

  Future<void> worker() async {
    while (true) {
      if (shouldAbortUpload || (cancelToken != null && cancelToken.isCompleted)) {
        break;
      }

      final index = currentIndex;
      if (index >= items.length) {
        break;
      }
      currentIndex++;

      final item = items[index];

      if (shouldSkip?.call(item) ?? false) {
        continue;
      }

      await processItem(item);
    }
  }

  // 启动 N 个 worker 并发处理
  final workerFutures = <Future<void>>[];
  for (int i = 0; i < concurrentWorkers; i++) {
    workerFutures.add(worker());
  }

  await Future.wait(workerFutures);
}
```

### 4.3 上传进度追踪与断点恢复

**文件位置**: `mobile/lib/repositories/upload.repository.dart`

```dart
class UploadRepository {
  void Function(TaskStatusUpdate)? onUploadStatus;
  void Function(TaskProgressUpdate)? onTaskProgress;

  UploadRepository() {
    FileDownloader().registerCallbacks(
      group: kBackupGroup,
      taskStatusCallback: (update) => onUploadStatus?.call(update),
      taskProgressCallback: (update) => onTaskProgress?.call(update),
    );
    // 注册其他分组的回调...
  }

  // 获取任务状态用于断点恢复
  Future<List<Task>> getActiveTasks(String group) {
    return FileDownloader().allTasks(group: group);
  }

  // 重置并清除数据库记录，可用于重新开始上传
  Future<int> reset(String group) {
    return FileDownloader().reset(group: group);
  }
}
```

### 4.4 单个文件上传与进度回调

```dart
Future<UploadResult> uploadFile({
  required File file,
  required String originalFileName,
  required Map<String, String> fields,
  required Completer<void>? cancelToken,
  void Function(int bytes, int totalBytes)? onProgress,
  required String logContext,
}) async {
  final baseRequest = ProgressMultipartRequest(
    'POST',
    Uri.parse('$savedEndpoint/assets'),
    abortTrigger: cancelToken?.future,
    onProgress: onProgress,
  );

  try {
    final fileStream = file.openRead();
    final assetRawUploadData = MultipartFile("assetData", fileStream, file.lengthSync(), filename: originalFileName);

    baseRequest.fields.addAll(fields);
    baseRequest.files.add(assetRawUploadData);

    final response = await NetworkRepository.client.send(baseRequest);
    // ... 处理响应
  } on RequestAbortedException {
    logger.warning("Upload $logContext was cancelled");
    return UploadResult.cancelled();
  } catch (error, stackTrace) {
    logger.warning("Error uploading $logContext: ${error.toString()}: $stackTrace");
    return UploadResult.error(errorMessage: error.toString());
  }
}
```

---

## 五、完整上传流程时序图

```
┌─────────┐     ┌─────────────┐     ┌─────────────┐     ┌──────────┐
│ 客户端   │     │ AssetUpload │     │ AssetMedia  │     │ 数据库    │
│ (Mobile) │     │ Interceptor │     │  Service    │     │ (PG)     │
└────┬────┘     └──────┬──────┘     └──────┬──────┘     └────┬─────┘
     │                 │                    │                  │
     │ 1. 本地计算 SHA1 │                    │                  │
     │    指纹并缓存    │                    │                  │
     ├─────────────────┘                    │                  │
     │                                      │                  │
     │ 2. 对比本地 checksum                 │                  │
     │    如已存在则跳过                    │                  │
     │                                      │                  │
     │ 3. 上传请求                         │                  │
     │    Header: x-immich-checksum        │                  │
     ├────────────────────────────────────>│                  │
     │                                      │                  │
     │                                      │ 4. SELECT by checksum + userId
     │                                      │─────────────────>│
     │                                      │                  │
     │                                      │ 5. 返回查询结果 │
     │                                      │<─────────────────┤
     │                                      │                  │
     │    6. 如已存在返回 DUPLICATE        │                  │
     │<────────────────────────────────────┤                  │
     │                                      │                  │
     │ 7. 如不存在，继续上传文件内容        │                  │
     ├────────────────────────────────────────────────────────>│
     │                                      │                  │
     │                                      │ 8. FileUploadInterceptor 流式处理
     │                                      │    - 分块计算 SHA1
     │                                      │    - 写入临时文件
     │                                      │                  │
     │                                      │ 9. 写入数据库    │
     │                                      │    (触发唯一约束)│
     │                                      │─────────────────>│
     │                                      │                  │
     │    10. 返回上传结果                  │                  │
     │<────────────────────────────────────────────────────────┤
     │                                      │                  │
     │ 11. 更新本地 remote_asset 表        │                  │
     │    记录已上传状态                    │                  │
     └─────────────────────────────────────────────────────────┘
```

---

## 六、关键技术总结

| 技术点 | 实现方案 | 优势 |
|-------|---------|------|
| **指纹算法** | SHA1 哈希 | 成熟稳定、碰撞概率极低 |
| **分块处理** | Node.js Stream / Web Worker | 内存高效、支持大文件 |
| **预检机制** | HTTP Header + 批量 API | 显著节省带宽和时间 |
| **唯一约束** | PostgreSQL 联合唯一索引 | 数据库层面最终保障 |
| **断点续传** | background_downloader | iOS 后台支持、Android 前台支持 |
| **并发控制** | Worker Pool 模式 | 控制并发数、避免网络拥塞 |
| **本地缓存** | SQLite 存储指纹 | 避免重复计算、快速预检 |

---

## 七、代码溯源索引

| 功能模块 | 文件路径 |
|---------|---------|
| 服务端文件上传拦截器 | `server/src/middleware/file-upload.interceptor.ts` |
| 上传前指纹预检拦截器 | `server/src/middleware/asset-upload.interceptor.ts` |
| 资产媒体服务（去重逻辑） | `server/src/services/asset-media.service.ts` |
| 密码学仓库（指纹计算） | `server/src/repositories/crypto.repository.ts` |
| 资产表 Schema 定义 | `server/src/schema/tables/asset.table.ts` |
| 指纹格式转换工具 | `server/src/utils/request.ts` |
| Web 端文件上传器 | `web/src/lib/utils/file-uploader.ts` |
| 移动端后台上传服务 | `mobile/lib/services/background_upload.service.dart` |
| 移动端前台上传服务 | `mobile/lib/services/foreground_upload.service.dart` |
| 移动端上传仓库 | `mobile/lib/repositories/upload.repository.dart` |
| 移动端备份仓库（本地去重） | `mobile/lib/infrastructure/repositories/backup.repository.dart` |
