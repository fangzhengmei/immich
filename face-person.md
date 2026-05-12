# Immich 人脸向量聚类与 Person 管理协同机制

## 一、完整数据流决策链

### 1.1 端到端处理流程

```
资产上传
    ↓
[队列: FaceDetection]
    ↓
handleDetectFaces() ── 人脸检测 + 向量生成 ── ML服务
    │
    ├─ 匹配已有人脸(IOU算法) → 更新embedding
    └─ 新检测人脸 → 插入asset_face + face_search
    ↓
[队列: FacialRecognition]  deferred: false
    ↓
handleRecognizeFaces()
    │
    ├─ 预检查: 已分配Person? → Skipped
    ├─ 预检查: sourceType != ML? → Skipped
    ├─ 预检查: 无embedding? → Failed
    │
    ├─ 第一次向量搜索(numResults = minFaces)
    │   └─ matches = 相似人脸(含自身)
    │
    ├─ 核心脸判定:
    │   └─ matches >= minFaces AND visibility = Timeline
    │
    ├─ 非核心脸 + 首次处理:
    │   └─ 延迟入队(deferred: true) → 返回Skipped
    │
    ├─ 第二次向量搜索(hasPerson = true)
    │   └─ 寻找最近的已有Person
    │
    ├─ Person创建条件:
    │   └─ 是核心脸 AND 两次搜索均无匹配
    │       └─ 创建新Person + 生成缩略图
    │
    └─ 分配PersonId: reassignFaces()
        ↓
[背景任务: PersonCleanup]
        ↓
    删除无任何人脸的空Person
```

---

## 二、向量聚类核心算法

### 2.1 向量索引设计

```sql
-- 使用pgvector的HNSW索引，余弦相似度
CREATE INDEX face_index ON face_search
USING hnsw (embedding vector_cosine_ops)
WITH (m = 16, ef_construction = 300);
```

**索引参数说明**：
- `m = 16`: 每个节点的邻居数量，平衡精度/内存
- `ef_construction = 300`: 构建时动态候选列表大小
- `vector_cosine_ops`: 余弦相似度，适合人脸特征

### 2.2 两次搜索策略

**第一次搜索**：
```typescript
searchFaces({
  userIds: [ownerId],
  embedding: faceEmbedding,
  maxDistance: config.maxDistance,
  numResults: config.minFaces,
  minBirthDate: assetCreatedAt,
})
```
**目的**：判断是否为"核心脸"，即该人脸在库中已有足够多的相似样本

**第二次搜索**：
```typescript
searchFaces({
  ...,
  numResults: 1,
  hasPerson: true,
})
```
**目的**：寻找最近的已有Person，避免创建重复Person

### 2.3 核心脸判定逻辑

| 条件 | 说明 |
|------|------|
| `matches >= minFaces` | 相似人脸数达到阈值 |
| `visibility = Timeline` | 资产在时间线中可见 |
| `deferred = false` | 首次处理 → 延迟；延迟后处理 → 强制匹配 |

**延迟处理机制**：
- 非核心脸首次入队时标记 `deferred: false`
- 处理时发现非核心 → 重新入队 `deferred: true`
- 第二次处理时无论是否核心都强制进行Person分配

---

## 三、Person 管理操作与聚类联动

### 3.1 创建 Person

**入口**：`POST /people` + `PersonCreate` 权限

```typescript
// 手动创建空Person
create({ ownerId, name, birthDate, isHidden, isFavorite, color })

// 聚类自动创建Person
create({ ownerId, faceAssetId }) → 关联特征脸
→ 触发 PersonGenerateThumbnail 任务
```

**创建后联动**：
- 新Person的 `birthDate` 作为后续向量搜索的过滤条件
- `faceAssetId` 指向的人脸embedding成为该Person的"锚点向量"

### 3.2 Person 字段更新的影响差异

**入口**：`PUT /people/:id` + `PersonUpdate` 权限

```typescript
update(id, { name, birthDate, isHidden, featureFaceAssetId, isFavorite, color })
```

