# 媒资入库时存储模板表达式向真实路径转换过程分析（Round 2）

> **更正说明**：本文档修正了前一版中的两处事实偏差：
> 1. 预设模板实际为 **21 个**（而非 22 个），明确定义来源
> 2. 全量迁移**并非模板变更自动触发**，需通过显式队列任务触发，厘清配置更新与迁移的关系

---

## 一、整体架构与流程概述

### 1.1 核心流程时序

```
资产上传入库
    ↓
asset-media.service.ts:create() → 创建资产记录，originalPath 指向上传临时目录
    ↓
触发 AssetExtractMetadata 任务 → 元数据提取
    ↓
metadata.service → 提取 EXIF 信息，完成后触发 AssetMetadataExtracted 事件
    ↓
storage-template.service.ts:onAssetMetadataExtracted() → 监听事件，排队 StorageTemplateMigrationSingle 任务
    ↓
handleMigrationSingle() → 调用 moveAsset() 应用存储模板
    ↓
getTemplatePath() → 模板编译 → 字段插值 → 冲突检测 → 生成最终路径
    ↓
storage.core.ts:moveFile() → 原子化文件移动 + 数据库更新
```

### 1.2 核心组件关系

| 组件 | 职责 | 关键文件 |
|------|------|----------|
| StorageTemplateService | 模板编译、渲染、路径生成、迁移调度 | `server/src/services/storage-template.service.ts` |
| StorageCore | 实际文件移动、用户路径隔离、原子操作保障 | `server/src/cores/storage.core.ts` |
| MoveRepository | 移动历史记录、断点续迁支持 | `server/src/repositories/move.repository.ts` |
| AssetMediaService | 资产上传入口、初始路径设置 | `server/src/services/asset-media.service.ts` |
| StorageRepository | 文件系统底层操作 | `server/src/repositories/storage.repository.ts` |
| SystemConfigService | 配置管理、ConfigUpdate 事件广播 | `server/src/services/system-config.service.ts` |
| QueueService | 任务队列控制、迁移任务显式触发 | `server/src/services/queue.service.ts` |

---

## 二、字段插值机制详解

### 2.1 模板引擎与编译

**核心代码**：`storage-template.service.ts:391-398`

```typescript
private compile(template: string) {
  return {
    raw: template,
    compiled: handlebar.compile(template, { knownHelpers: undefined, strict: true }),
    needsAlbum: template.includes('album'),
    needsAlbumMetadata: template.includes('album-startDate') || template.includes('album-endDate'),
  };
}
```

**设计要点**：
- 使用 **Handlebars** 作为模板引擎，启用 `strict` 模式
- 编译时静态分析模板依赖：`needsAlbum`、`needsAlbumMetadata`
- 依赖分析用于**懒加载优化**：仅当模板需要相册信息时才查询数据库

### 2.2 可用插值变量

**核心代码**：`storage-template.service.ts:400-430`

#### 2.2.1 基础变量

| 变量 | 说明 | 示例 |
|------|------|------|
| `{{filename}}` | 原始文件名（不含扩展名） | `IMG_1234` |
| `{{ext}}` | 文件扩展名（小写化） | `jpg` |
| `{{filetype}}` | 文件类型简码 | `IMG` / `VID` |
| `{{filetypefull}}` | 文件类型全写 | `IMAGE` / `VIDEO` |
| `{{assetId}}` | 资产 UUID | `d587e44b-f8c0-4832-9ba3-43268bbf5d4e` |
| `{{assetIdShort}}` | 资产 ID 后 12 位 | `43268bbf5d4e` |

#### 2.2.2 日期时间变量

基于 `fileCreatedAt` 字段，使用 Luxon 格式化：

| 类别 | 可用标记 |
|------|----------|
| 年 | `y`, `yy` |
| 月 | `M`, `MM`, `MMM`, `MMMM` |
| 周 | `W`, `WW` |
| 日 | `d`, `dd` |
| 时 | `h`, `hh`, `H`, `HH` |
| 分 | `m`, `mm` |
| 秒 | `s`, `ss`, `SSS` |

