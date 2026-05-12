# Immich 智能搜索技术架构分析

## 概述

Immich 智能搜索功能允许用户通过自然语言描述搜索照片，背后采用了 CLIP 模型将文本和图像编码到同一向量空间，并结合向量索引和关键词检索实现高效的混合搜索。

---

## 一、向量模型 (CLIP)

### 1.1 模型架构

Immich 使用 OpenCLIP 架构，包含两个编码器：

- **文本编码器 (Textual Encoder)**: 将自然语言文本编码为 512 维向量
- **图像编码器 (Visual Encoder)**: 将图像内容编码为 512 维向量

### 1.2 文本编码流程

**文件位置**: `machine-learning/immich_ml/models/clip/textual.py`

```python
class OpenClipTextualEncoder(BaseCLIPTextualEncoder):
    def tokenize(self, text: str, language: str | None = None) -> dict[str, NDArray[np.int32]]:
        # 1. 文本清洗与规范化
        text = clean_text(text, canonicalize=self.canonicalize)
        
        # 2. 多语言支持 (NLLB 模型)
        if self.is_nllb and language is not None:
            flores_code = WEBLATE_TO_FLORES200.get(language)
            if flores_code is None:
                flores_code = 'eng_Latn'  # 默认英语
            text = f"{flores_code}{text}"
        
        # 3. Tokenizer 编码 (context length = 77)
        tokens: Encoding = self.tokenizer.encode(text)
        return {"text": np.array([tokens.ids], dtype=np.int32)}
    
    def _predict(self, inputs: str, language: str | None = None) -> str:
        tokens = self.tokenize(inputs, language=language)
        # 4. 通过 ONNX 模型推理得到 512 维向量
        res: NDArray[np.float32] = self.session.run(None, tokens)[0][0]
        # 5. 序列化为字符串存储
        return serialize_np_array(res)
```

### 1.3 图像编码流程

**文件位置**: `machine-learning/immich_ml/models/clip/visual.py`

```python
class OpenClipVisualEncoder(BaseCLIPVisualEncoder):
    def transform(self, image: Image.Image) -> dict[str, NDArray[np.float32]]:
        # 1. 图像预处理：调整大小
        image = resize_pil(image, self.size)  # size = 224
        
        # 2. 中心裁剪
        image = crop_pil(image, self.size)
        
        # 3. 转换为 numpy 数组
        image_np = to_numpy(image)
        
        # 4. 归一化 (mean/std 基于 ImageNet 统计)
        image_np = normalize(image_np, self.mean, self.std)
        
        # 5. 转置为 CHW 格式并添加 batch 维度
        return {"image": np.expand_dims(image_np.transpose(2, 0, 1), 0)}
    
    def _predict(self, inputs: Image.Image | bytes) -> str:
        image = decode_pil(inputs)
        # 6. ONNX 模型推理得到 512 维向量
        res: NDArray[np.float32] = self.session.run(None, self.transform(image))[0][0]
        return serialize_np_array(res)
```

### 1.4 模型关键参数

| 参数 | 值 | 说明 |
|------|-----|------|
| 向量维度 | 512 | `DatabaseLock.CLIPDimSize = 512` |
| 文本上下文长度 | 77 | Tokenizer 最大长度 |
| 图像输入尺寸 | 224x224 | 视觉编码器输入 |
| 支持模型 | OpenCLIP, mCLIP, NLLB | 多语言支持 |

---

## 二、向量索引实现

### 2.1 数据库表结构

**文件位置**: `server/src/schema/tables/smart-search.table.ts`

```typescript
@Table({ name: 'smart_search' })
@Index({
  name: 'clip_index',
  using: 'hnsw',  // 分层可导航小世界图索引
  expression: `embedding vector_cosine_ops`,  // 余弦相似度
  with: `ef_construction = 300, m = 16`,  // 索引构建参数
  synchronize: false,
})
export class SmartSearchTable {
  @ForeignKeyColumn(() => AssetTable, { onDelete: 'CASCADE', primaryKey: true })
  assetId!: string;

  @Column({ type: 'vector', length: 512, storage: 'external', synchronize: false })
  embedding!: string;  // 512 维向量
}
```

### 2.2 HNSW 索引参数说明

| 参数 | 值 | 说明 |
|------|-----|------|
| `m` | 16 | 每个节点在构建时的邻居数量 |
| `ef_construction` | 300 | 构建时动态邻居列表大小 |
| `probes` | 1 | 查询时探索的邻居数量 |
| 距离度量 | `vector_cosine_ops` | 余弦相似度 (1 - cosine) |

### 2.3 向量相似度搜索实现

**文件位置**: `server/src/repositories/search.repository.ts`

