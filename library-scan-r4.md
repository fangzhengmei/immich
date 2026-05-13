# Immich Change 事件链路条件分支梳理与核验表

## 一、关键前置证据

### 1.1 唯一约束定义（非 originalPath）

```sql
-- 证据来源：schema/migrations/1752267649968-StandardizeNames.ts:545
CREATE UNIQUE INDEX "asset_ownerId_libraryId_checksum_idx" 
ON "asset" ("ownerId", "libraryId", "checksum") 
WHERE ("libraryId" IS NOT NULL);
```

**关键结论**：唯一约束基于 `ownerId + libraryId + checksum`，而非 `originalPath`。

### 1.2 外部库 checksum 生成逻辑（纯路径哈希）

```typescript
// library.service.ts:407-408
checksum: this.cryptoRepository.hashSha1(`path:${assetPath}`),
checksumAlgorithm: ChecksumAlgorithm.sha1Path,

// enum.ts:44-49
export enum ChecksumAlgorithm {
  /** sha1 checksum of the whole file contents */
  sha1File = 'sha1',
  /** sha1 checksum of "path:" plus the file path, currently used in external libraries, deprecated */
  sha1Path = 'sha1-path',
}
```

**关键结论**：外部库的 checksum 仅由**文件路径**决定，与文件内容、mtime 完全无关。只要路径不变，checksum 就不变。

---

## 二、LibrarySyncFiles 条件分支核验

### 2.1 执行流程图

```
                触发 LibrarySyncFiles
                        ↓
            ┌───────────────────────────┐
            │     processEntity 执行     │
            │  基于路径生成 sha1Path    │
            └─────────────┬─────────────┘
                          ↓
            ┌───────────────────────────┐
            │      createAll 执行       │
            │  INSERT INTO asset (...)  │
            └─────────────┬─────────────┘
                          ↓
              ┌───────────┴───────────┐
              │                       │
        无冲突 ✓                 有冲突 ✗
              │                       │
              ↓                       ↓
    ┌───────────────────┐   ┌───────────────────────┐
    │ assetIds = [ids]  │   │  异常被 Promise.catch │
    │  进入后续任务      │   │  assetIds 为空数组    │
    └───────────┬───────┘   └───────────┬───────────┘
                ↓                         ↓
    queuePostSyncJobs(非空)    queuePostSyncJobs(空数组)
                ↓                         ↓
    SidecarCheck + 元数据提取     无任何后续任务执行
```

### 2.2 分支一：媒体文件不存在于数据库中

| 步骤 | 行为 | 结果 | 证据 |
|------|------|------|------|
| **processEntity** | 基于路径生成 `sha1Path` checksum | 正常生成 Insertable 对象 | library.service.ts:400-419 |
| **createAll** | 执行纯 INSERT | 无冲突，返回新生成的 assetIds | asset.repository.ts:441-445 |
| **queuePostSyncJobs** | 传入非空 assetIds | ✅ 触发 SidecarCheck → MetadataExtraction → 缩略图生成 → 人脸识别 | library.service.ts:421-430 |

**结论**：新增文件场景下，完整后续任务链正常触发。

### 2.3 分支二：媒体文件已存在于数据库中

| 步骤 | 行为 | 结果 | 证据 |
|------|------|------|------|
| **processEntity** | 基于路径生成 `sha1Path` checksum | **路径不变 → checksum 不变** | library.service.ts:407 |
| **createAll** | 执行纯 INSERT | 唯一索引冲突 (`ownerId + libraryId + checksum`)，**异常被 Promise.catch 捕获** | library.service.ts:263-266 |
| **assetImports** | 路径对应的对象已生成，但插入失败 | **该资产不会出现在 assetIds 返回值中** | |
| **queuePostSyncJobs** | 传入空/不含该资产的 assetIds | ❌ 该资产不触发任何后续任务 | library.service.ts:278 |

**关键证据 - Promise.catch 隔离**：
```typescript
// library.service.ts:261-266
await Promise.all(
  job.paths.map((path) =>
    this.processEntity(path, library.ownerId, job.libraryId)
      .then((asset) => assetImports.push(asset))
      .catch((error: any) => this.logger.error(`Error processing ${path}: ${error}`)),
      // ⚠️ 单个文件处理失败被隔离，不影响其他文件
  ),
);
```

**结论**：文件已存在时，change 事件被静默丢弃，不触发任何元数据更新。

---

## 三、定时巡检 mtime 检测链路核验

### 3.1 触发条件与判定逻辑

