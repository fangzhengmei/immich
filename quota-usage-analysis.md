# Immich 配额与用量分析

本文以代码为准，把 "配额怎么算"、"什么时候加/减"、"什么时候报警" 这三个问题串起来说明。配额字段存元数据里（`user` 表），真正的文件占用属于存储层（磁盘上的原图、缩略图、编码视频等），但 Immich 只把**原图字节数**计入用户配额，统计口径在元数据侧（`asset_exif.fileSizeInByte`）与用户表的缓存字段（`user.quotaUsageInBytes`）之间来回维护。

## 1. 数据模型与字段来源

### 1.1 `user` 表上的两个核心字段

文件：`server/src/schema/tables/user.table.ts`

| 字段 | 类型 | 含义 |
| --- | --- | --- |
| `quotaSizeInBytes` | `bigint \| null` | 配额上限。`null` 表示不限额。 |
| `quotaUsageInBytes` | `bigint` | 已用字节数，运行时缓存值，默认 `0`。 |

两者都参与鉴权上下文：在 `server/src/database.ts:342` 把这两个字段加入 `authUser` 投影；任何通过 session / API key / shared link 解析出的 `auth.user` 都会带着这两个字段，供服务代码随时访问。

### 1.2 真实口径：`asset_exif.fileSizeInByte`

`quotaUsageInBytes` 只是一个缓存值。权威口径是所有 `asset` 记录（`deletedAt IS NULL`、`libraryId IS NULL`）对应的 `asset_exif.fileSizeInByte` 之和。
见 `server/src/repositories/user.repository.ts:310` 的 `syncUsage()`：

```sql
UPDATE "user"
SET "quotaUsageInBytes" = (
    SELECT COALESCE(SUM(asset_exif."fileSizeInByte"), 0)
    FROM asset
    LEFT JOIN asset_exif ON asset_exif."assetId" = asset.id
    WHERE asset."libraryId" IS NULL
      AND asset."ownerId" = "user".id
),
"updatedAt" = now()
WHERE "user"."deletedAt" IS NULL;
```

**口径要点**：
- 只统计 `asset.libraryId IS NULL` 的资产（外部挂载 Library 的文件不计入用户配额）。
- 直接使用 `asset_exif.fileSizeInByte`，因此不区分图片/视频/sidecar/缩略图/转码产物——缩略图、转码视频等派生文件的字节数一律不占用配额，只有原始上传文件计入。

## 2. 配额的设置与来源

### 2.1 管理员手动设置

`server/src/services/user-admin.service.ts:53` 的 `update()`：

```ts
if (dto.quotaSizeInBytes && user.quotaSizeInBytes !== dto.quotaSizeInBytes) {
  await this.userRepository.syncUsage(id);
}
```

**关键点**：修改配额上限时会先触发一次按资产重算的 `syncUsage(id)`，把缓存值与真实值对齐后再应用新的上限，避免缓存漂移导致的"刚改完配额就提示超限"问题。

### 2.2 OAuth 自动注册时从 Claim 注入

`server/src/services/auth.service.ts:371`：

```ts
quotaSizeInBytes: storageQuota === null ? null : storageQuota * HumanReadableSize.GiB,
```

从 OAuth 配置的 `storageQuotaClaim` 读取，单位按 GiB 解析；`null` 表示不限额。

## 3. 用量累加：三条路径

`user.quotaUsageInBytes` 的增量更新只走一个方法：`UserRepository.updateUsage(id, delta)`，实现是一条原子 SQL：

```sql
UPDATE "user"
SET "quotaUsageInBytes" = "quotaUsageInBytes" + $1,
    "updatedAt" = now()
WHERE id = $2 AND "deletedAt" IS NULL;
```

（见 `server/src/repositories/user.repository.ts:300`）

调用 `updateUsage` 的地方只有三处，覆盖了所有"改变用户实际占用量"的场景：

| 触发点 | 位置 | 场景 | delta |
| --- | --- | --- | --- |
| `AssetMediaService.uploadAsset` | `server/src/services/asset-media.service.ts:155` | 通过上传接口成功创建资产 | `+file.size` |
| `MetadataService`（运动照片视频提取） | `server/src/services/metadata.service.ts:750` | 从 HEIC/JPEG Motion Photo 中拆出独立的 mp4 资产（`!asset.isExternal` 时） | `+video.byteLength` |
| `AssetService.handleAssetDeletion` | `server/src/services/asset.service.ts:334` | 后台任务真正执行资产删除（仅 `asset.libraryId IS NULL` 时） | `-(asset.exifInfo?.fileSizeInByte \|\| 0)` |

### 3.1 上传路径的完整链路

`AssetMediaService.uploadAsset`（`server/src/services/asset-media.service.ts:127`）的顺序非常关键：

1. `requireAccess`：权限检查。
2. `requireQuota(auth, file.size)`：**先校验**，见第 4 节。
3. `create(...)`：写 `asset` 表、写 `asset_exif.fileSizeInByte`、落盘、触发 `AssetExtractMetadata` 任务。
4. `updateUsage(auth.user.id, file.size)`：写成功后再累加缓存。

