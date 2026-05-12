# Immich 人脸向量与 Person 管理协同机制

## 一、整体架构

Immich 的人脸管理系统由三部分协同工作：
1. **机器学习服务**：人脸检测 + 512维人脸嵌入向量生成
2. **PostgreSQL pgvector**：向量存储与近似最近邻（ANN）搜索
3. **Person 管理服务**：Person 实体管理、合并、拆分、重命名

---

## 二、向量聚类流程

### 2.1 数据表设计

| 表名 | 核心字段 | 说明 |
|------|---------|------|
| `asset_face` | `id`, `assetId`, `personId`, `boundingBox*`, `sourceType` | 人脸元数据，关联资产与 Person |
| `face_search` | `faceId`, `embedding` | 人脸向量表，512维向量 |
| `person` | `id`, `name`, `birthDate`, `faceAssetId`, `thumbnailPath` | Person 实体表 |

**关键索引设计**（`face-search.table.ts`）：

```typescript
@Index({
  name: 'face_index',
  using: 'hnsw',
  expression: `embedding vector_cosine_ops`,
  with: 'ef_construction = 300, m = 16',
})
```

- 使用 HNSW（Hierarchical Navigable Small World）索引
- 采用余弦相似度（`vector_cosine_ops`）
- `ef_construction = 300`：构建时动态列表大小
- `m = 16`：每个节点的邻居数量

### 2.2 人脸检测与向量生成流程

**入口**：`PersonService.handleDetectFaces()` (`person.service.ts:302`)

```
1. 获取资源文件（previewPath）
2. 调用机器学习服务 detectFaces()
   ├─ 人脸检测（YOLO/RetinaFace等模型）
   ├─ 返回：imageHeight, imageWidth, faces[boundingBox, embedding, score]
3. 与现有faces进行IOU匹配（交并比）
   ├─ IOU > 0.5：认为是同一张脸，更新embedding
   └─ IOU ≤ 0.5：新检测到的脸，插入新记录
4. personRepository.refreshFaces(facesToAdd, faceIdsToRemove, embeddings)
5. 对新增人脸，提交 FacialRecognition 任务
```

**IOU 匹配算法（`person.service.ts:386-401`）：

```typescript
private iou(face, newBox): number {
  // 计算交并比，过滤重复检测
  intersection = max(0, x2 - x1) * max(0, y2 - y1)
  union = area1 + area2 - intersection
  return intersection / union
}
```

### 2.3 人脸识别与聚类流程

**入口**：`PersonService.handleRecognizeFaces()` (`person.service.ts:461`)

```
1. 获取 faceId（未分配 personId）
2. 向量相似度搜索（searchFaces）：
   ├─ userIds 限定用户范围
   ├─ embedding 目标向量
   ├─ maxDistance 最大距离阈值（配置项：machine-learning.facialRecognition.maxDistance）
   └─ numResults 返回结果数量
3. 判断是否为"核心脸"：
   ├─ matches ≥ minFaces（配置
   └─ asset.visibility === Timeline
4. 核心脸且无匹配Person → 创建新Person
5. 非核心脸 → defer 延迟处理
6. 分配 personId → reassignFaces()
```

**向量搜索实现**（`search.repository.ts:315`）：

```typescript
searchFaces({ userIds, embedding, numResults, maxDistance, hasPerson, minBirthDate }) {
  // 设置 probes 控制搜索精度，HNSW搜索时检查的邻居数量
  set local vchordrq.probes = ${probes[VectorIndex.Face]}
  
  // 余弦相似度搜索
  face_search.embedding <=> ${embedding} AS distance
  
  // 过滤条件
  where distance <= maxDistance
  order by distance asc
  limit numResults
}
```

**聚类策略：

- **核心脸优先处理，非核心脸延迟处理
- 当匹配到已有Person时，直接分配
- 未匹配到且是核心脸时，创建新Person
- Person 与 birthDate 年龄一致性检查

---

## 三、Person 合并查询

### 3.1 Person 查询排序

**Person 列表查询**（`person.repository.ts:152`）：

```
排序优先级：
1. isHidden = false 优先
2. isFavorite = true 优先
3. 有 name 不为空优先
4. 人脸数量多优先
5. name 字典序
6. 创建时间

支持按"最近人脸"排序（closestFaceAssetId）
```

### 3.2 Person 合并机制

**入口**：`PersonService.mergePerson()` (`person.service.ts:557`)

