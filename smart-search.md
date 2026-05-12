# Immich 智能搜索技术架构分析

## 概述

Immich 智能搜索功能允许用户通过自然语言描述搜索照片。核心采用 CLIP 模型将文本/图像编码为向量，通过 HNSW 向量索引进行相似度检索，并与元数据过滤、OCR 全文检索组合工作。**不存在向量与关键词的权重融合公式**，两者是独立查询链路。

---

## 一、向量模型 (CLIP)

### 1.1 模型架构

Immich 使用 OpenCLIP 双编码器架构：

- **文本编码器 (Textual Encoder)**: 将自然语言文本编码为向量
- **图像编码器 (Visual Encoder)**: 将图像内容编码为向量
- **同构向量空间**: 文本与图像向量共享同一空间，可直接计算余弦相似度

### 1.2 维度配置说明

**重要修正**: 向量维度**由模型决定，通过配置驱动**，并非固定为 512。

- **模型配置项**: `machineLearning.clip.modelName`（系统配置中可指定不同模型）
- **数据库 Schema 当前实现**: `smart_search.embedding` 字段当前定义为 `vector(512)`，但维度上限随模型更换而变化
- **支持的模型变体**:
  - OpenCLIP 系列（如 ViT-B/32: 512 维）
  - mCLIP 多语言系列
  - 其他自定义 ONNX 格式模型

配置路径: `server/src/dtos/model-config.dto.ts`

```typescript
export const CLIPConfigSchema = ModelConfigSchema.extend({
  modelName: z.string().describe('Name of the model to use'),  // 模型名称决定维度
});
```

### 1.3 文本编码流程

**文件位置**: `machine-learning/immich_ml/models/clip/textual.py`

```python
class OpenClipTextualEncoder(BaseCLIPTextualEncoder):
    def tokenize(self, text: str, language: str | None = None) -> dict[str, NDArray[np.int32]]:
        # 1. 文本清洗与规范化
        text = clean_text(text, canonicalize=self.canonicalize)
        
        # 2. 多语言支持 (NLLB 模型语种前缀)
        if self.is_nllb and language is not None:
            flores_code = WEBLATE_TO_FLORES200.get(language)
            if flores_code is None:
                flores_code = 'eng_Latn'
            text = f"{flores_code}{text}"
        
        # 3. Tokenizer 编码 (context length = 77)
        tokens: Encoding = self.tokenizer.encode(text)
        return {"text": np.array([tokens.ids], dtype=np.int32)}
    
    def _predict(self, inputs: str, language: str | None = None) -> str:
        tokens = self.tokenize(inputs, language=language)
        # 4. ONNX 模型推理，输出维度由模型决定
        res: NDArray[np.float32] = self.session.run(None, tokens)[0][0]
        return serialize_np_array(res)  # 序列化为 JSON 数组字符串
```

### 1.4 图像编码流程

**文件位置**: `machine-learning/immich_ml/models/clip/visual.py`

```python
class OpenClipVisualEncoder(BaseCLIPVisualEncoder):
    def transform(self, image: Image.Image) -> dict[str, NDArray[np.float32]]:
        # 1. 图像预处理：调整大小 → 中心裁剪 → 归一化
        image = resize_pil(image, self.size)  # size 由模型配置决定 (通常 224)
        image = crop_pil(image, self.size)
        image_np = to_numpy(image)
        image_np = normalize(image_np, self.mean, self.std)  # ImageNet 统计值
        
        # 2. 转置为 CHW 格式并添加 batch 维度
        return {"image": np.expand_dims(image_np.transpose(2, 0, 1), 0)}
    
    def _predict(self, inputs: Image.Image | bytes) -> str:
        image = decode_pil(inputs)
        # 3. ONNX 模型推理，输出维度由模型决定
        res: NDArray[np.float32] = self.session.run(None, self.transform(image))[0][0]
        return serialize_np_array(res)
```

---

## 二、向量索引实现

### 2.1 数据库表结构

**文件位置**: `server/src/schema/tables/smart-search.table.ts`

```typescript
@Table({ name: 'smart_search' })
@Index({
  name: 'clip_index',
  using: 'hnsw',  // 分层可导航小世界图 (Hierarchical Navigable Small World)
  expression: `embedding vector_cosine_ops`,  // 余弦相似度算子
  with: `ef_construction = 300, m = 16`,  // 索引构建参数
  synchronize: false,
})
export class SmartSearchTable {
  @ForeignKeyColumn(() => AssetTable, { onDelete: 'CASCADE', primaryKey: true })
  assetId!: string;

  @Column({ type: 'vector', length: 512, storage: 'external', synchronize: false })
  embedding!: string;  // 向量维度当前为 512，随模型配置可变更
}
```

