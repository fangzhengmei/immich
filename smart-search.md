# Immich 智能搜索技术架构分析

## 概述

Immich 智能搜索功能允许用户通过自然语言描述搜索照片。核心采用 CLIP 模型将文本/图像编码为向量，通过 vchordrq/pgvector 向量索引进行相似度检索，并与元数据过滤、OCR 全文检索组合工作。**不存在向量与关键词的权重融合公式**，两者是独立查询链路。

---

## 一、向量模型与维度配置

### 1.1 模型架构

Immich 使用 OpenCLIP 双编码器架构：

- **文本编码器 (Textual Encoder)**：将自然语言文本编码为向量
- **图像编码器 (Visual Encoder)**：将图像内容编码为向量
- **同构向量空间**：文本与图像向量共享同一空间，可直接计算余弦相似度

### 1.2 模型配置变更与维度判定完整链路

#### 配置驱动流程

**当前实现 (硬编码 + 运行时可调)**：

```
1. 管理员配置 modelName
   └─ 系统配置项: machineLearning.clip.modelName (无默认维度校验)

2. 维度判定与数据库同步
   ├─ 2.1 获取当前维度: getDimensionSize(table, column)
   │    └─ 读取 pg_attribute.atttypmod，无有效值则默认 512
   │
   ├─ 2.2 设置新维度: setDimensionSize(dimSize)
   │    ├─ 事务1: truncate smart_search → 删维度约束(若有) → 加新维度约束
   │    ├─ 事务2: DROP INDEX clip_index → ALTER COLUMN embedding TYPE vector(dimSize)
   │    ├─          重建向量索引 (vchordrq / hnsw)
   │    ├─          删除临时维度约束
   │    └─ 重置 probes = 1
   │
   └─ 2.3 向量数据清理（当前实现）
        └─ delete from smart_search 删除所有向量，无自动触发全量重编码（TODO）
           真正 truncate 操作发生在 deleteAllSearchEmbeddings() 路径
           手动触发入口：JobName.SmartSearchQueueAll 任务队列

3. 索引参数动态计算 (vchord 专属)
   ├─ targetListCount(rowCount): 根据行数计算 lists 值
   │   └─ <128k:1, <2M:按1000取上靠近2幂, >2M:按行数平方根算
   └─ targetProbeCount(lists): probes = ceil(lists / 8)
```

#### 能力边界说明

| 能力项 | 当前实现状态 | 代码位置 |
|--------|-------------|---------|
| **运行时维度变更** | ✅ 已实现 | `database.repository.ts:301-330` |
| **多模型自动维度检测** | ❌ 未实现，需手动调用 `setDimensionSize` | - |
| **维度变更渐进式迁移** | ❌ 全表清空重建，无增量迁移 | - |
| **不同资产混合维度** | ❌ 单表统一列维度约束 | - |
| **pgvector 索引参数配置** | ✅ 建索引时支持 `ef_construction` (migration 中写死 300) | `vectorIndexQuery()` |
| **vchord probes 动态调整** | ✅ 根据 lists 自动计算 `targetProbeCount` | `database.repository.ts:346-348` |
| **查询时 probes 会话级设置** | ✅ `SET LOCAL vchordrq.probes = N` | `search.repository.ts:286` |

### 1.3 文本编码流程

**文件位置**：`machine-learning/immich_ml/models/clip/textual.py`

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

**文件位置**：`machine-learning/immich_ml/models/clip/visual.py`

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

**文件位置**：`server/src/schema/tables/smart-search.table.ts`

```typescript
@Table({ name: 'smart_search' })
@Index({
  name: 'clip_index',
  using: 'hnsw',  // pgvector 兼容写法，实际 vchord 使用 vchordrq
  expression: `embedding vector_cosine_ops`,  // 余弦相似度算子
  with: `ef_construction = 300, m = 16`,  // 索引构建参数
  synchronize: false,
})
export class SmartSearchTable {
  @ForeignKeyColumn(() => AssetTable, { onDelete: 'CASCADE', primaryKey: true })
  assetId!: string;

  @Column({ type: 'vector', length: 512, storage: 'external', synchronize: false })
  embedding!: string;  // 向量维度当前为 512，支持通过 setDimensionSize 变更
}
```