```typescript
searchSmart(pagination: SearchPaginationOptions, options: SmartSearchOptions) {
  return this.db.transaction().execute(async (trx) => {
    // 设置查询时的 probe 数量
    await sql`set local vchordrq.probes = ${sql.lit(probes[VectorIndex.Clip])}`.execute(trx);
    
    const items = await searchAssetBuilder(trx, options)
      .selectAll('asset')
      .innerJoin('smart_search', 'asset.id', 'smart_search.assetId')
      // 余弦相似度距离排序
      .orderBy(sql`smart_search.embedding <=> ${options.embedding}`)
      .limit(pagination.size + 1)
      .offset((pagination.page - 1) * pagination.size)
      .execute();
      
    return paginationHelper(items, pagination.size);
  });
}
```

### 2.4 向量扩展支持

Immich 支持两种 PostgreSQL 向量扩展：

| 扩展 | 说明 | 版本要求 |
|------|------|----------|
| `vector` (pgvector) | 官方标准向量扩展 | `>= 0.5.0` |
| `vchord` | pgvector 的性能优化分支 | `>= 0.1.0` |

**文件位置**: `server/src/repositories/database.repository.ts`

```typescript
export const VECTOR_EXTENSIONS = [DatabaseExtension.VectorChord, DatabaseExtension.Vector];

export async function getVectorExtension(runner: Kysely<DB>): Promise<VectorExtension> {
  // 1. 检查环境变量配置
  const envExtension = new ConfigRepository().getEnv().database.vectorExtension;
  if (envExtension) return envExtension;

  // 2. 检测数据库可用扩展
  const query = `SELECT name FROM pg_available_extensions WHERE name IN (${VECTOR_EXTENSIONS.map(ext => `'${ext}'`).join(', ')})`;
  const { rows: availableExtensions } = await sql.raw<{ name: VectorExtension }>(query).execute(runner);
  
  // 3. 优先选择 vchord
  const extensionNames = new Set(availableExtensions.map(row => row.name));
  return VECTOR_EXTENSIONS.find(ext => extensionNames.has(ext));
}
```

---

## 三、智能搜索服务流程

### 3.1 搜索流程总览

```
用户输入查询文本
    ↓
[ 缓存检查 ] → 命中? → 直接使用缓存向量
    ↓ 未命中
[ 调用 ML 服务编码文本 ]
    ↓
[ 获取用户可访问的资产范围 ]
    ↓
[ 向量相似度搜索 ]
    ↓
[ 分页返回结果 ]
```

### 3.2 核心实现代码

**文件位置**: `server/src/services/search.service.ts`

```typescript
@Injectable()
export class SearchService extends BaseService {
  // 文本向量缓存 (LRU, 容量 100)
  private embeddingCache = new LRUMap<string, string>(100);

  async searchSmart(auth: AuthDto, dto: SmartSearchDto): Promise<SearchResponseDto> {
    // 1. 检查智能搜索是否启用
    const { machineLearning } = await this.getConfig({ withCache: false });
    if (!isSmartSearchEnabled(machineLearning)) {
      throw new BadRequestException('Smart search is not enabled');
    }

    // 2. 获取用户可访问的用户 ID (包含伙伴共享)
    const userIds = this.getUserIdsToSearch(auth);
    
    let embedding: string;
    
    // 3. 处理查询输入 (文本或参考图像)
    if (dto.query) {
      // 缓存键: 模型名 + 查询文本 + 语言
      const key = machineLearning.clip.modelName + dto.query + dto.language;
      embedding = this.embeddingCache.get(key);
      
      if (!embedding) {
        // 调用机器学习服务编码文本
        embedding = await this.machineLearningRepository.encodeText(dto.query, {
          modelName: machineLearning.clip.modelName,
          language: dto.language,
        });
        this.embeddingCache.set(key, embedding);
      }
    } else if (dto.queryAssetId) {
      // 以图搜图: 使用已有资产的 embedding
      await this.requireAccess({ auth, permission: Permission.AssetRead, ids: [dto.queryAssetId] });
      const getEmbeddingResponse = await this.searchRepository.getEmbedding(dto.queryAssetId);
      const assetEmbedding = getEmbeddingResponse?.embedding;
      if (!assetEmbedding) {
        throw new BadRequestException(`Asset ${dto.queryAssetId} has no embedding`);
      }
      embedding = assetEmbedding;
    } else {
      throw new BadRequestException('Either `query` or `queryAssetId` must be set');
    }

    // 4. 执行向量搜索
    const page = dto.page ?? 1;
    const size = dto.size || 100;
    const { hasNextPage, items } = await this.searchRepository.searchSmart(
      { page, size },
      { ...dto, userIds: await userIds, embedding },
    );

    return this.mapResponse(items, hasNextPage ? (page + 1).toString() : null, { auth });
  }

  // 获取用户可搜索的用户 ID 列表 (包含伙伴共享)
  private async getUserIdsToSearch(auth: AuthDto): Promise<string[]> {
    const partnerIds = await getMyPartnerIds({
      userId: auth.user.id,
      repository: this.partnerRepository,
      timelineEnabled: true,
    });
    return [auth.user.id, ...partnerIds];
  }
}
```