**示例**：`{{y}}/{{MM}}/{{dd}}` → `2024/05/23`

#### 2.2.3 相册变量

| 变量 | 说明 |
|------|------|
| `{{album}}` | 相册名称（经过 sanitize 处理，移除 `.`） |
| `{{album-startDate-*}}` | 相册开始日期，支持所有日期标记 |
| `{{album-endDate-*}}` | 相册结束日期，支持所有日期标记 |

#### 2.2.4 设备变量

| 变量 | 说明 |
|------|------|
| `{{make}}` | 相机制造商 | `FUJIFILM` |
| `{{model}}` | 相机型号 | `X-T50` |
| `{{lensModel}}` | 镜头型号 | `XF27mm F2.8 R WR` |

### 2.3 日期插值实现

**核心代码**：`storage-template.service.ts:416-427`

```typescript
const dt = DateTime.fromJSDate(asset.fileCreatedAt);

for (const token of Object.values(storageTokens).flat()) {
  substitutions[token] = dt.toFormat(token);
  if (albumName) {
    substitutions['album-startDate-' + token] = albumStartDate
      ? DateTime.fromJSDate(albumStartDate).toFormat(token)
      : '';
    substitutions['album-endDate-' + token] = albumEndDate 
      ? DateTime.fromJSDate(albumEndDate).toFormat(token) 
      : '';
  }
}
```

### 2.4 预设模板（**更正：实际 21 个，非 22 个**）

**核心代码**：`storage-template.service.ts:35-57`

**证据 1**：源码定义（storage-template.service.ts:35-57）
```typescript
const storagePresets = [
  '{{y}}/{{y}}-{{MM}}-{{dd}}/{{filename}}',          // 1
  '{{y}}/{{MM}}-{{dd}}/{{filename}}',                // 2
  '{{y}}/{{MMMM}}-{{dd}}/{{filename}}',              // 3
  '{{y}}/{{MM}}/{{filename}}',                       // 4
  '{{y}}/{{#if album}}{{album}}{{else}}Other/{{MM}}{{/if}}/{{filename}}',    // 5
  '{{#if album}}{{album-startDate-y}}/{{album}}{{else}}{{y}}/Other/{{MM}}{{/if}}/{{filename}}',  // 6
  '{{y}}/{{MMM}}/{{filename}}',                      // 7
  '{{y}}/{{MMMM}}/{{filename}}',                     // 8
  '{{y}}/{{MM}}/{{dd}}/{{filename}}',                // 9
  '{{y}}/{{MMMM}}/{{dd}}/{{filename}}',              // 10
  '{{y}}/{{y}}-{{MM}}/{{y}}-{{MM}}-{{dd}}/{{filename}}',  // 11
  '{{y}}-{{MM}}-{{dd}}/{{filename}}',                // 12
  '{{y}}-{{MMM}}-{{dd}}/{{filename}}',               // 13
  '{{y}}-{{MMMM}}-{{dd}}/{{filename}}',              // 14
  '{{y}}/{{y}}-{{MM}}/{{filename}}',                 // 15
  '{{y}}/{{y}}-{{WW}}/{{filename}}',                 // 16
  '{{y}}/{{y}}-{{MM}}-{{dd}}/{{assetId}}',           // 17
  '{{y}}/{{y}}-{{MM}}/{{assetId}}',                  // 18
  '{{y}}/{{y}}-{{WW}}/{{assetId}}',                  // 19
  '{{album}}/{{filename}}',                          // 20
  '{{make}}/{{model}}/{{lensModel}}/{{filename}}',   // 21
];
```

