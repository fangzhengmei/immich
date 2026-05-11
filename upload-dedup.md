# Immich 移动端大批量上传去重与断点续传技术报告

## 概述

Immich 移动端实现了多层级的上传去重机制，通过本地候选筛选、平台原生哈希、服务端预检确保重复文件不上传。关于后台断点续传能力，**iOS 与 Android 的实现存在显著差异**，本报告基于实际代码证据进行严谨表述。

---

## 一、候选筛选去重 (Candidate Filtering)

### 1.1 本地数据库级去重实现

**文件位置**：`mobile/lib/infrastructure/repositories/backup.repository.dart:84-116`

```dart
Future<List<LocalAsset>> getCandidates(String userId, {bool onlyHashed = true}) async {
  final selectedAlbumIds = _db.localAlbumEntity.selectOnly(distinct: true)
    ..addColumns([_db.localAlbumEntity.id])
    ..where(_db.localAlbumEntity.backupSelection.equalsValue(BackupSelection.selected));

  final query = _db.localAssetEntity.select()
    ..where(
      (lae) =>
          // 条件1: 资产属于已选择的备份相册
          existsQuery(
            _db.localAlbumAssetEntity.selectOnly()
              ..addColumns([_db.localAlbumAssetEntity.assetId])
              ..where(
                _db.localAlbumAssetEntity.albumId.isInQuery(selectedAlbumIds) &
                    _db.localAlbumAssetEntity.assetId.equalsExp(lae.id),
              ),
          ) &
          // 条件2: 核心去重逻辑 - LEFT JOIN 远程资产表
          // 通过 checksum + owner_id 双字段匹配判断是否已上传
          notExistsQuery(
            _db.remoteAssetEntity.selectOnly()
              ..addColumns([_db.remoteAssetEntity.checksum])
              ..where(
                _db.remoteAssetEntity.checksum.equalsExp(lae.checksum) &
                _db.remoteAssetEntity.ownerId.equals(userId),
              ),
          ) &
          // 条件3: 不在排除的相册中
          lae.id.isNotInQuery(_getExcludedSubquery()),
    )
    ..orderBy([(localAsset) => OrderingTerm.desc(localAsset.createdAt)]);

  if (onlyHashed) {
    query.where((lae) => lae.checksum.isNotNull()); // 条件4: 只选择已计算哈希的资产
  }

  return query.map((localAsset) => localAsset.toDto()).get();
}
```

### 1.2 查询复杂度分析与谨慎结论

**SQL 查询结构特征**：
1.  **主表**：`local_asset_entity`（本地资产表）
2.  **关联1**：`local_album_asset_entity` + `local_album_entity`（EXISTS 子查询，判断是否在选中相册）
3.  **关联2**：`remote_asset_entity`（NOT EXISTS 子查询，判断是否已上传）
4.  **关联3**：`local_album_asset_entity` + `local_album_entity`（NOT IN 子查询，判断是否在排除相册）

**复杂度评估的谨慎表述**：

> ⚠️ **基于代码证据的分析**：
>
> 该查询使用了 **3 个子查询关联**（1 个 EXISTS + 1 个 NOT EXISTS + 1 个 NOT IN）。SQLite 在执行此类关联查询时，若无恰当索引支持可能导致 `O(N*M)` 的时间复杂度。
>
> **可观察的优化点**：
> - `checksum` 字段在 `remote_asset_entity` 表有索引（`idx_remote_asset_checksum`）
> - `checksum` 字段在 `local_asset_entity` 表有索引（`idx_local_asset_checksum`）
> - 两表间存在 `UNIQUE` 约束：`(checksum, owner_id)`，加速去重匹配
>
> **性能边界**：
> - 子查询关联在资产数量超过 10,000 条时可能产生可观察的查询延迟
> - 实际性能受 SQLite 查询优化器决策影响，若无基准测试数据，**不宜作出"高效"或"O(1)"的绝对化结论**

### 1.3 统计查询（三值统计）

**文件位置**：`mobile/lib/infrastructure/repositories/backup.repository.dart:39-82`

