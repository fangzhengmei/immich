# Immich 移动端大批量上传去重与断点续传技术报告

## 概述

Immich 移动端实现了完整的多层级去重机制和断点续传能力，通过**本地候选筛选**、**平台原生哈希**、**服务端预检**、**数据库唯一约束**四层防护确保重复文件不上传，同时通过 `background_downloader` 库实现 iOS/Android 双平台的后台任务断点续传。

---

## 一、候选筛选去重 (Candidate Filtering)

### 1.1 本地数据库级去重

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
          existsQuery(...) &
          // 条件2: ⭐ 核心去重逻辑 - LEFT JOIN 远程资产表
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
    query.where((lae) => lae.checksum.isNotNull()); // 条件4: 只选已计算哈希的资产
  }

  return query.map((localAsset) => localAsset.toDto()).get();
}
```

**去重机制详解**：
| 层级 | 实现方式 | 作用 |
|-----|---------|------|
| **数据库层** | `LEFT JOIN` + `NOT EXISTS` | 在 SQL 查询层面直接过滤掉已存在的资产，避免在 Dart 层处理大量数据 |
| **主键匹配** | `checksum + ownerId` 双字段匹配 | 确保同一用户的相同文件不会重复上传，不同用户即使文件相同也各自独立 |
| **预筛选** | `onlyHashed = true` | 只选择已完成哈希计算的资产，避免上传无指纹文件 |

### 1.2 统计查询（三值统计）

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

### 1.3 本地哈希持久化

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

func hashAssets(assetIds: [String], allowNetworkAccess: Bool, completion: @escaping (Result<[HashResult], Error>) -> Void) {
  // 并发任务组处理
  await withTaskGroup(of: HashResult?.self) { taskGroup in
    var results = [HashResult]()
    for asset in assets {
      taskGroup.addTask {
        guard let self = self else { return nil }
        return await self.hashAsset(asset, allowNetworkAccess: allowNetworkAccess)
      }
    }
    // ...
  }
}

private func hashAsset(_ asset: PHAsset, allowNetworkAccess: Bool) async -> HashResult? {
  return await withTaskCancellationHandler(operation: {
    guard let resource = asset.getResource() else {
      return HashResult(assetId: asset.localIdentifier, error: "Cannot get asset resource", hash: nil)
    }

    let options = PHAssetResourceRequestOptions()
    options.isNetworkAccessAllowed = allowNetworkAccess  // iCloud 下载控制

    return await withCheckedContinuation { continuation in
      var hasher = Insecure.SHA1()  // ⭐ iOS 使用 CryptoKit.SHA1

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

**iOS 实现特点**：
- **哈希算法**：`CryptoKit.Insecure.SHA1()`
- **流式处理**：通过 `dataReceivedHandler` 分块接收数据，实时更新哈希
- **iCloud 支持**：`allowNetworkAccess` 控制是否允许从 iCloud 下载
- **可取消**：支持任务取消，中断哈希计算

### 2.2 Android 哈希实现

**文件位置**：`mobile/android/app/src/main/kotlin/app/alextran/immich/sync/MessagesImplBase.kt:380-449`

```kotlin
import java.security.MessageDigest
import android.util.Base64
import kotlinx.coroutines.sync.Semaphore

companion object {
  private const val MAX_CONCURRENT_HASH_OPERATIONS = 16  // 最大并发数
  private val hashSemaphore = Semaphore(MAX_CONCURRENT_HASH_OPERATIONS)
  const val HASH_BUFFER_SIZE = 2 * 1024 * 1024  // 2MB 缓冲区
}

fun hashAssets(
  assetIds: List<String>,
  allowNetworkAccess: Boolean,
  callback: (Result<List<HashResult>>) -> Unit
) {
  hashTask = CoroutineScope(Dispatchers.IO).launch {
    try {
      val results = assetIds.map { assetId ->
        async {
          hashSemaphore.withPermit {  // 并发控制
            ensureActive()
            hashAsset(assetId)
          }
        }
      }.awaitAll()
      completeWhenActive(callback, Result.success(results))
    } catch (e: CancellationException) {
      // ... 处理取消
    }
  }
}