**证据 2**：单元测试验证（storage-template.service.spec.ts:67-89）
```typescript
presetOptions: [
  '{{y}}/{{y}}-{{MM}}-{{dd}}/{{filename}}',
  '{{y}}/{{MM}}-{{dd}}/{{filename}}',
  // ... 共 21 项，与源码完全一致
  '{{make}}/{{model}}/{{lensModel}}/{{filename}}',
],
```

**证据 3**：API 接口返回
```typescript
// storage-template.service.ts:132-134
getStorageTemplateOptions(): SystemConfigTemplateStorageOptionDto {
  return { ...storageTokens, presetOptions: storagePresets };
}
```

**来源说明**：
- 预设模板定义在 `storage-template.service.ts` 顶部的 `storagePresets` 常量数组
- 通过 `/api/system-config/storage-template-options` API 返回给前端供用户选择
- 默认模板为第 1 个：`'{{y}}/{{y}}-{{MM}}-{{dd}}/{{filename}}'`（见 `config.ts:321`）

**预设模板分类**：
| 分类 | 数量 | 示例 |
|------|------|------|
| 按日期层级 | 14 | `{{y}}/{{MM}}/{{dd}}/{{filename}}` |
| 按相册 | 3 | `{{album}}/{{filename}}` |
| 按设备 | 1 | `{{make}}/{{model}}/{{lensModel}}/{{filename}}` |
| 按 assetId | 3 | `{{y}}/{{y}}-{{MM}}-{{dd}}/{{assetId}}` |

### 2.5 默认模板配置

**核心代码**：`config.ts:318-322`

```typescript
storageTemplate: {
  enabled: false,                    // 默认禁用
  hashVerificationEnabled: true,     // 默认启用哈希校验
  template: '{{y}}/{{y}}-{{MM}}-{{dd}}/{{filename}}',  // 默认模板
},
```

---

## 三、冲突重命名策略

### 3.1 扩展名规范化

**核心代码**：`storage-template.service.ts:278-302`

在生成路径前，先对扩展名进行规范化处理：

```typescript
switch (extension) {
  case 'jpeg': case 'jpe': extension = 'jpg'; break;
  case 'tif': extension = 'tiff'; break;
  case '3gpp': extension = '3gp'; break;
  case 'mpeg': case 'mpe': extension = 'mpg'; break;
  case 'm2ts': case 'm2t': extension = 'mts'; break;
}
```

### 3.2 路径穿越防护

**核心代码**：`storage-template.service.ts:341-344`

```typescript
if (!fullPath.startsWith(rootPath)) {
  this.logger.warn(`Skipped attempt to access an invalid path: ${fullPath}`);
  return source;
}
```

### 3.3 重复后缀检测

**核心代码**：`storage-template.service.ts:364-370`

针对已重命名文件的智能识别：

```typescript
if (source.startsWith(fullPath) && source.endsWith(`.${extension}`)) {
  const diff = source.replace(fullPath, '').replace(`.${extension}`, '');
  const hasDuplicationAnnotation = /^\+\d+$/.test(diff);
  if (hasDuplicationAnnotation) {
    return source;  // 已正确重命名，跳过
  }
}
```

**场景示例**：
- 源路径：`upload/abc/FullSizeRender+7.heic`
- 目标路径：`upload/abc/FullSizeRender.heic`
- 检测到 `+7` 后缀，判定为已迁移，跳过重复处理

### 3.4 增量重命名机制

**核心代码**：`storage-template.service.ts:372-384`

```typescript
let duplicateCount = 0;
while (true) {
  const exists = await this.storageRepository.checkFileExists(destination);
  if (!exists) break;
  duplicateCount++;
  destination = `${fullPath}+${duplicateCount}.${extension}`;
}
```

**重命名规则**：
- 冲突时追加 `+N` 后缀，N 从 1 开始递增
- 例如：`photo.jpg` → `photo+1.jpg` → `photo+2.jpg`

---

## 四、跨用户隔离机制

### 4.1 用户路径根目录计算

**核心代码**：`storage.core.ts:104-106`