```sql
SELECT
  COUNT(*) AS total_count,                           -- 总资产数
  COUNT(*) FILTER (WHERE lae.checksum IS NULL) AS processing_count,  -- 待计算哈希数
  COUNT(*) FILTER (WHERE rae.id IS NULL) AS remainder_count          -- 实际待上传数
FROM local_asset_entity lae
LEFT JOIN main.remote_asset_entity rae
    ON lae.checksum = rae.checksum AND rae.owner_id = ?1  -- 核心关联条件
WHERE ...
```

### 1.4 本地哈希持久化

**文件位置**：`mobile/lib/infrastructure/repositories/local_asset.repository.dart:51-65`

```dart
Future<void> updateHashes(Map<String, String> hashes) {
  if (hashes.isEmpty) return Future.value();
  return _db.batch((batch) async {
    for (final entry in hashes.entries) {
      batch.update(
        _db.localAssetEntity,
        LocalAssetEntityCompanion(checksum: Value(entry.value)),
        where: (e) => e.id.equals(entry.key),
      );
    }
  });
}
```

---

## 二、平台原生哈希实现 (Hash Calculation)

### 2.1 iOS 哈希实现

**文件位置**：`mobile/ios/Runner/Sync/MessagesImpl.swift:271-380`

```swift
import CryptoKit

private func hashAsset(_ asset: PHAsset, allowNetworkAccess: Bool) async -> HashResult? {
  return await withTaskCancellationHandler(operation: {
    guard let resource = asset.getResource() else {
      return HashResult(assetId: asset.localIdentifier, error: "Cannot get asset resource", hash: nil)
    }

    let options = PHAssetResourceRequestOptions()
    options.isNetworkAccessAllowed = allowNetworkAccess  // iCloud 下载控制

    return await withCheckedContinuation { continuation in
      var hasher = Insecure.SHA1()  // iOS 使用 CryptoKit 框架的 SHA-1

      PHAssetResourceManager.default().requestData(
        for: resource,
        options: options,
        dataReceivedHandler: { data in
          hasher.update(data: data)  // 流式分块更新哈希
        },
        completionHandler: { error in
          // ...
          HashResult(
            assetId: asset.localIdentifier,
            error: nil,
            hash: Data(hasher.finalize()).base64EncodedString()  // Base64 输出
          )
        }
      )
    }
  }, onCancel: {
    guard let requestId = requestRef.id else { return }
    PHAssetResourceManager.default().cancelDataRequest(requestId)  // 支持取消
  })
}
```

**iOS 实现确认**：
- ✅ **哈希算法**：`CryptoKit.Insecure.SHA1()`（代码第 349 行确认）
- ✅ **流式处理**：通过 `dataReceivedHandler` 分块接收数据，实时更新哈希
- ✅ **iCloud 支持**：`allowNetworkAccess` 参数控制是否允许从 iCloud 下载
- ✅ **可取消**：支持任务取消，中断哈希计算

### 2.2 Android 哈希实现

**文件位置**：`mobile/android/app/src/main/kotlin/app/alextran/immich/sync/MessagesImplBase.kt:380-444`

```kotlin
import java.security.MessageDigest
import kotlinx.coroutines.sync.Semaphore

companion object {
  private const val MAX_CONCURRENT_HASH_OPERATIONS = 16  // 最大并发数
  private val hashSemaphore = Semaphore(MAX_CONCURRENT_HASH_OPERATIONS)
  const val HASH_BUFFER_SIZE = 2 * 1024 * 1024  // 2MB 缓冲区
}

private suspend fun hashAsset(assetId: String): HashResult {
  return try {
    val assetUri = ContentUris.withAppendedId(
      MediaStore.Files.getContentUri(MediaStore.VOLUME_EXTERNAL),
      assetId.toLong()
    )

    val digest = MessageDigest.getInstance("SHA-1")  // ⭐ Android 使用 Java MessageDigest
    ctx.contentResolver.openInputStream(assetUri)?.use { inputStream ->
      var bytesRead: Int
      val buffer = ByteArray(HASH_BUFFER_SIZE)  // 2MB 分块读取
      while (inputStream.read(buffer).also { bytesRead = it } > 0) {
        currentCoroutineContext().ensureActive()
        digest.update(buffer, 0, bytesRead)
      }
    } ?: return HashResult(assetId, "Cannot open input stream for asset", null)

    val hashString = Base64.encodeToString(digest.digest(), Base64.NO_WRAP)
    HashResult(assetId, null, hashString)
  } catch (e: Exception) {
    HashResult(assetId, "Failed to hash asset: ${e.message}", null)
  }
}
```