### 2.2 vchordrq 索引参数准确说明

| 参数 | 值/来源 | 说明 | 代码位置 |
|------|---------|------|---------|
| `m` | 16 | 每个节点在构建时的邻居数量，控制索引大小和精度 | migration 写死 |
| `ef_construction` | 300 | pgvector 构建时动态邻居列表大小，vchord 用 lists 替代 | pgvector 仅 |
| `lists` | 动态计算 | vchord 聚类中心数，由行数决定 targetListCount() | `database.repository.ts:336-344` |
| `vchordrq.probes` | 动态/会话级 | 查询时探测聚类数，`probes = ceil(lists / 8)` | `search.repository.ts:286` |
| 距离度量 | `vector_cosine_ops` | 余弦相似度 (距离 = 1 - cosθ) | - |

**重要修正**：Immich 当前实现**未使用 pgvector 的 `ef_search` 参数**，仅对 vchord 扩展设置 `vchordrq.probes`，值硬编码为 `1` 或根据 lists 动态计算。

### 2.3 向量相似度搜索实现

**文件位置**：`server/src/repositories/search.repository.ts`

```typescript
searchSmart(pagination: SearchPaginationOptions, options: SmartSearchOptions) {
  return this.db.transaction().execute(async (trx) => {
    // 设置会话级 probes 参数 (仅 vchord 有效，pgvector 忽略此语句)
    await sql`set local vchordrq.probes = ${sql.lit(probes[VectorIndex.Clip])}`.execute(trx);
    
    const items = await searchAssetBuilder(trx, options)
      .selectAll('asset')
      .innerJoin('smart_search', 'asset.id', 'smart_search.assetId')
      // 纯余弦相似度距离排序 (仅按向量距离，无权重融合)
      .orderBy(sql`smart_search.embedding <=> ${options.embedding}`)
      .limit(pagination.size + 1)
      .offset((pagination.page - 1) * pagination.size)
      .execute();
      
    return paginationHelper(items, pagination.size);
  });
}
```

### 2.4 向量扩展支持

Immich 支持两种 PostgreSQL 向量扩展，自动检测并优先使用 vchord：

| 扩展 | 索引类型 | 特有参数 | 说明 |
|------|---------|---------|------|
| `vchord` (VectorChord) | vchordrq | `lists`, `probes` | pgvector 高性能分支，支持残差量化 |
| `vector` (pgvector) | hnsw | `m`, `ef_construction` | 官方标准向量扩展 |

**文件位置**：`server/src/repositories/database.repository.ts`

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

**关键架构要点**：

1. **智能搜索 = 纯向量相似度排序**：按 `embedding <=> query_embedding` 余弦距离升序排列，无任何关键词权重融合
2. **关键词检索 = 独立链路**：通过 OCR 全文检索走 PostgreSQL trigram 索引，使用 `%>>` 运算符
3. **元数据过滤 = SQL WHERE 条件**：日期、地点、相机、人物、标签等作为过滤条件，不参与排序

### 3.2 智能搜索核心实现

**文件位置**：`server/src/services/search.service.ts`

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
    
    // 3. 查询向量化 (两种输入模式)
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

---

## 四、混合检索机制深度解析

### 4.1 不存在权重融合公式

**明确结论：Immich 智能搜索不涉及向量与关键词的权重融合**。两者是完全独立的检索链路：

| 检索类型 | 排序依据 | 实现方式 |
|----------|----------|----------|
| **向量检索** | 纯余弦相似度距离 | HNSW/vchordrq 索引，`<=>` 运算符 |
| **OCR 关键词检索** | trigram 文本相似度 | PostgreSQL `%>>` 全文检索运算符 |
| **元数据过滤** | 不参与排序，仅过滤 | SQL WHERE 条件 |

### 4.2 API 端点与 DTO 字段映射