这样即使第 4 步之前抛错（磁盘满、校验重复等），缓存也不会提前累加。`handleUploadError` 会在失败时清理磁盘文件，但不反向扣减缓存——因为缓存还没加过。

### 3.2 删除时为何是在 `handleAssetDeletion` 里扣

- 用户侧 `POST /trash` 只是把 `asset.deletedAt` 置为当前时间，并触发 `AssetTrash`/`AssetTrashAll` 事件，**不扣用量**。
- 真正扣减发生在后台任务 `JobName.AssetDelete`（由 `AssetDeleteCheck` 或管理员强制删除派发），对应 `AssetService.handleAssetDeletion`。
- 只有 `asset.libraryId IS NULL` 才扣，与 `syncUsage` 的口径一致。
- 使用 `asset.exifInfo?.fileSizeInByte || 0` 兜底：exif 若缺失就按 0 处理，不致引发崩溃。

### 3.3 运动视频的特殊处理

`metadata.service.ts:729` 分支里，如果拆出的 motion asset 是新创建的，才会 `updateUsage`；如果按 checksum 已存在（`isAssetChecksumConstraint` 捕获），则复用老资产，不再重复累加，避免同一个视频被不同 motion photo 多次计入。

## 4. 配额校验与"超限提示"下发

配额校验只在上传入口做，并且是**前置同步校验**：

`server/src/services/asset-media.service.ts:366`

```ts
private requireQuota(auth: AuthDto, size: number) {
  if (auth.user.quotaSizeInBytes !== null
      && auth.user.quotaSizeInBytes < auth.user.quotaUsageInBytes + size) {
    throw new BadRequestException('Quota has been exceeded!');
  }
}
```

调用时机：`uploadAsset` 第 2 步，在落盘 / 写元数据之前。

### 4.1 校验用的是哪个值

- `auth.user.quotaUsageInBytes` 来自 `session.repository` / `api.key.repository` / `shared.link.repository` 里 `authUser` 投影在**请求开始时**从 `user` 表读到的缓存值。
- 因此这是**乐观检查**：在请求开始到 `updateUsage` 落库之间若有其他请求同时上传，理论上会出现竞态（两个请求都看到相同的 `quotaUsageInBytes`，都通过校验，都成功），最终缓存值会短暂大于上限，但下次 `syncUsage` 会把缓存拉回到真实值，后续上传将被拒。Immich 选择接受这种短暂越界，以避免在每次上传里加数据库事务。

### 4.2 错误响应结构

`BadRequestException('Quota has been exceeded!')` 会被全局异常过滤器捕获并标准化：

`server/src/middleware/global-exception.filter.ts:30` 的 `fromError()`：

```ts
if (error instanceof HttpException) {
  const status = error.getStatus();           // 400
  const response = error.getResponse();       // 'Quota has been exceeded!'
  const body = typeof response === 'string'
    ? { message: response }
    : { ...response };
  delete body['error'];
  delete body['statusCode'];
  return { status, body };
}
```

最终发往客户端的 HTTP 响应为：

```
HTTP/1.1 400 Bad Request
x-immich-correlation-id: <uuid>
Content-Type: application/json

{ "message": "Quota has been exceeded!" }
```

### 4.3 "提示"以什么形式下发

Immich 没有独立的"配额告警"事件/邮件/websocket 推送；所谓的"上限提示"就是上传接口抛出的 HTTP 400。代码里没有 `NotificationService` 监听 `AssetCreate` / `UserSyncUsage` 来发送配额邮件或推送，`NotificationService` 的 `onAssetTrash` / `onAssetDelete` 等 handler 也不涉及配额（`server/src/services/notification.service.ts:151` 起）。

因此"告警下发"的边界应理解为：**同步阻塞上传并返回 400**，没有异步通知通道。

## 5. 配额超限提示的端到端链路

从后端抛异常到用户看到提示，Web 端和移动端走了两条不同的路径。

### 5.1 Web 端（SvelteKit）处理流程

**触发条件**：用户在网页端拖拽或选择文件上传，且 `quotaUsageInBytes + file.size > quotaSizeInBytes`。

完整调用链：

```
用户选择文件
  ↓
fileUploadHandler()  web/src/lib/utils/file-uploader.ts:80
  ↓
uploadRequest<AssetMediaResponseDto>()  POST /assets
  ↓
SDK 抛出 HttpError  @oazapfts/runtime
  ↓
catch (error) { ... }  file-uploader.ts:255
  ↓
handleError(error, $t('errors.unable_to_upload_file'))
  web/src/lib/utils/handle-error.ts:38
    ├─ getServerErrorMessage(error)  handle-error.ts:4
    │   ├─ isHttpError(error)  packages/sdk/src/fetch-errors.ts:20
    │   ├─ data = JSON.parse(error.data)
    │   └─ return data.message  → "Quota has been exceeded!"
    ├─ 截断前 75 字符 + 后缀 "(Immich Server Error)"
    └─ toastManager.danger(errorMessage)  显示红色 toast
  ↓
uploadAssetsStore.updateItem(id, { state: UploadState.ERROR, error: errorMessage })
  ↓
UploadAssetPreview.svelte  显示红色错误图标 + 错误文案 + 重试按钮
```