**Android 实现确认**：
- ✅ **哈希算法**：`java.security.MessageDigest.getInstance("SHA-1")`（代码第 399 行确认）
- ✅ **缓冲区大小**：2MB（`HASH_BUFFER_SIZE = 2 * 1024 * 1024`，代码第 95 行确认）
- ✅ **并发控制**：`Semaphore` 限制最大 16 个并发哈希操作（代码第 48-49 行确认）
- ✅ **协程支持**：使用 `CoroutineScope(Dispatchers.IO)` 异步处理，支持取消

### 2.3 跨平台一致性

| 对比项 | iOS | Android | 一致性 |
|-------|-----|---------|--------|
| 算法 | SHA-1 | SHA-1 | ✅ 一致 |
| 输出格式 | Base64 | Base64.NO_WRAP | ✅ 一致 |
| 取消支持 | ✅ | ✅ | 一致 |
| 并发数 | 无限制（TaskGroup） | 16 (Semaphore) | 平台差异 |

---

## 三、上传链路与服务端预检 (Upload Pipeline & Server Check)

### 3.1 完整上传链路去重层级

```
┌────────────────────────────────────────────────────────────────────────────┐
│                    Immich 完整上传去重链路                                   │
├────────────────────────────────────────────────────────────────────────────┤
│                                                                            │
│  ┌─────────────────┐  ┌─────────────────┐  ┌─────────────────┐             │
│  │ 层级1: 本地筛选 │  │ 层级2: Header  │  │ 层级3: 数据库约束│             │
│  │ SQLite 过滤     │  │预检 (服务端拦截)│  │唯一兜底          │             │
│  └────────┬────────┘  └────────┬────────┘  └────────┬────────┘             │
│           │                     │                     │                       │
│           ▼                     ▼                     ▼                       │
│      getCandidates()      AssetUploadInterceptor  UNIQUE(checksum, owner_id)│
│                                                                            │
│  执行时机: 选择待上传时   执行时机: 上传请求到达    执行时机: 数据插入时        │
└────────────────────────────────────────────────────────────────────────────┘
```

### 3.2 层级1：本地候选过滤

**执行时机**：选择待上传资产时
**触发位置**：`backup.repository.dart:getCandidates()`
**过滤方式**：SQLite `LEFT JOIN` 查询
**代码证据**：见 1.1 节

### 3.3 层级2：HTTP Header 预检拦截

**文件位置**：`server/src/middleware/asset-upload.interceptor.ts`

```typescript
@Injectable()
export class AssetUploadInterceptor implements NestInterceptor {
  async intercept(context: ExecutionContext, next: CallHandler<any>) {
    const req = context.switchToHttp().getRequest<AuthenticatedRequest>();
    const res = context.switchToHttp().getResponse<Response<AssetMediaResponseDto>>();

    // 从 HTTP Header 读取客户端预计算的 checksum
    const checksum = fromMaybeArray(req.headers[ImmichHeader.Checksum]);
    const response = await this.service.getUploadAssetIdByChecksum(req.user, checksum);

    if (response) {
      res.status(200);
      return of({ status: AssetMediaStatus.DUPLICATE, id: response.id });
    }

    return next.handle();
  }
}
```

**执行时机**：服务端接收到上传请求时，在 Multer 解析文件之前
**触发位置**：NestJS 拦截器
**性能特点**：只传输 Header，不上传文件内容，节省带宽

### 3.4 层级3：数据库唯一约束兜底

**文件位置**：`server/src/schema/tables/asset.table.ts`