```typescript
static getLibraryFolder(user: { storageLabel: string | null; id: string }) {
  return join(StorageCore.getBaseFolder(StorageFolder.Library), user.storageLabel || user.id);
}
```

### 4.2 storageLabel 字段设计

**核心代码**：`schema/tables/user.table.ts:64-65`

```typescript
@Column({ unique: true, nullable: true, default: null })
storageLabel!: string | null;
```

**隔离层级**：

```
<UPLOAD_LOCATION>/
  └── library/
      ├── user1_storage_label/     # 用户1（自定义 storageLabel）
      │   └── 2024/05/23/photo.jpg
      ├── a1b2c3d4-e5f6-7890-abcd-ef1234567890/  # 用户2（使用 UUID）
      │   └── 2024/05/23/photo.jpg
      └── another_user_label/       # 用户3
          └── 2024/05/23/photo.jpg
```

**设计特点**：
- `storageLabel` 唯一且可为空，空值时 fallback 到用户 UUID
- 支持 OAuth 声明映射：`storageLabelClaim` 可配置从 OAuth token 中提取
- 路径拼接使用 `path.join()` 保证跨平台兼容性

### 4.3 其他目录的用户隔离

**缩略图目录**：`storage.core.ts:116-122`
```typescript
static getImagePath(asset: ThumbnailPathEntity, options: ImagePathOptions) {
  return StorageCore.getNestedPath(
    StorageFolder.Thumbnails,
    asset.ownerId,  // 按 ownerId 隔离
    `${asset.id}_${fileType}${isEdited ? '_edited' : ''}.${format}`,
  );
}
```

**嵌套路径优化**：`storage.core.ts:328-334`
```typescript
static getNestedFolder(folder: StorageFolder, ownerId: string, filename: string): string {
  return join(
    StorageCore.getFolderLocation(folder, ownerId),
    filename.slice(0, 2),   // 第一级：文件名前 2 字符
    filename.slice(2, 4)    // 第二级：文件名 2-4 字符
  );
}
```

**作用**：避免单目录下文件过多导致性能问题。

---

## 五、重组迁移机制（**更正：全量迁移需显式触发**）

### 5.1 迁移任务类型与触发方式

| 任务类型 | JobName | 触发方式 | 适用场景 |
|----------|---------|----------|----------|
| 单资产迁移 | `StorageTemplateMigrationSingle` | **自动触发**：监听 `AssetMetadataExtracted` 事件 | 新上传资产、元数据更新后 |
| 全量迁移 | `StorageTemplateMigration` | **显式触发**：通过 Queue API 手动启动 | 模板变更后批量迁移所有资产 |

### 5.2 单资产迁移触发链（自动）

**核心代码**：`storage-template.service.ts:136-171`

```typescript
@OnEvent({ name: 'AssetMetadataExtracted' })
async onAssetMetadataExtracted({ source, assetId }: ArgOf<'AssetMetadataExtracted'>) {
  await this.jobRepository.queue({ 
    name: JobName.StorageTemplateMigrationSingle, 
    data: { source, id: assetId } 
  });
}
```

**触发链**：
```
资产上传 → create() → 排队 AssetExtractMetadata
  → metadata.service 提取 EXIF 完成
  → emit('AssetMetadataExtracted')
  → onAssetMetadataExtracted() 监听事件
  → 自动排队 StorageTemplateMigrationSingle 任务
  → handleMigrationSingle() 执行单资产迁移
```

### 5.3 全量迁移触发链（**更正：显式触发，非自动**）

**核心代码 1**：队列服务触发入口（`queue.service.ts:198-200`）
```typescript
case QueueName.StorageTemplateMigration: {
  return this.jobRepository.queue({ name: JobName.StorageTemplateMigration });
}
```