**关键代码解读**：

- `handle-error.ts:48` 的 `getServerErrorMessage()` 优先取 `error.data.message`，这正是后端返回的 `"Quota has been exceeded!"`。
- `handle-error.ts:50` 做了长度截断并标注 `(Immich Server Error)`，最终 toast 显示：
  ```
  Quota has been exceeded!
  (Immich Server Error)
  ```
- `file-uploader.ts:262` 把错误消息同时写入上传状态，用户在上传面板里可以看到每个失败文件的错误原因。
- Web 端**不做字符串匹配**，配额超限错误和其他上传错误（如网络错误、文件格式错误）走同一套展示逻辑。

### 5.2 移动端（Flutter）处理流程

**触发条件**：App 开启自动备份，后台或前台上传时遇到配额超限。移动端有特殊的逻辑来识别配额超限并中止整个备份队列。

完整调用链：

```
DriftBackupNotifier.startForegroundBackup(userId)
  drift_backup.provider.dart:259
  ↓
ForegroundUploadService.uploadCandidates(userId, cancelToken, callbacks)
  foreground_upload.service.dart:77
  ↓
_executeWithWorkerPool(
    processItem: (asset) => _uploadSingleAsset(asset, cancelToken, callbacks)
  )
  foreground_upload.service.dart:95
  ↓
_uploadSingleAsset(asset, cancelToken, callbacks)
  foreground_upload.service.dart:235
  ├─ [Live Photo] 先上传视频: _uploadRepository.uploadFile(livePhotoFile)
  │    └─ 失败时静默忽略（不设置 shouldAbortUpload）
  └─ [仍像] 上传主文件: _uploadRepository.uploadFile(file)
       foreground_upload.service.dart:375
  ↓
UploadRepository.uploadFile(file, ...)
  upload.repository.dart:91
  ↓
ProgressMultipartRequest → NetworkRepository.client.send()
  ↓
response.statusCode != 200/201  upload.repository.dart:117
  ↓
jsonDecode(responseBody)['message']  → "Quota has been exceeded!"
  upload.repository.dart:126-127
  ↓
return UploadResult.error(statusCode: 400, errorMessage: message)
  ↓
回到 _uploadSingleAsset()  foreground_upload.service.dart:386
  ├─ result.isSuccess → callbacks.onSuccess
  ├─ result.isCancelled → shouldAbortUpload = true
  └─ result.errorMessage != null:
       callbacks.onError(asset.localId!, errorMessage)  → UI 展示
       if (result.errorMessage == "Quota has been exceeded!")  关键字符串匹配
         shouldAbortUpload = true  → 中止 worker 循环
```

**关键代码解读**：

- `upload.repository.dart:126` 解析响应体时用 `error['message'] ?? error['error']` 提取错误信息。
- `foreground_upload.service.dart:399` 对配额超限做了**硬编码字符串匹配**：
  ```dart
  if (result.errorMessage == "Quota has been exceeded!") {
    shouldAbortUpload = true;
  }
  ```
- 匹配成功后设置 `shouldAbortUpload = true`，`_executeWithWorkerPool` 的 worker 循环（`foreground_upload.service.dart:207`）会在处理完当前资产后检查该标志并 break，不再继续上传剩余文件。
- `_uploadSequentially`（`foreground_upload.service.dart:117`）同样在循环开头检查 `shouldAbortUpload`，效果一致。
- 这是**唯一一处对特定错误消息做分支逻辑**的地方——后端 `BadRequestException` 的 message 文本如果改动，会直接破坏移动端中止备份的能力。

### 5.3 多语言文案与本地化

Immich 的 i18n 文件（`i18n/*.json`）中**没有独立的 key 对应配额超限错误**。相关的文案只有：

| key | 说明 |
| --- | --- |
| `storage_quota` | 设置页面的"存储配额"标签 |
| `has_quota` | 用户列表中的"配额大小"列标题 |
| `quota_size_gib` | 管理员编辑用户时的"配额大小（GiB）"字段 |
| `quota_higher_than_disk_size` | 管理员设置配额大于磁盘容量时的提示 |
| `nightly_tasks_sync_quota_usage_setting` | 夜间任务开关标签 |

真正的超限提示 `"Quota has been exceeded!"` 是后端硬编码的英文字符串，直接透传给客户端显示，不经过 i18n 翻译。Web 端会原样显示（附 `(Immich Server Error)` 后缀），移动端也原样显示并做字符串匹配。

### 5.4 移动端配额使用量展示

移动端在侧边栏（App Bar Dialog）实时展示配额进度，数据来源是 `AuthUser` entity 中的 `quotaSizeInBytes` 和 `quotaUsageInBytes`：

`mobile/lib/widgets/common/app_bar_dialog/app_bar_dialog.dart:147`

```dart
if (user != null && user.hasQuota) {
  usedDiskSpace = formatBytes(user.quotaUsageInBytes);
  totalDiskSpace = formatBytes(user.quotaSizeInBytes);
  percentage = user.quotaUsageInBytes / user.quotaSizeInBytes;
}
```