private suspend fun hashAsset(assetId: String): HashResult {
  return try {
    val assetUri = ContentUris.withAppendedId(
      MediaStore.Files.getContentUri(MediaStore.VOLUME_EXTERNAL),
      assetId.toLong()
    )

    val digest = MessageDigest.getInstance("SHA-1")  // ⭐ Android 使用 Java SHA-1
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

**Android 实现特点**：
- **哈希算法**：`java.security.MessageDigest.getInstance("SHA-1")`
- **缓冲区大小**：2MB（`HASH_BUFFER_SIZE = 2 * 1024 * 1024`）
- **并发控制**：`Semaphore` 限制最大 16 个并发哈希操作
- **协程支持**：使用 `CoroutineScope(Dispatchers.IO)` 异步处理，支持取消

### 2.3 跨平台一致性保证

| 对比项 | iOS | Android | 一致性 |
|-------|-----|---------|--------|
| 算法 | SHA-1 | SHA-1 | ✅ 一致 |
| 输出格式 | Base64 | Base64.NO_WRAP | ✅ 一致 |
| 取消支持 | ✅ | ✅ | 一致 |
| 并发数 | 无限制（TaskGroup） | 16 (Semaphore) | 平台差异 |

---

## 三、上传链路与服务端预检 (Upload Pipeline & Server Check)

### 3.1 完整上传链路

```
┌─────────────────────────────────────────────────────────────────────────┐
│                    Immich 完整上传去重链路                                │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                         │
│  ┌─────────────┐   ┌─────────────┐   ┌─────────────┐   ┌─────────────┐  │
│  │ 本地 SQLite  │   │ HTTP Header │   │ 批量预检 API │   │ PostgreSQL  │  │
│  │   去重      │   │    预检     │   │   (可选)    │   │ 唯一约束   │  │
│  └──────┬──────┘   └──────┬──────┘   └──────┬──────┘   └──────┬──────┘  │
│         │                  │                  │                  │         │
│         ▼                  ▼                  ▼                  ▼         │
│    getCandidates()    AssetUploadInterceptor  bulkUploadCheck   UNIQUE    │
│                         /assets                                (checksum,  │
│                         endpoint                              ownerId)   │
│                                                                         │
│  层级1: 本地过滤        层级2: 上传前过滤   层级3: 批量优化   层级4: 兜底  │
└─────────────────────────────────────────────────────────────────────────┘
```

### 3.2 层级1：本地候选过滤

**执行时机**：选择待上传资产时
**触发位置**：`backup.repository.dart:getCandidates()`
**过滤方式**：SQLite `LEFT JOIN` 查询
**性能特点**：数据库级操作，O(1) 复杂度，不产生网络请求

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
      // ⭐ 发现重复，直接返回 200，跳过 Multer 文件处理
      return of({ status: AssetMediaStatus.DUPLICATE, id: response.id });
    }

    return next.handle();
  }
}
```

**执行时机**：服务端接收到上传请求时，在 Multer 解析文件之前
**触发位置**：NestJS 拦截器
**性能特点**：只传输 Header，不上传文件内容，节省 99% 以上带宽

### 3.4 层级3：批量预检 API（Web 端使用）

**文件位置**：`server/src/services/asset-media.service.ts`

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
          action: AssetUploadAction.REJECT,  // 标记拒绝
          reason: AssetRejectReason.DUPLICATE,
          assetId: duplicate.id,
          isTrashed: duplicate.isTrashed,
        };
      }
      return {
        id,
        action: AssetUploadAction.ACCEPT,  // 标记允许上传
      };
    }),
  };
}
```

**执行时机**：批量上传前一次性预检
**适用场景**：Web 端批量上传
**性能特点**：一次 API 调用检查 N 个文件，HTTP 请求次数减少 N 倍

### 3.5 层级4：数据库唯一约束兜底

**文件位置**：`server/src/schema/tables/asset.table.ts`

