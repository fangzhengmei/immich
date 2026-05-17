# Immich 移动端备份状态同步完整分析（校正版）

## 1. 概述

Immich 移动端备份系统采用了"双层上传 + 三阶段确认"架构设计：
- **双层上传**：前台（Foreground）HTTP 并发上传 + 后台（Background）iOS URLSession 保活上传
- **三阶段确认**：HTTP 响应即时确认 → 内存计数乐观更新 → Sync Stream 最终落库确认

系统通过 Riverpod 状态管理、SQLite 本地持久化（Drift）和 WebSocket 同步流实现完整的备份状态同步。

---

## 2. 核心组件概览

### 2.1 状态管理层

| 组件 | 文件路径 | 核心职责 |
|------|---------|---------|
| `DriftBackupNotifier` | `mobile/lib/providers/backup/drift_backup.provider.dart` | 全局备份状态管理、内存计数、进度追踪、错误处理 |
| `AssetUploadProgressNotifier` | `mobile/lib/providers/backup/asset_upload_progress.provider.dart` | 单资产上传进度追踪 |
| `BackupNotifier` | `mobile/lib/providers/backup/backup.provider.dart` | 服务端磁盘信息同步 |

### 2.2 服务层

| 组件 | 文件路径 | 核心职责 |
|------|---------|---------|
| `ForegroundUploadService` | `mobile/lib/services/foreground_upload.service.dart` | 前台 HTTP 同步上传，支持并发 Worker 池（默认3并发） |
| `BackgroundUploadService` | `mobile/lib/services/background_upload.service.dart` | iOS 后台 URLSession 上传，支持应用挂起 |
| `SyncStreamService` | `mobile/lib/domain/services/sync_stream.service.dart` | 服务端事件流同步，最终写入远程资产表 |
| `BackgroundSyncManager` | `mobile/lib/domain/utils/background_sync.dart` | 同步任务调度、隔离执行、任务取消 |

### 2.3 仓库层

| 组件 | 文件路径 | 核心职责 |
|------|---------|---------|
| `UploadRepository` | `mobile/lib/repositories/upload.repository.dart` | 实际 HTTP 上传执行、进度回调 |
| `DriftBackupRepository` | `mobile/lib/infrastructure/repositories/backup.repository.dart` | 备份候选查询、本地状态统计 |
| `SyncStreamRepository` | `mobile/lib/infrastructure/repositories/sync_stream.repository.dart` | 同步流数据落库（remote_asset 表） |
| `RemoteAssetRepository` | `mobile/lib/infrastructure/repositories/remote_asset.repository.dart` | 远程资产缓存查询 |

### 2.4 网络层

| 组件 | 文件路径 | 核心职责 |
|------|---------|---------|
| `WebsocketNotifier` | `mobile/lib/providers/websocket.provider.dart` | WebSocket 连接管理、事件批处理防抖 |
| `Debouncer` | `mobile/lib/utils/debounce.dart` | 通用防抖器，支持最大等待时间 |

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
    ├─ UploadSpeedManager 更新速度（滑动窗口算法）
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
      if (shouldAbortUpload || (cancelToken != null && cancelToken.isCompleted)) {
        break;
      }
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

**注意**：并发模式下，`shouldAbortUpload` 被设置后，已在处理的资产会继续完成，只有新的资产会被跳过。

---

## 4. 服务端确认机制（三阶段确认）

### 4.1 阶段 1：HTTP 响应即时确认（乐观更新）

```
_uploadSingleAsset()
    ↓
_uploadRepository.uploadFile() → 返回 remoteAssetId
    ↓
callbacks.onSuccess(localAssetId, remoteAssetId)
    ↓
_handleForegroundBackupSuccess()  [drift_backup.provider.dart:336-343]
    ├─ ✅ 内存更新：backupCount + 1, remainderCount - 1
    ├─ ✅ UI更新：延迟 1s 后移除上传项
    └─ ❌ 数据库：remote_asset_entity 表仍无记录
```

**关键代码** (`drift_backup.provider.dart:336-343`):
```dart
void _handleForegroundBackupSuccess(String localAssetId, String remoteAssetId) {
  // ⚠️ 注意：这里只更新内存状态，不写入数据库
  state = state.copyWith(
    backupCount: state.backupCount + 1,
    remainderCount: state.remainderCount - 1,
  );
  _uploadSpeedManager.removeTask(localAssetId);
  Future.delayed(const Duration(milliseconds: 1000), () {
    _removeUploadItem(localAssetId);
  });
}
```

### 4.2 阶段 2：WebSocket 事件批处理（异步落库）

服务端处理完上传资产后（转码、缩略图生成等），通过 WebSocket 推送 `AssetUploadReady` 事件。

```
服务端 AssetUploadReadyV1/V2 事件
    ↓
WebsocketNotifier._handleSyncAssetUploadReadyV1/V2()
    ↓ 加入批处理队列
_batchedAssetUploadReady.add(data)
    ↓ 防抖触发（5s 间隔 / 10s 最大等待）
_batchDebouncer.run(_processBatchedAssetUploadReadyV1/V2)
    ↓
BackgroundSyncManager.syncWebsocketBatchV1/V2()
    ↓ 隔离执行
runInIsolateGentle(_handleWsAssetUploadReadyV1Batch)
    ↓
SyncStreamService.handleWsAssetUploadReadyV1Batch()
    ├─ 解析 SyncAssetV1 + SyncAssetExifV1
    ├─ ✅ 写入 remote_asset_entity 表（checksum 关联）
    └─ ✅ 写入 remote_exif_entity 表
```

**批处理防抖配置** (`websocket.provider.dart:49-52`):
```dart
final Debouncer _batchDebouncer = Debouncer(
  interval: const Duration(seconds: 5),    // 最少 5s 间隔
  maxWaitTime: const Duration(seconds: 10), // 最多等待 10s
);
```

**落库实现** (`sync_stream.repository.dart:196-272`):
```dart
Future<void> updateAssetsV1(Iterable<SyncAssetV1> data, {String debugLabel = 'user'}) async {
  await _db.batch((batch) {
    for (final asset in data) {
      final companion = RemoteAssetEntityCompanion(
        checksum: Value(asset.checksum),  // ⚠️ 关键：通过 checksum 关联本地资产
        ownerId: Value(asset.ownerId),
        uploadedAt: Value(asset.createdAt),
        // ... 其他字段
      );
      batch.insert(
        _db.remoteAssetEntity,
        companion.copyWith(id: Value(asset.id)),
        onConflict: DoUpdate((_) => companion),  // 冲突则更新
      );
    }
  });
}
```