用 `LinearProgressIndicator` 展示进度条，下方显示 `X used of Y`。这是**被动展示**，进度条不会在即将超限时变红或弹出警告——只有真正发起上传被 400 拒绝时用户才会收到提示。

### 5.5 iOS 后台 URLSession 上传的配额超限处理（关键差异）

iOS 后台备份使用 `background_downloader: ^9.5.4`（`mobile/pubspec.yaml:14`），基于 iOS 原生 `URLSession` 进行异步上传，与前台上传的错误处理路径有本质区别。本节所有结论均明确标注**证据等级**：

| 证据等级 | 说明 |
| --- | --- |
| ✅ **代码事实** | 项目源码中直接可验证 |
| 📚 **SDK 文档事实** | `background_downloader` 9.5.4 官方 API 契约 |
| ⚠️ **合理推断** | 基于 SDK 行为惯例和现有代码逻辑推断 |

#### 5.5.1 事件流与消费链路

```
FileDownloader() 原生回调
  ↓  （upload.repository.dart:23-37）
UploadRepository.onUploadStatus 回调
  ↓  （background_upload.service.dart:108）
BackgroundUploadService._onUploadCallback(update)
  ├─ 向 taskStatusController.add(update) （broadcast stream）
  └─ _handleTaskStatusUpdate(update)  ← 唯一的业务处理入口
```

**代码事实 ✅**：
- `background_upload.service.dart:108` 在构造函数中绑定回调：`_uploadRepository.onUploadStatus = _onUploadCallback`
- `background_upload.service.dart:122` 暴露 `taskStatusStream` 是 `broadcast()` Stream，但**整个代码库中没有任何地方 `.listen()` 它**：
  ```bash
  $ grep -n "taskStatusStream\.listen\|listen.*taskStatus" mobile/lib -r
  # 无结果
  ```
- 所有业务逻辑仅存在于 `_handleTaskStatusUpdate(update)`（`background_upload.service.dart:207`），且**不返回 Future**（`void` 返回类型 + 内部 `unawaited`），调用方无法 await。
- `_taskStatusController` 是 `broadcast()`，没有消费者订阅时事件被丢弃（Dart Stream 语义）。

#### 5.5.2 `TaskStatusUpdate.responseBody` 的可用性

**代码事实 ✅**：
- `_handleLivePhoto`（`background_upload.service.dart:239`）直接读取 `update.responseBody` 解析 JSON 中的 `response['id']`：
  ```dart
  if (update.responseBody == null || update.responseBody!.isEmpty) {
    return;
  }
  final response = jsonDecode(update.responseBody!);
  ```
- 这表明 `background_downloader` 9.5.4 在 **`TaskStatus.complete` 状态下**会填充 `responseBody`。

**⚠️ 合理推断（非代码事实）**：
- `background_downloader` 的 API 设计中，`responseBody` 字段在 `TaskStatus.failed` 状态下同样可用（SDK 文档对 `TaskStatusUpdate` 的描述）。但 Immich 代码从未读取这个状态下的 `responseBody`，因此**此结论属于推断**，不是已验证的代码行为。

#### 5.5.3 `_handleTaskStatusUpdate` 的失败分支处理

**代码事实 ✅**：
`background_upload.service.dart:207-226` 的 switch 结构：

```dart
void _handleTaskStatusUpdate(TaskStatusUpdate update) async {
  switch (update.status) {
    case TaskStatus.complete:
      unawaited(_handleLivePhoto(update));
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
      break;  // TaskStatus.failed 直接忽略，无任何逻辑
  }
}
```

`TaskStatus.failed`、`TaskStatus.waitingToRetry`、`TaskStatus.canceled`、`TaskStatus.paused` 全部进入 `default` 分支，静默忽略。

#### 5.5.4 重试次数与行为

**代码事实 ✅**：
- `buildUploadTask`（`background_upload.service.dart:435`）中 `retries: 3` 是硬编码的。
- `background_downloader` 的重试语义：`retries` 指**额外重试次数**，即总尝试次数 = 1（原始）+ `retries`。因此每个任务最多 4 次 HTTP 请求。
- `updates: Updates.statusAndProgress`（`background_upload.service.dart:434`）意味着每次状态变更（包括每次重试失败）都会触发回调。

**⚠️ 合理推断**：
- 配额超限是确定性 400 错误，服务器每次都会返回相同的错误。因此配额超限时，每个资产会产生 4 次 400 请求。

#### 5.5.5 与前台上传的对比