**核心代码 2**：任务处理器（`storage-template.service.ts:173-213`）
```typescript
@OnJob({ name: JobName.StorageTemplateMigration, queue: QueueName.StorageTemplateMigration })
async handleMigration(): Promise<JobStatus> {
  // 全量迁移逻辑
  const assets = this.assetJobRepository.streamForStorageTemplateJob();
  for await (const asset of assets) {
    await this.moveAsset(asset, { storageLabel, filename });
  }
  await this.storageRepository.removeEmptyDirs(libraryFolder);
}
```

**触发链**：
```
管理员操作 → POST /api/queue/storageTemplateMigration/start
  → queue.service:runCommandLegacy()
  → case QueueName.StorageTemplateMigration
  → 显式排队 JobName.StorageTemplateMigration 任务
  → handleMigration() 执行全量迁移
```

### 5.4 配置更新与迁移任务的关系（**更正：配置更新≠自动迁移**）

**配置更新流程**：`system-config.service.ts:59-79`

```typescript
async updateSystemConfig(dto: SystemConfigDto): Promise<SystemConfigDto> {
  // 1. 验证配置
  await this.eventRepository.emit('ConfigValidate', { newConfig, oldConfig });
  
  // 2. 保存到数据库
  const newConfig = await this.updateConfig(dto);
  
  // 3. 广播配置更新事件
  await this.eventRepository.emit('ConfigUpdate', { newConfig, oldConfig });
  
  return mapConfig(newConfig);
}
```

**配置更新事件处理**：`storage-template.service.ts:101-104`

```typescript
@OnEvent({ name: 'ConfigUpdate', server: true })
onConfigUpdate({ newConfig }: ArgOf<'ConfigUpdate'>) {
  this.onConfigInit({ newConfig });  // 仅重新编译模板，不触发迁移！
}
```

**配置初始化**：`storage-template.service.ts:92-99`

```typescript
@OnEvent({ name: 'ConfigInit' })
onConfigInit({ newConfig }: ArgOf<'ConfigInit'>) {
  const template = newConfig.storageTemplate.template;
  if (!this._template || template !== this.template.raw) {
    this.logger.debug(`Compiling new storage template: ${template}`);
    this._template = this.compile(template);  // 仅编译，不迁移
  }
}
```

**完整关系图**：

```
用户修改存储模板
    ↓
PUT /api/system-config
    ↓
system-config.service:updateSystemConfig()
    ├─→ emit('ConfigValidate')  # 验证模板语法
    ├─→ 保存配置到 system_metadata 表
    └─→ emit('ConfigUpdate', { newConfig, oldConfig })
            ↓
            ├─→ storage-template.service:onConfigUpdate()
            │       └─→ onConfigInit() → 重新编译模板到内存
            │           （仅此而已，不触发迁移）
            │
            ├─→ system-config.service:onConfigUpdate()
            │       └─→ clearConfigCache()
            │
            └─→ 其他服务的 ConfigUpdate 处理...

配置更新完成 ✅
    ↓
（此时模板已更新，但已有资产仍在旧路径）
    ↓
管理员手动触发全量迁移 ⚠️ （必须显式操作）
    ↓
POST /api/queue/storageTemplateMigration/start
    ↓
queue.service:runCommandLegacy()
    ↓
排队 JobName.StorageTemplateMigration 任务
    ↓
handleMigration() → 遍历所有资产 → 逐个 moveAsset()
```

**关键结论**：
1. **配置更新仅重新编译模板**，不会自动迁移已有资产
2. **全量迁移必须通过队列 API 显式触发**
3. 单资产迁移（新上传）是自动的，全量迁移（历史资产）是手动的
4. 这种设计避免了模板变更意外触发大规模 IO 操作

### 5.5 原子化移动保障

**核心代码**：`storage.core.ts:181-257`

#### 5.5.1 移动历史记录

```typescript
// 开始移动前创建记录
move = await this.moveRepository.create({ entityId, pathType, oldPath, newPath });

// 移动完成后删除记录
await this.moveRepository.delete(move.id);
```

#### 5.5.2 断点续迁

