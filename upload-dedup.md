# Immich 移动端上传去重与断点续传技术报告

## 概述

本报告旨在**严格基于代码证据**，清晰区分「移动端实际上传路径中已实现的去重能力」与「服务端可用但移动端未使用的去重能力」。报告中所有结论均有对应代码证据支撑，避免将服务端能力误表述为移动端已实现功能。

---

## 第一部分：移动端实际上传去重链路（代码可证实）

### 1.1 唯一可证实的移动端去重机制：本地 SQLite 数据库级去重

**文件位置**: `mobile/lib/infrastructure/repositories/backup.repository.dart:84-116`

```dart
Future<List<LocalAsset>> getCandidates(String userId, {bool onlyHashed = true}) async {
  final selectedAlbumIds = _db.localAlbumEntity.selectOnly(distinct: true)
    ..addColumns([_db.localAlbumEntity.id])
    ..where(_db.localAlbumEntity.backupSelection.equalsValue(BackupSelection.selected));

  final query = _db.localAssetEntity.select()
    ..where(
      (lae) =>
          // 条件1：资产属于已选择的备份相册
          existsQuery(...) &
          // 条件2：⭐ 核心去重逻辑 - LEFT JOIN 远程资产表
          // 通过本地缓存的远程资产 checksum 匹配判断是否已上传
          notExistsQuery(
            _db.remoteAssetEntity.selectOnly()
              ..addColumns([_db.remoteAssetEntity.checksum])
              ..where(
                _db.remoteAssetEntity.checksum.equalsExp(lae.checksum) &
                _db.remoteAssetEntity.ownerId.equals(userId),
              ),
          ) &
          // 条件3：不在排除的相册中
          lae.id.isNotInQuery(_getExcludedSubquery()),
    )
    ..orderBy([(localAsset) => OrderingTerm.desc(localAsset.createdAt)]);

  if (onlyHashed) {
    query.where((lae) => lae.checksum.isNotNull()); // 条件4：只选择已计算哈希的资产
  }

  return query.map((localAsset) => localAsset.toDto()).get();
}
```

**代码证据结论**:
- ✅ **唯一的移动端去重机制**：在上传候选筛选阶段，通过 SQLite 的 `LEFT JOIN` + `NOT EXISTS` 查询过滤掉已上传的资产
- ✅ **去重依据**：`local_asset_entity.checksum` = `remote_asset_entity.checksum` + `owner_id` 匹配
- ✅ **数据来源**：`remote_asset_entity` 表是通过「远程资产同步」操作从服务端拉取后本地缓存的数据
- ⚠️ **此机制不产生任何网络请求**，完全是本地数据库查询

### 1.2 本地哈希计算（iOS / Android 原生实现）

**iOS 实现**: `mobile/ios/Runner/Sync/MessagesImpl.swift:271-380`

```swift
private func hashAsset(_ asset: PHAsset, allowNetworkAccess: Bool) async -> HashResult? {
  return await withTaskCancellationHandler(operation: {
    var hasher = Insecure.SHA1()  // CryptoKit SHA-1 算法
    PHAssetResourceManager.default().requestData(
      for: resource,
      options: options,
      dataReceivedHandler: { data in hasher.update(data: data) },  // 流式分块哈希
      completionHandler: { error in
        // 输出 Base64 格式
        HashResult(assetId: asset.localIdentifier, error: nil,
          hash: Data(hasher.finalize()).base64EncodedString())
      }
    )
  }, onCancel: { ... })
}
```

**Android 实现**: `mobile/android/app/src/main/kotlin/app/alextran/immich/sync/MessagesImplBase.kt:380-444`

```kotlin
private suspend fun hashAsset(assetId: String): HashResult {
  val digest = MessageDigest.getInstance("SHA-1")  // Java MessageDigest SHA-1
  ctx.contentResolver.openInputStream(assetUri)?.use { inputStream ->
    val buffer = ByteArray(HASH_BUFFER_SIZE)  // 2MB 分块读取
    while (inputStream.read(buffer).also { bytesRead = it } > 0) {
      currentCoroutineContext().ensureActive()
      digest.update(buffer, 0, bytesRead)
    }
  }
  val hashString = Base64.encodeToString(digest.digest(), Base64.NO_WRAP)  // 输出 Base64
  return HashResult(assetId, null, hashString)
}
```