### 3.3 机器学习服务通信

**文件位置**: `server/src/repositories/machine-learning.repository.ts`

```typescript
@Injectable()
export class MachineLearningRepository {
  private healthyMap: Record<string, boolean> = {};

  // 编码文本为向量
  async encodeText(text: string, { language, modelName }: TextEncodingOptions) {
    const request = { 
      [ModelTask.SEARCH]: { 
        [ModelType.TEXTUAL]: { modelName, options: { language } } 
      } 
    };
    const response = await this.predict<ClipTextualResponse>({ text }, request);
    return response[ModelTask.SEARCH];
  }

  // 编码图像为向量
  async encodeImage(imagePath: string, { modelName }: CLIPConfig) {
    const request = { [ModelTask.SEARCH]: { [ModelType.VISUAL]: { modelName } } };
    const response = await this.predict<ClipVisualResponse>({ imagePath }, request);
    return response[ModelTask.SEARCH];
  }

  // 统一的预测接口 (支持多 ML 服务器负载均衡)
  private async predict<T>(payload: ModelPayload, config: MachineLearningRequest): Promise<T> {
    const formData = await this.getFormData(payload, config);

    // 健康服务器优先, 其次尝试不健康的
    for (const url of [
      ...this.config.urls.filter(url => this.isHealthy(url)),
      ...this.config.urls.filter(url => !this.isHealthy(url)),
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
        // 记录失败, 尝试下一个服务器
      }
      this.setHealthy(url, false);
    }

    throw new Error(`Machine learning request failed for all URLs`);
  }
}
```

---

## 四、后台任务与索引构建

### 4.1 任务队列系统

**文件位置**: `server/src/enum.ts`

```typescript
export enum QueueName {
  SmartSearch = 'smartSearch',  // 智能搜索编码队列
  // ... 其他队列
}

export enum JobName {
  SmartSearchQueueAll = 'SmartSearchQueueAll',  // 全量编码任务
  SmartSearch = 'SmartSearch',                   // 单个资产编码
}
```

### 4.2 向量存储与更新

**文件位置**: `server/src/repositories/search.repository.ts`

```typescript
async upsert(assetId: string, embedding: string): Promise<void> {
  await this.db
    .insertInto('smart_search')
    .values({ assetId, embedding })
    // 冲突时更新 (原子操作)
    .onConflict(oc => oc.column('assetId').doUpdateSet(eb => ({
      embedding: eb.ref('excluded.embedding')
    })))
    .execute();
}
```

---

## 五、混合检索机制说明

**重要澄清**: Immich 当前实现中, "混合检索"并非传统意义上的关键词+向量加权融合, 而是通过以下方式实现检索能力的组合:

### 5.1 多维度过滤机制

在 `SmartSearchOptions` 中, 向量搜索与多个过滤条件并行工作:

```typescript
export type SmartSearchOptions = 
  SearchDateOptions &        // 日期范围过滤
  SearchEmbeddingOptions &   // 向量相似度搜索 (核心)
  SearchExifOptions &        // EXIF 元数据过滤 (地点、相机等)
  SearchOneToOneRelationOptions &
  SearchStatusOptions &      // 资产状态 (收藏、归档等)
  SearchUserIdOptions &      // 用户权限
  SearchPeopleOptions &      // 人物过滤
  SearchTagOptions &         // 标签过滤
  SearchOcrOptions;          // OCR 文本过滤
```

### 5.2 检索流程详解

```
输入: 自然语言查询 + 可选过滤条件
    ↓
[ 步骤 1: 文本向量化 ]
    ↓ CLIP 文本编码器
    512维文本向量
    ↓
[ 步骤 2: 向量 ANN 搜索 ]
    ↓ HNSW 索引 + 余弦相似度
    按相似度排序的候选资产集合
    ↓
[ 步骤 3: 应用多维度过滤 ]
    ├─ 用户权限过滤 (ownerId IN userIds)
    ├─ 日期范围过滤 (fileCreatedAt, localDateTime)
    ├─ 资产状态过滤 (isFavorite, visibility, status)
    ├─ EXIF 元数据过滤 (city, country, make, model, lensModel)
    ├─ 人物过滤 (personIds)
    ├─ 标签过滤 (tagIds)
    └─ OCR 文本过滤 (ocr 关键词)
    ↓
[ 步骤 4: 分页返回结果 ]
```

### 5.3 与元数据搜索的互补

Immich 同时提供两种搜索方式, 用户可按需选择:

| 搜索类型 | 实现方式 | 适用场景 |
|----------|----------|----------|
| **智能搜索** | CLIP 向量 + HNSW 索引 | 语义内容搜索: "海滩上的狗", "日落时分的人群" |
| **元数据搜索** | 传统 SQL 查询 + B 树索引 | 精确条件搜索: "2024年1月拍摄", "iPhone 15", "东京" |

### 5.4 OCR 文本检索

对于包含文字的图像, Immich 通过 OCR 提取文本并存储, 支持文本关键词的精确匹配。这与 CLIP 的语义理解形成互补:

- **CLIP**: 理解图像整体语义内容
- **OCR**: 精确提取图像中的文字信息

---

## 六、性能优化策略

### 6.1 缓存策略

| 缓存层级 | 实现 | 容量 | 作用 |
|----------|------|------|------|
| 文本向量缓存 | LRUMap | 100 条 | 避免重复编码相同查询 |
| ML 服务健康状态 | Map | - | 故障转移与负载均衡 |

### 6.2 索引优化

1. **HNSW 参数调优**:
   - `m = 16`: 平衡索引大小和查询精度
   - `ef_construction = 300`: 较高的构建精度
   - `probes = 1`: 查询时的性能优先配置

2. **向量存储**:
   - 使用 `storage = 'external'` 优化 TOAST 存储
   - 避免行内存储大向量数据

### 6.3 查询优化

```typescript
// 1. 事务内设置 probe 参数, 会话级生效
await sql`set local vchordrq.probes = ${sql.lit(probes[VectorIndex.Clip])}`.execute(trx);

// 2. 分页 + limit/offset
.limit(pagination.size + 1)
.offset((pagination.page - 1) * pagination.size)
```

---

## 七、系统架构图

```
┌─────────────────────────────────────────────────────────────────┐
│                        Immich 智能搜索系统                       │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  ┌──────────────┐                                               │
│  │  用户界面    │  - 自然语言输入                               │
│  │  (Web/Mobile)│  - 搜索结果展示                               │
│  └──────┬───────┘                                               │
│         │                                                       │
│  ┌──────▼───────┐      ┌────────────────┐                      │
│  │ Search API   │──────►│  权限验证      │                      │
│  │  Controller  │      │  (伙伴共享)     │                      │
│  └──────┬───────┘      └────────────────┘                      │
│         │                                                       │
│  ┌──────▼───────┐      ┌────────────────┐                      │
│  │ SearchService│──────►│  Embedding     │                      │
│  │              │      │  Cache (LRU)    │                      │
│  └──────┬───────┘      └────────────────┘                      │
│         │                                                       │
│  ┌──────▼──────────────────────────────────┐                   │
│  │     Machine Learning Repository          │                   │
│  │  (多服务器负载均衡 + 健康检查)            │                   │
│  └──────┬──────────────────────────────────┘                   │
│         │                                                       │
│  ┌──────▼───────┐      ┌────────────────┐                      │
│  │   ML Server  │──────►│  OpenCLIP      │                      │
│  │  (Python/ONNX)│      │  Text Encoder  │                      │
│  │              │      │  Visual Encoder │                      │
│  └──────────────┘      └────────────────┘                      │
│         │                                                       │
│  ┌──────▼──────────────────────────────────┐                   │
│  │      Search Repository (PostgreSQL)      │                   │
│  │  ┌────────────────────────────────────┐ │                   │
│  │  │ smart_search 表                     │ │                   │
│  │  │ - assetId (FK)                      │ │                   │
│  │  │ - embedding (vector(512))           │ │                   │
│  │  │ - HNSW 索引 (clip_index)            │ │                   │
│  │  └────────────────────────────────────┘ │                   │
│  └─────────────────────────────────────────┘                   │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

## 八、关键技术总结

| 技术组件 | 选型 | 优势 |
|----------|------|------|
| **向量模型** | OpenCLIP / mCLIP / NLLB | 多语言支持, 开源成熟 |
| **向量数据库** | pgvector / vchord | PostgreSQL 原生集成 |
| **索引算法** | HNSW | 对数级查询复杂度, 高召回率 |
| **距离度量** | 余弦相似度 | 归一化向量的标准度量 |
| **运行时** | ONNX Runtime | 跨平台高性能推理 |
| **缓存策略** | LRU Map | 高频查询复用 |
| **高可用** | 多 ML 服务器 + 健康检查 | 故障自动转移 |

---

## 参考文件路径

- 搜索服务: `server/src/services/search.service.ts`
- 搜索仓库: `server/src/repositories/search.repository.ts`
- ML 仓库: `server/src/repositories/machine-learning.repository.ts`
- CLIP 文本编码器: `machine-learning/immich_ml/models/clip/textual.py`
- CLIP 图像编码器: `machine-learning/immich_ml/models/clip/visual.py`
- 数据库 Schema: `server/src/schema/tables/smart-search.table.ts`
- 枚举定义: `server/src/enum.ts`
