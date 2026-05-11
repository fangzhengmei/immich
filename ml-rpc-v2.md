# Immich ML 服务 RPC 接口与模型管理 v2

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

- **配置校验**：模型变更前验证合法性
- **维度同步**：自动检测并同步向量维度变更
- **故障转移**：支持多 ML 服务器健康检查与重试
- **任务调度**：模型变更时触发重建任务

### 核心组件关系

```
┌─────────────────────────────────────────────────────────────────┐
│                        Main Service (NestJS)                     │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────────────┐   │
│  │ SystemConfig │──│  HealthCheck │──│ MachineLearningRepo  │   │
│  └──────────────┘  └──────────────┘  └──────────────────────┘   │
│         │                 │                      │               │
│  ┌──────▼───────┐  ┌─────▼──────┐      ┌────────▼─────────┐     │
│  │ SmartInfoSvc │──│ PersonSvc  │──────│ DatabaseRepository│     │
│  └──────────────┘  └────────────┘      └──────────────────┘     │
└─────────────────────────────────────────────────────────────────┘
                             │
                    HTTP/REST (FormData)
                             │
┌─────────────────────────────────────────────────────────────────┐
│                    ML Service (FastAPI)                          │
│  ┌─────────────┐  ┌─────────────┐  ┌───────────────────────┐    │
│  │  ModelCache │──│  ONNX/ARMNN │──│     InsightFace       │    │
│  └─────────────┘  └─────────────┘  └───────────────────────┘    │
└─────────────────────────────────────────────────────────────────┘
```

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

### 1. 支持模型清单 (v2)

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

### 2. 配置校验机制 (ConfigValidate)

**触发时机**：用户在管理界面更新 ML 配置时

```typescript
// SmartInfoService.onConfigValidate
onConfigValidate({ newConfig }: ArgOf<'ConfigValidate'>) {
  try {
    // 校验 CLIP 模型是否在支持列表中
    getCLIPModelInfo(newConfig.machineLearning.clip.modelName);
  } catch {
    throw new Error(
      `Unknown CLIP model: ${newConfig.machineLearning.clip.modelName}. ` +
      `Please check the model name for typos.`
    );
  }
}
```

**校验流程**：
```
用户提交配置更新
         │
         ▼
┌─────────────────────┐
│  SystemConfig.validate  │
│  - ML 服务 URL 格式      │
│  - 模型名称合法性        │
└─────────────────────┘
         │
         ▼  失败
    抛出错误，终止更新
         │  成功
         ▼
┌─────────────────────┐
│  ConfigUpdate 事件   │
│  触发维度同步流程     │
└─────────────────────┘
```

### 3. 模型版本切换流程

**触发条件**：旧配置与新配置的 `modelName` 不一致

```typescript
const modelChange = oldConfig?.machineLearning.clip.modelName 
    !== newConfig.machineLearning.clip.modelName;
```

**切换策略**：

| 场景 | 操作 |
|------|------|
| 模型不同，维度相同 | 删除所有搜索嵌入，触发重新编码 |
| 模型不同，维度不同 | 更新 DB 维度 + 删除所有嵌入 |
| 模型相同，维度意外不一致 | 仅更新 DB 维度 |

> **注意**：目前人脸识别模型切换**不会**自动触发人脸向量重建，需用户手动执行 "重新检测所有人脸"。

---

## 向量维度与重建任务联动

### 1. 向量维度管理

#### CLIP 搜索向量表 (`smart_search`)

```sql
-- 列定义
assetId: UUID
embedding: vector(<dimSize>)   -- 维度由模型决定
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

> **固定维度**：人脸向量固定为 512 维，因为所有 InsightFace 模型输出维度一致。

### 2. 维度同步与重建流程

```typescript
// SmartInfoService.init() - 核心逻辑
await databaseRepository.withLock(DatabaseLock.CLIPDimSize, async () => {
  const { dimSize } = getCLIPModelInfo(newConfig.machineLearning.clip.modelName);
  const dbDimSize = await databaseRepository.getDimensionSize('smart_search');
  
  const modelChange = oldConfig?.clip.modelName !== newConfig.clip.modelName;
  const dimSizeChange = dbDimSize !== dimSize;
  
  if (!modelChange && !dimSizeChange) return;

  if (dimSizeChange) {
    // 维度不同：更新数据库列类型，ALTER COLUMN vector(n)
    await databaseRepository.setDimensionSize(dimSize);
  } else {
    // 同维度但模型不同：仅清空现有嵌入
    await databaseRepository.deleteAllSearchEmbeddings();
  }
  
  // TODO: 自动触发全量重建任务 (目前需手动触发)
});
```

### 3. 锁机制与任务同步

**锁名称**：`DatabaseLock.CLIPDimSize`

```typescript
// 正在进行的编码任务会检查锁状态
if (databaseRepository.isBusy(DatabaseLock.CLIPDimSize)) {
  await databaseRepository.wait(DatabaseLock.CLIPDimSize);
}

// 编码完成后二次校验，防止编码期间模型被更改
const newConfig = await getConfig();
if (oldConfig.clip.modelName !== newConfig.clip.modelName) {
  return JobStatus.Skipped;  // 丢弃过时的嵌入
}
```

### 4. 手动重建触发

用户可在管理界面触发 "重新编码所有 CLIP 嵌入"：

```typescript
// JobName.SmartSearchQueueAll (force=true)
async handleQueueEncodeClip({ force }) {
  if (force) {
    // 强制模式：重置维度 + 重新处理所有资产
    const { dimSize } = getCLIPModelInfo(config.clip.modelName);
    await databaseRepository.setDimensionSize(dimSize);
  }
  
  // 流式分页处理所有资产
  for await (const asset of streamForEncodeClip(force)) {
    await jobRepository.queue({ name: JobName.SmartSearch, data: { id: asset.id } });
  }
}
```

**人脸识别重建**：
```typescript
// JobName.AssetDetectFacesQueueAll (force=true)
async handleQueueDetectFaces({ force }) {
  if (force) {
    // 删除所有 ML 生成的人脸 + 向量
    await personRepository.deleteFaces({ sourceType: SourceType.MachineLearning });
    await personRepository.vacuum({ reindexVectors: true });  // 重建向量索引
  }
  // 重新检测所有资产人脸
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
  
  // IOU (交并比) 匹配已有人脸
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

**IOU 阈值**：0.5，避免重复检测同一张脸。

### 3. 人脸向量搜索

```sql
-- 人脸相似度搜索 (person.repository.sql)
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
  // 定期 ping 所有 ML 服务器
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
  // 先尝试健康服务器，再尝试不健康的
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

## 注意事项与最佳实践

### 1. 模型切换
- **CLIP 模型切换**会触发嵌入删除，但不会自动重建，需用户手动触发
- **人脸识别模型切换**不会自动触发重建，需手动执行 "重新检测所有人脸"
- 切换前评估维度变化影响，大模型可能显著增加存储和搜索延迟

### 2. 性能优化
- 预加载常用模型避免首次冷启动延迟
- 高并发场景启用多 ML 服务器负载均衡
- 批量任务（如全库重建）建议在低峰期执行
- 考虑使用更大的 batch size 提高推理吞吐量

### 3. 故障处理
- ML 服务崩溃不影响主服务可用性，仅禁用智能功能
- 多 ML 服务器配置可提供高可用性
- 向量索引损坏时可通过重建任务恢复