```typescript
@Table('asset')
@Index({
  name: 'IDX_asset_owner_checksum_unique',
  columns: ['ownerId', 'checksum'],
  unique: true,
  where: '"libraryId" IS NULL',
})
@Index({
  columns: ['ownerId', 'libraryId', 'checksum'],
  unique: true,
  where: '"libraryId" IS NOT NULL',
})
export class AssetTable {
  @Column({ type: 'bytea', index: true })
  checksum!: Buffer;
}
```

**执行时机**：资产插入数据库时
**作用**：最终兜底，防止并发上传导致的重复数据

---

## 四、后台断点续传实现边界与证据 (Background Upload Resume)

### 4.1 架构概述

Immich 使用 `background_downloader: ^9.5.4` 库（代码证据：`pubspec.yaml` 第 14 行）实现双平台后台上传。**但两平台实现路径存在显著差异**：

### 4.2 iOS 断点续传：URLSession 支持

**文件位置**：`mobile/lib/domain/services/background_worker.service.dart:223-224`

```dart
if (Platform.isIOS) {
  return _ref?.read(driftBackupProvider.notifier).startBackupWithURLSession(currentUser.id);
}
```

**iOS 实现证据**：
- ✅ 使用 `background_downloader` 库封装的 iOS `URLSession` 后台任务
- ✅ 任务状态持久化到 SQLite 数据库（`FileDownloader().database`）
- ✅ 支持 5 种任务状态查询：`enqueued` / `running` / `canceled` / `waitingToRetry` / `paused`
- ✅ 支持 `start()` / `reset()` 控制断点续传
- ✅ Live Photo 两阶段断点：视频上传完成后回调触发照片上传

**iOS 断点续传能力确认**：
> ✅ **确认**：iOS 端通过 URLSession 后台任务机制具备断点续传能力。App 重启后可通过 `FileDownloader().database` 恢复未完成任务。

### 4.3 Android 断点续传：实现边界声明

**文件位置**：`mobile/lib/domain/services/background_worker.service.dart:227-229`

```dart
// Android 后台执行路径
return _ref
    ?.read(foregroundUploadServiceProvider)
    .uploadCandidates(currentUser.id, _cancellationToken, useSequentialUpload: true);
```

**Android 实现代码分析**：

**Android WorkManager 入口**：`mobile/android/app/src/main/kotlin/app/alextran/immich/background/BackgroundWorker.kt:63-102`

```kotlin
override fun startWork(): ListenableFuture<Result> {
  Log.i(TAG, "Starting background upload worker")
  // ... 初始化 FlutterEngine
  engine!!.dartExecutor.executeDartEntrypoint(
    DartExecutor.DartEntrypoint(
      loader.findAppBundlePath(),
      "package:immich_mobile/domain/services/background_worker.service.dart",
      "backgroundSyncNativeEntrypoint"  // 调用 Dart 侧入口
    )
  )
  return completionHandler
}
```

**Dart 侧执行路径**：`mobile/lib/domain/services/background_worker.service.dart:227-229`

```dart
// Android 实际调用的是 foregroundUploadService，且 useSequentialUpload = true
return _ref
    ?.read(foregroundUploadServiceProvider)
    .uploadCandidates(currentUser.id, _cancellationToken, useSequentialUpload: true);
```

**顺序上传实现**：`mobile/lib/services/foreground_upload.service.dart:112-134`

```dart
/// Sequential upload - used for background isolate where concurrent HTTP clients may cause issues
Future<void> _uploadSequentially({
  required List<LocalAsset> items,
  required Completer<void> cancelToken,
  required bool hasWifi,
  required UploadCallbacks callbacks,
}) async {
  await _storageRepository.clearCache();
  shouldAbortUpload = false;

  for (final asset in items) {  // 顺序循环，单文件上传
    if (shouldAbortUpload || cancelToken.isCompleted) {
      break;
    }
    await _uploadSingleAsset(asset, cancelToken, callbacks: callbacks);
  }
}
```

**Android 断点续传能力边界声明**：