**各字段对聚类结果的影响差异对照表**：

| 字段 | 对聚类的实际影响 | 作用机制 | 是否影响自动匹配 |
|------|-----------------|---------|-----------------|
| **`name`** | **无直接影响** | 仅用于UI显示，不参与任何向量搜索或匹配逻辑 | ❌ |
| **`birthDate`** | **影响显著** | 作为 `minBirthDate` 过滤条件传入 `searchFaces` | ✅ |
| **`featureFaceAssetId`** | **仅影响排序** | 改变 Person 列表按相似度排序的基准向量 | ⚠️ |
| **`isHidden`** | **影响范围** | 隐藏的 Person 不会出现在 getAll 结果中，但仍可被匹配 | ⚠️ |
| **`isFavorite`** | **无直接影响** | 仅用于UI排序，不参与匹配逻辑 | ❌ |
| **`color`** | **无直接影响** | 仅用于UI显示 | ❌ |

---

**详细联动机制**：

1. **`birthDate` 更新** → 直接改变匹配可能性
   - **过滤逻辑**：
     ```sql
     WHERE (person.birthDate IS NULL OR person.birthDate <= minBirthDate)
     ```
   - 示例：Person生日从 2000年 → 1990年
     - 搜索范围扩大了10年
     - 1995年拍摄的同人人脸现在可以匹配到该Person

2. **`featureFaceAssetId` 更新** → 仅影响相似度排序
   - 改变 "按相似度展示Person" 的排序顺序
   - 人脸自动聚类的 `searchFaces` 调用完全不使用 faceAssetId

3. **`name` 更新** → 纯UI变化
   - 向量搜索和聚类完全不参与

### 3.3 合并 Person

**入口**：`POST /people/:id/merge` + `PersonMerge` 权限

**合并后对聚类的影响**：
- 所有被合并Person的人脸embedding现在归属于同一Person
- 后续新人脸搜索时匹配概率显著提升
- 减少Person碎片化，提高跨相册识别准确率

### 3.4 拆分 Person

**拆分 = 人脸重新分配到新/其他Person**

**入口1**：`PUT /people/:id/reassign` + `PersonReassign` 权限
**入口2**：`PUT /faces/:id` + `FaceUpdate` 权限

**拆分联动逻辑**：
- 接收方Person无特征脸 → 选新移入的脸作为特征脸
- 移出方Person失去特征脸 → 随机选一张脸作为新特征脸
- 触发 `PersonGenerateThumbnail` 任务生成新缩略图

### 3.5 删除 Person

**入口**：`DELETE /people/:id` + `PersonDelete` 权限

**注意**：删除Person不会删除对应的人脸，这些人脸将在后续FacialRecognition任务中重新聚类

---

## 四、跨相册识别机制

### 4.1 向量空间共享设计

```
所有相册的所有人脸共享同一个 face_search 表
    ↓
搜索时通过 userIds 过滤确保只匹配当前用户的人脸
    ↓
跨相册的同人人脸因为embedding相似 → 距离近 → 匹配到同一Person
```

**关键约束**：
- Person是用户级隔离的，不同用户即使人脸相似也不会互相匹配

### 4.2 年龄一致性过滤

```typescript
minBirthDate: new Date(face.asset.fileCreatedAt)

WHERE (person.birthDate IS NULL OR person.birthDate <= minBirthDate)
```

**原理**：照片拍摄时间必定晚于人物出生日期 → 排除时空矛盾的匹配

### 4.3 手动人脸优先

```typescript
sourceType = SourceType.Manual | MachineLearning
```

**Manual人脸特性**：
- 用户手动框选/分配的人脸
- **不参与自动聚类**（非ML来源人脸直接跳过）
- 作为"黄金标准"锚定Person的向量中心
- 提高该Person后续自动匹配的准确率

---

## 五、失败分支与异常处理

### 5.1 任务状态返回值