合并流程：

```
1. 权限检查（PersonMerge权限）
2. 遍历待合并的 personIds：
   ├─ 获取待合并Person信息
   ├─ 属性合并策略：
   │  ├─ 主Person无name → 继承待合并Person的name
   │  └─ 主Person无birthDate → 继承待合并Person的birthDate
   │  └─ 其他属性保留主Person
   ├─ personRepository.reassignFaces({ oldPersonId, newPersonId })
   │  └─ SQL: UPDATE asset_face SET personId = newPersonId WHERE personId = oldPersonId
   ├─ 删除待合并Person（清理thumbnail文件）
   └─ 记录审计日志
```

### 3.3 人脸重新分配

**入口**：`PersonService.reassignFaces()` (`person.service.ts:82`)

```
1. 检查人脸所属 Person 特征脸更新：
   ├─ 源Person失去了最后一张脸 → 创建新的随机特征脸
   └─ 目标Person之前无特征脸 → 分配新特征脸
```

---

## 四、跨相册识别

### 4.1 跨相册识别原理

```
所有用户所有相册的人脸共享同一向量空间
personId作为关联标识
搜索时通过userIds过滤用户范围
```

**向量搜索跨相册能力：

```typescript
searchFaces({
  userIds: [face.asset.ownerId],  // 限定用户范围
  embedding: face.faceSearch.embedding,
  maxDistance: machineLearning.facialRecognition.maxDistance,
  numResults: machineLearning.facialRecognition.minFaces,
  minBirthDate: new Date(face.asset.fileCreatedAt),  // 年龄一致性
})
```

### 4.2 年龄一致性检查

搜索时加入 `minBirthDate` 约束：

```typescript
where (person.birthDate is null OR person.birthDate <= minBirthDate
```

确保人脸年龄必须早于照片拍摄时间，防止将老人年轻照片矛盾

---

## 五、关键配置项

```typescript
// facialRecognition: {
  minFaces: number,           // 最小人脸数（创建Person阈值
  maxDistance: number,      // 向量相似度阈值
  minScore: number,         // 人脸检测置信度
}
```

---

## 六、数据流图

```
┌─────────────────────────────────────────────────────────────────┐
│                    机器学习服务                               │
├─────────────────────────────────────────────────────────────────┤
│  detectFaces(imagePath)                                        │
│  ├─ 人脸检测(boundingBox)                                │
│  └─ 人脸嵌入(512d vector)                               │
└──────────────────────────┬────────────────────────────────────┘
                           │
                           ▼
┌─────────────────────────────────────────────────────────────────┐
│                    PostgreSQL                               │
├─────────────────────────────────────────────────────────────────┤
│  asset_face 表                                              │
│  ├─ id, assetId, personId, boundingBox*                │
│  └─ sourceType: ML / MANUAL                              │
│                                                               │
│  face_search 表 (pgvector HNSW索引)                    │
│  ├─ faceId                                                  │
│  └─ embedding: vector(512)                                     │
│                                                               │
│  person 表                                                  │
│  ├─ id, name, birthDate, faceAssetId, thumbnailPath     │
│  └─ isHidden, isFavorite                                  │
└──────────────────────────┬────────────────────────────────────┘
                           │
                           ▼
┌─────────────────────────────────────────────────────────────────┐
│                    PersonService                               │
├─────────────────────────────────────────────────────────────────┤
│  handleDetectFaces() ────► 人脸检测与入库             │
│  handleRecognizeFaces() ───► 向量聚类分配 Person       │
│  mergePerson() ─────────────► Person 合并                  │
│  reassignFaces() ───────► 人脸重新分配              │
└─────────────────────────────────────────────────────────────────┘
```

---

## 七、核心代码路径

| 功能 | 文件位置 |
|------|---------|
| Person 服务 | `server/src/services/person.service.ts` |
| Person Repository | `server/src/repositories/person.repository.ts` |
| 搜索 Repository | `server/src/repositories/search.repository.ts` |
| ML Repository | `server/src/repositories/machine-learning.repository.ts` |
| 人脸表定义 | `server/src/schema/tables/asset-face.table.ts` |
| Person 表定义 | `server/src/schema/tables/person.table.ts` |
| 向量搜索表 | `server/src/schema/tables/face-search.table.ts` |
| Python ML 服务 | `machine-learning/immich_ml/models/facial_recognition/` |