### 4.3 阶段 3：全量同步（最终一致性保障）

应用启动或用户手动触发时执行全量同步，确保即使 WebSocket 事件丢失也能最终对齐。

```
BackgroundSyncManager.syncRemote()
    ↓
runInIsolateGentle(syncStreamService.sync())
    ↓
SyncStreamService.sync()
    ├─ 版本检查 + 数据迁移
    ├─ 调用 syncApiRepository.streamChanges()
    ├─ 按批次处理所有同步事件
    ├─ 每批处理完成后发送 ACK
    └─ ✅ 全量写入 remote_asset_entity 表
```

### 4.4 状态差异时间窗口分析

**时间窗口 1：HTTP 成功 → WebSocket 落库前**
- 状态：`DriftBackupState.backupCount` 已更新，但 `remote_asset_entity` 无记录
- 影响：
  - ✅ UI 显示"已备份"计数正确
  - ❌ 如果此时调用 `getBackupStatus()`，SQL 查询会显示不一致（因为 SQL 通过 checksum 关联判断）
  - ✅ 下次备份时，`getCandidates()` 仍会排除该资产吗？**不会！**
    - `getCandidates()` 通过 `NOT EXISTS (remote_asset where checksum = local.checksum)` 判断
    - HTTP 成功后 checksum 未写入 remote_asset，所以该资产会被重新查询为候选
    - **但实际上传时服务端会检测到重复并返回已存在的 assetId**

**时间窗口 2：WebSocket 落库 → UI 刷新前**
- 状态：`remote_asset_entity` 已更新，但 `DriftBackupState` 内存计数未变
- 影响：
  - ❌ UI 显示的 `backupCount` 可能与实际数据库不一致
  - ✅ 下次调用 `getBackupStatus()` 时会重新计算并对齐

---

## 5. WebSocket 批事件并发边界深度分析

### 5.1 防抖器核心实现分析

**Debouncer 类** (`debounce.dart:1-64`):

```dart
class Debouncer {
  Debouncer({required this.interval, this.maxWaitTime});
  
  final Duration interval;
  final Duration? maxWaitTime;
  Timer? _timer;
  FutureOr<void> Function()? _lastAction;  // ⚠️ 只保存最后一个 action
  DateTime? _lastActionTime;
  Future<void>? _actionFuture;

  void run(FutureOr<void> Function() action) {
    _lastAction = action;  // ⚠️ 每次调用覆盖之前的 action
    _timer?.cancel();      // ⚠️ 取消之前的 timer

    if (maxWaitTime != null &&
        (_lastActionTime == null || 
         DateTime.now().difference(_lastActionTime!) > maxWaitTime!)) {
      _callAndRest();  // 超过最大等待时间立即执行
      return;
    }
    _timer = Timer(interval, _callAndRest);  // 设置新 timer
  }

  void _callAndRest() {
    _lastActionTime = DateTime.now();
    final action = _lastAction;
    _lastAction = null;  // 清空，防止重复执行

    final result = action!();
    if (result is Future) {
      _actionFuture = result.whenComplete(() {
        _actionFuture = null;
      });
    }
    _timer = null;
  }
}
```

**关键特性**：
1. **单 Action 保存**：`_lastAction` 只保存最后一个传入的 action
2. **Timer 取消重设**：每次 `run()` 取消之前的 timer，设置新的 timer
3. **最大等待时间**：超过 `maxWaitTime` 立即执行，防止无限延迟
4. **异步 Future 追踪**：`_actionFuture` 保存当前执行中的 Future

### 5.2 WebSocket 批处理链路完整分析

**共享资源问题** (`websocket.provider.dart:53,173,178`):
```dart
final List<dynamic> _batchedAssetUploadReady = [];  // ⚠️ V1 和 V2 共享同一个队列！

void _handleSyncAssetUploadReadyV1(dynamic data) {
  _batchedAssetUploadReady.add(data);  // 加入共享队列
  _batchDebouncer.run(_processBatchedAssetUploadReadyV1);  // 触发 V1 处理
}

void _handleSyncAssetUploadReadyV2(dynamic data) {
  _batchedAssetUploadReady.add(data);  // 加入同一个共享队列
  _batchDebouncer.run(_processBatchedAssetUploadReadyV2);  // 触发 V2 处理
}
```

**批处理执行** (`websocket.provider.dart:190-230`):
```dart
void _processBatchedAssetUploadReadyV1() {
  if (_batchedAssetUploadReady.isEmpty) return;

  try {
    unawaited(
      _ref.read(backgroundSyncProvider).syncWebsocketBatchV1(
        _batchedAssetUploadReady.toList()  // 快照当前队列
      ).then((_) {
        if (isSyncAlbumEnabled) {
          _ref.read(backgroundSyncProvider).syncLinkedAlbum();
        }
      }),
    );
  } catch (error) {
    _log.severe("Error processing batched AssetUploadReadyV1 events: $error");
  }

  _batchedAssetUploadReady.clear();  // ⚠️ 立即清空，不等待异步任务完成
}
```

**任务合并逻辑** (`background_sync.dart:189-207`):
```dart
Future<void> syncWebsocketBatchV1(List<dynamic> batchData) {
  if (_syncWebsocketTask != null) {
    return _syncWebsocketTask!.future;  // ⚠️ 上一批未完成，直接返回，丢弃新数据
  }
  _syncWebsocketTask = _handleWsAssetUploadReadyV1Batch(batchData);
  return _syncWebsocketTask!.whenComplete(() {
    _syncWebsocketTask = null;
  });
}

// ⚠️ V1 和 V2 共享同一个 _syncWebsocketTask 变量！
Future<void> syncWebsocketBatchV2(List<dynamic> batchData) {
  if (_syncWebsocketTask != null) {
    return _syncWebsocketTask!.future;  // V1 正在执行时，V2 数据被丢弃
  }
  _syncWebsocketTask = _handleWsAssetUploadReadyV2Batch(batchData);
  return _syncWebsocketTask!.whenComplete(() {
    _syncWebsocketTask = null;
  });
}
```