| API 端点 | HTTP | DTO 类型 | 特有字段 | 共享过滤字段 |
|----------|------|---------|---------|-------------|
| `/search/smart` | POST | `SmartSearchDto` | `query`, `queryAssetId`, `language` | BaseSearchSchema 全部字段 |
| `/search/metadata` | POST | `MetadataSearchDto` | `id`, `checksum`, `description`, `originalFileName`, `originalPath`, `previewPath`, `thumbnailPath`, `encodedVideoPath`, `order` | BaseSearchSchema 全部字段 |
| `/search/statistics` | POST | `StatisticsSearchDto` | `description` | BaseSearchSchema 全部字段 |
| `/search/random` | POST | `RandomSearchDto` | `withStacked`, `withPeople` | BaseSearchWithResultsSchema |
| `/search/large-assets` | POST | `LargeAssetSearchDto` | `minFileSize` | BaseSearchWithResultsSchema |

### 4.3 关键字段查询实现映射

| 前端输入字段 | DTO 字段 | SQL 实现 | 代码位置 |
|-------------|---------|---------|---------|
| **语义搜索查询** | `SmartSearchDto.query` | `embedding <=> query_vector` 排序 | `search.repository.ts:290` |
| **OCR 文本搜索** | `*.ocr` (共享字段) | `f_unaccent(ocr_search.text) %>> f_unaccent(tokenize(...))` | `database.ts:457-461` |
| **描述搜索** | `MetadataSearchDto.description` | `f_unaccent(asset_exif.description) ILIKE '%...%'` | `database.ts:452-456` |
| **文件名搜索** | `MetadataSearchDto.originalFileName` | `f_unaccent(asset.originalFileName) ILIKE` | `database.ts:445-451` |
| **文件路径搜索** | `MetadataSearchDto.originalPath` | `f_unaccent(asset.originalPath) ILIKE` | `database.ts:442-444` |
| **地点/相机过滤** | `*.city/state/country/make/model/lensModel` | `asset_exif.X = value` (INNER JOIN) | `database.ts:393-422` |

### 4.4 SmartSearchOptions 过滤条件组合

**文件位置**：`server/src/repositories/search.repository.ts`

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
  SearchOcrOptions;          // OCR 文本过滤 (独立全文检索 INNER JOIN)
```

---

## 五、端到端时序流程

### 5.1 完整时序图

```
用户前端                          Immich Server                          ML Service                          PostgreSQL
  │                                 │                                     │                                 │
  ├─ 输入自然语言查询 ─────────────▶│                                     │                                 │
  │ (POST /search/smart)            │                                     │                                 │
  │                                 │ 1. 权限验证: Permission.AssetRead    │                                 │
  │                                 │                                     │                                 │
  │                                 │ 2. 检查智能搜索启用配置               │                                 │
  │                                 │    machineLearning.enabled           │                                 │
  │                                 │                                     │                                 │
  │                                 │ 3. LRU 缓存检查                      │                                 │
  │                                 │    key = modelName + query + lang    │                                 │
  │                                 │    ├─ 命中 → 直接使用缓存向量         │                                 │
  │                                 │    └─ 未命中                         │                                 │
  │                                 │        └─ encodeText 请求 ───────────▶│                                 │
  │                                 │                                     │ 4. CLIP 文本编码器推理          │
  │                                 │◀────────────────────────────────────┤ 5. 返回 JSON 序列化向量        │
  │                                 │ 6. 写入 LRU 缓存                     │                                 │
  │                                 │                                     │                                 │
  │                                 │ 7. 获取用户可访问范围                 │                                 │
  │                                 │    (含伙伴共享 userIds)              │                                 │
  │                                 │                                     │                                 │
  │                                 │ 8. searchAssetBuilder 组装查询        │                                 │
  │                                 │    ├─ 权限 ownerId IN (...)          │                                 │
  │                                 │    ├─ 日期范围/taken/created          │                                 │
  │                                 │    ├─ EXIF 地点/相机 JOIN            │                                 │
  │                                 │    ├─ 人物/标签 INNER JOIN           │                                 │
  │                                 │    ├─ OCR 文本过滤 (如有) JOIN       │                                 │
  │                                 │    └─ visibility/isFavorite 等状态    │                                 │
  │                                 │                                     │                                 │
  │                                 │ 9. 开启事务                          │                                 │
  │                                 │    └─ SET LOCAL vchordrq.probes = 1──▶│                                 │
  │                                 │                                     │                                 │
  │                                 │ 10. 执行 SQL 查询 ───────────────────▶│                                 │
  │                                 │                                     │ 11. HNSW/vchordrq 索引扫描     │
  │                                 │                                     │     按余弦距离排序 + 过滤条件    │
  │                                 │◀─────────────────────────────────────┤ 12. 分页结果集                 │
  │                                 │ 13. 提交事务                         │                                 │
  │◀────────────────────────────────┤ 14. 返回按相似度排序的资产列表        │
  ▼                                 ▼                                     ▼                                 ▼