```typescript
@Table('asset')
// 索引1：无库时的用户级唯一约束
@Index({
  name: 'IDX_asset_owner_checksum_unique',
  columns: ['ownerId', 'checksum'],
  unique: true,
  where: '"libraryId" IS NULL',
})
// 索引2：有库时的库级唯一约束
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

## 四、后台断点续传与任务状态管理 (Background Upload & Resume)

### 4.1 架构概述

Immich 使用 `background_downloader` 库实现双平台后台上传：
- **iOS**：基于 `URLSession` 后台会话
- **Android**：基于 `WorkManager` 前台服务

### 4.2 任务状态统计

**文件位置**：`mobile/lib/repositories/upload.repository.dart:69-89`

```dart
Future<void> getUploadInfo() async {
  // ⭐ 状态统计：从数据库查询所有状态的任务数
  final [enqueuedTasks, runningTasks, canceledTasks, waitingTasks, pausedTasks] = await Future.wait([
    FileDownloader().database.allRecordsWithStatus(TaskStatus.enqueued, group: kBackupGroup),
    FileDownloader().database.allRecordsWithStatus(TaskStatus.running, group: kBackupGroup),
    FileDownloader().database.allRecordsWithStatus(TaskStatus.canceled, group: kBackupGroup),
    FileDownloader().database.allRecordsWithStatus(TaskStatus.waitingToRetry, group: kBackupGroup),
    FileDownloader().database.allRecordsWithStatus(TaskStatus.paused, group: kBackupGroup),
  ]);

  dPrint(() => """
    Upload Info:
    Enqueued: ${enqueuedTasks.length}   // 队列中等待
    Running: ${runningTasks.length}     // 正在上传
    Canceled: ${canceledTasks.length}   // 已取消
    Waiting: ${waitingTasks.length}     // 等待重试
    Paused: ${pausedTasks.length}       // 已暂停
  """);
}
```

**⭐ 状态统计边界说明**：
- **用途**：仅用于日志输出、调试展示、进度统计
- **数据源**：SQLite 数据库持久化存储
- **调用时机**：主动调用查询，非实时
- **影响范围**：只读，不影响任务执行

### 4.3 状态回调处理

**文件位置**：`mobile/lib/services/background_upload.service.dart:209-228`

```dart
void _handleTaskStatusUpdate(TaskStatusUpdate update) async {
  switch (update.status) {
    // ⭐ 只处理 complete 状态，其他状态交由 background_downloader 内部管理
    case TaskStatus.complete:
      unawaited(_handleLivePhoto(update));  // 处理 Live Photo 关联

      // iOS 平台清理临时文件
      if (CurrentPlatform.isIOS) {
        try {
          final path = await update.task.filePath();
          await File(path).delete();
        } catch (e) {
          _logger.severe('Error deleting file path for iOS: $e');
        }
      }
      break;

    default:
      break;  // 其他状态不做处理
  }
}
```

**⭐ 状态回调边界说明**：
| 状态 | 是否处理 | 处理逻辑 |
|-----|---------|---------|
| `TaskStatus.complete` | ✅ 处理 | 1. 触发 Live Photo 后续上传<br>2. iOS 清理临时文件 |
| `TaskStatus.running` | ❌ 不处理 | 由库内部管理进度 |
| `TaskStatus.enqueued` | ❌ 不处理 | 由库内部管理队列 |
| `TaskStatus.waitingToRetry` | ❌ 不处理 | 由库内部管理重试 |
| `TaskStatus.paused` | ❌ 不处理 | 由用户手动控制 |
| `TaskStatus.canceled` | ❌ 不处理 | 用户取消操作 |
| `TaskStatus.failed` | ❌ 不处理 | 由库内部管理失败逻辑 |

### 4.4 回调注册与分组

**文件位置**：`mobile/lib/repositories/upload.repository.dart:22-38`

```dart
UploadRepository() {
  // 注册三个分组的状态回调
  FileDownloader().registerCallbacks(
    group: kBackupGroup,          // 普通备份任务组
    taskStatusCallback: (update) => onUploadStatus?.call(update),
    taskProgressCallback: (update) => onTaskProgress?.call(update),
  );
  FileDownloader().registerCallbacks(
    group: kBackupLivePhotoGroup, // Live Photo 高优先级任务组
    taskStatusCallback: (update) => onUploadStatus?.call(update),
    taskProgressCallback: (update) => onTaskProgress?.call(update),
  );
  FileDownloader().registerCallbacks(
    group: kManualUploadGroup,    // 手动上传任务组
    taskStatusCallback: (update) => onUploadStatus?.call(update),
    taskProgressCallback: (update) => onTaskProgress?.call(update),
  );
}
```

### 4.5 断点续传控制

**文件位置**：`mobile/lib/services/background_upload.service.dart:193-207`

```dart
/// 取消所有后台上传并重置队列
Future<int> cancel() async {
  shouldAbortQueuingTasks = true;
  await _storageRepository.clearCache();
  await _uploadRepository.reset(kBackupGroup);           // 重置任务状态
  await _uploadRepository.deleteDatabaseRecords(kBackupGroup);  // 清除数据库记录
  final activeTasks = await _uploadRepository.getActiveTasks(kBackupGroup);
  return activeTasks.length;
}