| 状态 | 触发场景 |
|------|---------|
| `JobStatus.Success` | 正常完成，人脸分配/创建Person成功 |
| `JobStatus.Skipped` | 功能关闭、已分配、非核心脸延迟、匹配不足 |
| `JobStatus.Failed` | 资源不存在、ML服务失败、数据缺失 |

### 5.2 详细失败分支

**handleRecognizeFaces() 失败路径**：
```
1. 人脸识别未启用(config) → Skipped
2. face不存在 / asset关联丢失 → Failed
3. sourceType != MachineLearning → Skipped (手动人脸不参与聚类)
4. 无embedding向量 → Failed
5. 已有personId → Skipped (避免重复处理)
6. matches <= 1 (minFaces > 1) → Skipped (仅匹配到自己)
7. 非核心脸 + 非延迟处理 → 重新入队(deferred=true) → Skipped
```

---

## 六、权限控制矩阵

### 6.1 Person 相关权限

| 权限 | 对应操作 | 所有权检查 |
|------|---------|-----------|
| `PersonCreate` | 创建Person、创建人脸 | `checkFaceOwnerAccess` |
| `PersonRead` | 获取Person列表/详情/统计/缩略图 | `checkOwnerAccess` |
| `PersonUpdate` | 修改Person信息、批量更新 | `checkOwnerAccess` |
| `PersonDelete` | 删除Person | `checkOwnerAccess` |
| `PersonMerge` | 合并Person | `checkOwnerAccess` |
| `PersonReassign` | 重新分配人脸归属 | `checkFaceOwnerAccess` |

### 6.2 Face 相关权限

| 权限 | 对应操作 | 说明 |
|------|---------|------|
| `FaceCreate` | 手动创建人脸 | 绕过ML检测，用户框选 |
| `FaceRead` | 获取资产人脸列表 | 查看人脸bounding box |
| `FaceUpdate` | 重新分配人脸 | 等价PersonReassign |
| `FaceDelete` | 删除人脸 | soft delete / force delete |

---

## 七、核心数据表关系

```
┌──────────────┐          ┌──────────────┐          ┌──────────────┐
│    person    │          │  asset_face  │          │  face_search │
├──────────────┤          ├──────────────┤          ├──────────────┤
│ id (PK)      │←──────┐  │ id (PK)      │┌────────→│ faceId (PK)  │
│ ownerId      │       │  │ assetId      ││         │ embedding    │
│ name         │       └──│ personId     ││         └──────────────┘
│ birthDate    │          │ boundingBox* ││
│ faceAssetId  │──┐      │ imageWidth   ││
│ thumbnailPath│  │      │ imageHeight  ││
│ isHidden     │  │      │ sourceType   ││
│ isFavorite   │  │      │ deletedAt    ││
│ color        │  │      │ isVisible    ││
└──────────────┘  │      └──────────────┘│
                  └──────────────────────┘
```

---

## 八、关键配置参数

```typescript
// machineLearning.facialRecognition 配置
{
  minFaces: number,      // 创建Person所需最小匹配数
  maxDistance: number,   // 向量相似度阈值(余弦距离)
  minScore: number,      // 人脸检测置信度阈值
}
```

---

## 九、Deferred 重试与 PersonCleanup 触发时机

### 9.1 完整时间线与先后关系

```
时间轴:

T0  资产上传 → AssetDetectFacesQueueAll 入队
    ↓
T1  handleQueueDetectFaces() 执行
    ├─ 遍历所有未检测人脸的资产
    ├─ 批量入队 AssetDetectFaces 任务
    └─ 所有资产检测完成后 → PersonCleanup 入队
    ↓
T2  各 AssetDetectFaces 并行执行 → 生成人脸+embedding
    ↓
T3  FacialRecognitionQueueAll 入队
    ↓
T4  handleQueueRecognizeFaces() 执行
    ├─ 等待 FaceDetection 队列完成
    ├─ 遍历所有 personId=null 的ML人脸
    └─ 批量入队 FacialRecognition(id, deferred: false)
    ↓
T5  handleRecognizeFaces(deferred: false) 执行
    ├─ 是核心脸？
    │  ├─ 是 → 分配/创建Person → Success
    │  └─ 否 → 重新入队 deferred=true → Skipped
    ↓
T6  handleRecognizeFaces(deferred: true) 执行
    └─ 无论是否核心脸 → 强制尝试匹配
```

