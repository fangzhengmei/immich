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

**第一次搜索（第489-495行）**：
```typescript
searchFaces({
  userIds: [ownerId],
  embedding: faceEmbedding,
  maxDistance: config.maxDistance,
  numResults: config.minFaces,  // = minFaces
  minBirthDate: assetCreatedAt,
})
```
**目的**：判断是否为"核心脸"，即该人脸在库中已有足够多的相似样本

**第二次搜索（第516-528行）**：
```typescript
searchFaces({
  ...,
  numResults: 1,
  hasPerson: true,  // 只搜索已分配Person的人脸
})
```
**目的**：寻找最近的已有Person，避免创建重复Person

### 2.3 核心脸判定逻辑

| 条件 | 说明 |
|------|------|
| `matches >= minFaces` | 相似人脸数达到阈值（默认值通常为2-3） |
| `visibility = Timeline` | 资产在时间线中可见（非归档/隐藏） |
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

// 聚类自动创建Person（第530-535行）
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
   - **过滤逻辑**（`search.repository.ts:337-341`）：
     ```sql
     WHERE (person.birthDate IS NULL OR person.birthDate <= minBirthDate)
     ```
   - 示例：Person生日从 2000年 → 1990年
     - 搜索范围扩大了10年
     - 1995年拍摄的同人人脸现在可以匹配到该Person
     - 之前因出生太晚被过滤掉的人脸现在可能匹配成功

2. **`featureFaceAssetId` 更新** → 仅影响相似度排序
   - **Person列表排序**（`person.repository.ts:175-190`）：
     - `closestPersonId` 参数传入时，会用该 Person 的 faceAssetId 对应的 embedding 作为基准
     - 其他 Person 按 embedding 余弦距离排序
   - **不影响自动匹配**：
     - 人脸自动聚类的 `searchFaces` 调用不使用 faceAssetId
     - 只是改变 "这个人像谁" 的展示顺序

3. **`name` 更新** → 纯UI变化
   - 只在用户手动合并、重分配人脸时帮助识别
   - 向量搜索和聚类完全不参与

### 3.3 合并 Person

**入口**：`POST /people/:id/merge` + `PersonMerge` 权限

```
合并流程:
1. 权限检查: 目标Person需PersonUpdate权限
2. 逐个处理待合并PersonIds:
   ├─ 权限检查: 待合并Person需PersonMerge权限
   ├─ 属性继承: 主Person无name/birthDate → 继承待合并的
   ├─ reassignFaces({ oldPersonId, newPersonId })
   │   └─ SQL: UPDATE asset_face SET personId = newPersonId
   │      WHERE personId = oldPersonId
   ├─ 删除待合并Person(清理thumbnail文件)
   └─ PersonCleanup 任务确保最终清理
```

**合并后对聚类的影响**：
- 所有被合并Person的人脸embedding现在归属于同一Person
- 后续新人脸搜索时匹配概率显著提升（更多匹配样本）
- 减少Person碎片化，提高跨相册识别准确率

### 3.4 拆分 Person

**拆分 = 人脸重新分配到新/其他Person**

**入口1**：`PUT /people/:id/reassign` + `PersonReassign` 权限
```typescript
// 批量重新分配
reassignFaces(personId, { data: [{ personId, assetId }] })
```

**入口2**：`PUT /faces/:id` + `FaceUpdate` 权限
```typescript
// 单个人脸重新分配
reassignFacesById(newPersonId, { id: faceId })
```

**拆分联动逻辑（第82-125行）**：
```
人脸A从PersonX移到PersonY:
1. 若PersonY之前无特征脸 → 选新移入的脸作为特征脸
2. 若PersonX移走的是它的特征脸 → PersonX需要重新选特征脸
3. createNewFeaturePhoto([...受影响的PersonIds])
   └─ 选随机人脸作为新特征脸 + 生成新缩略图
```

### 3.5 删除 Person

**入口**：`DELETE /people/:id` + `PersonDelete` 权限

```typescript
deleteAll(ids)
  ├─ 检查所有权 access.person.checkOwnerAccess()
  ├─ 删除存储的缩略图文件
  ├─ 数据库删除person记录
  └─ 关联人脸自动变为未分配状态(ON DELETE SET NULL)
```

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
- `userIds: [face.asset.ownerId]` 确保跨用户隔离

### 4.2 年龄一致性过滤

```typescript
minBirthDate: new Date(face.asset.fileCreatedAt)

WHERE (person.birthDate IS NULL OR person.birthDate <= minBirthDate)
```

**原理**：照片拍摄时间必定晚于人物出生日期 → 排除时空矛盾的匹配

**示例**：
- 2020年拍摄的照片A → 只能匹配 birthDate ≤ 2020年的Person
- 2010年拍摄的照片B → 可以匹配 birthDate ≤ 2010年的Person
- 若Person无birthDate → 不限制

### 4.3 手动人脸优先

```typescript
sourceType = SourceType.Manual | MachineLearning
```

**Manual人脸特性**：
- 用户手动框选/分配的人脸
- 不参与自动聚类（第474-477行：非ML来源人脸直接跳过）
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

**handleRecognizeFaces() 失败路径（第462-543行）**：
```
1. 人脸识别未启用(config) → Skipped
2. face不存在 / asset关联丢失 → Failed
3. sourceType != MachineLearning → Skipped (手动人脸不参与聚类)
4. 无embedding向量 → Failed
5. 已有personId → Skipped (避免重复处理)
6. matches <= 1 (minFaces > 1) → Skipped (仅匹配到自己)
7. 非核心脸 + 非延迟处理 → 重新入队(deferred=true) → Skipped
```

**handleDetectFaces() 失败路径（第302-384行）**：
```
1. 人脸识别未启用 → Skipped
2. 资产文件丢失(previewPath) → Failed
3. 资产visibility = Hidden → Skipped
4. ML服务调用失败(网络/模型错误) → 抛出异常 → Failed
```

### 5.3 边界情况处理

**空Person清理**（第262-267行）：
```typescript
getAllWithoutFaces() → 找出所有 face_count = 0 的Person
  → 删除文件 + 数据库删除
```
**触发时机**：每次全量人脸检测完成后自动入队

**Person特征脸丢失**：
```typescript
// 当Person的faceAssetId指向的人脸被删除/移动时
createNewFeaturePhoto(personId)
  → getRandomFace(personId) 选一个随机脸作为新特征脸
  → 触发 PersonGenerateThumbnail 任务
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

### 6.3 权限检查实现

```typescript
// access.ts 第287-300行
case Permission.PersonCreate:
case Permission.PersonReassign:
  return access.person.checkFaceOwnerAccess(userId, faceIds);

case Permission.PersonRead | PersonUpdate | PersonDelete | PersonMerge:
  return access.person.checkOwnerAccess(userId, personIds);
```

**关键设计**：
- Person操作检查Person所有权
- 人脸分配操作检查人脸所在资产的所有权
- 确保用户只能操作自己的人脸和Person

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
                      1:N 关系
                   一个Person有多张脸
                   一张脸属于0或1个Person
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

**配置对聚类的影响**：
- `minFaces` 越大 → 创建Person越保守 → Person越少但更准确
- `maxDistance` 越小 → 匹配越严格 → Person越多但错误率低
- 两者需要根据数据集大小和准确率要求平衡调整

---

## 九、代码路径索引

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