**代码证据结论**:
- ✅ 双平台均使用 **SHA-1** 算法
- ✅ 输出格式统一为 **Base64**（与服务端兼容）
- ✅ 哈希结果持久化存储在 `local_asset_entity.checksum` 字段

### 1.3 关键发现：移动端上传时**未设置** Checksum HTTP Header

**代码证据**:
1. **`buildUploadTask` 方法**（iOS 后台上传路径）: `mobile/lib/services/background_upload.service.dart:375-439`
   ```dart
   Future<UploadTask> buildUploadTask(...) async {
     final headers = ApiService.getRequestHeaders();  // ⭐ 只获取自定义 headers
     // ...
     return UploadTask(
       // ...
       headers: headers,  // ⭐ 未添加 checksum header
       // ...
     );
   }
   ```

2. **`getRequestHeaders` 方法**：`mobile/lib/services/api.service.dart:195-202`
   ```dart
   static Map<String, String> getRequestHeaders() {
     var customHeadersStr = Store.get(StoreKey.customHeaders, "");
     if (customHeadersStr.isEmpty) {
       return const {};  // ⭐ 默认返回空 map，无 checksum header
     }
     return (jsonDecode(customHeadersStr) as Map).cast<String, String>();
   }
   ```

3. **`uploadFile` 方法**（前台上传路径）：`mobile/lib/repositories/upload.repository.dart:91-150`
   - 代码中**无任何 `checksum` 相关 header 设置逻辑**
   - 只设置了 multipart form fields（设备ID、创建时间等）

**最终结论**:
> ❗ **重要修正**：在当前所有上传路径（前台/后台/Android/iOS）中，**移动端代码并未将文件 checksum 放入 HTTP 请求头中传递给服务端**。
>
> 服务端的 `AssetUploadInterceptor` 预检拦截器虽然存在，但**移动端未触发此优化路径**。去重最终仅依赖服务端上传文件后的数据库唯一约束。

---

## 第二部分：服务端可用的去重能力（但移动端未使用）

### 2.1 服务端预检拦截器（HTTP Header Checksum）

**文件位置**: `server/src/middleware/asset-upload.interceptor.ts`

```typescript
@Injectable()
export class AssetUploadInterceptor implements NestInterceptor {
  async intercept(context: ExecutionContext, next: CallHandler<any>) {
    const req = context.switchToHttp().getRequest<AuthenticatedRequest>();
    const res = context.switchToHttp().getResponse<Response<AssetMediaResponseDto>>();

    // 从 Header 读取客户端预计算的 checksum
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

**能力说明**:
- ✅ **服务端具备此能力**：可在解析文件前通过 header 中的 checksum 快速判断重复
- ✅ **Web 端在使用**：`web/src/lib/utils/file-uploader.ts` 中可以看到此逻辑调用
- ❌ **移动端未使用**：无代码证据表明移动端上传时设置了此 header

### 2.2 批量预检 API（/assets/bulk-check）

**文件位置**: `server/src/services/asset-media.service.ts`

```typescript
async bulkUploadCheck(auth: AuthDto, dto: AssetBulkUploadCheckDto): Promise<AssetBulkUploadCheckResponseDto> {
  const checksums = dto.assets.map((asset) => fromChecksum(asset.checksum));
  const results = await this.assetRepository.getByChecksums(auth.user.id, checksums);
  // ... 返回重复结果
}
```

**能力说明**:
- ✅ **服务端具备此能力**：批量预检 API 存在
- ✅ **Web 端在使用**：用于批量上传前的优化
- ❌ **移动端未使用**：代码搜索结果中未发现移动端调用此 API

### 2.3 数据库唯一约束兜底（最终保障）

**文件位置**: `server/src/schema/tables/asset.table.ts`

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
  checksum!: Buffer;  // 服务端计算后存储
}
```