### 5.3 并发场景 1：上一批未完成时新批到达

**触发条件**：
- 批处理 A 正在执行（`_syncWebsocketTask != null`）
- 新事件 X、Y 到达，加入 `_batchedAssetUploadReady`
- 防抖触发，调用 `_processBatchedAssetUploadReadyV1()`

```
时序图：
T0: 批处理 A 开始执行 → _syncWebsocketTask = futureA
T1: 新事件 X、Y 到达 → _batchedAssetUploadReady = [X, Y]
T2: 防抖触发 → _processBatchedAssetUploadReadyV1() 被调用
T3: syncWebsocketBatchV1([X, Y]) 被调用
    → 检测到 _syncWebsocketTask != null
    → 直接返回 futureA，**丢弃 [X, Y]**
T4: _batchedAssetUploadReady.clear() → 队列清空
T5: futureA 完成 → _syncWebsocketTask = null

结果：事件 X、Y 被永久丢失！
```

**保护点**：
- ✅ 无保护！直接丢弃新数据
- ✅ 最终由全量同步兜底

**缺口**：
- ❌ 新事件被静默丢弃，无日志、无重试
- ❌ 队列在异步任务开始前就被清空
- ❌ V1 和 V2 互相阻塞（共享同一个 `_syncWebsocketTask`）

### 5.4 并发场景 2：V1/V2 事件交替到达

**触发条件**：
- V1 事件到达 → 加入队列 → `run(V1_handler)`
- 5s 内 V2 事件到达 → 加入同一队列 → `run(V2_handler)`
- 防抖 timer 触发

```
时序图：
T0: V1 事件 A 到达
    → _batchedAssetUploadReady = [A]
    → _batchDebouncer.run(V1_handler)
    → _lastAction = V1_handler, timer = 5s
T1: (2s 后) V2 事件 B 到达
    → _batchedAssetUploadReady = [A, B]
    → _batchDebouncer.run(V2_handler)
    → _lastAction = V2_handler (覆盖 V1_handler!), timer 重置为 5s
T2: (5s 后) timer 触发
    → _callAndRest() 执行 V2_handler
    → _processBatchedAssetUploadReadyV2() 被调用
    → 传入队列 [A, B] 给 syncWebsocketBatchV2()
    → V2 处理器尝试解析 V1 事件 A → 类型不匹配 → 跳过

结果：V1 事件 A 被 V2 处理器处理，可能被静默跳过！
```

**保护点**：
- ✅ `handleWsAssetUploadReadyV2Batch()` 有类型检查，不匹配的事件会被 `continue` 跳过
- ✅ 最终由全量同步兜底

**缺口**：
- ❌ V1 和 V2 事件共享同一个队列，可能混合
- ❌ 防抖器的 `_lastAction` 被覆盖，导致事件类型与处理器不匹配
- ❌ 跳过的事件无日志、无重试

### 5.5 并发场景 3：防抖器最大等待时间触发

**触发条件**：
- 事件持续以 < 5s 的间隔到达
- 累计超过 10s（`maxWaitTime`）

```
时序图：
T0: 事件 A 到达 → run(V1_handler) → _lastActionTime = T0, timer = 5s
T3: (3s 后) 事件 B 到达 → run(V1_handler) → timer 重置为 5s
T6: (3s 后) 事件 C 到达 → run(V1_handler) → timer 重置为 5s
T9: (3s 后) 事件 D 到达 → run(V1_handler)
    → 检测到 DateTime.now() - _lastActionTime (T0) > 10s
    → 立即调用 _callAndRest()，执行批处理
    → 队列 [A, B, C, D] 被处理

结果：最大等待时间防止无限延迟，正确触发批处理
```

**保护点**：
- ✅ `maxWaitTime` 机制确保不会无限延迟
- ✅ 队列中的所有事件被一次性处理

### 5.6 并发场景 4：批处理失败

**触发条件**：
- 批处理执行过程中抛出异常

```dart
Future<void> handleWsAssetUploadReadyV1Batch(List<dynamic> batchData) async {
  try {
    for (final data in batchData) {
      // 解析和处理...
    }
    if (assets.isNotEmpty) {
      await _syncStreamRepository.updateAssetsV1(assets);
      await _syncStreamRepository.updateAssetsExifV1(exifs);
    }
  } catch (error, stackTrace) {
    _logger.severe("Error processing batch", error, stackTrace);
    // ⚠️ 异常被捕获后，没有重试机制
  }
}
```

**保护点**：
- ✅ 异常被捕获，不会崩溃应用
- ✅ `_syncWebsocketTask` 在 `whenComplete` 中被置空，不会永久阻塞

**缺口**：
- ❌ 失败的批次没有重试机制
- ❌ 失败的事件永久丢失（除非全量同步）
- ❌ 部分成功部分失败的场景没有处理（例如前 5 个成功，第 6 个失败，后 4 个未处理）

### 5.7 对失败恢复的影响

| 并发问题 | 失败恢复影响 | 兜底机制 |
|---------|-------------|---------|
| 上一批未完成时新批被丢弃 | 新事件丢失，直到下次全量同步 | 全量 sync |
| V1/V2 交替到达导致类型不匹配 | 部分事件被跳过，直到下次全量同步 | 全量 sync |
| 批处理异常失败 | 整批事件丢失，直到下次全量同步 | 全量 sync |
| 部分成功部分失败 | 未处理的事件丢失，直到下次全量同步 | 全量 sync |

**关键结论**：WebSocket 批事件的所有并发问题最终都依赖**全量同步**来兜底。如果用户长时间不重启应用（不触发全量同步），这些事件可能永久丢失，导致备份状态显示"未完成"但实际上传已成功。

### 5.8 对状态最终一致性的影响

**一致性模型**：最终一致性（Eventual Consistency）
- **收敛时间**：取决于全量同步触发频率（应用启动时）
- **不一致窗口**：从事件丢失到下次全量同步的时间
- **用户感知**：备份计数显示不准确，可能显示"还有 X 项待备份"但实际上已全部上传成功

**不一致场景链**：
```
1. 用户上传 100 张照片
   ↓
2. HTTP 全部成功 → 内存 backupCount = 100
   ↓
3. WebSocket 推送 100 个 AssetUploadReady 事件
   ↓
4. 并发场景触发 → 丢失 20 个事件
   ↓
5. remote_asset_entity 只有 80 条记录
   ↓
6. getBackupStatus() 查询显示 remainderCount = 20
   ↓
7. 用户看到"还有 20 项待备份"，困惑为什么一直在"上传"
   ↓
8. 用户重启应用 → 全量同步 → 补全 20 条记录 → 状态对齐
```