```typescript
let move = await this.moveRepository.getByEntity(entityId, pathType);
if (move) {
  // 检测文件实际位置
  const oldPathExists = await this.storageRepository.checkFileExists(move.oldPath);
  const newPathExists = await this.storageRepository.checkFileExists(move.newPath);
  const actualPath = oldPathExists ? move.oldPath : (newPathExists ? move.newPath : null);
  // 从实际位置继续执行
}
```

#### 5.5.3 跨设备移动降级

```typescript
try {
  await this.storageRepository.rename(move.oldPath, newPath);
} catch (error: any) {
  if (error.code !== 'EXDEV') throw;
  // 跨设备（EXDEV）时降级为：复制 → 验证 → 删除
  await this.storageRepository.copyFile(move.oldPath, newPath);
  if (!(await this.verifyNewPathContentsMatchesExpected(...))) {
    await this.storageRepository.unlink(newPath);
    return;
  }
  await this.storageRepository.utimes(newPath, atime, mtime);
  await this.storageRepository.unlink(move.oldPath);
}
```

### 5.6 文件完整性校验

**核心代码**：`storage.core.ts:259-293`

```typescript
private async verifyNewPathContentsMatchesExpected(
  oldPath: string, newPath: string, 
  assetInfo?: { sizeInBytes: number; checksum: Buffer }
) {
  // 1. 文件大小校验
  if (newPathSize !== oldPathSize) return false;
  
  // 2. 可选：哈希校验（需配置 hashVerificationEnabled）
  if (assetInfo && config.storageTemplate.hashVerificationEnabled) {
    const newChecksum = await this.cryptoRepository.hashFile(newPath);
    if (!newChecksum.equals(checksum)) return false;
  }
  return true;
}
```

### 5.7 数据库与文件系统一致性

**核心代码**：`storage.core.ts:308-326`

```typescript
private savePath(pathType: PathType, id: string, newPath: string) {
  switch (pathType) {
    case AssetPathType.Original:
      return this.assetRepository.update({ id, originalPath: newPath });
    case AssetFileType.FullSize:
    case AssetFileType.Thumbnail:
    case AssetFileType.Preview:
      return this.assetRepository.upsertFile({ assetId: id, type, path: newPath });
    // ...
  }
}
```

### 5.8 媒体位置变更迁移

**核心代码**：`storage.service.ts:97-132`

当检测到 `UPLOAD_LOCATION` 环境变量变更时：

```typescript
if (previous !== current) {
  this.logger.warn(`Media location changed, performing automatic migration...`);
  await this.databaseRepository.migrateFilePaths(previous, current);
}
```

---

## 六、配置验证机制

**核心代码**：`storage-template.service.ts:106-130`

系统启动或配置变更时，使用模拟数据验证模板：

```typescript
@OnEvent({ name: 'ConfigValidate' })
onConfigValidate({ newConfig }: ArgOf<'ConfigValidate'>) {
  this.render(compiled, {
    asset: {
      fileCreatedAt: new Date(),
      originalPath: '/upload/test/IMG_123.jpg',
      type: AssetType.Image,
      id: 'd587e44b-f8c0-4832-9ba3-43268bbf5d4e',
    },
    filename: 'IMG_123',
    extension: 'jpg',
    albumName: 'album',
    // ...
  });
}
```

---

## 七、特殊场景处理

### 7.1 动态照片（Live Photo）

**核心代码**：`storage-template.service.ts:160-169`

```typescript
if (asset.livePhotoVideoId) {
  const livePhotoVideo = await this.assetJobRepository.getForStorageTemplateJob(asset.livePhotoVideoId);
  const motionFilename = getLivePhotoMotionFilename(filename, livePhotoVideo.originalPath);
  await this.moveAsset(livePhotoVideo, { storageLabel, filename: motionFilename }, asset);
}
```