> ⚠️ **基于代码证据的边界声明**：
>
> 1. **Android 实际执行路径**：`WorkManager` 启动独立 FlutterEngine → 调用 Dart 的 `_uploadSequentially()` → 使用 `foregroundUploadService.uploadSingleAsset()` 进行**顺序单文件上传**
>
> 2. **关于 HTTP 级断点续传**：
>    - 代码中**未发现** `Range` 请求头相关实现
>    - 代码中**未发现** 服务端上传偏移量（upload offset）查询/设置逻辑
>    - `ProgressMultipartRequest` 仅实现进度追踪，不支持断点续传
>    - `uploadFile()` 方法始终创建全新的 `MultipartRequest`，无断点恢复逻辑
>
> 3. **任务级恢复能力**：
>    - ✅ WorkManager 任务在进程被杀后可重启整个上传流程
>    - ✅ 本地去重机制确保已上传成功的文件不会重复上传
>    - ❌ **单文件级别的断点续传**：代码证据不足以支撑此结论
>
> 4. **证据缺口**：
>    - 未找到 `Content-Range` 响应头处理逻辑
>    - 未找到上传进度持久化（已上传字节数存储）的代码
>    - 未找到与服务端协商上传偏移量的 API 调用

### 4.4 任务状态统计与状态回调边界

**任务状态统计**：`mobile/lib/repositories/upload.repository.dart:69-89`

```dart
Future<void> getUploadInfo() async {
  final [enqueuedTasks, runningTasks, canceledTasks, waitingTasks, pausedTasks] = await Future.wait([
    FileDownloader().database.allRecordsWithStatus(TaskStatus.enqueued, group: kBackupGroup),
    FileDownloader().database.allRecordsWithStatus(TaskStatus.running, group: kBackupGroup),
    FileDownloader().database.allRecordsWithStatus(TaskStatus.canceled, group: kBackupGroup),
    FileDownloader().database.allRecordsWithStatus(TaskStatus.waitingToRetry, group: kBackupGroup),
    FileDownloader().database.allRecordsWithStatus(TaskStatus.paused, group: kBackupGroup),
  ]);
  // ... 日志输出
}
```

**状态统计边界说明**：
- **数据源**：SQLite 数据库持久化存储（由 background_downloader 库维护）
- **适用范围**：主要用于日志输出、调试展示，不直接驱动断点续传逻辑
- **调用时机**：主动查询调用，非实时回调
- **Android 限制**：由于 Android 使用顺序上传而非任务队列，此统计功能对 Android 作用有限

**状态回调处理**：`mobile/lib/services/background_upload.service.dart:209-228`

```dart
void _handleTaskStatusUpdate(TaskStatusUpdate update) async {
  switch (update.status) {
    case TaskStatus.complete:
      unawaited(_handleLivePhoto(update));  // 仅处理 Live Photo 关联

      if (CurrentPlatform.isIOS) {
        try {
          final path = await update.task.filePath();
          await File(path).delete();  // iOS 清理临时文件
        } catch (e) {
          _logger.severe('Error deleting file path for iOS: $e');
        }
      }
      break;

    default:
      break;  // 其他状态不做任何处理！
  }
}
```

**状态回调边界说明**：

| 状态 | 是否处理 | 处理逻辑 |
|-----|---------|---------|
| `TaskStatus.complete` | ✅ 处理 | 1. 触发 Live Photo 后续上传<br>2. iOS 清理临时文件 |
| `TaskStatus.running` | ❌ **不处理** | 由库内部管理进度 |
| `TaskStatus.enqueued` | ❌ **不处理** | 由库内部管理队列 |
| `TaskStatus.waitingToRetry` | ❌ **不处理** | 由库内部管理重试 |
| `TaskStatus.paused` | ❌ **不处理** | 由用户手动控制 |
| `TaskStatus.canceled` | ❌ **不处理** | 用户取消操作 |
| `TaskStatus.failed` | ❌ **不处理** | 由库内部管理失败逻辑 |

> ⚠️ **关键发现**：应用层回调仅处理 `complete` 状态，**其他状态的恢复/重试逻辑完全依赖于 background_downloader 库的内部实现**。此架构设计使得断点续传能力高度受限于第三方库的行为。

### 4.5 Live Photo 两阶段上传