### 5.9 现有保护点总结

| 保护机制 | 位置 | 作用 |
|---------|------|------|
| `_syncWebsocketTask` 单任务 | `background_sync.dart:190-207` | 防止并发写入数据库（但会丢弃新数据） |
| `maxWaitTime` 最大等待 | `debounce.dart:20-24` | 防止批处理无限延迟 |
| 类型检查 `continue` | `sync_stream.service.dart:340-358` | 防止类型不匹配导致崩溃 |
| try-catch 异常捕获 | `sync_stream.service.dart:338-368` | 防止批处理失败导致应用崩溃 |
| `onConflict: DoUpdate` | `sync_stream.repository.dart:215` | 重复写入时更新而非报错 |
| 全量同步兜底 | `background_sync.dart:160-187` | 应用启动时补全所有缺失数据 |

### 5.10 编辑事件与上传批事件的互相阻塞分析

#### 5.10.1 共用并发门闩的设计

**编辑事件处理路径**（无防抖，直接执行） (`websocket.provider.dart:182-188`):
```dart
void _handleSyncAssetEditReadyV1(dynamic data) {
  unawaited(_ref.read(backgroundSyncProvider).syncWebsocketEditV1(data));
}

void _handleSyncAssetEditReadyV2(dynamic data) {
  unawaited(_ref.read(backgroundSyncProvider).syncWebsocketEditV2(data));
}
```

**关键发现**：编辑事件（AssetEditReadyV1/V2）**没有防抖机制**，收到后立即调用 `syncWebsocketEditV1/V2()`。

**共用并发门闩** (`background_sync.dart:189-227`):
```dart
Cancelable<void>? _syncWebsocketTask;  // ⚠️ 所有事件类型共享同一个任务变量！

// 上传批事件 V1
Future<void> syncWebsocketBatchV1(List<dynamic> batchData) {
  if (_syncWebsocketTask != null) {
    return _syncWebsocketTask!.future;  // 有任务在执行，直接丢弃新数据
  }
  _syncWebsocketTask = _handleWsAssetUploadReadyV1Batch(batchData);
  return _syncWebsocketTask!.whenComplete(() {
    _syncWebsocketTask = null;
  });
}

// 上传批事件 V2
Future<void> syncWebsocketBatchV2(List<dynamic> batchData) {
  if (_syncWebsocketTask != null) {
    return _syncWebsocketTask!.future;  // 有任务在执行，直接丢弃新数据
  }
  _syncWebsocketTask = _handleWsAssetUploadReadyV2Batch(batchData);
  return _syncWebsocketTask!.whenComplete(() {
    _syncWebsocketTask = null;
  });
}

// 编辑事件 V1
Future<void> syncWebsocketEditV1(dynamic data) {
  if (_syncWebsocketTask != null) {
    return _syncWebsocketTask!.future;  // 有任务在执行，直接丢弃新数据
  }
  _syncWebsocketTask = _handleWsAssetEditReadyV1(data);
  return _syncWebsocketTask!.whenComplete(() {
    _syncWebsocketTask = null;
  });
}

// 编辑事件 V2
Future<void> syncWebsocketEditV2(dynamic data) {
  if (_syncWebsocketTask != null) {
    return _syncWebsocketTask!.future;  // 有任务在执行，直接丢弃新数据
  }
  _syncWebsocketTask = _handleWsAssetEditReadyV2(data);
  return _syncWebsocketTask!.whenComplete(() {
    _syncWebsocketTask = null;
  });
}
```

**核心问题**：
1. **单个任务变量**：`_syncWebsocketTask` 被所有 4 个事件类型（上传 V1/V2、编辑 V1/V2）共享
2. **直接丢弃策略**：`_syncWebsocketTask != null` 时，新数据被直接丢弃，无队列、无延迟消费
3. **编辑事件无防抖**：编辑事件高频到达时更容易触发冲突

#### 5.10.2 时序 1：编辑事件在先，上传批事件在后

```
时序图：
T0: 编辑事件 E1 到达 → syncWebsocketEditV1(E1)
    → _syncWebsocketTask == null
    → _syncWebsocketTask = futureEdit(E1)
    → 开始在 Isolate 中执行编辑操作（更新 remote_asset_entity）

T1: (100ms 后) 上传批事件 U1 防抖触发 → syncWebsocketBatchV1([A,B,C])
    → 检测到 _syncWebsocketTask != null (futureEdit 仍在执行)
    → 直接返回 futureEdit，**丢弃 [A,B,C]**
    → _batchedAssetUploadReady 已被 clear()

T2: futureEdit(E1) 完成 → _syncWebsocketTask = null

结果：
- ✅ 编辑事件 E1 成功执行
- ❌ 上传批事件 [A,B,C] 被永久丢弃
- ❌ 无日志、无重试
```

**实际结果**：上传批事件被**丢弃**，不是延迟消费。

#### 5.10.3 时序 2：上传批事件在先，编辑事件在后

```
时序图：
T0: 上传批事件 U1 防抖触发 → syncWebsocketBatchV1([A,B,C])
    → _syncWebsocketTask == null
    → _syncWebsocketTask = futureBatch([A,B,C])
    → 开始在 Isolate 中执行批量插入（可能有 100+ 条记录，耗时较长）

T1: (100ms 后) 编辑事件 E1 到达 → syncWebsocketEditV1(E1)
    → 检测到 _syncWebsocketTask != null (futureBatch 仍在执行)
    → 直接返回 futureBatch，**丢弃 E1**

T2: futureBatch([A,B,C]) 完成 → _syncWebsocketTask = null

结果：
- ✅ 上传批事件 [A,B,C] 成功执行
- ❌ 编辑事件 E1 被永久丢失
- ❌ 用户在 Web 端修改的元数据（如标题、描述、地理位置）不会同步到移动端
```

**实际结果**：编辑事件被**丢弃**，不是延迟消费。

#### 5.10.4 时序 3：高频编辑事件连续到达