| 维度 | 前台 `_uploadSingleAsset` | 后台 `BackgroundUploadService` |
| --- | --- | --- |
| 错误解析 | 同步检查 `UploadResult.errorMessage` ✅ | **无解析**，`TaskStatus.failed` 被忽略 ✅ |
| 配额超限检测 | `== "Quota has been exceeded!"` ✅ | **无检测**，不读取 `responseBody` ✅ |
| 中止机制 | `shouldAbortUpload=true` → worker 循环检查后 break ✅ | **无中止**，`shouldAbortQueuingTasks` 只在 `cancel()` 中设 true ✅ |
| 重试行为 | 无自动重试，每次 `uploadFile` 调用是单次请求 ✅ | `retries:3` → 最多 4 次请求 ✅ |
| 用户可见 | `callbacks.onError` → UI 展示错误图标 + 文案 ✅ | **静默**，无 UI 回调，无日志 ✅ |
| 批量继续 | 命中超限后 worker 循环 break，不再处理剩余资产 ✅ | 单次入队 100 个后自然停止，无后续循环 ✅ |
| Live Photo 处理 | 视频预传失败被静默忽略，仍尝试上传主图 ✅ | 视频上传失败则主图任务不被创建，两者均不入库 ✅ |

**代码事实 ✅**：
- `uploadBackupCandidates`（`background_upload.service.dart:171-185`）只取前 100 个候选资产入队一次，没有 `while` 循环。一批处理完（无论成功失败）后不会自动拉取下一批。
- 下一批只有在 iOS 系统再次触发后台任务时才会入队。因此**不会无限循环**，每次系统调度最多浪费 100 × 4 = 400 次请求。

#### 5.5.6 taskStatusStream 的断点分析

`taskStatusStream` 被定义（`background_upload.service.dart:119-122`）但从未被消费，存在以下断点：

1. **事件黑洞**：`_taskStatusController.add(update)` 向 broadcast stream 投递事件，但没有 listener，事件立即被丢弃。
2. **无二次处理**：DriftBackupNotifier（`drift_backup.provider.dart:189`）持有 `_backgroundUploadService` 引用，但从未订阅 `taskStatusStream`。
3. **UI 无感知**：没有任何 Widget 或 Notifier 监听失败状态更新，因此 UI 无法展示配额超限错误。
4. **日志缺失**：`default` 分支甚至没有 `_logger.warning()` 或 `_logger.severe()` 调用，错误完全不留痕迹。

### 5.6 后台备份触发机制

iOS 后台备份通过原生 `BackgroundWorker` 触发，而非 App 内调度：

- Swift 端：`mobile/ios/Runner/Background/BackgroundWorker.swift` 处理系统级后台任务回调
- Pigeon 桥接：`mobile/pigeon/background_worker_api.dart:44` 的 `onIosUpload` 方法被原生调用时，Dart 侧执行 `startBackupWithURLSession`
- 系统调度：iOS 根据设备空闲、充电、网络等条件决定何时触发后台任务

这意味着用户**无法在 App 内实时看到后台上传的配额超限错误**——错误只存在于服务器日志和客户端 `background_downloader` 的本地数据库中，不进入 UI。

### 5.7 Live Photo 视频预传失败对配额判断的影响

iOS Live Photo 由一张静态图（HEIC/JPEG）和一段短视频（MOV/MP4）组成。前台 `_uploadSingleAsset` 和后台 `BackgroundUploadService` 对 Live Photo 采用了不同的上传策略，两者在视频预传失败时的行为差异会直接影响配额判断的准确性。

#### 5.7.1 前台 `_uploadSingleAsset` 的 Live Photo 流程

`foreground_upload.service.dart:332-356`：

```dart
// Upload live photo video first if available
String? livePhotoVideoId;
if (entity.isLivePhoto && livePhotoFile != null) {
  final livePhotoResult = await _uploadRepository.uploadFile(
    file: livePhotoFile, ...
  );

  if (livePhotoResult.isSuccess && livePhotoResult.remoteAssetId != null) {
    livePhotoVideoId = livePhotoResult.remoteAssetId;
  }
  // ⚠️ livePhotoResult.isSuccess == false 时，静默忽略，不设 shouldAbortUpload
  // ⚠️ livePhotoResult.errorMessage == "Quota has been exceeded!" 时，也不处理
}

if (livePhotoVideoId != null) {
  fields['livePhotoVideoId'] = livePhotoVideoId;
}

// Upload still image (always executed, regardless of video result)
final result = await _uploadRepository.uploadFile(file: file, ...);
```

**代码事实 ✅**：
1. 视频预传失败时，代码仅将 `livePhotoVideoId` 保持为 `null`，不调用 `callbacks.onError`，不设置 `shouldAbortUpload`。
2. 主图上传始终会执行，即使视频因配额超限失败。
3. 主图上传失败并返回 `"Quota has been exceeded!"` 时，才触发 `shouldAbortUpload = true`（`foreground_upload.service.dart:399`）。
4. 视频预传的 `errorMessage` 从未被检查——如果视频上传因配额超限失败而主图成功，用户不会收到任何配额超限提示。

**风险分析**：

| 场景 | 视频预传 | 主图上传 | `shouldAbortUpload` | 用户感知 |
| --- | --- | --- | --- | --- |
| 配额刚满 | 失败（400） | 成功 | ❌ 不设置 | 静默：Live Photo 降级为静态图 |
| 配额已满 | 失败（400） | 失败（400） | ✅ 设置 | 主图失败提示触发中止 |
| 配额已满 | 失败（400） | 成功（极小概率） | ❌ 不设置 | 静默：视频丢失但主图入库 |
| 配额充足 | 成功 | 成功 | — | 正常上传 |