**实际生效机制**:
- ✅ **这是移动端实际上传路径中最终生效的服务端去重机制**
- 服务端在接收完整文件后重新计算 SHA-1 checksum
- 插入数据库时触发唯一约束冲突，返回重复状态
- 冲突处理逻辑在 `asset-media.service.ts` 中实现

---

## 第三部分：断点续传能力分析

### 3.1 iOS 后台上传断点续传

**文件位置**: `mobile/lib/domain/services/background_worker.service.dart:223-224`

```dart
if (Platform.isIOS) {
  return _ref?.read(driftBackupProvider.notifier).startBackupWithURLSession(currentUser.id);
}
```

**iOS 实现证据**:
- ✅ 使用 `background_downloader` 库封装的 iOS `URLSession` 后台任务
- ✅ 任务状态持久化到 SQLite 数据库（`FileDownloader().database`）
- ✅ 支持 5 种任务状态查询：`enqueued` / `running` / `canceled` / `waitingToRetry` / `paused`
- ✅ 支持 `start()` / `reset()` 控制断点续传

**iOS 断点续传能力确认**:
> ✅ **确认**：iOS 端通过 URLSession 后台任务机制具备断点续传能力。App 重启后可通过 `FileDownloader().database` 恢复未完成任务。

### 3.2 Android 断点续传实现边界

**文件位置**: `mobile/lib/domain/services/background_worker.service.dart:227-229`

```dart
// Android 后台执行路径
return _ref
    ?.read(foregroundUploadServiceProvider)
    .uploadCandidates(currentUser.id, _cancellationToken, useSequentialUpload: true);
```

**Android 实际执行路径**:
1. WorkManager 启动独立 FlutterEngine
2. 调用 `backgroundSyncNativeEntrypoint()` 入口
3. 调用 `_uploadSequentially()` 进行**顺序单文件上传**（非任务队列）
4. 使用 `foregroundUploadService.uploadFile()` 创建全新 MultipartRequest

**代码证据缺失说明**:
> ⚠️ **基于代码证据的边界声明**：
>
> 1. **HTTP 级断点续传（Range 请求）**：代码中**未发现** `Range` / `Content-Range` 相关实现
> 2. **上传偏移量持久化**：代码中**未发现**已上传字节数存储逻辑
> 3. **任务级恢复**：✅ WorkManager 在进程被杀后可重启整个上传流程
> 4. **文件级去重**：✅ 本地数据库 checksum 过滤确保已上传成功的文件不会重复上传
> 5. **单文件断点续传**：❌ 代码证据不足以支撑此结论

### 3.3 状态回调处理边界

**文件位置**: `mobile/lib/services/background_upload.service.dart:209-228`

```dart
void _handleTaskStatusUpdate(TaskStatusUpdate update) async {
  switch (update.status) {
    case TaskStatus.complete:
      unawaited(_handleLivePhoto(update));  // 仅处理 Live Photo 后续上传
      if (CurrentPlatform.isIOS) {
        // iOS 清理临时文件
      }
      break;
    default:
      break;  // ⭐ 其他状态（running/enqueued/paused 等）完全不处理！
  }
}
```

**状态回调边界说明**:
| 状态 | 是否处理 | 处理逻辑 |
|-----|---------|---------|
| `complete` | ✅ 处理 | 触发 Live Photo 后续上传 + iOS 清理临时文件 |
| `running/enqueued/waitingToRetry/paused/canceled/failed` | ❌ **不处理** | 完全交由 background_downloader 库内部管理 |

> ⚠️ **关键发现**：应用层回调仅处理 `complete` 状态，**其他状态的恢复/重试逻辑完全依赖于第三方库的内部实现**。

---

## 第四部分：关键结论汇总表