```
时序图：
T0: 编辑事件 E1 到达 → syncWebsocketEditV1(E1)
    → _syncWebsocketTask = futureEdit(E1)

T1: (50ms 后) 编辑事件 E2 到达 → syncWebsocketEditV1(E2)
    → _syncWebsocketTask != null
    → 直接返回 futureEdit(E1)，**丢弃 E2**

T2: (50ms 后) 编辑事件 E3 到达 → syncWebsocketEditV1(E3)
    → _syncWebsocketTask != null
    → 直接返回 futureEdit(E1)，**丢弃 E3**

T3: futureEdit(E1) 完成 → _syncWebsocketTask = null

结果：
- ✅ E1 成功执行
- ❌ E2、E3 被永久丢失
- ❌ 用户的连续编辑操作只有第一个生效
```

**实际结果**：后续编辑事件被**丢弃**。

### 5.11 触发条件矩阵

| 触发场景 | 先到达事件 | 后到达事件 | 实际结果 | 现有保护点 | 缺口 | 对失败恢复的影响 | 对最终一致性的影响 |
|---------|-----------|-----------|---------|-----------|------|-----------------|------------------|
| **场景 1**：上一批未完成 | 上传批 V1 | 上传批 V1 | ❌ 新批被丢弃 | ✅ 防止并发写入数据库 | ❌ 新数据静默丢失 | 无影响（全量同步兜底） | 不一致直到全量同步 |
| **场景 2**：上一批未完成 | 上传批 V1 | 上传批 V2 | ❌ 新批被丢弃 | ✅ 防止并发写入数据库 | ❌ V1/V2 共享任务变量 | 无影响（全量同步兜底） | 不一致直到全量同步 |
| **场景 3**：交替到达 | 上传批 V1 | 编辑 V1 | ❌ 编辑被丢弃 | ✅ 防止并发写入数据库 | ❌ 编辑无重试机制 | ❌ 用户编辑丢失，需手动重新编辑 | ❌ 元数据不一致，需全量同步 |
| **场景 4**：交替到达 | 编辑 V1 | 上传批 V1 | ❌ 上传批被丢弃 | ✅ 防止并发写入数据库 | ❌ 上传批无重试机制 | 无影响（全量同步兜底） | 不一致直到全量同步 |
| **场景 5**：交替到达 | 编辑 V1 | 编辑 V2 | ❌ 后编辑被丢弃 | ✅ 防止并发写入数据库 | ❌ 编辑无防抖、无队列 | ❌ 用户编辑丢失，需手动重新编辑 | ❌ 元数据不一致，需全量同步 |
| **场景 6**：V1/V2 混合队列 | 上传批 V1 | 上传批 V2 | ⚠️ 部分事件被跳过 | ✅ 类型检查 continue | ❌ V1/V2 共享队列 | 无影响（全量同步兜底） | 不一致直到全量同步 |
| **场景 7**：批处理异常 | 任意 | 任意 | ❌ 整批丢失 | ✅ 异常捕获不崩溃 | ❌ 无重试机制 | 无影响（全量同步兜底） | 不一致直到全量同步 |
| **场景 8**：高频编辑 | 编辑 V1 | 编辑 V1 | ❌ 后续编辑被丢弃 | ✅ 防止并发写入数据库 | ❌ 编辑无防抖、无队列 | ❌ 用户编辑丢失 | ❌ 元数据不一致 |

### 5.12 缺口总结（含编辑事件）

| 缺口 | 风险 | 严重程度 |
|------|------|---------|
| 新批数据静默丢弃 | 事件永久丢失（直到全量同步） | ⚠️ 中 |
| V1/V2 共享队列和任务变量 | 事件类型与处理器不匹配 | ⚠️ 中 |
| 批处理失败无重试 | 整批事件丢失 | ⚠️ 中 |
| 部分成功无断点续处理 | 部分事件丢失 | ⚠️ 低 |
| 无事件持久化队列 | 应用重启后丢失未处理事件 | ⚠️ 低（全量同步兜底） |
| 无丢弃事件日志 | 问题难以排查 | ⚠️ 低 |
| **编辑事件与上传事件共享门闩** | **编辑事件可能被上传阻塞丢弃** | ⚠️ **高**（用户编辑丢失） |
| **编辑事件无防抖** | **高频编辑连续丢弃** | ⚠️ **高**（用户编辑丢失） |
| **编辑事件无重试** | **丢弃后无恢复机制** | ⚠️ **高**（用户编辑丢失） |

---

## 6. 失败恢复机制

### 6.1 失败分类与停止条件

| 失败类型 | 触发条件 | 停止队列？ | 处理策略 |
|---------|---------|-----------|---------|
| **用户取消** | `cancelToken.complete()` | ✅ 立即 | 所有 worker 检测到后退出循环 |
| **配额超限** | 服务端返回 "Quota has been exceeded!" | ✅ 立即 | 设置 `shouldAbortUpload = true`，剩余资产跳过 |
| **文件过大 (413)** | 服务端返回 413 错误 | ❌ 不停止 | 标记单资产失败，继续下一个 |
| **网络错误** | 连接超时、DNS 失败等 | ❌ 不停止 | 标记单资产失败，继续下一个 |
| **文件不存在** | 本地资产已被删除 | ❌ 不停止 | 标记单资产失败，继续下一个 |
| **iCloud 下载失败** | iOS 云端资产无法加载 | ❌ 不停止 | 标记单资产失败，继续下一个 |

**关键代码** (`foreground_upload.service.dart:390-406`):
```dart
if (result.isSuccess && result.remoteAssetId != null) {
  callbacks.onSuccess?.call(asset.localId!, result.remoteAssetId!);
} else if (result.isCancelled) {
  _logger.warning(() => "Backup was cancelled by the user");
  shouldAbortUpload = true;  // ✅ 停止队列
} else if (result.errorMessage != null) {
  callbacks.onError?.call(asset.localId!, result.errorMessage!);
  
  if (result.errorMessage == "Quota has been exceeded!") {
    shouldAbortUpload = true;  // ✅ 只有配额超限才停止队列
  }
  // ❌ 413 文件过大等其他错误不停止队列
}
```

**队列停止检测点** (`foreground_upload.service.dart:211-213`):
```dart
Future<void> worker() async {
  while (true) {
    // 每次循环开始检测是否应该停止
    if (shouldAbortUpload || (cancelToken != null && cancelToken.isCompleted)) {
      break;
    }
    // ... 处理下一个资产
  }
}
```