**最关键的风险**：视频预传因配额超限失败后，如果主图上传恰好成功（比如配额刚好够主图但不够视频，或缓存值滞后），该 Live Photo 会被"静默降级"为只有静态图的资产，用户不会收到任何提示。

#### 5.7.2 后台 `BackgroundUploadService` 的 Live Photo 流程

`background_upload.service.dart:228-259` 的 `_handleLivePhoto`：

```dart
Future<void> _handleLivePhoto(TaskStatusUpdate update) async {
  try {
    if (update.responseBody == null || update.responseBody!.isEmpty) {
      return;  // 视频上传失败时 responseBody 可能为空，直接返回
    }
    final response = jsonDecode(update.responseBody!);
    // 解析 response['id'] → livePhotoVideoId
    // 构建 getLivePhotoUploadTask → enqueueTasks([uploadTask])
  } catch (error, stackTrace) {
    // 失败时仅 dPrint，不做任何错误上报
  }
}
```

**代码事实 ✅**：
1. 后台 Live Photo 是两段式：先入队视频任务，视频任务成功后在 `_handleLivePhoto` 中解析 `responseBody` 获取 `id`，再构建主图任务并入队。
2. 如果视频任务失败（`TaskStatus.failed`），`_handleTaskStatusUpdate` 的 `default` 分支直接忽略，`_handleLivePhoto` 永远不会被调用，主图任务也不会被创建。
3. 整个过程无错误回调、无 UI 提示、无日志（`catch` 用的是 `dPrint` 而非 `_logger`）。

**与前台的对比**：

| 维度 | 前台 `_uploadSingleAsset` | 后台 `BackgroundUploadService` |
| --- | --- | --- |
| 视频失败后主图 | 仍尝试上传 | 不创建主图任务，两者均丢失 |
| 视频失败的配额判断 | 依赖主图上传结果 | 不做任何配额判断 |
| 视频失败的用户提示 | 依赖主图上传失败才触发 | 完全静默 |
| 降级行为 | 降级为静态图（静默） | 整个 Live Photo 丢失（静默） |

#### 5.7.3 结论

1. **前台**：Live Photo 视频预传失败不会触发配额中止，仅当主图上传也失败时才中止。存在"视频静默丢失但主图成功入库"的降级风险。
2. **后台**：Live Photo 视频上传失败导致整个 Live Photo（视频+主图）静默丢失，无任何配额判断和用户提示。
3. **两者均缺失**对视频预传阶段的错误检查——`livePhotoResult.errorMessage` 和 `update.responseBody` 在失败路径上从未被用于判断配额状态。

## 6. 缓存对齐机制

`quotaUsageInBytes` 是缓存，真实口径在 `asset_exif.fileSizeInByte`。对齐手段有两类：

### 6.1 主动全量 / 单用户重算

- `UserRepository.syncUsage(id?)`（`server/src/repositories/user.repository.ts:310`）：
  - 传 `id` 时只重算该用户；
  - 不传时全量重算所有 `deletedAt IS NULL` 的用户。
- `UserAdminService.update`（`user-admin.service.ts:61`）：管理员改配额时对该用户重算一次。
- `UserService.handleUserSyncUsage`（`server/src/services/user.service.ts:233`）：处理 `JobName.UserSyncUsage` 任务，全量重算。

### 6.2 定时触发

`server/src/services/queue.service.ts:280` 的 `handleNightlyJobs`：

```ts
if (config.nightlyTasks.syncQuotaUsage) {
  jobs.push({ name: JobName.UserSyncUsage });
}
```

由 system config 的 `nightlyTasks.syncQuotaUsage` 开关控制；开启后，每个维护夜（`NightlyJobs`）队列会派发一次 `UserSyncUsage`，把所有用户的缓存值拉回到和 `asset_exif` 一致。

所以整体策略是：

- 热路径上传 / 删除用 `updateUsage` 做增量维护，保证用户当下看到的"已用空间"基本准确；
- 冷路径用夜间 `syncUsage` 做兜底重算，修复因崩溃、外部库删除、人工 SQL 等造成的漂移；
- 管理员改配额时立即重算一次，避免缓存滞后影响判断。

## 6. 统计展示（给管理后台用）

管理员端展示的"每个用户用量"不直接读 `quotaUsageInBytes`，而是独立聚合：

`server/src/repositories/user.repository.ts:232` 的 `getUserStats()` 直接 `SUM(asset_exif.fileSizeInByte)`（`asset.libraryId IS NULL`），并拆出 `usagePhotos` / `usageVideos`。`ServerService.getStatistics`（`server/src/services/server.service.ts:133`）把这个结果填到 `UsageByUserDto`，但 `quotaSizeInBytes` 直接取 `user.quotaSizeInBytes`——也就是说**展示层的"已用"是实时聚合计，"配额"是 user 表值**，两者的口径并不完全同步，这在夜间 `syncUsage` 之前会有差异。

## 7. 完整端到端链路全景图