---

### 9.2 Deferred 重试机制详解

| 阶段 | deferred 值 | 触发条件 | 队列入口 | 返回状态 | 对Person归属的影响 |
|------|---------|---------|---------|---------|
| **首次处理** | `false` | 批量入队所有未分配人脸 | `FacialRecognition` 队列 | **Skipped** 非核心脸时重新入队 | 不分配，等待更多样本积累 |
| **延迟重试** | `true` | 非核心脸首次处理后重新入队 | `FacialRecognition` 队列 | **Success/Skipped** | 强制匹配，可能分配也可能找不到匹配 |

**关键代码说明**：
- deferred=false 时，非核心脸 → "等等看"，需要更多相似人脸积累后再匹配
- deferred=true 时，强制匹配 → 不等待，即使只有1张人脸也尝试匹配现有Person
- deferred 仅重试 **1次** → 不是无限循环，失败后不会再自动入队

---

### 9.3 PersonCleanup 触发时机

| 触发场景 | 触发位置 | 执行条件 |
|---------|---------|---------|
| **全量检测后** | `handleQueueDetectFaces` 末尾 | 非强制模式下自动触发 |
| **强制检测前** | `handleQueueDetectFaces` 开头 | force = true 先清空旧人脸后清理 |
| **强制识别前** | `handleQueueRecognizeFaces` 开头 | force = true 先清空所有旧分配后清理 |
| **手动API触发** | API `/api/job` | 手动调用 job API |

---

### 9.4 先后关系与依赖

```
执行顺序优先级：

1. FaceDetection 队列 (人脸检测 → 生成新人脸)
2. PersonCleanup 入队 (但不等待 FaceDetection 全部完成)
   ↓
3. FacialRecognition 队列 (人脸聚类 → 分配Person)
   ├─ deferred=false 首次处理
   │  └─ 非核心脸 → deferred=true 入队
   └─ deferred=true 延迟处理
```

**竞态条件说明**：
- PersonCleanup 只删除当前已经 face_count=0 的Person
- 正在 FacialRecognition 中正在分配的人脸不会被删除（有数据库事务一致性保证）

---

## 十、误区对照速查表

| 动作类型 | 具体操作 | 触发条件 | 进入的队列/任务入口 | 可能返回状态 | 对Person归属的实际影响 | 影响聚类结果？ |
|---------|---------|---------|---------|---------|
| **仅重命名name** | 修改Person.name字段 | 用户调用 `PUT /people/:id` | 无队列（同步执行） | Success | ❌ name仅UI显示，完全不参与任何向量匹配或聚类逻辑 | ❌ 不改变 |
| **修改birthDate** | 修改Person.birthDate字段 | 用户调用 `PUT /people/:id` | 无队列（同步执行） | Success | ✅ birthDate作为 `minBirthDate` 过滤条件直接传入 `searchFaces` SQL，扩大/缩小可匹配的时间范围 | ✅ 直接改变 |
| **修改featureFaceAssetId** | 更新Person.faceAssetId | 用户调用 `PUT /people/:id` | `PersonGenerateThumbnail` 缩略图生成队列 | Success | ⚠️ 仅改变Person列表按相似度排序的基准向量，影响closestPersonId搜索的展示顺序，自动聚类完全不使用 | ⚠️ 仅影响排序展示 |
| **非核心脸deferred=false** | 首次聚类处理 | `matches < minFaces` 或 `asset.visibility != Timeline` | `FacialRecognition` 队列 | Skipped | 🔄 不分配Person，重新入队 deferred=true 延迟重试，等待更多相似人脸积累 | 🔄 延迟待定 |
| **非核心脸deferred=true** | 延迟后强制重试 | 自身触发重新入队 | `FacialRecognition` 队列 | Success / Skipped | ✅ 强制匹配现有Person，成功则分配，失败则保持未分配状态（仅重试1次） | ✅ 可能改变 |
| **PersonCleanup清理** | 删除空Person | 全量人脸检测完成后 | `PersonCleanup` 背景任务队列 | Success | 🗑️ 仅删除 `face_count=0` 的空Person（无任何人脸关联的孤立Person），不影响已有分配关系 | ❌ 仅清理，不改变 |