```

### 5.2 关键时序步骤详解

| 步骤 | 说明 | 关键代码位置 |
|------|------|------------|
| **1-2** | 前端发起 `/search/smart`，验证用户身份与智能搜索启用状态 | `search.controller.ts:86-87` |
| **3** | LRU 缓存键 = modelName + query + language，容量 100 | `search.service.ts:29` |
| **4-5** | ML 服务负载均衡，健康检查机制，故障自动转移 | `machine-learning.repository.ts:164` |
| **6** | 向量序列化格式：JSON 数字数组字符串 | `visual.py:32` |
| **7** | `getUserIdsToSearch` 包含用户 + 伙伴共享 ID | `search.service.ts:189-196` |
| **8** | `searchAssetBuilder` 组装所有过滤条件 | `database.ts:371-486` |
| **9** | 事务内会话级设置 probes，pgvector 安全忽略此语句 | `search.repository.ts:286` |
| **10-12** | PostgreSQL 执行计划：向量索引扫描 → 结果集过滤 | `search.repository.ts:288-295` |
| **13** | 排序仅按向量余弦距离升序，无其他排序因子 | `search.repository.ts:290` |

---

## 六、后台任务与索引构建

### 6.1 任务队列系统

**文件位置**：`server/src/enum.ts`

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

1. **vchord 参数调优**：
   - `lists`：根据行数动态计算，聚类中心数量
   - `probes = ceil(lists / 8)`：查询时探测聚类数
   - 索引变更自动触发 `reindexVectorsIfNeeded()`

2. **pgvector 参数调优**：
   - `m = 16`：平衡索引大小和查询精度
   - `ef_construction = 300`：较高构建精度
   - **注**：Immich 未暴露 `ef_search` 查询参数

3. **向量存储优化**：
   - `storage = 'external'`：使用 PostgreSQL TOAST 外部存储，避免行内膨胀

### 7.3 查询优化

```typescript
// 事务内会话级设置 probes 参数 (仅 vchord 生效)
await sql`set local vchordrq.probes = ${sql.lit(probes[VectorIndex.Clip])}`.execute(trx);
// probes 硬编码默认值 = 1，vchord 下根据 lists 动态调整
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
│  │  (Web/Mobile)│  - 过滤条件选择                                       │
│  │              │  - OCR 关键词搜索                                      │
│  └──────┬───────┘                                                        │
│         │                                                                  │
│  ┌──────▼───────────────────────────────────────────────────┐             │
│  │                  Search Controller                         │             │
│  │  ┌─────────────────┐  ┌───────────────────────────────┐  │             │
│  │  │ POST /smart     │  │  POST /metadata               │  │             │
│  │  │ SmartSearchDto  │  │  MetadataSearchDto            │  │             │
│  │  └─────────────────┘  └───────────────────────────────┘  │             │
│  └────────────────────┬─────────────────────────────────────┘             │
│                       │                                                     │
│  ┌────────────────────▼────────────────────────────┐                        │
│  │              Search Service                      │                        │
│  │  ┌──────────────────────────────────────────┐   │                        │
│  │  │ LRU Cache: modelName + query + lang      │   │                        │
│  │  │ Capacity: 100                            │   │                        │
│  │  └──────────────────────────────────────────┘   │                        │
│  │  ┌─────────────────┐  ┌───────────────────────┐   │                        │
│  │  │ searchSmart()   │  │  getUserIdsToSearch() │   │                        │
│  │  └─────────────────┘  └───────────────────────┘   │                        │
│  └────────────────────┬────────────────────────────┘                        │
│                       │                                                     │
│  ┌────────────────────▼────────────────────────────────────┐                 │
│  │           MachineLearning Repository                     │                 │
│  │  (多服务器负载均衡 + 健康检查 + 故障转移 + encodeText)   │                 │
│  └────────────────────┬────────────────────────────────────┘                 │
│                       │                                                     │
│  ┌────────────────────▼─────────┐                                           │
│  │   ML Service (Python ONNX)    │                                           │
│  │  ┌──────────────────────────┐│                                           │
│  │  │ OpenClipTextualEncoder   ││                                           │
│  │  │ OpenClipVisualEncoder    ││                                           │
│  │  └──────────────────────────┘│                                           │
│  └──────────────────────────────┘                                           │
│                       │                                                     │
│  ┌────────────────────▼──────────────────────────────────────────────────┐ │
│  │                       Search Repository (PostgreSQL)                      │ │
│  │  ┌──────────────────────────────────────────────────────────────────┐ │ │
│  │  │ smart_search 表                                                    │ │ │
│  │  │  - assetId (FK, PK)                                               │ │ │
│  │  │  - embedding vector(dimSize)                                      │ │ │
│  │  │  - 索引: clip_index USING vchordrq/hnsw (embedding vector_cosine_ops) │ │
│  │  └──────────────────────────────────────────────────────────────────┘ │ │
│  │  ┌──────────────────────────────────────────────────────────────────┐ │ │
│  │  │ ocr_search 表 (独立全文检索)                                       │ │ │
│  │  │  - trigram 索引，%>> 运算符                                        │ │ │
│  │  └──────────────────────────────────────────────────────────────────┘ │ │
│  └──────────────────────────────────────────────────────────────────────────┘ │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

