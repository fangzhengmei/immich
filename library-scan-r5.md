# Immich LibrarySyncFiles 异常传播路径核验

## 一、异常传播路径逐行证明

### 1.1 Promise.all 层异常隔离

```typescript
// library.service.ts:260-267
const assetImports: Insertable<AssetTable>[] = [];
await Promise.all(
  job.paths.map((path) =>
    this.processEntity(path, library.ownerId, job.libraryId)
      .then((asset) => assetImports.push(asset))
      .catch((error: any) => this.logger.error(`Error processing ${path}: ${error}`)),
      // ⚠️ 关键证据 1: 每个文件的异常被独立 .catch 捕获，仅打日志，不向外抛出
  ),
);
```

**结论 1**：单个文件处理异常被隔离，不会导致整个 `Promise.all` reject。

### 1.2 createAll 执行条件

```typescript
// library.service.ts:269
const assetIds = await this.assetRepository.createAll(assetImports);
```

- `assetImports` 只包含 `processEntity` 成功的资产
- 失败的资产不会被 push 到数组中
- `createAll` 接收的数组可能为空，但**不会因为之前的异常失败**

**关键证据 - ChunkedArray 装饰器对空数组的处理**：
```typescript
// decorators.ts:81-86
// Early return if argument length is less than or equal to the chunk size.
if (
  (Array.isArray(argument) && argument.length <= chunkSize) ||
  (argument instanceof Set && argument.size <= chunkSize)
) {
  return originalMethod.apply(this, arguments_);
}
```

- 空数组 `length = 0 <= 4000` → 直接调用原方法
- kysely 的 `insertInto(...).values([])` 对空数组的处理：生成合法的 INSERT 0 rows，不会抛异常

**结论 2**：`createAll` 必然成功执行，返回空数组 `[]` 或部分成功的 ID 数组。

### 1.3 queuePostSyncJobs 的执行确定性

```typescript
// library.service.ts:278
await this.queuePostSyncJobs(assetIds);

// library.service.ts:421-430
async queuePostSyncJobs(assetIds: string[]) {
  this.logger.debug(`Queuing sidecar discovery for ${assetIds.length} asset(s)`);
  // queueAll 接收空数组时是安全的 no-op
  await this.jobRepository.queueAll(
    assetIds.map((assetId) => ({
      name: JobName.SidecarCheck,
      data: { id: assetId, source: 'upload' },
    })),
  );
}
```

**结论 3**：`queuePostSyncJobs` 必然被调用，不受之前异常的影响。
- 有成功资产 → 触发 SidecarCheck
- 所有资产失败（assetIds = []）→ 空操作，不触发任何任务

### 1.4 任务最终状态

```typescript
// library.service.ts:280
return JobStatus.Success;
```

**结论 4**：LibrarySyncFiles 任务最终返回 `JobStatus.Success`，不会被标记为失败。

---

## 二、异常传播路径总结图

```
                    触发 LibrarySyncFiles
                             ↓
                    ┌──────────────────┐
                    │   Promise.all    │
                    │  并行处理 N 个文件│
                    └────────┬─────────┘
                             ↓
                ┌────────────┴────────────┐
                │                         │
       processEntity 成功          processEntity 失败
                │                         │
                ↓                         ↓
       assetImports.push(asset)    .catch 打 error 日志
                │                         │
                └────────────┬────────────┘
                             ↓
                    createAll(assetImports)
                    (只包含成功的资产)
                             ↓
                    queuePostSyncJobs(assetIds)
                    (空数组 = 无后续任务)
                             ↓
                    return JobStatus.Success
                             ✅
```

---

## 三、Change 事件在已存在文件场景的条件分析

### 3.1 必要条件 (Necessary Conditions)

| 编号 | 必要条件 | 验证方式 | 不满足时的结果 |
|------|---------|---------|--------------|
| **NC1** | 文件系统发出 change 事件 | inotify 监控触发 | ❌ 整个流程不启动 |
| **NC2** | 路径通过 mime 类型匹配过滤 | picomatch 匹配 | ❌ 事件被静默忽略，不入队 |
| **NC3** | 路径在 importPaths 下，不被 exclusionPatterns 排除 | 同 NC2 | ❌ 同上 |
| **NC4** | library 存在，未被删除 | handleSyncFiles 开头检查 | ❌ 任务提前返回 Failed |
| **NC5** | 唯一索引冲突 (`ownerId + libraryId + checksum`) | 路径不变 → sha1Path checksum 不变 | ⚠️ 该资产被过滤，不出现在结果中 |

### 3.2 充分条件 (Sufficient Conditions)

