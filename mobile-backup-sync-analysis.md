# Immich 移动端备份状态同步完整分析

## 1. 概述

Immich 移动端备份系统采用了双层架构设计，支持前台（Foreground）和后台（Background）两种上传模式。系统通过 Riverpod 状态管理、SQLite 本地持久化（Drift）和与服务端的同步流（Sync Stream）实现完整的备份状态同步。

## 2. 核心组件概览

### 2.1 状态管理层

| 组件 | 文件路径 | 核心职责 |
|------|---------|---------|
| `DriftBackupNotifier` | `mobile/lib/providers/backup/drift_backup.provider.dart` | 全局备份状态管理、进度追踪、错误处理 |
| `AssetUploadProgressNotifier` | `mobile/lib/providers/backup/asset_upload_progress.provider.dart` | 单资产上传进度追踪 |
| `BackupNotifier` | `mobile/lib/providers/backup/backup.provider.dart` | 服务端磁盘信息同步 |

### 2.2 服务层

| 组件 | 文件路径 | 核心职责 |
|------|---------|---------|
| `ForegroundUploadService` | `mobile/lib/services/foreground_upload.service.dart` | 前台 HTTP 同步上传，支持并发 Worker 池 |
| `BackgroundUploadService` | `mobile/lib/services/background_upload.service.dart` | iOS 后台 URLSession 上传，支持应用挂起 |
| `SyncStreamService` | `mobile/lib/domain/services/sync_stream.service.dart` | 服务端事件流同步，确认上传结果 |

### 2.3 仓库层

| 组件 | 文件路径 | 核心职责 |
|------|---------|---------|
| `UploadRepository` | `mobile/lib/repositories/upload.repository.dart` | 实际 HTTP 上传执行、进度回调 |
| `DriftBackupRepository` | `mobile/lib/infrastructure/repositories/backup.repository.dart` | 备份候选查询、本地状态持久化 |
| `RemoteAssetRepository` | `mobile/lib/infrastructure/repositories/remote_asset.repository.dart` | 远程资产缓存管理 |

---

## 3. 上传进度追踪机制

### 3.1 进度数据流

```
上传请求
    ↓
ProgressMultipartRequest.finalize()  [upload.repository.dart:162-180]
    ↓ 字节流转换，实时统计已传输字节
onProgress(bytes, totalBytes) 回调
    ↓
_handleForegroundBackupProgress()  [drift_backup.provider.dart:300-334]
    ├─ 计算 progress = bytes / totalBytes
    ├─ UploadSpeedManager 更新速度
    └─ 更新 DriftBackupState.uploadItems
```

### 3.2 关键实现细节

**自定义进度请求类** (`upload.repository.dart:153-181`):
```dart
class ProgressMultipartRequest extends MultipartRequest with Abortable {
  @override
  ByteStream finalize() {
    final byteStream = super.finalize();
    final total = contentLength;
    var bytes = 0;
    final stream = byteStream.transform(
      StreamTransformer.fromHandlers(
        handleData: (List<int> data, EventSink<List<int>> sink) {
          bytes += data.length;
          onProgress!(bytes, total);  // 实时进度回调
          sink.add(data);
        },
      ),
    );
    return ByteStream(stream);
  }
}
```

**上传状态模型** (`drift_backup.provider.dart:30-99`):
```dart
class DriftUploadStatus {
  final String taskId;           // 本地资产ID
  final String filename;         // 文件名
  final double progress;         // 0.0 - 1.0
  final int fileSize;            // 文件总大小
  final String networkSpeedAsString;  // 网速显示
  final bool? isFailed;          // 是否失败
  final String? error;           // 错误信息
}
```

### 3.3 上传速度计算

`upload_speed_calculator.dart` 实现了滑动窗口平均算法：

```dart
class UploadSpeedManager {
  final Map<String, UploadSpeedCalculator> _calculators = {};

  String updateProgress(String taskId, int currentBytes, int totalBytes) {
    final calculator = getCalculator(taskId);
    calculator.update(currentBytes, totalBytes);  // 滑动窗口更新
    return calculator.speedAsString;  // 格式化输出："X MB/s" 或 "X kB/s"
  }
}
```