**文件位置**：`mobile/lib/services/background_upload.service.dart:230-261`

```dart
Future<void> _handleLivePhoto(TaskStatusUpdate update) async {
  try {
    if (update.task.metaData.isEmpty) return;

    final metadata = UploadTaskMetadata.fromJson(update.task.metaData);
    if (!metadata.isLivePhotos) return;  // 非 Live Photo 跳过

    if (update.responseBody == null || update.responseBody!.isEmpty) return;
    final response = jsonDecode(update.responseBody!);

    final localAsset = await _localAssetRepository.getById(metadata.localAssetId);
    if (localAsset == null) return;

    // 阶段1完成后，入队阶段2任务
    final uploadTask = await getLivePhotoUploadTask(localAsset, response['id'] as String);
    if (uploadTask == null) return;

    await enqueueTasks([uploadTask]);  // 断点：继续上传另一半
  } catch (error, stackTrace) {
    // ...
  }
}
```

**两阶段上传说明**：
| 阶段 | 内容 | 优先级 | 分组 | 断点触发 |
|-----|------|--------|------|---------|
| 阶段 1 | Motion 视频文件 | 默认 | `kBackupGroup` | 视频上传完成 |
| 阶段 2 | 照片文件 | 最高 (0) | `kBackupLivePhotoGroup` | complete 回调中入队 |

---

## 五、关键技术参数汇总

| 参数 | 值 | 代码位置 | 说明 |
|-----|----|---------|------|
| **前台并发数** | **3** | `foreground_upload.service.dart:202` | Worker Pool 默认并发数 |
| **后台批次大小** | **100** | `background_upload.service.dart:173` | iOS 每次入队任务数 |
| **哈希算法** | **SHA-1** | iOS/Android 原生代码 | 文件内容指纹，双平台确认 |
| **哈希输出格式** | **Base64** | 双平台代码 | 与服务端兼容 |
| **Android 哈希并发** | **16** | `MessagesImplBase.kt:48` | Semaphore 控制 |
| **Android 哈希缓冲区** | **2MB** | `MessagesImplBase.kt:95` | 分块读取大小 |
| **后台下载器版本** | **9.5.4** | `pubspec.yaml:14` | background_downloader |
| **回调处理状态数** | **1/7** | `background_upload.service.dart:209-228` | 仅处理 complete |
| **iOS 断点续传** | **✅ 确认** | URLSession 机制 | 任务级断点恢复 |
| **Android 单文件断点** | **⚠️ 证据不足** | - | 代码未发现 Range 请求实现 |

---

## 六、代码溯源索引

| 功能模块 | 文件路径 |
|---------|---------|
| 备份候选查询（去重核心） | `mobile/lib/infrastructure/repositories/backup.repository.dart:84-116` |
| 本地资产 Hash 更新 | `mobile/lib/infrastructure/repositories/local_asset.repository.dart:51-65` |
| iOS 原生哈希实现 | `mobile/ios/Runner/Sync/MessagesImpl.swift:271-380` |
| Android 原生哈希实现 | `mobile/android/app/src/main/kotlin/app/alextran/immich/sync/MessagesImplBase.kt:380-449` |
| 前台并发 Worker Pool | `mobile/lib/services/foreground_upload.service.dart:190-237` |
| Android 顺序上传实现 | `mobile/lib/services/foreground_upload.service.dart:112-134` |
| Android WorkManager 入口 | `mobile/android/app/src/main/kotlin/app/alextran/immich/background/BackgroundWorker.kt:63-102` |
| 后台 worker 平台分发 | `mobile/lib/domain/services/background_worker.service.dart:207-235` |
| 后台上传服务（状态回调） | `mobile/lib/services/background_upload.service.dart:209-228` |
| 上传仓库（状态管理） | `mobile/lib/repositories/upload.repository.dart:69-89` |
| 服务端预检拦截器 | `server/src/middleware/asset-upload.interceptor.ts` |
| 数据库唯一约束定义 | `server/src/schema/tables/asset.table.ts` |
| Live Photo 两阶段上传 | `mobile/lib/services/background_upload.service.dart:230-261` |