/// 恢复断点续传
Future<void> resume() {
  return _uploadRepository.start();  // 启动后台下载器的恢复机制
}
```

**断点续传说明**：
- **任务持久化**：所有上传任务状态持久化在 SQLite 中
- **App 重启恢复**：启动时调用 `start()` 自动恢复队列中未完成的任务
- **失败重试**：`waitingToRetry` 状态由库自动管理重试策略
- **Live Photo 断点**：视频上传完成后才触发照片上传，实现两阶段断点

### 4.6 Live Photo 两阶段断点续传

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

    // ⭐ 阶段1完成后，入队阶段2任务
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
| 阶段 2 | 照片文件 | 最高 (0) | `kBackupLivePhotoGroup` | 回调中入队 |

---

## 五、关键技术参数汇总

| 参数 | 值 | 代码位置 | 说明 |
|-----|----|---------|------|
| **前台并发数** | **3** | `foreground_upload.service.dart:199` | Worker Pool 默认并发数 |
| **后台批次大小** | **100** | `background_upload.service.dart:173` | 每次入队任务数 |
| **哈希算法** | **SHA-1** | iOS/Android 原生代码 | 文件内容指纹 |
| **哈希输出格式** | **Base64** | 双平台代码 | 与服务端兼容 |
| **Android 哈希并发** | **16** | `MessagesImplBase.kt:48` | 最大并发哈希操作 |
| **Android 哈希缓冲区** | **2MB** | `MessagesImplBase.kt:95` | 分块读取大小 |
| **任务状态数** | **5 种** | `upload.repository.dart:71-75` | enqueued/running/canceled/waitingToRetry/paused |
| **Live Photo 优先级** | **0 (最高)** | `background_upload.service.dart:353` | 确保照片立即上传 |
| **去重层级** | **4 层** | - | 本地/Header/批量API/数据库约束 |

---

## 六、代码溯源索引

| 功能模块 | 文件路径 |
|---------|---------|
| 备份候选查询（去重核心） | `mobile/lib/infrastructure/repositories/backup.repository.dart:84-116` |
| 本地资产 Hash 更新 | `mobile/lib/infrastructure/repositories/local_asset.repository.dart:51-65` |
| iOS 原生哈希实现 | `mobile/ios/Runner/Sync/MessagesImpl.swift:271-380` |
| Android 原生哈希实现 | `mobile/android/app/src/main/kotlin/app/alextran/immich/sync/MessagesImplBase.kt:380-449` |
| 前台并发 Worker Pool | `mobile/lib/services/foreground_upload.service.dart:190-237` |
| 后台上传服务 | `mobile/lib/services/background_upload.service.dart` |
| 上传仓库（状态管理） | `mobile/lib/repositories/upload.repository.dart:69-89` |
| 服务端预检拦截器 | `server/src/middleware/asset-upload.interceptor.ts` |
| 批量预检 API 实现 | `server/src/services/asset-media.service.ts` |
| 数据库唯一约束定义 | `server/src/schema/tables/asset.table.ts` |
| Live Photo 断点续传 | `mobile/lib/services/background_upload.service.dart:230-261` |
