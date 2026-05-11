# Immich ML 服务 RPC 接口与模型管理

## 目录

1. [架构概述](#架构概述)
2. [RPC 接口规范](#rpc-接口规范)
3. [模型加载机制](#模型加载机制)
4. [模型版本管理与切换](#模型版本管理与切换)
5. [主服务调用流程](#主服务调用流程)
6. [配置与环境变量](#配置与环境变量)

---

## 架构概述

Immich 使用独立的 Python 微服务处理机器学习任务，包括人脸识别、图像搜索和 OCR。主服务（TypeScript/NestJS）通过 HTTP REST API 与 ML 服务通信。

### 核心组件

| 组件 | 语言/框架 | 职责 |
|------|-----------|------|
| ML 服务 | Python/FastAPI | 模型加载、推理执行、模型缓存 |
| 主服务 | TypeScript/NestJS | 请求编排、健康检查、故障转移 |
| 模型仓库 | Hugging Face Hub | 模型存储与分发 |

---

## RPC 接口规范

### 端点

| 端点 | 方法 | 说明 |
|------|------|------|
| `/` | GET | 服务信息 |
| `/ping` | GET | 健康检查 |
| `/predict` | POST | 执行推理任务 |

### 请求格式 (FormData)

```
entries: <JSON 配置字符串>
image: <二进制图像文件>  # 或
text: <文本字符串>
```

### entries JSON 结构

```typescript
type PipelineRequest = Record<ModelTask, Record<ModelType, PipelineEntry>>;

interface PipelineEntry {
  modelName: string;
  options?: Record<string, any>;
}

enum ModelTask {
  FACIAL_RECOGNITION = 'facial-recognition',
  SEARCH = 'clip',
  OCR = 'ocr'
}

enum ModelType {
  DETECTION = 'detection',
  RECOGNITION = 'recognition',
  TEXTUAL = 'textual',
  VISUAL = 'visual'
}
```

### 请求示例

#### 1. 人脸识别

```json
{
  "facial-recognition": {
    "detection": {
      "modelName": "buffalo_l",
      "options": { "minScore": 0.7 }
    },
    "recognition": {
      "modelName": "buffalo_l"
    }
  }
}
```

#### 2. 图像编码 (CLIP)

```json
{
  "clip": {
    "visual": { "modelName": "ViT-L-14-quickgelu" }
  }
}
```

#### 3. 文本编码 (CLIP)

```json
{
  "clip": {
    "textual": {
      "modelName": "ViT-L-14-quickgelu",
      "options": { "language": "zh" }
    }
  }
}
```

#### 4. OCR

```json
{
  "ocr": {
    "detection": {
      "modelName": "PP-OCRv4-mobile",
      "options": { "minScore": 0.5, "maxResolution": 2048 }
    },
    "recognition": {
      "modelName": "PP-OCRv4-mobile",
      "options": { "minScore": 0.5 }
    }
  }
}
```

### 响应格式

```typescript
interface InferenceResponse {
  "imageHeight"?: number;
  "imageWidth"?: number;
  "facial-recognition"?: Face[];
  "clip"?: string;  // base64 编码的 embedding
  "ocr"?: OCRResult;
}

interface Face {
  boundingBox: { x1: number; y1: number; x2: number; y2: number };
  embedding: string;  // base64 编码
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

## 模型加载机制

### 1. 模型缓存 (ModelCache)

使用 `aiocache` 的内存缓存管理模型实例：

```python
class ModelCache:
    def __init__(self, revalidate=False, timeout=None, profiling=False):
        self.cache = SimpleMemoryCache(timeout=timeout, ...)
    
    async def get(model_name, model_type, model_task, **kwargs):
        # 缓存键: model_name + model_type + model_task
        # 命中: 直接返回
        # 未命中: 实例化新模型
```

**缓存策略**：
- **TTL (Time-To-Live)**: 默认 300 秒，空闲后卸载
- **重新验证**: 访问时重置 TTL (`revalidate=True`)
- **乐观锁**: 防止并发加载同一模型

### 2. 模型生命周期

```
请求到达
    ↓
检查缓存 (ModelCache.get)
    ├─ 命中 → 返回模型实例
    └─ 未命中 → 实例化模型
        ↓
检查模型文件 (cached 属性)
    ├─ 存在 → 跳过下载
    └─ 不存在 → 从 Hugging Face Hub 下载
        ↓
加载到内存 (model.load())
    ├─ 根据硬件选择格式 (ONNX/ARMNN/RKNN)
    └─ 创建推理会话 (Session)
        ↓
执行推理 (model.predict())
    ↓
返回结果
```

### 3. 模型下载

```python
def _download(self):
    snapshot_download(
        f"immich-app/{clean_name(self.model_name)}",
        cache_dir=self.cache_dir,
        ignore_patterns=ignored_patterns[self.model_format]
    )
```

**目录结构**：
```
cache_folder/
├── facial-recognition/
│   └── buffalo_l/
│       └── detection/
│           └── model.onnx
├── clip/
│   └── ViT-L-14-quickgelu/
│       ├── visual/
│       └── textual/
└── ocr/
    └── PP-OCRv4-mobile/
        ├── detection/
        └── recognition/
```

### 4. 模型格式自动选择

根据可用硬件自动选择最佳模型格式：

```python
@property
def _model_format_default(self) -> ModelFormat:
    if rknn.is_available:
        return ModelFormat.RKNN       # Rockchip NPU
    elif ann.is_available and settings.ann:
        return ModelFormat.ARMNN      # ARM NPU
    else:
        return ModelFormat.ONNX       # CPU/GPU
```

**支持格式**：
- **ONNX**: 通用格式，支持 CUDA/ROCm/OpenVINO/CPU
- **ARMNN**: ARM 架构 NPU 加速
- **RKNN**: Rockchip NPU 加速

### 5. 预加载机制

启动时预加载常用模型：

```python
# config.py
class PreloadModelData(BaseModel):
    clip: ClipSettings
    facial_recognition: FacialRecognitionSettings
    ocr: OcrSettings

# 环境变量配置示例
MACHINE_LEARNING_PRELOAD__CLIP__VISUAL=ViT-L-14-quickgelu
MACHINE_LEARNING_PRELOAD__FACIAL_RECOGNITION__DETECTION=buffalo_l
```

---

## 模型版本管理与切换

### 1. 版本标识

模型版本通过 **模型名称** 标识，存储在 Hugging Face Hub 的 `immich-app` 组织下：

| 任务 | 可用模型 |
|------|----------|
| 人脸识别检测 | `buffalo_l`, `buffalo_s` |
| 人脸识别识别 | `buffalo_l`, `buffalo_s` |
| CLIP 视觉 | `ViT-L-14-quickgelu`, `ViT-B-32__openai` |
| CLIP 文本 | `ViT-L-14-quickgelu`, `ViT-B-32__openai`, `M-CLIP-B-32` |
| OCR 检测 | `PP-OCRv4-mobile` |
| OCR 识别 | `PP-OCRv4-mobile` |

### 2. 版本切换方式

#### 方式 1: 配置文件切换 (推荐)

在主服务 `systemConfig` 中修改：

```typescript
// server/src/config.ts
machineLearning: {
  clip: {
    enabled: true,
    modelName: "ViT-B-32__openai"  // 切换模型
  },
  facialRecognition: {
    enabled: true,
    modelName: "buffalo_s"         // 切换模型
  }
}
```

**生效机制**：
1. 更新配置后，新请求使用新模型名
2. 旧模型缓存超时（默认 300s）后自动卸载
3. 新模型按需加载

#### 方式 2: 环境变量覆盖

```bash
# ML 服务端
MACHINE_LEARNING_PRELOAD__CLIP__VISUAL=ViT-B-32__openai
```

#### 方式 3: 运行时热切换

通过 API 请求中的 `modelName` 参数动态指定：

```json
{
  "clip": {
    "visual": { "modelName": "another-model-version" }
  }
}
```

### 3. 模型源映射

```python
# models/__init__.py
def get_model_class(model_name, model_type, model_task):
    source = get_model_source(model_name)
    match source, model_type, model_task:
        case ModelSource.OPENCLIP, ModelType.VISUAL, ModelTask.SEARCH:
            return OpenClipVisualEncoder
        case ModelSource.INSIGHTFACE, ModelType.DETECTION, ModelTask.FACIAL_RECOGNITION:
            return FaceDetector
        case ModelSource.PADDLE, ModelType.RECOGNITION, ModelTask.OCR:
            return TextRecognizer
```

**模型源**：
- `openclip`: OpenAI CLIP 模型
- `mclip`: 多语言 CLIP
- `insightface`: 人脸识别 (InsightFace)
- `paddle`: OCR (PaddleOCR)

### 4. 模型依赖关系

某些模型依赖其他模型的输出：

```python
class FaceRecognizer(InferenceModel):
    depends = [(ModelType.DETECTION, ModelTask.FACIAL_RECOGNITION)]
    # 人脸识别需要先运行人脸检测
```

ML 服务自动处理依赖：
1. 分离无依赖和有依赖的任务
2. 并行执行无依赖任务
3. 等待依赖完成后执行有依赖任务

---

## 主服务调用流程

### 1. MachineLearningRepository 核心逻辑

```typescript
// server/src/repositories/machine-learning.repository.ts
@Injectable()
export class MachineLearningRepository {
  private healthyMap: Record<string, boolean> = {};

  private async predict<T>(payload: ModelPayload, config: MachineLearningRequest): Promise<T> {
    const formData = await this.getFormData(payload, config);

    // 故障转移: 先尝试健康的服务器
    for (const url of [
      ...this.config.urls.filter(url => this.isHealthy(url)),
      ...this.config.urls.filter(url => !this.isHealthy(url))
    ]) {
      try {
        const response = await fetch(new URL('/predict', url), {
          method: 'POST',
          body: formData
        });
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
}
```

### 2. 健康检查机制

```typescript
// 定期检查所有 ML 服务器
private tick() {
  for (const url of this.config.urls) {
    void this.check(url);
  }
}

private async check(url: string) {
  try {
    const response = await fetch(new URL('/ping', url), {
      signal: AbortSignal.timeout(this.config.availabilityChecks.timeout)
    });
    this.setHealthy(url, response.ok);
  } catch {
    this.setHealthy(url, false);
  }
}
```

### 3. 业务方法

```typescript
// 人脸识别
async detectFaces(imagePath: string, options: FaceDetectionOptions) {
  const request = {
    "facial-recognition": {
      detection: { modelName: options.modelName, options: { minScore: options.minScore } },
      recognition: { modelName: options.modelName }
    }
  };
  return this.predict({ imagePath }, request);
}

// 图像编码 (CLIP)
async encodeImage(imagePath: string, config: CLIPConfig) {
  const request = { clip: { visual: { modelName: config.modelName } } };
  return this.predict({ imagePath }, request);
}

// 文本编码 (CLIP)
async encodeText(text: string, options: TextEncodingOptions) {
  const request = {
    clip: {
      textual: { modelName: options.modelName, options: { language: options.language } }
    }
  };
  return this.predict({ text }, request);
}

// OCR
async ocr(imagePath: string, options: OcrOptions) {
  const request = {
    ocr: {
      detection: { modelName: options.modelName, options: { ... } },
      recognition: { modelName: options.modelName, options: { ... } }
    }
  };
  return this.predict({ imagePath }, request);
}
```

---

## 配置与环境变量

### ML 服务环境变量

| 变量 | 默认值 | 说明 |
|------|--------|------|
| `MACHINE_LEARNING_CACHE_FOLDER` | `~/.cache/immich_ml` | 模型缓存目录 |
| `MACHINE_LEARNING_MODEL_TTL` | `300` | 模型空闲超时 (秒) |
| `MACHINE_LEARNING_MODEL_TTL_POLL_S` | `10` | 空闲检查间隔 |
| `MACHINE_LEARNING_WORKERS` | `1` | Gunicorn worker 数 |
| `MACHINE_LEARNING_WORKER_TIMEOUT` | `300` | Worker 超时 |
| `MACHINE_LEARNING_REQUEST_THREADS` | CPU 核心数 | 推理线程池大小 |
| `MACHINE_LEARNING_ANN` | `true` | 启用 ARMNN 加速 |
| `MACHINE_LEARNING_PRELOAD__*` | `null` | 预加载模型配置 |
| `MACHINE_LEARNING_MAX_BATCH_SIZE__FACIAL_RECOGNITION` | 自动 | 人脸识别批量大小 |
| `IMMICH_HOST` | `[::]` | 监听地址 |
| `IMMICH_PORT` | `3003` | 监听端口 |

### 主服务 ML 配置

```typescript
// server/src/config.ts
machineLearning: {
  enabled: true;
  urls: ['http://immich-machine-learning:3003'];  // 支持多服务器
  availabilityChecks: {
    enabled: true;
    timeout: 5000;     // 5 秒
    interval: 30000;   // 30 秒
  };
  clip: {
    enabled: true;
    modelName: 'ViT-L-14-quickgelu';
  };
  facialRecognition: {
    enabled: true;
    modelName: 'buffalo_l';
    minScore: 0.7;
    minFaces: 1;
    maxDistance: 0.6;
  };
  ocr: {
    enabled: true;
    modelName: 'PP-OCRv4-mobile';
    minDetectionScore: 0.5;
    minRecognitionScore: 0.5;
    maxResolution: 2048;
  };
}
```

---

## 注意事项

1. **模型切换无需重启服务**：修改配置后，新请求自动使用新模型
2. **缓存清理**：模型损坏时会自动清理缓存并重试下载
3. **多 ML 服务器**：支持横向扩展和故障转移
4. **硬件加速**：根据运行环境自动选择最佳推理后端
5. **空闲关机**：长时间无请求时自动释放内存
6. **批量处理**：支持人脸识别批量推理以提高性能