---

## 九、技术总结

### 9.1 关键技术栈

| 技术组件 | 选型 | 说明 |
|----------|------|------|
| **向量模型** | OpenCLIP / mCLIP / NLLB | 多语言支持，modelName 配置驱动维度 |
| **向量数据库** | pgvector / vchord | PostgreSQL 原生扩展，自动检测优先 vchord |
| **索引算法** | HNSW / vchordrq | 对数级查询复杂度，高召回率 |
| **距离度量** | 余弦相似度 | 归一化向量标准度量 |
| **运行时** | ONNX Runtime | 跨平台高性能推理 |
| **全文检索** | pg_trgm | PostgreSQL trigram 索引，CJK 二元分词 |
| **高可用** | 多 ML 服务器 + 健康检查 | 故障自动转移，不健康服务器降级 |

### 9.2 架构设计要点

1. **无权重融合设计**：向量相似度与关键词检索完全独立，避免复杂调参负担
2. **配置驱动模型**：modelName 决定维度，支持模型更换无需代码修改，全量重建
3. **过滤后置策略**：元数据/OCR 作为 WHERE 过滤不参与排序，性能优先
4. **LRU 缓存热点查询**：避免重复编码相同文本，缓存键含模型名+语言
5. **ML 服务层抽象**：支持多实例负载均衡与故障转移，健康服务器优先
6. **vchord/pgvector 兼容层**：`vectorIndexQuery()` 统一索引创建语法

---

## 参考文件路径

- 搜索控制器：`server/src/controllers/search.controller.ts`
- 搜索服务：`server/src/services/search.service.ts`
- 搜索仓库：`server/src/repositories/search.repository.ts`
- 数据库仓库（索引/维度管理）：`server/src/repositories/database.repository.ts`
- ML 仓库：`server/src/repositories/machine-learning.repository.ts`
- CLIP 文本编码器：`machine-learning/immich_ml/models/clip/textual.py`
- CLIP 图像编码器：`machine-learning/immich_ml/models/clip/visual.py`
- 数据库 Schema：`server/src/schema/tables/smart-search.table.ts`
- 查询构建器：`server/src/utils/database.ts`
- 搜索 DTO：`server/src/dtos/search.dto.ts`
- 枚举定义：`server/src/enum.ts`