> **重要校正**：之前的分析错误地认为文件过大 (413) 会停止队列，实际上只有"配额超限"会停止整个上传队列。文件过大、网络错误等都只影响单个资产。

### 6.2 前台上传失败处理

**失败回调** (`drift_backup.provider.dart:345-374`):
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

**失败恢复边界**：
- ✅ 失败记录保存在 `DriftBackupState.uploadItems` 内存中
- ❌ 失败记录不持久化到数据库
- ❌ 没有自动重试机制
- ✅ 用户手动重新触发备份时，已成功的资产通过 checksum 去重不会重复上传
- ✅ 失败的资产会被重新尝试上传

### 6.3 后台上传重试机制

**任务配置** (`background_upload.service.dart:424-441`):
```dart
return UploadTask(
  retries: 3,           // ✅ background_downloader 自动重试 3 次
  updates: Updates.statusAndProgress,
  // ...
);
```

**后台上传失败处理** (`background_upload.service.dart:211-230`):
```dart
void _handleTaskStatusUpdate(TaskStatusUpdate update) async {
  switch (update.status) {
    case TaskStatus.complete:
      unawaited(_handleLivePhoto(update));  // 只处理 Live Photo 第二阶段
      // ⚠️ 注意：普通资产上传成功后，这里不更新任何状态！
      // 状态完全依赖 Sync Stream 最终确认
      break;
    // 其他状态（失败、重试等）没有特殊处理
    default:
      break;
  }
}
```

> **重要发现**：后台上传成功后，除了 Live Photo 需要触发第二阶段上传外，普通资产不会更新任何本地状态。所有状态更新完全依赖 WebSocket 事件或全量同步。

### 6.4 断点续传能力

| 能力 | 支持情况 | 说明 |
|------|---------|------|
| 应用重启后继续后台任务 | ✅ 支持 | iOS URLSession 系统级支持 |
| 文件级断点续传 | ❌ 不支持 | 每次失败后重新上传整个文件 |
| 资产级去重 | ✅ 支持 | 通过 checksum 避免重复上传 |
| 失败持久化 | ❌ 不支持 | 失败记录仅在内存中 |
| 失败自动重试 | ⚠️ 部分支持 | 后台上传自动重试3次，前台上传不自动重试 |

**去重逻辑** (`backup.repository.dart:84-116`):
```dart
Future<List<LocalAsset>> getCandidates(String userId, {bool onlyHashed = true}) async {
  final query = _db.localAssetEntity.select()
    ..where(
      (lae) =>
        existsQuery(...) &  // 在选中相册中
        notExistsQuery(     // 远程不存在相同 checksum
          _db.remoteAssetEntity.selectOnly()
            ..addColumns([_db.remoteAssetEntity.checksum])
            ..where(
              _db.remoteAssetEntity.checksum.equalsExp(lae.checksum) &
              _db.remoteAssetEntity.ownerId.equals(userId),
            ),
        ) &
        lae.id.isNotInQuery(_getExcludedSubquery()),
    );
  // ...
}
```

---

## 7. 前后端状态对齐机制

### 7.1 状态对齐架构

```
┌─────────────────────────────────────────────────────────────────────────┐
│                         移动端 (Client)                                  │
│                                                                          │
│  ┌────────────────────┐    ┌────────────────────┐    ┌────────────────┐ │
│  │  local_asset_entity │───▶│ DriftBackupState   │───▶│ remote_asset_  │ │
│  │  (本地资产清单)     │    │  (内存状态)        │    │ entity (远程   │ │
│  │  - checksum         │    │  - backupCount     │    │  资产缓存)     │ │
│  │  - createdAt        │    │  - remainderCount  │    │  - checksum    │ │
│  │                     │    │  - uploadItems     │    │  - ownerId     │ │
│  └─────────▲──────────┘    └──────────▲─────────┘    └────────┬───────┘ │
│            │                        │                           │         │
│            │ PhotoManager 扫描      │ HTTP 回调                │ Sync    │
│            │                        │                           │ Stream  │
│            │                        │                           ▼         │
└────────────┼────────────────────────┼─────────────────────────────────────┘
             │                        │                             │
             │                        │                             │
┌────────────┴────────────────────────┴─────────────────────────────┴───────┐
│                         服务端 (Server)                                    │
│                                                                           │
│  ┌──────────────┐     ┌──────────────┐     ┌──────────────┐     ┌────────┐ │
│  │  文件存储    │────▶│  PostgreSQL  │────▶│  Sync Stream │────▶│ WebSocket │
│  │  (对象存储)  │     │  (元数据)    │     │  (事件流)    │     │  (推送)   │ │
│  └──────────────┘     └──────────────┘     └──────────────┘     └────────┘ │
└───────────────────────────────────────────────────────────────────────────┘
```

### 7.2 四重状态对齐机制

| 对齐机制 | 触发时机 | 覆盖范围 | 延迟 | 可靠性 |
|---------|---------|---------|------|-------|
| **HTTP 回调乐观更新** | 每个资产上传成功 | 内存计数 | 即时 | ⚠️ 不可靠（可能与数据库不一致） |
| **WebSocket 批处理** | 服务端处理完成后推送 | 远程资产表 | 5-10s（防抖） | ⚠️ 中等（并发场景可能丢失） |
| **全量同步** | 应用启动 / 手动触发 | 全量远程数据 | 分钟级 | ✅ 可靠 |
| **备份统计查询** | 备份页面打开 / 刷新 | 重新计算所有计数 | 即时 | ✅ 可靠 |

### 7.3 本地数据库表结构

**local_asset_entity** - 本地资产清单（无备份状态字段）
- `id`: 本地资产唯一标识
- `checksum`: 文件哈希（用于关联远程资产）
- `cloudId`: iOS 云端标识（iCloud）
- `createdAt`, `updatedAt`: 文件时间戳
- **注意**：没有 `isBackedUp` 或类似字段，备份状态完全通过关联查询判断

**remote_asset_entity** - 远程资产缓存（Sync Stream 写入）
- `id`: 服务端资产 UUID
- `checksum`: 文件哈希（关联本地资产的关键）
- `ownerId`: 所属用户 ID
- `uploadedAt`: 上传时间
- `deletedAt`: 删除标记（软删除）
- `stackId`: 相册堆 ID