**滑动窗口算法** (`upload_speed_calculator.dart:41-75`):
- 窗口大小：5 个样本
- 最小采样间隔：100ms（避免除零错误）
- 速度计算：`bytesTransferred / elapsedSeconds` → 转换为 MB/s
- 剩余时间估算：`remainingBytes / bytesPerSecond`

### 3.4 并发上传控制

前台上传采用 Worker 池模式 (`foreground_upload.service.dart:197-237`):
```dart
Future<void> _executeWithWorkerPool<T>({
  required List<T> items,
  int concurrentWorkers = 3,  // 默认 3 个并发
  required Future<void> Function(T item) processItem,
}) async {
  int currentIndex = 0;
  Future<void> worker() async {
    while (true) {
      final index = currentIndex;
      if (index >= items.length) break;
      currentIndex++;
      await processItem(items[index]);
    }
  }
  final workerFutures = List.generate(concurrentWorkers, (_) => worker());
  await Future.wait(workerFutures);
}
```

---

## 4. 服务端确认机制

### 4.1 同步确认流程

上传成功后，系统通过两个独立通道确认资产状态：

```
通道 1：HTTP 响应即时确认
  uploadFile() 成功 → 返回 remoteAssetId
    ↓
  _handleForegroundBackupSuccess()  [drift_backup.provider.dart:336-343]
    ├─ 更新 backupCount + 1, remainderCount - 1
    ├─ 延迟 1s 后移除上传项（UI 过渡）
    └─ 本地数据库已通过 checksum 关联

通道 2：Sync Stream 最终确认
  服务端 AssetUploadReady 事件
    ↓
  SyncStreamService.handleWsAssetUploadReadyV1Batch()  [sync_stream.service.dart:328-369]
    ├─ 解析 SyncAssetV1 + SyncAssetExifV1
    ├─ 写入 remote_asset_entity 表
    └─ 通过 checksum 关联本地资产
```

### 4.2 HTTP 响应确认

`upload.repository.dart:91-150` 中 `uploadFile()` 方法：
```dart
Future<UploadResult> uploadFile({...}) async {
  final response = await NetworkRepository.client.send(baseRequest);
  final responseBodyString = await response.stream.bytesToString();

  if ([200, 201].contains(response.statusCode)) {
    final responseBody = jsonDecode(responseBodyString);
    return UploadResult.success(remoteAssetId: responseBody['id'] as String);
  }
  // 错误处理...
}
```

**成功响应结构**:
```json
{
  "id": "uuid-of-uploaded-asset",
  // ... 其他资产元数据
}
```

### 4.3 同步流最终确认

Sync Stream 是服务端状态的真实来源（Source of Truth）。服务端处理完上传的资产后（包括转码、缩略图生成等），会通过 WebSocket 发送 `AssetUploadReady` 事件。

**批处理确认** (`sync_stream.service.dart:328-412`):
```dart
Future<void> handleWsAssetUploadReadyV1Batch(List<dynamic> batchData) async {
  final List<SyncAssetV1> assets = [];
  final List<SyncAssetExifV1> exifs = [];

  for (final data in batchData) {
    final asset = SyncAssetV1.fromJson(data['asset']);
    final exif = SyncAssetExifV1.fromJson(data['exif']);
    assets.add(asset);
    exifs.add(exif);
  }

  // 批量写入本地数据库
  await _syncStreamRepository.updateAssetsV1(assets);
  await _syncStreamRepository.updateAssetsExifV1(exifs);
}
```

### 4.4 本地-远程关联机制

**通过 Checksum 关联** (`backup.repository.dart:39-82`):
```sql
SELECT
  COUNT(*) AS total_count,
  COUNT(*) FILTER (WHERE rae.id IS NULL) AS remainder_count
FROM local_asset_entity lae
LEFT JOIN main.remote_asset_entity rae
    ON lae.checksum = rae.checksum AND rae.owner_id = ?1
WHERE EXISTS (
    SELECT 1 FROM local_album_asset_entity laa
    INNER JOIN main.local_album_entity la ON laa.album_id = la.id
    WHERE laa.asset_id = lae.id AND la.backup_selection = ?2
)
```

