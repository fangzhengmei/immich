# Immich ML 服务 RPC 接口与模型管理 v3

## 目录

1. [架构概述](#架构概述)
2. [RPC 接口规范](#rpc-接口规范)
3. [模型加载与缓存机制](#模型加载与缓存机制)
4. [模型版本管理与校验](#模型版本管理与校验)
5. [向量维度与重建任务联动](#向量维度与重建任务联动)
6. [人脸识别与搜索链路详解](#人脸识别与搜索链路详解)
7. [配置与环境变量](#配置与环境变量)
8. [多 ML 服务器与故障转移](#多-ml-服务器与故障转移)

---

## 架构概述

Immich 采用独立的 Python 微服务（FastAPI）处理机器学习任务，包括人脸识别、CLIP 图像/文本编码和 OCR。主服务（TypeScript/NestJS）通过 HTTP REST API 调用 ML 服务，并提供：

- **配置校验**：仅 CLIP 模型名称合法性校验
- **维度同步**：CLIP 模型维度变更时同步数据库列
- **故障转移**：支持多 ML 服务器健康检查与重试

---

## RPC 接口规范

### 端点定义

| 端点 | 方法 | 说明 |
|------|------|------|
| `/` | GET | 服务信息 |
| `/ping` | GET | 健康检查（用于故障转移） |
| `/predict` | POST | 执行推理任务 |

### `/predict` 请求格式 (FormData)

```
entries: <JSON Pipeline配置>
image: <二进制文件>   # 图像任务
# 或
text: <字符串>        # 文本任务
```

### Pipeline 配置 (entries)

```typescript
type PipelineRequest = {
  [ModelTask.CLIP]?: {
    [ModelType.VISUAL]?: { modelName: string };
    [ModelType.TEXTUAL]?: { modelName: string; options?: { language?: string } };
  };
  [ModelTask.FACIAL_RECOGNITION]?: {
    [ModelType.DETECTION]?: { modelName: string; options: { minScore: number } };
    [ModelType.RECOGNITION]?: { modelName: string };
  };
  [ModelTask.OCR]?: {
    [ModelType.DETECTION]?: { modelName: string; options: { minScore: number; maxResolution: number } };
    [ModelType.RECOGNITION]?: { modelName: string; options: { minScore: number } };
  };
};
```

### 请求示例

#### 人脸识别

```json
{
  "facial-recognition": {
    "detection": { "modelName": "buffalo_l", "options": { "minScore": 0.7 } },
    "recognition": { "modelName": "buffalo_l" }
  }
}
```

#### CLIP 图像编码

```json
{
  "clip": { "visual": { "modelName": "ViT-B-32__openai" } }
}
```

#### OCR 文本检测

```json
{
  "ocr": {
    "detection": { "modelName": "PP-OCRv5_mobile", "options": { "minScore": 0.5, "maxResolution": 736 } },
    "recognition": { "modelName": "PP-OCRv5_mobile", "options": { "minScore": 0.8 } }
  }
}
```

### 响应格式

```typescript
interface InferenceResponse {
  imageHeight?: number;
  imageWidth?: number;
  "facial-recognition"?: Face[];
  clip?: string;               // base64 编码的向量 [dimSize]
  ocr?: OCRResult;
}

interface Face {
  boundingBox: { x1: number; y1: number; x2: number; y2: number };
  embedding: string;           // base64 编码的 512 维向量
  score: number;
}

interface OCRResult {
  text: string[];
  box: number[][];
  boxScore: number[];
  textScore: number[];
}
```

---

## 模型加载与缓存机制

### 1. ModelCache 内存缓存

```python
class ModelCache:
    """带 TTL 的模型内存缓存，使用乐观锁防止并发加载"""
    def __init__(self, revalidate: bool = False, timeout: int | None = None):
        self.cache = SimpleMemoryCache(timeout=timeout)

    async def get(model_name: str, model_type: ModelType, model_task: ModelTask):
        """获取模型实例，不存在则实例化"""
```

**缓存策略**：
- **TTL 超时**：默认 300 秒，空闲后自动卸载
- **重新验证**：访问时重置 TTL (`revalidate=True`)
- **乐观锁**：防止并发加载同一模型
- **依赖处理**：自动按拓扑顺序加载依赖模型

### 2. 模型加载流程

```
ML 服务收到 /predict 请求
         │
         ▼
┌─────────────────────┐
│  解析 entries 配置   │
│  (按任务类型分组)    │
└─────────────────────┘
         │
         ▼
┌─────────────────────┐
│  分离有/无依赖任务   │
└─────────────────────┘
         │
         ▼
┌─────────────────────┐   命中  ┌────────────────┐
│  ModelCache.get()   │──────▶│ 返回模型实例    │
└─────────────────────┘        └────────────────┘
         │ 未命中
         ▼
┌─────────────────────┐
│  getModelClass()    │ ──▶ 根据模型名+类型+任务映射到类
└─────────────────────┘
         │
         ▼
┌─────────────────────┐
│  检查本地缓存目录    │
│ ~/.cache/immich_ml/ │
└─────────────────────┘
         │  不存在
         ▼
┌──────────────────────────────────────┐
│  从 HuggingFace Hub 下载模型          │
│  repo: immich-app/{model_name_clean}  │
└──────────────────────────────────────┘
         │
         ▼
┌──────────────────────────────────────┐
│  按硬件选择后端格式                   │
│  ONNX (默认) / ARMNN / NPU / CUDA    │
└──────────────────────────────────────┘
         │
         ▼
┌──────────────────────────────────────┐
│  创建推理 Session 并加入缓存          │
└──────────────────────────────────────┘
```

### 3. 模型格式自动选择

| 硬件平台 | 优先格式 | 说明 |
|---------|---------|------|
| Rockchip NPU | RKNN | `rknn.is_available` 检测 |
| ARM NPU | ARMNN | `ann` 环境变量启用 |
| NVIDIA GPU | ONNX (CUDA) | CUDAExecutionProvider |
| Intel GPU | ONNX (OpenVINO) | OpenVINOExecutionProvider |
| CPU | ONNX (CPU) | 默认 |

### 4. 预加载机制 (Preload)

启动时可通过环境变量预加载常用模型，避免首次请求延迟：

```bash
MACHINE_LEARNING_PRELOAD__CLIP__VISUAL=ViT-B-32__openai
MACHINE_LEARNING_PRELOAD__CLIP__TEXTUAL=ViT-B-32__openai
MACHINE_LEARNING_PRELOAD__FACIAL_RECOGNITION__DETECTION=buffalo_l
MACHINE_LEARNING_PRELOAD__FACIAL_RECOGNITION__RECOGNITION=buffalo_l
MACHINE_LEARNING_PRELOAD__OCR__DETECTION=PP-OCRv5_mobile
MACHINE_LEARNING_PRELOAD__OCR__RECOGNITION=PP-OCRv5_mobile
```

---

## 模型版本管理与校验

### 1. 支持模型清单 (v3)

#### CLIP 模型 (OpenCLIP / M-CLIP)

| 模型名称 | 维度 | 说明 |
|---------|------|------|
| `ViT-B-32__openai` | 512 | **默认**，小而快 |
| `ViT-B-16__openai` | 512 | 中等 |
| `ViT-L-14-quickgelu__dfn2b` | 768 | 高质量 |
| `ViT-L-14__openai` | 768 | 高质量 |
| `RN50__openai` | 1024 | ResNet 系列 |
| `XLM-Roberta-Large-Vit-L-14` | 768 | 多语言 |
| `ViT-H-14-quickgelu__dfn5b` | 1024 | 超高质量 |
| `ViT-SO400M-14-SigLIP2-378__webli` | 1152 | 最新架构 |

完整列表参见 `CLIP_MODEL_INFO` 常量。

#### 人脸识别 (InsightFace)

| 模型名称 | 维度 | 说明 |
|---------|------|------|
| `buffalo_l` | 512 | **默认**，大模型，高精度 |
| `buffalo_m` | 512 | 中等 |
| `buffalo_s` | 512 | 小模型，速度快 |
| `antelopev2` | 512 | 最新，亚洲人脸优化 |

所有 InsightFace 模型输出 **512 维**人脸特征向量。

#### OCR (PaddleOCR v5)

| 模型名称 | 说明 |
|---------|------|
| `PP-OCRv5_mobile` | **默认**，移动端优化 |
| `PP-OCRv5_server` | 服务器版，高精度 |
| `CH__PP-OCRv5_mobile` | 中文专用 |
| `KOREAN__PP-OCRv5_mobile` | 韩文专用 |
| `EL__PP-OCRv5_mobile` | 希腊文专用 |

### 2. 配置校验机制 (ConfigValidate) - 代码真实实现

**触发时机**：用户在管理界面更新 ML 配置时，`updateSystemConfig` 先 emit `ConfigValidate` 事件。

**实际校验内容（代码证据）**：

```typescript
// server/src/services/smart-info.service.ts:23-32
// ONLY: 校验 CLIP 模型名称是否在支持列表中
@OnEvent({ name: 'ConfigValidate' })
onConfigValidate({ newConfig }: ArgOf<'ConfigValidate'>) {
  try {
    getCLIPModelInfo(newConfig.machineLearning.clip.modelName);
  } catch {
    throw new Error(
      `Unknown CLIP model: ${newConfig.machineLearning.clip.modelName}. ` +
      `Please check the model name for typos.`,
    );
  }
}
```

```typescript
// server/src/services/system-config.service.ts:51-57
// ONLY: 校验日志配置是否被环境变量锁定
@OnEvent({ name: 'ConfigValidate' })
onConfigValidate({ newConfig, oldConfig }: ArgOf<'ConfigValidate'>) {
  const { logLevel } = this.configRepository.getEnv();
  if (!_.isEqual(toPlainObject(newConfig.logging), oldConfig.logging) && logLevel) {
    throw new Error('Logging cannot be changed while the environment variable IMMICH_LOG_LEVEL is set.');
  }
}
```

> **⚠️ 重要事实**：
> - ❌ **人脸 modelName 无校验**：输入任意字符串都不会被拦截
> - ❌ **OCR modelName 无校验**：输入任意字符串都不会被拦截
> - ❌ **ML URL 格式无校验**：不检查 URL 是否可访问、格式是否正确
> - ✅ **仅 CLIP modelName 校验**：匹配 `CLIP_MODEL_INFO` 白名单

### 3. 模型版本切换流程 - 代码真实实现

**触发条件**：仅 CLIP 模型变化会被处理，人脸/OCR 模型变化**完全无处理**。

```typescript
// server/src/services/smart-info.service.ts:34-65
private async init(newConfig: SystemConfig, oldConfig?: SystemConfig) {
  if (!isSmartSearchEnabled(newConfig.machineLearning)) {
    return;
  }

  await this.databaseRepository.withLock(DatabaseLock.CLIPDimSize, async () => {
    const { dimSize } = getCLIPModelInfo(newConfig.machineLearning.clip.modelName);
    const dbDimSize = await this.databaseRepository.getDimensionSize('smart_search');

    // 仅检查 CLIP 模型变化
    const modelChange =
      oldConfig && oldConfig.machineLearning.clip.modelName !== newConfig.machineLearning.clip.modelName;
    const dimSizeChange = dbDimSize !== dimSize;
    
    if (!modelChange && !dimSizeChange) {
      return;
    }

    if (dimSizeChange) {
      // 维度不同：更新数据库列类型 ALTER COLUMN vector(n)
      await this.databaseRepository.setDimensionSize(dimSize);
    } else {
      // 同维度但模型不同：仅清空现有嵌入 ❌ 不会自动重建
      await this.databaseRepository.deleteAllSearchEmbeddings();
    }

    // TODO: A job to reindex all assets should be scheduled, though user
    // confirmation should probably be requested before doing that.
    // ⚠️ 截至当前版本，这个 TODO 仍未实现 - 不会自动触发任何重建任务
  });
}
```

**各类型模型切换行为总结**：

| 模型类型 | 校验 | 维度更新 | 清空现有向量 | 自动调度重建任务 | 需人工触发 |
|---------|------|---------|-------------|----------------|-----------|
| **CLIP (视觉)** | ✅ 白名单校验 | ✅ (dimSize 变化时) | ✅ (同维度时) | ❌ 完全无 | ✅ 必须手动点 "重新编码所有 CLIP" |
| **人脸识别** | ❌ 无任何校验 | ❌ (固定 512 维) | ❌ 什么都不做 | ❌ 完全无 | ✅ 必须手动点 "重新检测所有人脸" |
| **OCR** | ❌ 无任何校验 | N/A | ❌ 什么都不做 | ❌ 完全无 | ✅ 手动重新触发 OCR 任务 |

> **代码证据**：搜索 `facialRecognition.modelName`、`ocr.modelName`，在 ConfigUpdate/ConfigInit 事件链中完全无处理逻辑。

---

## 向量维度与重建任务联动

### 1. 向量维度管理

#### CLIP 搜索向量表 (`smart_search`)

```sql
-- 列定义
assetId: UUID
embedding: vector(<dimSize>)   -- 维度由模型决定，动态可变
```

维度存储在 `system_metadata` 中：
```typescript
await databaseRepository.getDimensionSize('smart_search');
await databaseRepository.setDimensionSize(dimSize);
```

#### 人脸向量表 (`face_search`)

```sql
-- 列定义
faceId: UUID
embedding: vector(512)   -- 固定 512 维 (InsightFace)
personId: UUID | null
```

> **固定维度**：所有 InsightFace 模型输出维度一致，数据库列永远是 512 维。

### 2. 维度同步流程（仅限 CLIP）

```
用户在管理界面修改 CLIP 模型名称
         │
         ▼
┌─────────────────────────────────────┐
│  ConfigValidate 事件                │
│  ✅ 校验 CLIP modelName 在白名单    │
│  ❌ 跳过人脸/OCR 模型校验           │
└─────────────────────────────────────┘
         │  校验通过
         ▼
┌─────────────────────────────────────┐
│  写入配置到数据库                   │
└─────────────────────────────────────┘
         │
         ▼
┌─────────────────────────────────────┐
│  ConfigUpdate 事件                  │
│  ────────────────────────────      │
│  1. 获取新模型 dimSize              │
│  2. 对比 DB 中存储的 dimSize        │
│  3. dimSize 变化 → ALTER COLUMN     │
│  4. 同维度模型变了 → DELETE 所有向量 │
│  5. ❌ 不会调度任何重建任务         │
│     (代码里是 TODO 注释)            │
└─────────────────────────────────────┘
```

### 3. 锁机制与任务同步

**锁名称**：`DatabaseLock.CLIPDimSize`

```typescript
// server/src/services/smart-info.service.ts:113-122
// 正在进行的编码任务会检查锁状态
if (this.databaseRepository.isBusy(DatabaseLock.CLIPDimSize)) {
  this.logger.verbose(`Waiting for CLIP dimension size to be updated`);
  await this.databaseRepository.wait(DatabaseLock.CLIPDimSize);
}

// 编码完成后二次校验，防止编码期间模型被更改
const newConfig = await this.getConfig({ withCache: true });
if (machineLearning.clip.modelName !== newConfig.machineLearning.clip.modelName) {
  // Skip the job if the model has changed since the embedding was generated.
  return JobStatus.Skipped;  // ⚠️ 直接丢弃，不写入 DB
}
```

**并发安全**：
- 维度更新期间，所有编码任务阻塞等待锁
- 维度更新完成后，任务继续执行，但写入前二次校验模型名
- 校验不通过直接跳过，避免混合新旧模型向量

### 4. 手动重建触发（唯一方式）

#### CLIP 重建：`SmartSearchQueueAll (force=true)`

```typescript
// server/src/services/smart-info.service.ts:67-93
@OnJob({ name: JobName.SmartSearchQueueAll, queue: QueueName.SmartSearch })
async handleQueueEncodeClip({ force }) {
  if (force) {
    const { dimSize } = getCLIPModelInfo(config.clip.modelName);
    // 确保维度正确
    await this.databaseRepository.setDimensionSize(dimSize);
  }
  
  // 流式分页：所有无嵌入资产 / force=true 时所有资产
  for await (const asset of streamForEncodeClip(force)) {
    await jobRepository.queue({ name: JobName.SmartSearch, data: { id: asset.id } });
  }
}
```

#### 人脸识别重建：`AssetDetectFacesQueueAll (force=true)`

```typescript
// PersonService.handleQueueDetectFaces
async handleQueueDetectFaces({ force }) {
  if (force) {
    // 删除所有 ML 生成的人脸记录 + 向量
    await personRepository.deleteFaces({ sourceType: SourceType.MachineLearning });
    await personRepository.vacuum({ reindexVectors: true });  // 重建向量索引
  }
  // 重新检测所有资产人脸
  for await (const asset of streamForDetectFaces(force)) {
    await jobRepository.queue({ name: JobName.AssetDetectFaces, data: { id: asset.id } });
  }
}
```

---

## 人脸识别与搜索链路详解

### 1. 完整处理链路

```
用户上传图片
      │
      ▼
┌─────────────────┐
│  生成缩略图      │
│  (preview.webp) │
└─────────────────┘
      │
      ▼
┌─────────────────────────────┐
│  Job: AssetDetectFaces      │ ◀── 人脸识别队列
│  ───────────────────────    │
│  1. 调用 ML /predict        │
│     (detection + recognition)│
│  2. 返回 N 个人脸 + 512D 向量│
│  3. IOU 匹配去重             │
│  4. 写入 asset_face 表       │
│  5. 写入 face_search 向量表  │
│  6. 触发 FacialRecognition   │
└─────────────────────────────┘
      │
      ▼
┌─────────────────────────────┐
│  Job: FacialRecognition     │ ◀── 聚类任务
│  ───────────────────────    │
│  1. K-means/DBSCAN 聚类      │
│  2. 生成/更新 person 实体    │
│  3. 关联 face <-> person     │
└─────────────────────────────┘
      │
      ▼
┌─────────────────────────────┐
│  Job: SmartSearch           │ ◀── CLIP 队列
│  ───────────────────────    │
│  1. 调用 ML /predict         │
│     (clip visual)            │
│  2. 写入 smart_search 表     │
└─────────────────────────────┘
```

### 2. 人脸检测与去重 (IOU)

```typescript
// PersonService.handleDetectFaces
for (const { boundingBox, embedding } of mlFaces) {
  // 缩放边界框到原图尺寸
  const scaledBox = scale(boundingBox, originalImageSize);
  
  // IOU (交并比) 匹配已有人脸，阈值 0.5
  const match = existingFaces.find(face => iou(face, scaledBox) > 0.5);
  
  if (match) {
    // 更新现有脸的 embedding
    embeddings.push({ faceId: match.id, embedding });
  } else {
    // 创建新脸
    const newFaceId = uuid();
    facesToAdd.push(newFaceId, ...);
    embeddings.push({ faceId: newFaceId, embedding });
  }
}

// 批量刷新：原子性更新 faces + embeddings
await personRepository.refreshFaces(facesToAdd, faceIdsToRemove, embeddings);
```

**IOU 阈值**：0.5，避免同一张脸被重复检测多次。

### 3. 人脸向量搜索

```sql
-- person.repository.sql: 人脸相似度搜索
SELECT
  face.id,
  face.boundingBoxX1,
  face.boundingBoxY1,
  face.boundingBoxX2,
  face.boundingBoxY2,
  fs.embedding <-> (SELECT embedding FROM face_search WHERE faceId = $targetFaceId) as distance
FROM asset_faces face
JOIN face_search fs ON fs.faceId = face.id
WHERE face.ownerId = $userId
  AND fs.embedding <-> (SELECT embedding FROM face_search WHERE faceId = $targetFaceId) < $maxDistance
ORDER BY distance ASC
LIMIT 100;
```

**距离阈值**：由 `machineLearning.facialRecognition.maxDistance` 控制（默认 0.5）。

### 4. CLIP 搜索链路

```
用户输入搜索关键词
         │
         ▼
┌────────────────────────────────────┐
│  SearchController.searchSmart      │
└────────────────────────────────────┘
         │
         ▼
┌────────────────────────────────────┐
│  1. 调用 ML encodeText             │
│     clip/textual -> 512D 向量      │
└────────────────────────────────────┘
         │
         ▼
┌────────────────────────────────────┐
│  2. 向量相似度搜索                  │
│     smart_search.embedding <-> q   │
└────────────────────────────────────┘
         │
         ▼
┌────────────────────────────────────┐
│  3. 过滤权限 + 元数据二次过滤       │
│     (时间、地点、人脸等)            │
└────────────────────────────────────┘
         │
         ▼
┌────────────────────────────────────┐
│  4. 返回排序结果                    │
└────────────────────────────────────┘
```

---

## 配置与环境变量

### ML 服务环境变量

| 变量 | 默认值 | 说明 |
|------|--------|------|
| `MACHINE_LEARNING_CACHE_FOLDER` | `~/.cache/immich_ml` | 模型下载缓存目录 |
| `MACHINE_LEARNING_MODEL_TTL` | `300` | 模型空闲超时（秒），0=永久 |
| `MACHINE_LEARNING_MODEL_TTL_POLL_S` | `10` | 空闲检查间隔 |
| `MACHINE_LEARNING_WORKERS` | `1` | Gunicorn worker 数 |
| `MACHINE_LEARNING_WORKER_TIMEOUT` | `300` | Worker 超时 |
| `MACHINE_LEARNING_REQUEST_THREADS` | CPU 核心数 | 推理线程池 |
| `MACHINE_LEARNING_ANN` | `true` | 启用 ARMNN NPU 加速 |
| `MACHINE_LEARNING_PRELOAD__*` | 无 | 启动预加载模型 |
| `IMMICH_HOST` | `[::]` | 监听地址 |
| `IMMICH_PORT` | `3003` | 监听端口 |

### 主服务 ML 配置 (默认值)

```typescript
machineLearning: {
  enabled: true,
  urls: ['http://immich-machine-learning:3003'],  // 支持多服务器
  
  availabilityChecks: {
    enabled: true,
    timeout: 2000,      // 2 秒健康检查超时
    interval: 30000,    // 30 秒检查一次
  },
  
  clip: {
    enabled: true,
    modelName: 'ViT-B-32__openai',  // 默认小模型
  },
  
  facialRecognition: {
    enabled: true,
    modelName: 'buffalo_l',
    minScore: 0.7,      // 人脸检测置信度阈值
    maxDistance: 0.5,   // 人脸聚类距离阈值
    minFaces: 3,        // 显示人物的最少脸数
  },
  
  ocr: {
    enabled: true,
    modelName: 'PP-OCRv5_mobile',  // v5 默认
    minDetectionScore: 0.5,
    minRecognitionScore: 0.8,
    maxResolution: 736,  // OCR 处理最大边长
  },
}
```

---

## 多 ML 服务器与故障转移

### 1. 健康检查机制

```typescript
// MachineLearningRepository
private healthyMap: Record<string, boolean> = {};

private tick() {
  // 定期 ping 所有 ML 服务器 /ping 端点
  for (const url of config.urls) {
    fetch(`${url}/ping`, { timeout: 2000 })
      .then(res => setHealthy(url, res.ok))
      .catch(() => setHealthy(url, false));
  }
}
```

**检查间隔**：30 秒（可配置）

### 2. 请求路由与故障转移

```typescript
private async predict<T>(payload, config): Promise<T> {
  // 优先级：先尝试健康服务器，再尝试不健康的
  const urls = [
    ...healthyServers,
    ...unhealthyServers,
  ];

  for (const url of urls) {
    try {
      const response = await fetch(`${url}/predict`, { method: 'POST', body: formData });
      if (response.ok) {
        this.setHealthy(url, true);
        return response.json();
      }
    } catch (error) {
      this.setHealthy(url, false);
    }
  }
  throw new Error('All ML servers failed');
}
```

**失败恢复**：失败的服务器仍会在后续请求中被重试（优先级降低）。

### 3. 水平扩展建议

| 工作负载 | 建议扩展方式 |
|---------|------------|
| CLIP 编码 (批量) | 2-4 个 ML 实例，GPU 加速 |
| 人脸识别 (批量) | 2+ 个 ML 实例 |
| OCR (偶尔) | 共享实例 |
| 搜索推理 (实时) | 独立集群 |

---

## 重要修正与当前实现限制总结

### 1. 模型版本管理真实行为

| 功能 | v2 描述 | v3 真实实现 |
|------|--------|-----------|
| 人脸模型校验 | ❌ 隐含有校验 | ✅ **完全无校验**，输入啥都过 |
| OCR 模型校验 | ❌ 隐含有校验 | ✅ **完全无校验**，输入啥都过 |
| CLIP 模型校验 | ✅ 正确 | ✅ 仅白名单校验 |
| 人脸模型变更后处理 | ❌ 提过手动触发 | ✅ **什么都不做**，旧向量继续用 |
| CLIP 自动重建 | ❌ 隐含会自动 | ✅ **仅清空/TODO 注释**，不会调度 |
| 配置校验范围 | ❌ 提到 ML URL 格式 | ✅ **完全无 URL 校验** |

### 2. 最佳实践

**模型切换操作指南**：
1. **切换 CLIP 模型** → 保存后**必须**手动点击 "重新编码所有 CLIP 嵌入"，否则搜索功能为空
2. **切换人脸识别模型** → 保存后**必须**手动点击 "重新检测所有人脸"，否则旧向量与新模型不兼容
3. **切换 OCR 模型** → 没有现成的全局重建按钮，需手动触发或等待资产重新扫描

**故障处理**：
- ML 服务崩溃不影响主服务可用性，仅禁用智能功能
- 多 ML 服务器配置可提供高可用性
- 向量索引损坏时可通过重建任务恢复
- 错误的模型名称会导致 ML 服务返回 500，但主服务不会提前拦截