**关键点**：
- 动态照片的视频部分使用静态照片的日期信息
- 确保照片和视频部分最终在同一目录
- 视频文件名继承照片文件名，仅变更扩展名

### 7.2 Sidecar 文件

**核心代码**：`storage-template.service.ts:247-255`

```typescript
const sidecarPath = getAssetFile(asset.files, AssetFileType.Sidecar, { isEdited: false })?.path;
if (sidecarPath) {
  await this.storageCore.moveFile({
    entityId: id,
    pathType: AssetFileType.Sidecar,
    oldPath: sidecarPath,
    newPath: `${newPath}.xmp`,
  });
}
```

### 7.3 外部资产豁免

**核心代码**：`storage-template.service.ts:222-226`

```typescript
if (asset.isExternal || StorageCore.isAndroidMotionPath(asset.originalPath)) {
  return;  // 外部资产不受存储模板影响
}
```

---

## 八、关键设计权衡

### 8.1 异步迁移 vs 同步应用

**选择**：异步迁移（通过任务队列）

**理由**：
- 避免阻塞上传请求
- 元数据提取完成后才具备完整插值数据
- 支持批量重试和断点续迁

### 8.2 重命名后缀设计

**选择**：`+N` 而非 `(N)` 或 `_N`

**理由**：
- `+` 在文件名中合法且少见
- 正则 `^\+\d+$` 可精确匹配，避免误判
- 不影响文件扩展名识别

### 8.3 数据库锁粒度

**核心代码**：`storage-template.service.ts:228`

```typescript
return this.databaseRepository.withLock(DatabaseLock.StorageTemplateMigration, async () => {
  // 单资产迁移全过程
});
```

**设计**：使用全局数据库锁 `StorageTemplateMigration`，避免并发迁移冲突。

### 8.4 配置更新与迁移分离

**选择**：配置更新仅编译模板，不自动触发迁移

**理由**：
- 避免模板误修改导致大规模 IO 风暴
- 给管理员留出验证和确认时间
- 全量迁移是重量级操作，应由管理员显式决策

---

## 九、代码调用链总结

### 9.1 新资产上传完整路径转换链

```
1. asset-media.service.ts:127-161
   uploadAsset()
   └── create() → 设置 originalPath = upload/{uuid前2位}/{uuid2-4位}/{uuid}.ext

2. asset-media.service.ts:361
   排队 JobName.AssetExtractMetadata

3. metadata.service
   提取 EXIF 完成后 emit('AssetMetadataExtracted')

4. storage-template.service.ts:136-139
   onAssetMetadataExtracted()
   └── 排队 JobName.StorageTemplateMigrationSingle  ✅ 自动触发

5. storage-template.service.ts:142-171
   handleMigrationSingle()
   └── moveAsset()

6. storage-template.service.ts:221-260
   moveAsset()
   ├── getTemplatePath()  # 模板插值
   └── storageCore.moveFile()

7. storage-template.service.ts:262-389
   getTemplatePath()
   ├── 扩展名规范化
   ├── rootPath = getLibraryFolder(user)  # 用户隔离
   ├── 编译模板插值
   ├── 路径安全校验
   ├── 重复后缀检测
   └── 冲突重命名循环

8. storage.core.ts:181-257
   moveFile()
   ├── 创建 move_history 记录
   ├── 执行 rename/copy+verify+delete
   ├── savePath() → 更新数据库
   └── 删除 move_history 记录
```

### 9.2 全量迁移触发链（**更正：显式触发**）

```
1. 管理员通过 UI/API 启动迁移
   ↓
2. POST /api/queue/storageTemplateMigration/start
   ↓
3. queue.service.ts:198-199
   runCommandLegacy()
   └── case QueueName.StorageTemplateMigration
       └── 显式排队 JobName.StorageTemplateMigration  ⚠️ 必须显式操作

4. storage-template.service.ts:173-213
   handleMigration()
   ├── 流式遍历所有资产
   ├── 按用户分组获取 storageLabel
   ├── 逐个 moveAsset()
   └── 清理空目录
```