**关键关联点**:
- `local_asset_entity.checksum` ↔ `remote_asset_entity.checksum`
- 关联时同时匹配 `owner_id`（多用户支持）
- 只有存在 `remote_asset` 记录的资产才被视为"已备份"

---

## 5. 失败恢复机制

### 5.1 失败分类与处理

| 失败类型 | 触发条件 | 处理策略 |
|---------|---------|---------|
| **取消上传** | 用户主动取消 | `RequestAbortedException` → 标记 `isCancelled`，终止队列 |
| **配额超限** | 服务端返回 413 或 "Quota has been exceeded" | 立即终止整个上传队列 |
| **网络错误** | 连接超时、DNS 失败等 | 单资产失败，不影响其他资产，`isFailed: true` |
| **文件不存在** | 本地资产已被删除 | 标记失败，跳过继续 |
| **iCloud 下载失败** | iOS 云端资产无法加载 | 标记失败，跳过继续 |

### 5.2 前台上传失败处理

`foreground_upload.service.dart:407-419`:
```dart
} catch (error, stackTrace) {
  _logger.severe("Error backup asset: ${error.toString()}", stackTrace);
  callbacks.onError?.call(asset.localId!, error.toString());
} finally {
  // iOS 清理临时文件
  if (Platform.isIOS) {
    await file?.delete();
    await livePhotoFile?.delete();
  }
}
```

**状态更新** (`drift_backup.provider.dart:345-374`):
```dart
void _handleForegroundBackupError(String localAssetId, String errorMessage) {
  final currentItem = state.uploadItems[localAssetId];
  state = state.copyWith(
    uploadItems: {
      ...state.uploadItems,
      localAssetId: currentItem.copyWith(isFailed: true, error: errorMessage),
    },
  );
  _uploadSpeedManager.removeTask(localAssetId);
}
```

### 5.3 后台上传重试机制

`background_upload.service.dart:424-441` 中任务配置：
```dart
return UploadTask(
  retries: 3,           // 自动重试 3 次
  updates: Updates.statusAndProgress,
  // ...
);
```

后台上传由系统 `background_downloader` 库管理：
- 自动重试网络临时故障
- 失败任务状态持久化到本地数据库
- 应用重启后可查询失败任务重新入队

### 5.4 断点续传

**当前实现限制**:
- ✅ 应用重启后可继续未完成的后台任务（iOS URLSession 支持）
- ❌ 不支持文件级别的断点续传（每次重新上传整个文件）
- ✅ 已上传成功的资产不会重复上传（通过 checksum 去重）

**去重逻辑** (`backup.repository.dart:84-116`):
```dart
Future<List<LocalAsset>> getCandidates(String userId, {bool onlyHashed = true}) async {
  final query = _db.localAssetEntity.select()
    ..where(
      (lae) =>
        // 存在于选中的相册
        existsQuery(...) &
        // 在远程资产中不存在相同 checksum
        notExistsQuery(
          _db.remoteAssetEntity.selectOnly()
            ..addColumns([_db.remoteAssetEntity.checksum])
            ..where(
              _db.remoteAssetEntity.checksum.equalsExp(lae.checksum) &
              _db.remoteAssetEntity.ownerId.equals(userId),
            ),
        ) &
        // 不在排除的相册中
        lae.id.isNotInQuery(_getExcludedSubquery()),
    );
  // ...
}
```

---

## 6. 前后端状态对齐

### 6.1 状态对齐架构

```
┌─────────────────────────────────────────────────────────────┐
│                     移动端 (Client)                         │
│  ┌──────────────┐     ┌──────────────┐     ┌──────────────┐ │
│  │  本地资产表  │────▶│  备份状态表  │────▶│  远程资产表  │ │
│  │ (local_asset)│     │ (Drift 内存) │     │ (remote_asset)│ │
│  └──────────────┘     └──────────────┘     └───────┬──────┘ │
│         ▲                                            │        │
│         │ PhotoManager 扫描                   Sync Stream │
│         │                                            ▼        │
└─────────┴────────────────────────────────────────────────────┘
          │                                            │
          │                                            │
┌─────────┴────────────────────────────────────────────┴────────┐
│                     服务端 (Server)                           │
│  ┌──────────────┐     ┌──────────────┐     ┌──────────────┐  │
│  │  资产存储    │────▶│  元数据DB    │────▶│  同步流服务  │  │
│  │ (文件系统)   │     │ (PostgreSQL) │     │ (WebSocket)  │  │
│  └──────────────┘     └──────────────┘     └──────────────┘  │
└─────────────────────────────────────────────────────────────┘
```