```
                                   ┌─────────────────────────┐
                                   │  用户上传文件（Web/APP） │
                                   └───────────┬─────────────┘
                                               │
                       ┌───────────────────────┼───────────────────────┐
                       │                       │                       │
                       ▼                       ▼                       ▼
              Web 拖拽上传             移动端备份任务          移动端手动上传
                       │                       │                       │
                       │             ┌─────────┴─────────┐             │
                       │             │                   │             │
                       │             ▼                   ▼             │
                       │       前台同步上传       iOS 后台 URLSession    │
                       │  (_uploadSingleAsset)  (BackgroundUpload)  │
                       │             │                   │             │
                       └─────────────┼───────────────────┼─────────────┘
                                     │                   │
                                     ▼                   ▼
                                   ┌─────────────────────────┐
                                   │  POST /assets  (HTTP)    │
                                   └───────────┬─────────────┘
                                               │
  ┌────────────────────────────────────────────┼────────────────────────────────────────────┐
  │ SERVER                                     │                                            │
  │                                            ▼                                            │
  │                          ┌──────────────────────────────────┐                           │
  │                          │ AssetMediaService.uploadAsset    │                           │
  │                          │  ① requireQuota(auth, file.size) │                           │
  │                          │    quotaUsage + size > quotaSize │                           │
  │                          │    throw BadRequestException     │                           │
  │                          └───────────────┬──────────────────┘                           │
  │                                          │                                              │
  │                                          ▼                                              │
  │                          ┌──────────────────────────────────┐                           │
  │                          │ GlobalExceptionFilter.fromError  │                           │
  │                          │  status=400                      │                           │
  │                          │  body={ message: "Quota has      │                           │
  │                          │          been exceeded!" }        │                           │
  │                          └───────────────┬──────────────────┘                           │
  │                                          │                                              │
  └──────────────────────────────────────────┼──────────────────────────────────────────────┘
                                             │
                                             ▼
                                   HTTP 400 JSON Response
                                             │
              ┌──────────────────────────────┼──────────────────────────────┐
              │                              │                              │
              ▼                              ▼                              ▼
        Web 端处理                前台移动端处理                后台移动端处理
              │                              │                              │
  ┌──────────────────────┐  ┌──────────────────────────┐  ┌──────────────────────────┐
  │ handleError()        │  │ 错误消息字符串匹配        │  │ _handleTaskStatusUpdate │
  │ getServerErrorMessage│  │ "Quota has been exceeded!"│  │ TaskStatus.failed → 忽略  │
  │ toastManager.danger  │  │ shouldAbortUpload=true  │  │ 无 responseBody 解析     │
  │ uploadAssetsStore    │  │ 中止 worker 循环         │  │ retries:3 → 3 次重试     │
  │ 红色 toast + 重试按钮 │  │ callbacks.onError→UI    │  │ 错误静默，用户不可见     │
  └──────────────────────┘  └──────────────────────────┘  └──────────────────────────┘
```

删除方向的对称链路：

```
  用户 / 管理员 删除资产
        │
        ▼
  asset.deletedAt = now()   （扣用量不在这步）
        │
        ▼
  JobName.AssetDeleteCheck  →  JobName.AssetDelete
        │
        ▼
  AssetService.handleAssetDeletion
        │
        └─ asset.libraryId IS NULL 时：
            updateUsage(ownerId, -exif.fileSizeInByte)
```

校准：

```
  夜间 / 管理员改配额  ──►  UserRepository.syncUsage()
                              以 asset_exif SUM 回写 user.quotaUsageInBytes
```

## 8. 设计注意事项与隐式约束

1. **硬编码字符串耦合**（代码事实 ✅）：移动端 `foreground_upload.service.dart:399` 用 `== "Quota has been exceeded!"` 做分支判断。后端如果修改这个异常消息文本，移动端将无法识别配额超限，备份队列不会中止，产生大量无意义的 400 请求。此耦合仅存在于前台路径，后台路径因无错误处理不受影响。

2. **无 i18n 的错误提示**：超限提示是英文硬编码，所有语言的用户看到的都是 `"Quota has been exceeded!"`，Web 端还会附加 `(Immich Server Error)` 后缀。

3. **乐观校验的短暂越界**：并发上传时多个请求可能同时通过校验，导致 `quotaUsageInBytes` 短暂超过上限，但夜间 `syncUsage` 会纠正，后续上传将被拒。

4. **移动端被动展示**：侧边栏进度条只做展示，不做阈值告警（如 90% 时变色、弹通知）。

5. **外部库资产不计入**：`libraryId IS NOT NULL` 的资产（外部挂载）不占用配额，上传、删除时都不触发 `updateUsage`。

6. **iOS 后台上传的静默失败**（代码事实 ✅）：
   - `background_upload.service.dart:207` 的 `_handleTaskStatusUpdate` 不处理 `TaskStatus.failed`，配额超限错误被完全忽略。
   - `taskStatusStream` 定义但未被任何地方 `.listen()`，事件被丢弃。
   - `default` 分支无日志，错误不留痕迹。
   - 配合 `retries: 3`，每次后台备份会产生最多 100 × 4 = 400 次无效请求（代码事实 ✅），但不会无限循环（单次入队 100 个后停止，下一批需等待系统再次触发）。
   - `responseBody` 在 `TaskStatus.failed` 状态下可用的结论属于合理推断 ⚠️，基于 SDK 文档，但 Immich 代码未验证。