### 9.3 配置更新流程（**更正：不自动触发迁移**）

```
1. 管理员更新存储模板配置
   ↓
2. PUT /api/system-config
   ↓
3. system-config.service.ts:59-79
   updateSystemConfig()
   ├── emit('ConfigValidate')  # 模板语法验证
   ├── 保存配置到数据库
   └── emit('ConfigUpdate')  # 广播配置变更

4. storage-template.service.ts:101-104
   onConfigUpdate()
   └── onConfigInit()
       └── compile(template)  # 仅重新编译内存中的模板
       （仅此而已，不触发迁移！）

5. 配置更新完成，但资产仍在旧路径
   ↓
6. 管理员手动触发全量迁移（如步骤 1-2 所示）
```

---

## 十、两处事实偏差更正对照表

| 事项 | Round 1（偏差） | Round 2（修正） | 证据来源 |
|------|----------------|----------------|----------|
| 预设模板数量 | 22 个 | **21 个** | `storage-template.service.ts:35-57` 数组定义 + `storage-template.service.spec.ts:67-89` 测试验证 |
| 预设模板来源 | 未明确 | 定义在 `storage-template.service.ts` 顶部 `storagePresets` 常量，通过 API 返回给前端 | `storage-template.service.ts:132-134` |
| 全量迁移触发 | "模板变更自动触发" | **必须通过队列 API 显式触发**，配置更新仅重新编译模板 | `storage-template.service.ts:101-104` 只调用 `onConfigInit` 编译；`queue.service.ts:198-199` 显示需显式排队 |
| 配置更新与迁移关系 | 未明确区分 | 配置更新 = 内存模板重编译；全量迁移 = 独立显式操作 | `system-config.service.ts:59-79` 完整流程 + `storage-template.service.ts:92-104` 事件处理 |

---

## 十一、关键文件速查表

| 文件 | 核心函数/类 | 行数 | 主要职责 |
|------|------------|------|----------|
| `storage-template.service.ts` | `StorageTemplateService` | 1-431 | 模板服务主类 |
| ↳ | `compile()` | 391-398 | 模板编译与依赖分析 |
| ↳ | `render()` | 400-430 | 字段插值渲染 |
| ↳ | `getTemplatePath()` | 262-389 | 目标路径生成（含冲突处理） |
| ↳ | `moveAsset()` | 221-260 | 单资产迁移入口 |
| ↳ | `handleMigrationSingle()` | 142-171 | 单资产迁移任务处理（自动触发） |
| ↳ | `handleMigration()` | 173-213 | 全量迁移任务处理（需显式触发） |
| ↳ | `onConfigUpdate()` | 101-104 | 配置更新处理（仅编译，不迁移） |
| ↳ | `storagePresets` | 35-57 | 21 个预设模板定义 |
| `storage.core.ts` | `StorageCore` | 1-361 | 存储核心类 |
| ↳ | `getLibraryFolder()` | 104-106 | 用户根目录计算 |
| ↳ | `moveFile()` | 181-257 | 原子化文件移动 |
| ↳ | `verifyNewPathContentsMatchesExpected()` | 259-293 | 文件完整性校验 |
| ↳ | `savePath()` | 308-326 | 数据库路径更新 |
| `move.repository.ts` | `MoveRepository` | 1-63 | 移动历史管理 |
| `user.table.ts` | `UserTable` | 27-85 | 用户表定义（含 storageLabel） |
| `asset-media.service.ts` | `AssetMediaService` | 45-371 | 资产上传入口 |
| `system-config.service.ts` | `SystemConfigService` | 1-85 | 配置管理、ConfigUpdate 事件广播 |
| `queue.service.ts` | `QueueService` | 198-200 | 全量迁移任务显式触发入口 |
| `config.ts` | `defaults` | 318-322 | 默认配置（模板默认禁用） |