### 6.2 状态同步触发点

| 触发时机 | 同步内容 | 执行方 |
|---------|---------|-------|
| 应用启动 | 全量同步远程资产 | SyncStreamService.sync() |
| 备份页面打开 | 查询备份统计 | DriftBackupNotifier.getBackupStatus() |
| 上传成功后 | 即时更新计数 | _handleForegroundBackupSuccess() |
| WebSocket 推送 | 增量更新远程资产 | handleWsAssetUploadReadyV1Batch() |
| 手动刷新 | 重新拉取备份统计 | 用户下拉刷新 |

### 6.3 本地数据库表结构

**local_asset_entity** - 本地资产清单
- `id`: 本地资产唯一标识
- `checksum`: 文件哈希（用于关联远程资产）
- `cloudId`: iOS 云端标识（iCloud）
- `createdAt`, `updatedAt`: 文件时间戳

**remote_asset_entity** - 远程资产缓存（同步流写入）
- `id`: 服务端资产 UUID
- `checksum`: 文件哈希（关联本地资产的关键）
- `ownerId`: 所属用户 ID
- `deletedAt`: 删除标记（软删除）
- `stackId`: 相册堆 ID

**local_album_entity** - 相册配置
- `backupSelection`: `selected` / `excluded` / `none`

### 6.4 一致性保障机制

1. **Checksum 作为事实关联键**
   - 上传前计算本地文件 checksum
   - 服务端返回的资产也包含 checksum
   - 同步时通过 checksum 匹配，避免重复上传

2. **Sync Stream 最终一致性**
   - 所有服务端变更通过同步流推送
   - 客户端按顺序处理事件，发送 ACK 确认
   - 支持 `syncResetV1` 事件重置客户端状态

3. **定期全量同步**
   - 每次应用启动执行完整同步
   - 处理同步流可能遗漏的变更
   - 执行数据迁移任务

### 6.5 状态查询 API

`backup.repository.dart:39-82` 单 SQL 查询获取所有统计：
```sql
SELECT
  COUNT(*) AS total_count,
  COUNT(*) FILTER (WHERE lae.checksum IS NULL) AS processing_count,
  COUNT(*) FILTER (WHERE rae.id IS NULL) AS remainder_count
FROM local_asset_entity lae
LEFT JOIN main.remote_asset_entity rae
    ON lae.checksum = rae.checksum AND rae.owner_id = ?1
WHERE EXISTS (
    SELECT 1 FROM local_album_asset_entity laa
    INNER JOIN main.local_album_entity la ON laa.album_id = la.id
    WHERE laa.asset_id = lae.id AND la.backup_selection = ?2
)
AND NOT EXISTS (
    SELECT 1 FROM local_album_asset_entity laa
    INNER JOIN main.local_album_entity la ON laa.album_id = la.id
    WHERE laa.asset_id = lae.id AND la.backup_selection = ?3
);
```

**状态字段说明**:
- `totalCount`: 选中相册中的资产总数
- `backupCount`: `totalCount - remainderCount`（已备份）
- `remainderCount`: 远程不存在的资产数（待上传）
- `processingCount`: checksum 为 NULL 的资产数（正在计算哈希）

---

## 7. Live Photo 特殊处理

### 7.1 两阶段上传

iOS Live Photo 包含照片 + 视频两个文件，需分开上传：

```
阶段 1：上传视频部分
  getUploadTask(asset) → 提取 motion file
    ↓
  上传成功 → 返回 remoteAssetId
    ↓
  _handleLivePhoto() → 构建第二阶段任务

阶段 2：上传照片部分
  getLivePhotoUploadTask(asset, livePhotoVideoId)
    ↓
  fields['livePhotoVideoId'] = videoId
    ↓
  上传照片，关联视频 ID
```