| 功能模块 | 结论 | 证据状态 | 代码位置 |
|---------|------|---------|---------|
| **本地 SQLite 去重** | ✅ 已实现 | 完整代码证据 | `backup.repository.dart:84-116` |
| **HTTP Header 预检** | ❌ 移动端未使用 | 代码搜索无结果 | 服务端有实现但移动端未传 header |
| **批量预检 API** | ❌ 移动端未调用 | 代码搜索无结果 | Web 端使用，移动端无调用 |
| **数据库唯一约束** | ✅ 最终生效 | 完整代码证据 | `asset.table.ts` |
| **iOS URLSession 断点续传** | ✅ 已实现 | 完整代码证据 | `background_worker.service.dart:223-224` |
| **Android HTTP 断点续传** | ⚠️ 证据不足 | 代码搜索无 Range 相关实现 | - |
| **Android 任务级重启** | ✅ 已实现 | WorkManager 入口代码 | `BackgroundWorker.kt:63-102` |
| **checksum header 传递** | ❌ 未实现 | 代码搜索无结果 | 所有上传路径均无设置 |
| **双平台 SHA-1 哈希** | ✅ 已实现 | 完整原生代码 | iOS/Swift + Android/Kotlin |

---

## 第五部分：完整上传链路示意图

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                    Immich 移动端实际上传去重链路                               │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│  ┌──────────────────────┐                                                   │
│  │ 1. 本地 SQLite 去重  │ ◀── 唯一可证实的移动端去重机制                        │
│  │   getCandidates()    │                                                   │
│  └─────────┬────────────┘                                                   │
│            │                                                                │
│            ▼                                                                │
│  ┌──────────────────────┐                                                   │
│  │ 2. 原生 SHA-1 哈希    │ ◀── iOS/Android 原生计算，结果持久化到本地            │
│  └─────────┬────────────┘                                                   │
│            │                                                                │
│            ▼                                                                │
│  ┌──────────────────────┐                                                   │
│  │ 3. 上传文件到服务端   │ ◀── ⚠️ 注意：未携带 checksum header                  │
│  │   (Multipart POST)   │                                                   │
│  └─────────┬────────────┘                                                   │
│            │                                                                │
│            ▼                                                                │
│  ┌──────────────────────┐                                                   │
│  │ 4. 服务端接收文件    │                                                   │
│  │   重新计算 SHA-1     │                                                   │
│  └─────────┬────────────┘                                                   │
│            │                                                                │
│            ▼                                                                │
│  ┌──────────────────────┐                                                   │
│  │ 5. 数据库唯一约束    │ ◀── 最终去重保障                                   │
│  │   UNIQUE(checksum,   │                                                   │
│  │        owner_id)     │                                                   │
│  └──────────────────────┘                                                   │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

---

## 第六部分：代码溯源索引

### 移动端代码
| 功能模块 | 文件路径 |
|---------|---------|
| 备份候选查询（本地去重核心） | `mobile/lib/infrastructure/repositories/backup.repository.dart:84-116` |
| iOS 原生哈希实现 | `mobile/ios/Runner/Sync/MessagesImpl.swift:271-380` |
| Android 原生哈希实现 | `mobile/android/app/src/main/kotlin/app/alextran/immich/sync/MessagesImplBase.kt:380-449` |
| 请求头获取方法（无 checksum） | `mobile/lib/services/api.service.dart:195-202` |
| 后台上传任务构建 | `mobile/lib/services/background_upload.service.dart:375-439` |
| 前台并发 Worker Pool | `mobile/lib/services/foreground_upload.service.dart:190-237` |
| Android 顺序上传实现 | `mobile/lib/services/foreground_upload.service.dart:112-134` |
| Android WorkManager 入口 | `mobile/android/app/src/main/kotlin/app/alextran/immich/background/BackgroundWorker.kt:63-102` |
| 后台 worker 平台分发 | `mobile/lib/domain/services/background_worker.service.dart:207-235` |

### 服务端代码
| 功能模块 | 文件路径 |
|---------|---------|
| 服务端预检拦截器 | `server/src/middleware/asset-upload.interceptor.ts` |
| 批量预检 API 实现 | `server/src/services/asset-media.service.ts` |
| 数据库唯一约束定义 | `server/src/schema/tables/asset.table.ts` |