**local_album_entity** - 相册配置
- `backupSelection`: `selected` / `excluded` / `none`

### 7.4 状态查询 API

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
- `backupCount`: `totalCount - remainderCount`（内存中维护，非数据库查询）
- `remainderCount`: 远程不存在的资产数（待上传）
- `processingCount`: checksum 为 NULL 的资产数（正在计算哈希）

> **关键发现**：`DriftBackupState.backupCount` 是内存维护的增量计数器，而 `getBackupStatus()` 返回的是数据库查询结果。两者可能在 HTTP 成功后、WebSocket 落库前出现短暂不一致。

### 7.5 一致性保障机制

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

4. **备份页面重新查询**
   - 每次打开备份页面调用 `getBackupStatus()`
   - 从数据库重新计算所有统计
   - 修正内存计数可能的偏差

---

## 8. Live Photo 特殊处理

### 8.1 两阶段上传

iOS Live Photo 包含照片 + 视频两个文件，需分开上传：

```
阶段 1：上传视频部分（普通优先级）
  getUploadTask(asset) → 提取 motion file
    ↓
  后台上传成功 → 返回 remoteAssetId
    ↓
  _handleLivePhoto() → 构建第二阶段任务

阶段 2：上传照片部分（高优先级）
  getLivePhotoUploadTask(asset, livePhotoVideoId)
    ↓
  fields['livePhotoVideoId'] = videoId
    ↓
  上传照片，关联视频 ID（priority=0, 最高优先级）
```

### 8.2 任务分组与优先级

`background_upload.service.dart:274-283`:
```dart
/// iOS LivePhoto 两阶段上传：
/// 1. 视频文件：普通优先级组 (kBackupGroup)
/// 2. 照片文件：高优先级组 (kBackupLivePhotoGroup, priority=0)
/// 取消操作只取消视频组，照片组不受影响（视频已上传）
```

**不同组的取消行为** (`background_upload.service.dart:195-204`):
```dart
Future<int> cancel() async {
  shouldAbortQueuingTasks = true;
  await _storageRepository.clearCache();
  await _uploadRepository.reset(kBackupGroup);           // 只重置普通备份组
  await _uploadRepository.deleteDatabaseRecords(kBackupGroup);  // 只清理普通备份组
  // ⚠️ Live Photo 照片组 (kBackupLivePhotoGroup) 不会被取消
  final activeTasks = await _uploadRepository.getActiveTasks(kBackupGroup);
  return activeTasks.length;
}
```

---

## 9. 网络条件适配

### 9.1 WiFi / 蜂窝网络控制

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

### 9.2 后台上传网络约束

`background_upload.service.dart:437`:
```dart
return UploadTask(
  requiresWiFi: requiresWiFi,  // 系统级网络约束
  // ...
);
```

iOS URLSession 会自动遵守此约束，在不满足网络条件时暂停。

---

## 10. 取消机制

### 10.1 前台上传取消

`drift_backup.provider.dart:281-286`:
```dart
void stopForegroundBackup() {
  _cancelToken?.complete();      // 触发取消信号
  _cancelToken = null;
  _uploadSpeedManager.clear();   // 清理速度统计
  state = state.copyWith(uploadItems: {}, iCloudDownloadProgress: {});
}
```

**取消传播链**:
1. `Completer.complete()` 触发 future 完成
2. `ProgressMultipartRequest` 监听 `abortTrigger` future
3. 抛出 `RequestAbortedException` 终止当前上传
4. Worker 池检测到 `cancelToken.isCompleted` 退出循环

### 10.2 后台上传取消

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

> **注意**：后台上传取消只影响 `kBackupGroup` 组，`kBackupLivePhotoGroup`（Live Photo 照片）不会被取消。

---

## 11. 各组件衔接关系全景

### 11.1 完整数据流图

```
用户触发备份
    │
    ▼
┌──────────────────────────────────────────────────────────────────┐
│ DriftBackupNotifier.startForegroundBackup()                     │
│  ├─ 创建 cancelToken                                            │
│  └─ 调用 ForegroundUploadService.uploadCandidates()              │
└───────────────────────────────────┬──────────────────────────────┘
                                    │
    ┌───────────────────────────────┘
    │
    ▼
┌──────────────────────────────────────────────────────────────────┐
│ ForegroundUploadService._executeWithWorkerPool()                 │
│  ├─ 3 个并发 Worker                                              │
│  └─ 每个 Worker 调用 _uploadSingleAsset()                        │
└───────────────────────────────────┬──────────────────────────────┘
                                    │
    ┌───────────────────────────────┘
    │
    ▼
┌──────────────────────────────────────────────────────────────────┐
│ ForegroundUploadService._uploadSingleAsset()                     │
│  ├─ 检查本地文件 / iCloud 下载                                    │
│  ├─ 调用 UploadRepository.uploadFile()                           │
│  ├─ 成功 → callbacks.onSuccess()                                 │
│  ├─ 失败 → callbacks.onError()                                   │
│  └─ 配额超限 → 设置 shouldAbortUpload = true                     │
└───────────────────────────────────┬──────────────────────────────┘
                                    │
          ┌─────────────────────────┴─────────────────────────┐
          │                                                   │
          ▼                                                   ▼
┌──────────────────────────────┐                   ┌──────────────────────────────┐
│ _handleForegroundBackupSuccess │                   │ _handleForegroundBackupError  │
│  ├─ backupCount + 1            │                   │  ├─ 标记 isFailed = true      │
│  ├─ remainderCount - 1         │                   │  └─ 保存 errorMessage        │
│  └─ 延迟 1s 移除上传项          │                   └──────────────────────────────┘
└───────────────┬────────────────┘
                │
                ▼  [ 时间窗口：HTTP 成功，WebSocket 未落库 ]
                │
                ▼
┌──────────────────────────────────────────────────────────────────┐
│ 服务端处理资产（转码、缩略图、元数据提取）                        │
└───────────────────────────────────┬──────────────────────────────┘
                                    │
                                    ▼
┌──────────────────────────────────────────────────────────────────┐
│ WebSocket 推送 AssetUploadReadyV1/V2 事件                        │
└───────────────────────────────────┬──────────────────────────────┘
                                    │
    ┌───────────────────────────────┘
    │
    ▼
┌──────────────────────────────────────────────────────────────────┐
│ WebsocketNotifier._handleSyncAssetUploadReadyV1/V2()             │
│  ├─ 加入 _batchedAssetUploadReady 队列                           │
│  └─ Debouncer 防抖（5s / 10s）                                   │
└───────────────────────────────────┬──────────────────────────────┘
                                    │
    ┌───────────────────────────────┘
    │
    ▼
┌──────────────────────────────────────────────────────────────────┐
│ BackgroundSyncManager.syncWebsocketBatchV1/V2()                  │
│  └─ runInIsolateGentle() 隔离执行                                │
└───────────────────────────────────┬──────────────────────────────┘
                                    │
    ┌───────────────────────────────┘
    │
    ▼
┌──────────────────────────────────────────────────────────────────┐
│ SyncStreamService.handleWsAssetUploadReadyV1Batch()              │
│  ├─ 解析 SyncAssetV1 + SyncAssetExifV1                           │
│  └─ 调用 SyncStreamRepository.updateAssetsV1()                    │
└───────────────────────────────────┬──────────────────────────────┘
                                    │
    ┌───────────────────────────────┘
    │
    ▼
┌──────────────────────────────────────────────────────────────────┐
│ SyncStreamRepository.updateAssetsV1()                            │
│  ├─ batch insert 到 remote_asset_entity                          │
│  └─ onConflict: DoUpdate（冲突则更新）                            │
└──────────────────────────────────────────────────────────────────┘
                                    │
                                    ▼  [ 状态最终一致 ]
```