```typescript
// library.service.ts:576-613
private checkExistingAsset(
  asset: { isOffline: boolean; fileModifiedAt: Date; ... },
  stat: Stats | null,
): AssetSyncResult {
  // 离线判定省略...

  // ⚠️ 唯一更新触发条件：文件 mtime 变化
  if (stat.mtime.valueOf() !== asset.fileModifiedAt.valueOf()) {
    this.logger.verbose(`Asset ${asset.originalPath} needs metadata extraction`);
    return AssetSyncResult.UPDATE;
  }

  return AssetSyncResult.DO_NOTHING;
}
```

### 3.2 LibrarySyncAssets 执行流程

| 步骤 | 行为 | 触发条件 |
|------|------|----------|
| **并行 stat** | 获取所有待检资产的当前文件状态 | 所有资产 |
| **mtime 比对** | `stat.mtime` vs 数据库 `asset.fileModifiedAt` | 值不同 → UPDATE 状态 |
| **收集 assetIdsToUpdate** | 所有 mtime 变化的资产 ID 数组 | 至少有一个文件变化 |
| **queuePostSyncJobs** | 批量触发后续任务 | 同 LibrarySyncFiles 相同逻辑 |

### 3.3 关键差异：两条链路触发条件对比

| 触发维度 | 实时 Change 事件链路 | 定时巡检链路 |
|----------|---------------------|--------------|
| **触发信号** | 文件系统 inotify 事件 | Cron 定时调度 |
| **判定逻辑** | 无任何内容判定，直接 INSERT | mtime 精确比对，变化才触发 |
| **checksum 感知** | 仅路径哈希，不感知内容 | 不依赖 checksum |
| **内容变化感知** | ❌ 完全不感知（只要路径不变就冲突） | ✅ 通过 mtime 间接感知 |
| **对已存在文件** | 静默丢弃 + 日志 error | 检测到变化则完整更新 |

---

## 四、综合结论对比表

| 场景 | Change 事件 (实时) | LibrarySyncAssets (定时) | 是否一致 |
|------|---------------------|--------------------------|----------|
| **全新文件导入** | ✅ 完整任务链触发 | ✅ 同左 (扫描后也会发现) | 一致 |
| **文件内容修改 (mtime 变化)** | ❌ 静默丢弃，不更新 | ✅ 检测到 mtime 变化，触发更新 | **不一致** |
| **文件内容修改 (mtime 不变)** | ❌ 静默丢弃，不更新 | ❌ 无法检测，不更新 | 一致 |
| **文件被删除** | ❌ LibraryRemoveAsset 硬删除 | ✅ 标记 isOffline = true | **不一致** |
| **Sidecar 文件单独修改** | ❌ 被 mime 过滤器忽略 | ❌ 主文件 mtime 不变，无法检测 | 一致 |

---

## 五、可验证的日志观察点

执行以下命令可验证上述结论：

```bash
# 1. 观察 change 事件接收情况
grep "File change event received" immich.log

# 2. 观察单个文件处理异常
grep "Error processing" immich.log
# ⚠️ 预期：change 事件触发的路径会出现唯一约束冲突错误

# 3. 观察 LibrarySyncFiles 导入计数
grep "Imported .* file(s) into library" immich.log
# ⚠️ 预期：change 事件不会增加导入计数

# 4. 观察定时扫描的更新检测
grep "needs metadata extraction" immich.log
grep -E "offlined|onlined|updated" immich.log
# ⚠️ 预期：仅定时扫描期间才会出现这些日志
```

---

## 六、设计本质与改进空间

### 6.1 当前设计的本质

**实时事件链路 = 仅负责新文件发现**  
**定时巡检链路 = 负责所有状态变化（修改 + 删除 + 恢复）**

change 事件的设计意图并非处理文件更新，而是：
1. 快速发现扫描间隔内新增的文件
2. 与定时扫描形成互补，减少新增文件的可见延迟

### 6.2 现存问题

1. **更新延迟不可控**：文件修改后需等待下一次定时扫描才会被处理
2. **错误日志噪音**：change 事件必然触发的唯一约束冲突容易被误判为系统异常
3. **checksum 设计缺陷**：`sha1Path` 已被标记 deprecated，但外部库仍在使用

### 6.3 改进方向建议

在不改变现有架构的前提下，可优化的点：
1. Change 事件处理器在入队前先通过 `filterNewExternalAssetPaths` 预过滤，避免无意义的数据库冲突
2. 对于已存在文件的 change 事件，可直接调用 `queuePostSyncJobs` 触发元数据提取，不依赖 mtime 检测
3. 冲突日志级别从 error 降级为 debug/verbose，减少噪音