---

### 10.1 核心结论

**会真实改变聚类结果的动作（仅2个）**：
1. ✅ **修改 `birthDate`** → 直接改变 `searchFaces` 的过滤范围，影响后续人脸可匹配到的Person集合
2. ✅ **非核心脸 `deferred=true` 强制重试** → 第二次处理时跳过核心脸判定，强行分配Person

**仅影响展示/排序的动作（4个）**：
1. ❌ **仅重命名 `name`** → 纯UI展示，与向量搜索、聚类算法完全无关
2. ⚠️ **修改 `featureFaceAssetId`** → 仅影响 "按相似度展示Person" 的排序顺序，自动聚类不使用
3. ❌ **`deferred=false` 跳过处理** → 只是延迟，不改变最终归属，只是暂缓决定
4. ❌ **PersonCleanup 清理** → 仅删除无用的空Person，是垃圾回收，不影响正常Person的归属关系

---

> **一句话总结**：别再纠结改名字、换封面是否影响人脸识别了 —— 只有改生日才真的能让你在老照片里被认出！

---

## 十一、扩展常见误区对照表

| 动作/场景 | 误区认知 | 实际行为 | 触发条件 | 队列入口 | 返回状态 | 对Person归属实际影响 |
|---------|---------|---------|---------|---------|
| **删除Person** | Person删除后人脸也一起删除 | Person删除后关联人脸personId置空，**人脸保留**，下次聚类重新匹配 | `DELETE /people/:id` | 无队列(同步执行) | - | 🔄 人脸在下轮聚类重新分配 |
| **手动分配人脸** | 手动分配的人脸会被聚类算法重新分配 | `sourceType=Manual` 的人脸**永不参与自动聚类** | `PUT /faces/:id` | 无队列(同步执行) | Success | ✅ 永久锁定归属，不会被ML算法重新分配 |
| **合并Person** | 合并的Person人脸会被重新聚类 | 合并后所有脸personId直接更新，**不会重新计算** | `POST /people/:id/merge` | 无队列(同步执行) | Success | ✅ 批量人脸永久归属 |

---

## 十二、代码路径索引

| 模块 | 文件路径 | 核心函数 |
|------|---------|---------|
| 人脸/ Person服务 | `server/src/services/person.service.ts` | `handleDetectFaces`, `handleRecognizeFaces`, `mergePerson`, `reassignFaces` |
| Person仓库 | `server/src/repositories/person.repository.ts` | `reassignFaces`, `getAllWithoutFaces`, `refreshFaces` |
| 搜索仓库 | `server/src/repositories/search.repository.ts` | `searchFaces`, HNSW查询 |
| ML仓库 | `server/src/repositories/machine-learning.repository.ts` | `detectFaces`, 向量生成 |
| Person控制器 | `server/src/controllers/person.controller.ts` | CRUD + merge + reassign |
| 人脸控制器 | `server/src/controllers/face.controller.ts` | 人脸手动操作 |
| 权限定义 | `server/src/utils/access.ts` | 权限检查矩阵 |
| Person表 | `server/src/schema/tables/person.table.ts` | 表结构定义 |
| 人脸表 | `server/src/schema/tables/asset-face.table.ts` | 人脸元数据 |
| 向量搜索表 | `server/src/schema/tables/face-search.table.ts` | pgvector索引定义 |
| DTO定义 | `server/src/dtos/person.dto.ts` | 请求/响应结构 |