### 2.2 HNSW 索引参数说明

| 参数 | 值 | 说明 |
|------|-----|------|
| `m` | 16 | 每个节点在构建时的邻居数量，控制索引大小和精度 |
| `ef_construction` | 300 | 构建时动态邻居列表大小，值越大构建越慢精度越高 |
| `ef_search` | 动态 | 查询时探索邻居数量，默认值 |
| 距离度量 | `vector_cosine_ops` | 余弦相似度 (距离 = 1 - cos(θ) |

### 2.3 向量相似度搜索实现

**文件位置**: `server/src/repositories/search.repository.ts`

```typescript
searchSmart(pagination: SearchPaginationOptions, options: SmartSearchOptions) {
  return this.db.transaction().execute(async (trx) => {
    // 设置会话级 probe 配置 (仅对 vchord 有效)
    await sql`set local vchordrq.probes = ${sql.lit(probes[VectorIndex.Clip])}`.execute(trx);
    
    const items = await searchAssetBuilder(trx, options)
      .selectAll('asset')
      .innerJoin('smart_search', 'asset.id', 'smart_search.assetId')
      // 纯余弦相似度距离排序 (仅按向量距离，无权重融合
      .orderBy(sql`smart_search.embedding <=> ${options.embedding}`)
      .limit(pagination.size + 1)
      .offset((pagination.page - 1) * pagination.size)
      .execute();
      
    return paginationHelper(items, pagination.size);
  });
}
```

### 2.4 向量扩展支持

Immich 支持两种 PostgreSQL 向量扩展，检测顺序优先使用 vchord：

| 扩展 | 说明 |
|------|------|
| `vchord` | pgvector 的高性能分支，支持残差量化 |
| `vector` (pgvector) | 官方标准向量扩展 |

**文件位置**: `server/src/repositories/database.repository.ts`

```typescript
export const VECTOR_EXTENSIONS = [DatabaseExtension.VectorChord, DatabaseExtension.Vector];

export async function getVectorExtension(runner: Kysely<DB>): Promise<VectorExtension> {
  // 1. 环境变量优先配置
  const envExtension = new ConfigRepository().getEnv().database.vectorExtension;
  if (envExtension) return envExtension;

  // 2. 自动检测数据库可用扩展
  const query = `SELECT name FROM pg_available_extensions WHERE name IN (${VECTOR_EXTENSIONS.map(ext => `'${ext}'`).join(', ')})`;
  const { rows: availableExtensions } = await sql.raw<{ name: VectorExtension }>(query).execute(runner);
  
  // 3. 优先选择 vchord
  const extensionNames = new Set(availableExtensions.map(row => row.name));
  return VECTOR_EXTENSIONS.find(ext => extensionNames.has(ext));
}
```

---

## 三、智能搜索服务流程

### 3.1 检索链路澄清

**关键架构要点**:

1. **智能搜索 = 纯向量相似度排序**：按 `embedding <=> query_embedding` 余弦距离升序排列，无任何关键词权重融合
2. **关键词检索 = 独立链路**：通过 OCR 全文检索走 PostgreSQL trigram 索引，使用 `%>>` 运算符
3. **元数据过滤 = SQL WHERE 条件**：日期、地点、相机、人物、标签等作为过滤条件，不参与排序

### 3.2 智能搜索核心实现

**文件位置**: `server/src/services/search.service.ts`

```typescript
@Injectable()
export class SearchService extends BaseService {
  // 文本向量缓存 (LRU 策略，容量 100)
  private embeddingCache = new LRUMap<string, string>(100);

  async searchSmart(auth: AuthDto, dto: SmartSearchDto): Promise<SearchResponseDto> {
    // 1. 验证智能搜索启用状态
    const { machineLearning } = await this.getConfig({ withCache: false });
    if (!isSmartSearchEnabled(machineLearning)) {
      throw new BadRequestException('Smart search is not enabled');
    }

    // 2. 获取用户可访问范围 (包含伙伴共享)
    const userIds = this.getUserIdsToSearch(auth);
    
    let embedding: string;
    
    // 3. 查询向量化 (两种输入模式
    if (dto.query) {
      // 缓存键：模型名 + 查询文本 + 语言
      const key = machineLearning.clip.modelName + dto.query + dto.language;
      embedding = this.embeddingCache.get(key);
      
      if (!embedding) {
        // 调用机器学习服务编码文本
        embedding = await this.machineLearningRepository.encodeText(dto.query, {
          modelName: machineLearning.clip.modelName,  // 配置驱动模型选择
          language: dto.language,
        });
        this.embeddingCache.set(key, embedding);
      }
    } else if (dto.queryAssetId) {
      // 以图搜图：使用已有资产的 embedding 作为查询向量
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

    // 4. 执行向量相似度搜索 + 元数据过滤
    const page = dto.page ?? 1;
    const size = dto.size || 100;
    const { hasNextPage, items } = await this.searchRepository.searchSmart(
      { page, size },
      { ...dto, userIds: await userIds, embedding },  // 过滤条件在此传入 SQL WHERE 生效
    );

    return this.mapResponse(items, hasNextPage ? (page + 1).toString() : null, { auth });
  }
}
```

### 3.3 机器学习服务通信

```typescript
@Injectable()
export class MachineLearningRepository {
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

  // 统一预测接口，支持多服务器负载均衡与故障转移
  private async predict<T>(payload: ModelPayload, config: MachineLearningRequest): Promise<T> {
    const formData = await this.getFormData(payload, config);

    // 健康服务器优先，不健康服务器次之
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
        // 记录失败，尝试下一个服务器
      }
      this.setHealthy(url, false);
    }

    throw new Error(`Machine learning request failed for all URLs`);
  }
}
```

---

## 四、混合检索机制深度解析

### 4.1 不存在权重融合公式

**明确结论：Immich 智能搜索不涉及向量与关键词的权重融合**。两者是完全独立的检索链路：

| 检索类型 | 排序依据 | 实现方式 |
|----------|----------|----------|
| **向量检索 | 纯余弦相似度距离 | HNSW 向量索引，`<=>` 运算符 |
| **OCR 关键词检索** | trigram 文本相似度 | PostgreSQL `%>>` 全文检索运算符 |
| **元数据过滤** | 不参与排序，仅过滤 | SQL WHERE 条件 |

### 4.2 SmartSearchOptions 过滤条件组合

**文件位置**: `server/src/repositories/search.repository.ts

```typescript
export type SmartSearchOptions = 
  SearchDateOptions &        // 日期范围过滤 (WHERE 条件)
  SearchEmbeddingOptions &   // 向量相似度搜索 (ORDER BY 排序)
  SearchExifOptions &        // EXIF 元数据过滤 (WHERE 条件)
  SearchOneToOneRelationOptions &
  SearchStatusOptions &      // 资产状态过滤 (WHERE 条件)
  SearchUserIdOptions &      // 用户权限过滤 (WHERE 条件)
  SearchPeopleOptions &      // 人物 ID 过滤 (WHERE 条件)
  SearchTagOptions &         // 标签 ID 过滤 (WHERE 条件)
  SearchOcrOptions;          // OCR 文本过滤 (独立全文检索)
```

### 4.3 OCR 全文检索独立实现

**文件位置**: `server/src/utils/database.ts:457-461`

```typescript
.$if(!!options.ocr, (qb) =>
  qb
    .innerJoin('ocr_search', 'asset.id', 'ocr_search.assetId')
    .where(() => sql`f_unaccent(ocr_search.text) %>> f_unaccent(${tokenizeForSearch(options.ocr!).join(' ')})`),
)
```

OCR 检索使用 PostgreSQL trigram 索引，与向量检索完全独立：
- 分词策略：CJK 字符二元分词，英文按空格分词
- 相似度运算符：`%>>` (pg_trgm 扩展的 "strict word similarity"
- 走 `ocr_search` 表独立存储文本索引

---

## 五、端到端时序流程

### 5.1 完整时序图

```
用户前端                          Immich Server                          ML Service                          PostgreSQL
  │                                 │                                     │                                 │
  ├─ 输入自然语言查询 "海滩日落 ───▶│                                     │                                 │
  │                                 │                                     │                                 │
  │ 1. 权限验证                     │                                     │                                 │
  │  │                              │                                     │                                 │
  │ 2. 检查智能搜索启用配置         │                                     │                                 │
  │  │                              │                                     │                                 │
  │ 3. LRU 缓存检查                 │                                     │                                 │
  │  ├─ 命中？                       │                                     │                                 │
  │  │  └─ 使用缓存向量 ───────────▶│ 4. 构建用户可访问范围 (含伙伴)        │                                 │
  │  │                              │                                     │                                 │
  │  └─ 未命中                      │                                     │                                 │
  │     └─ 编码文本请求 ───────────────▶├────────────────────────────────────▶│                                 │
  │                                 │                                     │ 5. CLIP 文本编码器推理          │
  │                                 │◀────────────────────────────────────┤                                 │
  │                                 │ 6. 返回向量结果                       │                                 │
  │                                 │ 7. 写入 LRU 缓存                     │                                 │
  │                                 │                                     │                                 │
  │                                 │ 8. 构建查询执行计划                  │                                 │
  │                                 │  ├─ 用户 ID IN 过滤                    │                                 │
  │                                 │  ├─ 日期范围过滤                    │                                 │
  │                                 │  ├─ 地点/相机/人物/标签过滤          │                                 │
  │                                 │  ├─ OCR 全文检索过滤 (如有)           │                                 │
  │                                 │  └─ 向量相似度排序                  │                                 │
  │                                 │                                     │                                 │
  │                                 │ 9. 执行 SQL 查询 ───────────────────▶│                                 │
  │                                 │                                     │ 10. HNSW 索引扫描 + 过滤       │
  │                                 │◀─────────────────────────────────────┤                                 │
  │                                 │ 11. 分页结果集                      │                                 │
  │◀────────────────────────────────┤                                     │                                 │
  │ 12. 返回按相似度排序的照片      │                                     │                                 │
  ▼                                 ▼                                     ▼                                 ▼
```

### 5.2 关键时序步骤详解

| 步骤 | 说明 | 关键代码位置 |
|------|------|------------|
| **1-2** | 前端发起 `/api/search/smart，验证用户身份与智能搜索启用状态 | `search.controller.ts` |
| **3** | LRU 缓存键 = modelName + query + language，容量 100 | `search.service.ts:30` |
| **4-6** | ML 服务负载均衡，健康检查机制，故障自动转移 | `machine-learning.repository.ts:164` |
| **7** | 向量序列化格式：JSON 数字数组字符串 | `visual.py:32` |
| **8** | `searchAssetBuilder` 组装所有过滤条件 | `database.ts:371-486` |
| **9-10** |  PostgreSQL 执行计划：HNSW 索引扫描 → 结果集过滤 | `search.repository.ts:280-296` |
| **11** | 排序仅按向量余弦距离升序，无其他排序因子 | `search.repository.ts:290` |

---

## 六、后台任务与索引构建

### 6.1 任务队列系统

**文件位置**: `server/src/enum.ts`

```typescript
export enum QueueName {
  SmartSearch = 'smartSearch',  // 智能搜索编码队列
}

export enum JobName {
  SmartSearchQueueAll = 'SmartSearchQueueAll',  // 全量编码任务
  SmartSearch = 'SmartSearch',                   // 单个资产编码
}
```

### 6.2 向量存储与更新

```typescript
async upsert(assetId: string, embedding: string): Promise<void> {
  await this.db
    .insertInto('smart_search')
    .values({ assetId, embedding })
    // 冲突时原子更新
    .onConflict(oc => oc.column('assetId').doUpdateSet(eb => ({
      embedding: eb.ref('excluded.embedding')
    })))
    .execute();
}
```

---

## 七、性能优化策略

### 7.1 缓存策略

| 缓存层级 | 实现 | 容量 | 作用 |
|----------|------|------|------|
| 文本向量缓存 | LRU Map | 100 条 | 避免重复编码相同查询 |
| ML 服务健康状态 | Map | - | 故障转移与负载均衡 |

### 7.2 索引优化

1. **HNSW 参数调优**：
   - `m = 16`：平衡索引大小和查询精度
   - `ef_construction = 300`：较高构建精度
   - `probes = 1`：vchord 查询时的性能优先配置

2. **向量存储优化**：
   - `storage = 'external'`：使用 PostgreSQL TOAST 外部存储，避免行内膨胀

### 7.3 查询优化

```typescript
// 事务内会话级设置 probe 参数
await sql`set local vchordrq.probes = ${sql.lit(probes[VectorIndex.Clip])}`.execute(trx);
```

---

## 八、系统架构总览

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                         Immich 智能搜索系统架构                              │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│  ┌──────────────┐                                                        │
│  │  用户前端    │  - 自然语言查询输入                                    │
│  │  (Web/Mobile)│  - 过滤条件选择                                      │
│  └──────┬───────┘                                                        │
│         │                                                                  │
│  ┌──────▼───────┐      ┌───────────────────┐                           │
│  │ Search API   │─────▶│ 权限/启用状态检查  │                           │
│  │  Controller  │      │ (伙伴共享范围)    │                           │
│  └──────┬───────┘      └───────────────────┘                           │
│         │                                                                  │
│  ┌──────▼───────────────────┐      ┌───────────────────┐                  │
│  │   SearchService       │─────▶│ LRU 向量缓存       │                  │
│  │                       │      │ (modelName+query+lang)│                  │
│  └──────┬───────────────┘      └───────────────────┘                  │
│         │                                                                  │
│  ┌──────▼──────────────────────────────────────────┐                     │
│  │      MachineLearning Repository                    │                     │
│  │  (多服务器负载均衡 + 健康检查 + 故障转移) │                     │
│  └──────┬──────────────────────────────────────────┘                     │
│         │                                                                  │
│  ┌──────▼─────────┐      ┌───────────────────────────────────┐              │
│  │   ML Server   │──────▶│  OpenCLIP ONNX 模型            │              │
│  │  (Python)      │      │  - Textual Encoder             │              │
│  │               │      │  - Visual Encoder              │              │
│  └────────────────┘      └───────────────────────────────────┘              │
│         │                                                                  │
│  ┌──────▼──────────────────────────────────────────────────────────┐   │
│  │               Search Repository (PostgreSQL)                          │   │
│  │  ┌────────────────────────────────────────────────────────┐ │   │
│  │  │ smart_search 表                                          │ │   │
│  │  │  - assetId (FK, PK)                                   │ │   │
│  │  │  - embedding (vector(512))                             │ │   │
│  │  │  - HNSW 索引 (clip_index) USING hnsw                │ │   │
│  │  └────────────────────────────────────────────────────────┘ │   │
│  │  ┌────────────────────────────────────────────────────────┐ │   │
│  │  │ ocr_search 表                                            │ │   │
│  │  │  - 全文检索 trigram 索引                               │ │   │
│  │  └────────────────────────────────────────────────────────┘ │   │
│  └──────────────────────────────────────────────────────────────────┘   │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

---

## 九、技术总结

### 9.1 关键技术栈

| 技术组件 | 选型 | 说明 |
|----------|------|------|
| **向量模型** | OpenCLIP / mCLIP / NLLB | 多语言支持，modelName 配置驱动维度 |
| **向量数据库** | pgvector / vchord | PostgreSQL 原生扩展 |
| **索引算法** | HNSW | 对数级查询复杂度，高召回率 |
| **距离度量** | 余弦相似度 | 归一化向量标准度量 |
| **运行时** | ONNX Runtime | 跨平台高性能推理 |
| **全文检索** | pg_trgm | PostgreSQL trigram 索引 |
| **高可用** | 多 ML 服务器 + 健康检查 | 故障自动转移 |

### 9.2 架构设计要点

1. **无权重融合设计**：向量相似度与关键词检索完全独立，避免复杂调参负担
2. **配置驱动模型**：modelName 决定维度，支持模型更换无需代码修改
3. **过滤后置策略**：元数据/OCR 作为 WHERE 过滤不参与排序，性能优先
4. **LRU 缓存热点查询**：避免重复编码相同文本
5. **ML 服务层抽象**：支持多实例负载均衡与故障转移

---

## 参考文件路径

- 搜索服务: `server/src/services/search.service.ts`
- 搜索仓库: `server/src/repositories/search.repository.ts`
- ML 仓库: `server/src/repositories/machine-learning.repository.ts`
- CLIP 文本编码器: `machine-learning/immich_ml/models/clip/textual.py`
- CLIP 图像编码器: `machine-learning/immich_ml/models/clip/visual.py`
- 数据库 Schema: `server/src/schema/tables/smart-search.table.ts`
- 查询构建器: `server/src/utils/database.ts`
- 模型配置 DTO: `server/src/dtos/model-config.dto.ts`
- 枚举定义: `server/src/enum.ts`