| 场景 | 充分条件组合 | 最终结果 |
|------|-------------|----------|
| **S1 - 纯已存在文件场景** | NC1 ✅ + NC2 ✅ + NC3 ✅ + NC4 ✅ + NC5 ✅ (必然冲突) | 1. 打一条 error 日志<br>2. assetIds = []<br>3. 不触发 queuePostSyncJobs<br>4. **任务返回 JobStatus.Success** ✅ |
| **S2 - 混合场景 (部分新文件)** | NC1 ✅ + NC2 ✅ + NC3 ✅ + NC4 ✅ + 部分文件 NC5 ❌ (不冲突) | 1. 已存在文件打 error 日志<br>2. 新文件正常插入<br>3. 新文件触发 queuePostSyncJobs<br>4. **任务返回 JobStatus.Success** ✅ |

### 3.3 条件逻辑关系

```
NC1 (change 事件)
  ↓
NC2 (mime 匹配)
  ↓
NC3 (路径在 import 范围内)
  ↓
NC4 (library 存在)
  ↓
┌─────────────────────────────────────────────┐
│  NC5 (唯一索引冲突) 对每个文件独立判定       │
│  每个文件的异常被 .catch 隔离，不影响其他文件 │
└──────────────────────┬──────────────────────┘
                       ↓
               createAll 执行成功
               (只包含不冲突的文件)
                       ↓
               queuePostSyncJobs 执行
               (空数组 = 无后续任务)
                       ↓
               return JobStatus.Success ✅
```

---

## 四、结论对照表（无歧义版）

| 判定项 | 结果 | 证据来源 |
|--------|------|---------|
| **createAll 抛错后任务是否失败？** | ❌ 不会，异常在 Promise.all 层面被隔离 | library.service.ts:265 |
| **已存在文件是否会让 createAll 抛错？** | ✅ 会，但只在单个 Promise 层面 | 唯一索引约束定义 |
| **该异常是否会影响其他文件插入？** | ❌ 不会，每个文件独立处理 | Promise.all + .catch 隔离 |
| **queuePostSyncJobs 是否还会执行？** | ✅ 必然执行，只是传入的 assetIds 可能为空 | library.service.ts:278 |
| **已存在文件是否会触发元数据提取？** | ❌ 完全不会，不在 assetIds 返回值中 | 异常资产未 push 到 assetImports |
| **任务最终状态是什么？** | ✅ JobStatus.Success | library.service.ts:280 |
| **用户可见的异常痕迹是什么？** | 一条 "Error processing {path}" error 日志 | library.service.ts:265 |

---

## 五、可验证的代码片段集合

### 验证点 1：Promise.all 异常隔离
```typescript
// library.service.ts:261-267
await Promise.all(
  job.paths.map((path) =>
    this.processEntity(path, library.ownerId, job.libraryId)
      .then((asset) => assetImports.push(asset))
      .catch((error: any) => this.logger.error(`Error processing ${path}: ${error}`)),
      // 无 rethrow，异常被吞噬
  ),
);
```

### 验证点 2：createAll 无 try-catch
```typescript
// library.service.ts:269
const assetIds = await this.assetRepository.createAll(assetImports);
// 不包裹 try-catch，直接 await
// assetImports 只包含成功的对象
```

### 验证点 3：queuePostSyncJobs 无条件执行
```typescript
// library.service.ts:276-278
this.logger.log(`Imported ${assetIds.length} ... file(s) into library ${job.libraryId}`);
await this.queuePostSyncJobs(assetIds);
// 在 createAll 之后，无任何条件判断
```

### 验证点 4：任务成功返回
```typescript
// library.service.ts:280
return JobStatus.Success;
// 函数体内只有两处提前返回：library 不存在/已删除时返回 Failed
// 其余所有路径最终都返回 Success
```

---

## 六、关键修正与澄清

对比之前版本的结论修正：

| 之前的表述 | 修正后的准确表述 |
|-----------|-----------------|
| "任务失败，异常被 catch" | ❌ 不准确 |
| "queuePostSyncJobs 永远不会被调用" | ❌ 错误，**实际上必然被调用**，只是 assetIds 可能为空导致无实际效果 |
| "静默丢弃" | ✅ 准确，但需补充：伴随一条 error 日志，且任务状态为 Success |

**准确表述总结**：

> 当 change 事件触发已存在文件的 LibrarySyncFiles 时：
> 1. 该文件的 INSERT 会因唯一索引约束失败，被记录一条 error 日志
> 2. 该异常不会向上传播，不会影响同批次其他文件的导入
> 3. createAll 和 queuePostSyncJobs 都会正常执行
> 4. 任务最终返回 JobStatus.Success
> 5. 唯一的副作用是日志中有一条数据库约束冲突错误，但该资产不会触发任何后续的元数据提取任务