### 7.2 任务分组与优先级

`background_upload.service.dart:274-283`:
```dart
/// iOS LivePhoto 两阶段上传：
/// 1. 视频文件：普通优先级组 (kBackupGroup)
/// 2. 照片文件：高优先级组 (kBackupLivePhotoGroup, priority=0)
/// 取消操作只取消视频组，照片组不受影响（视频已上传）
```

---

## 8. 网络条件适配

### 8.1 WiFi / 蜂窝网络控制

`foreground_upload.service.dart:457-467`:
```dart
bool _shouldRequireWiFi(LocalAsset asset) {
  bool requiresWiFi = true;
  if (asset.isVideo && _appSettingsService.getSetting(AppSettingsEnum.useCellularForUploadVideos)) {
    requiresWiFi = false;
  } else if (!asset.isVideo && _appSettingsService.getSetting(AppSettingsEnum.useCellularForUploadPhotos)) {
    requiresWiFi = false;
  }
  return requiresWiFi;
}
```

**Worker 池中的跳过逻辑**:
```dart
await _executeWithWorkerPool<LocalAsset>(
  items: candidates,
  shouldSkip: (asset) {
    final requireWifi = _shouldRequireWiFi(asset);
    return requireWifi && !hasWifi;  // 需要 WiFi 但当前没有
  },
  processItem: ...
);
```

### 8.2 后台上传网络约束

`background_upload.service.dart:437`:
```dart
return UploadTask(
  requiresWiFi: requiresWiFi,  // 系统级网络约束
  // ...
);
```

iOS URLSession 会自动遵守此约束，在不满足网络条件时暂停。

---

## 9. 取消机制

### 9.1 前台上传取消

`drift_backup.provider.dart:281-286`:
```dart
void stopForegroundBackup() {
  _cancelToken?.complete();      // 触发取消信号
  _cancelToken = null;
  _uploadSpeedManager.clear();   // 清理速度统计
  state = state.copyWith(uploadItems: {}, iCloudDownloadProgress: {});
}
```

**取消传播**:
1. `Completer.complete()` 触发 future 完成
2. `ProgressMultipartRequest` 监听 `abortTrigger` future
3. 抛出 `RequestAbortedException` 终止上传
4. Worker 池检测到 `cancelToken.isCompleted` 退出循环

### 9.2 后台上传取消

`background_upload.service.dart:195-204`:
```dart
Future<int> cancel() async {
  shouldAbortQueuingTasks = true;
  await _storageRepository.clearCache();
  await _uploadRepository.reset(kBackupGroup);           // 重置队列
  await _uploadRepository.deleteDatabaseRecords(kBackupGroup);  // 清理记录
  final activeTasks = await _uploadRepository.getActiveTasks(kBackupGroup);
  return activeTasks.length;
}
```

---

## 10. 总结

### 10.1 设计亮点

1. **双层上传架构**：前台并发 + 后台保活，兼顾速度和可靠性
2. **Checksum 关联机制**：无需保存上传记录，通过文件哈希自然去重
3. **Sync Stream 最终一致性**：WebSocket 推送确保状态最终对齐
4. **滑动窗口速度计算**：平滑的上传速度和剩余时间估算
5. **Live Photo 优雅处理**：两阶段上传，高优先级确保完整性

### 10.2 可改进点

1. **缺乏真正的断点续传**：大文件上传失败需重新开始
2. **失败资产无自动重试队列**：需用户手动重新触发备份
3. **上传进度无持久化**：应用重启后进度重置
4. **并发数固定**：未根据网络条件动态调整并发数

### 10.3 关键文件速查

| 功能 | 文件 | 行号范围 |
|------|------|---------|
| 上传进度回调 | `upload.repository.dart` | 153-181 |
| 上传状态管理 | `drift_backup.provider.dart` | 103-398 |
| 前台上传服务 | `foreground_upload.service.dart` | 81-467 |
| 后台上传服务 | `background_upload.service.dart` | 98-442 |
| 速度计算 | `upload_speed_calculator.dart` | 1-182 |
| 同步流服务 | `sync_stream.service.dart` | 30-543 |
| 备份统计查询 | `backup.repository.dart` | 39-116 |