7. **前后台行为不一致**（代码事实 ✅）：
   - 前台上传能检测到配额超限并中止整个备份循环；
   - 后台上传（iOS URLSession）不做任何检测，单次入队 100 个后自然停止。
   - 两者的错误可见性差异：前台通过 `callbacks.onError` 提示用户，后台完全静默。

8. **Live Photo 视频预传的静默降级**（代码事实 ✅）：
   - 前台 `_uploadSingleAsset`（`foreground_upload.service.dart:332-352`）中，Live Photo 视频预传失败（包括配额超限）被静默忽略，不设置 `shouldAbortUpload`，不调用 `callbacks.onError`。
   - 主图上传始终会执行，可能出现"视频丢失但主图成功入库"的静默降级场景。
   - 后台 `_handleLivePhoto`（`background_upload.service.dart:228-259`）中，视频上传失败导致主图任务不被创建，整个 Live Photo 静默丢失。
   - 两者均未在视频预传阶段检查 `errorMessage` 或 `responseBody` 中的配额信息。

## 9. 关键代码索引

| 主题 | 文件 | 行号 |
| --- | --- | --- |
| 配额字段定义 | `server/src/schema/tables/user.table.ts` | 72, 75 |
| 鉴权上下文携带配额 | `server/src/database.ts` | 342 |
| 上传入口 & 校验 | `server/src/services/asset-media.service.ts` | 127, 141, 155, 366 |
| 全局异常过滤器 | `server/src/middleware/global-exception.filter.ts` | 30 |
| 运动视频累加 | `server/src/services/metadata.service.ts` | 729, 750 |
| 删除时扣减 | `server/src/services/asset.service.ts` | 307, 334 |
| 原子累加 SQL | `server/src/repositories/user.repository.ts` | 300 |
| 重算缓存 | `server/src/repositories/user.repository.ts` | 310 |
| 管理员改配额 → 重算 | `server/src/services/user-admin.service.ts` | 60 |
| 夜间 sync 任务 | `server/src/services/user.service.ts` | 233 |
| 夜间任务派发 | `server/src/services/queue.service.ts` | 280 |
| 管理端用量聚合 | `server/src/repositories/user.repository.ts` | 232 |
| OAuth 配额注入 | `server/src/services/auth.service.ts` | 371 |
| Web 端错误处理 | `web/src/lib/utils/handle-error.ts` | 4, 38 |
| Web 端上传错误捕获 | `web/src/lib/utils/file-uploader.ts` | 255, 262 |
| 前台上传配额检测 | `mobile/lib/services/foreground_upload.service.dart` | 399 |
| 前台上传中止循环（worker pool） | `mobile/lib/services/foreground_upload.service.dart` | 207 |
| 前台上传主路径 `_uploadSingleAsset` | `mobile/lib/services/foreground_upload.service.dart` | 235 |
| 前台 Live Photo 视频预传 | `mobile/lib/services/foreground_upload.service.dart` | 332-352 |
| 前台上传入口 `uploadCandidates` | `mobile/lib/services/foreground_upload.service.dart` | 77 |
| 前台 worker pool `_executeWithWorkerPool` | `mobile/lib/services/foreground_upload.service.dart` | 193 |
| 前台顺序上传 `_uploadSequentially` | `mobile/lib/services/foreground_upload.service.dart` | 108 |
| 后台上传服务入口 `uploadBackupCandidates` | `mobile/lib/services/background_upload.service.dart` | 159 |
| 后台上传错误处理（缺失） | `mobile/lib/services/background_upload.service.dart` | 207-226 |
| 后台上传任务构建 | `mobile/lib/services/background_upload.service.dart` | 372, 435 |
| 后台 Live Photo 处理 `_handleLivePhoto` | `mobile/lib/services/background_upload.service.dart` | 228-259 |
| 后台上传 Repository `uploadFile` | `mobile/lib/repositories/upload.repository.dart` | 91 |
| 前台上传 Repository `uploadFile` | `mobile/lib/repositories/upload.repository.dart` | 91 |
| Repository 错误解析 | `mobile/lib/repositories/upload.repository.dart` | 117, 126-127 |
| 前台备份触发入口 | `mobile/lib/providers/backup/drift_backup.provider.dart` | 259 |
| 后台备份触发入口 | `mobile/lib/providers/backup/drift_backup.provider.dart` | 376 |
| 前台备份错误回调 UI | `mobile/lib/providers/backup/drift_backup.provider.dart` | 345 |
| 移动端配额进度展示 | `mobile/lib/widgets/common/app_bar_dialog/app_bar_dialog.dart` | 147 |
| SDK HttpError 类型 | `packages/sdk/src/fetch-errors.ts` | 16, 20 |
| iOS 后台 Worker（Swift） | `mobile/ios/Runner/Background/BackgroundWorker.swift` | — |
| Pigeon 桥接 | `mobile/pigeon/background_worker_api.dart` | 44 |