### 11.2 失败恢复边界

```
前台上传失败
    │
    ├─ isFailed: true（内存状态）
    ├─ 不写入数据库
    ├─ 不自动重试
    └─ 用户重新备份 → 重新查询候选 → 已成功的 checksum 去重 → 失败的重新上传

后台上传失败
    │
    ├─ background_downloader 自动重试 3 次
    ├─ 失败后任务状态持久化到系统数据库
    ├─ 应用重启后可查询 pending tasks
    └─ 调用 resume() 继续

WebSocket 批事件丢失
    │
    ├─ 并发场景：上一批未完成时新批被丢弃
    ├─ 并发场景：V1/V2 交替到达类型不匹配
    ├─ 并发场景：批处理异常失败
    └─ 下次全量同步时补全

应用冷启动
    │
    ├─ 执行全量同步 syncRemote()
    ├─ 同步所有远程资产到 remote_asset_entity
    └─ 状态完全对齐
```

---

## 12. 总结

### 12.1 设计亮点

1. **双层上传架构**：前台并发 + 后台保活，兼顾速度和可靠性
2. **Checksum 关联机制**：无需保存上传记录，通过文件哈希自然去重
3. **三阶段确认**：HTTP 即时反馈 + WebSocket 批处理 + 全量同步兜底
4. **滑动窗口速度计算**：平滑的上传速度和剩余时间估算
5. **Live Photo 优雅处理**：两阶段上传，高优先级确保完整性
6. **任务隔离执行**：使用 Isolate 避免阻塞 UI 线程
7. **最大等待时间防抖**：防止批处理无限延迟

### 12.2 可改进点

1. **缺乏真正的断点续传**：大文件上传失败需重新开始
2. **失败资产无持久化队列**：应用重启后丢失失败记录
3. **内存计数与数据库查询不一致**：HTTP 成功后到 WebSocket 落库前存在时间窗口
4. **并发数固定**：未根据网络条件动态调整并发数
5. **前台上传无自动重试**：网络临时故障导致的失败需用户手动重试
6. **WebSocket 批事件并发安全问题**：
   - V1/V2 共享队列和任务变量，可能导致事件丢失
   - 上一批未完成时新批被静默丢弃
   - 批处理失败无重试机制
   - 部分成功部分失败无断点续处理

### 12.3 关键校正点

| 校正项 | 之前结论 | 校正后结论 |
|--------|---------|-----------|
| 文件过大 (413) 是否停止队列 | 是 | ❌ 否，只有配额超限才停止队列 |
| HTTP 成功后是否写入数据库 | 是 | ❌ 否，只更新内存计数，数据库写入依赖 Sync Stream |
| 后台上传成功后是否更新状态 | 是 | ❌ 否，除 Live Photo 外，普通资产不更新任何状态 |
| backupCount 来源 | 数据库查询 | ❌ 内存增量计数器，非数据库查询结果 |
| WebSocket 批处理并发安全 | 未提及 | ❌ 存在多个并发边界问题，可能导致事件丢失 |

### 12.4 WebSocket 并发边界关键结论

1. **上一批未完成时新批会被静默丢弃**：`_syncWebsocketTask` 非空时直接返回现有 future，新数据被丢弃，队列同时被清空
2. **V1/V2 事件互相阻塞**：共享同一个 `_syncWebsocketTask` 变量，V1 执行时 V2 事件被丢弃，反之亦然
3. **V1/V2 共享队列可能导致类型不匹配**：交替到达时，最后一个处理器会处理混合队列，不匹配的事件被静默跳过
4. **所有并发问题最终依赖全量同步兜底**：如果用户长时间不重启应用，事件可能永久丢失
5. **批处理失败无重试**：异常被捕获后没有重试机制，整批事件丢失

### 12.5 关键文件速查

| 功能 | 文件 | 行号范围 |
|------|------|---------|
| 上传进度回调 | `upload.repository.dart` | 153-181 |
| 上传状态管理 | `drift_backup.provider.dart` | 103-398 |
| 前台上传服务 | `foreground_upload.service.dart` | 81-467 |
| 后台上传服务 | `background_upload.service.dart` | 98-442 |
| 速度计算 | `upload_speed_calculator.dart` | 1-182 |
| 同步流服务 | `sync_stream.service.dart` | 30-543 |
| 备份统计查询 | `backup.repository.dart` | 39-116 |
| WebSocket 批处理 | `websocket.provider.dart` | 43-231 |
| 后台同步管理 | `background_sync.dart` | 1-278 |
| 同步流落库 | `sync_stream.repository.dart` | 196-272 |
| 防抖器实现 | `debounce.dart` | 1-64 |
| WebSocket 并发任务控制 | `background_sync.dart` | 189-227 |
| 编辑事件处理（无防抖） | `websocket.provider.dart` | 182-188 |
| 编辑事件落库 | `sync_stream.service.dart` | 414-487 |
