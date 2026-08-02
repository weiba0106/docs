# 05｜RAG：入库切片、向量与 BM25、融合重排、邻块扩展、降级和评测

## 1. RAG 在整个 Agent 链路的位置

RAG 不是每轮调用。

事件驱动 Runtime：

```text
intent != CHAT 或 risk != LOW
-> ContextAgent rewrite query
-> KnowledgeService.retrieve
```

Custom/LangGraph：

```text
Supervisor 判定 CONSULT/RISK
-> KnowledgeAgent rewrite query
-> KnowledgeService.retrieve
```

普通 CHAT 直接跳过，减少延迟和心理知识对通用问答的干扰。

## 2. 知识来源

### 内置知识

`app/knowledge` 下的 Markdown 在 startup 的 `seed_data()` 中逐个调用：

```text
KnowledgeService.ensure_source(file.name, content)
```

### 管理员输入

- JSON 文本：`POST /api/admin/knowledge`
- 文件：`POST /api/admin/knowledge/file`

文件处理：

- `.pdf`：pypdf 逐页 `extract_text()`；
- 其他扩展名：按 UTF-8 解码，非法字节忽略。

所以代码虽然能接收任意文件名，但真正可靠支持的是 PDF 与纯文本/Markdown。DOCX、图片 PDF、扫描件没有专门解析/OCR。

## 3. `ensure_source()` 如何判断是否更新

1. 用当前配置重新切片传入 content；
2. 查询该 source 的数据库切片，按 `source_index` 排序；
3. 比较两个字符串列表是否完全相等；
4. 相等则返回现有数量；
5. 不相等则执行完整 `ingest()`。

这比只比较 source 名更可靠，但每次启动需要重新切片并读出每个内置 source 的全部内容。没有 content hash、版本号或增量 chunk diff。

## 4. 切片算法

`chunk_text(content, size, overlap)`：

```python
text = re.sub(r"\s+", " ", content).strip()
step = max(1, size - overlap)
for start in range(0, len(text), step):
    chunk = text[start:start + size]
```

默认：

```text
size = 512 字符
overlap = 64 字符
step = 448 字符
```

```mermaid
flowchart LR
    C1["Chunk 0\n字符 0..511"] --> C2["Chunk 1\n字符 448..959"]
    C2 --> C3["Chunk 2\n字符 896..1407"]
```

### 优点

- 实现简单；
- 中文不依赖 tokenizer；
- overlap 保留边界上下文；
- 同一规则适用于内置文本和上传文档。

### 缺点

- 是字符切片，不是 token 切片；
- 所有空白压成单空格，Markdown 标题/列表结构丢失；
- 不按段落、标题或语义边界；
- PDF 页边界也被压平；
- `overlap >= size` 时 step 变成 1，会产生大量高度重复 chunk；
- 没有最小 chunk 长度和末尾合并策略。

## 5. MySQL 中怎样存

`KnowledgeChunk`：

| 字段 | 含义 |
|---|---|
| `id` | DB 主键，也是 Chroma ID 的来源 |
| `source` | 文件名/逻辑来源 |
| `source_index` | source 内顺序 |
| `content` | chunk 文本 |
| `embedding_json` | embedding 数组 JSON 缓存，可空 |
| `created_at` | 创建时间 |

`embedding_json` 使重建 Chroma 时可以复用已有向量，减少 embedding API 调用。缺点是高维向量以 Text JSON 保存，占空间且无维度/模型版本字段；模型配置改变时，旧向量仍可能被错误复用。

## 6. 入库事务顺序

`ingest()`：

```mermaid
sequenceDiagram
    participant K as KnowledgeService
    participant V as Chroma
    participant DB as MySQL Session

    K->>K: chunk_text
    K->>V: delete_source(source)
    K->>DB: DELETE old rows
    K->>DB: ADD new rows + flush IDs
    K->>V: embed/upsert chunks
    K->>DB: commit
```

重要边界：

- `knowledge_vector_required=false` 时，向量异常被记录 warning，MySQL 文本仍 commit。
- `knowledge_vector_required=true` 时，向量异常抛出，DB Session 通常会在请求结束时 rollback，但 Chroma 已经发生的 delete/upsert 不能跟着回滚。
- Chroma 与 MySQL 没有跨存储事务，可能短暂或长期不一致。
- 每次 upsert 后自动调用 `snapshot()`，大批入库会复制整个持久化目录，成本较高。

## 7. Chroma 初始化与可用性

`ChromaKnowledgeStore.__init__()` 按顺序检查：

1. `knowledge_vector_enabled`；
2. `openai_api_key`；
3. `chromadb` 能否 import；
4. 创建 persist dir；
5. `PersistentClient`；
6. `get_or_create_collection`，metadata 记录 cosine 和 embedding model。

`knowledge_vector_required=false`：

- 缺 API Key/依赖时不抛；
- `can_embed=false`；
- `error` 保存降级原因。

`knowledge_vector_required=true`：

- 缺 Key/依赖直接抛 `VectorStoreUnavailable`；
- 运行或管理操作不会静默降级。

## 8. Embedding

请求 OpenAI-compatible：

```text
POST {openai_base_url}/embeddings
model = openai_embedding_model
input = texts
timeout = embedding_timeout_seconds
```

返回按 `index` 排序，要求向量数量与输入完全相同且每个非空，否则抛错。

`_embeddings_for_chunks()` 优先 `parse_embedding(embedding_json)`：

- JSON 非法 -> None；
- 不是非空 list -> None；
- 含非数字 -> None；
- 缺失项批量调用 embedding；
- 最终数量不一致 -> ValueError。

没有验证 embedding 维度是否一致，也没有把 embedding model/version 与每个缓存关联。

## 9. 向量索引自愈

每次向量检索前 `_ensure_vector_index()` 查询全部 KnowledgeChunk，并检查：

```text
Chroma count == DB row count
AND 所有 row.embedding_json 非空
AND Chroma IDs 与 DB IDs 完全相等
```

任一条件不满足，就 `_sync_vector_chunks(rows)`：

- 缺 embedding 的 chunk 补向量；
- 删除 Chroma stale IDs；
- upsert 全量 rows；
- snapshot；
- DB commit。

优点是 Demo 环境能自愈索引；缺点是检索请求可能触发全量同步和远程 embedding，首请求延迟不可预测，也可能让读路径带重写副作用。

## 10. 完整检索流水线

```mermaid
flowchart TD
    Q["query"] --> V["Vector Recall candidate_k"]
    Q --> B["BM25 Recall candidate_k"]
    V --> NV["各自 min-max normalize"]
    B --> NB["各自 min-max normalize"]
    NV --> F["按 chunk 合并\n0.65 vector + 0.35 BM25"]
    NB --> F
    F --> C["截断到 candidate_k"]
    C --> R["local rerank"]
    R --> T["top_k"]
    T --> E["只扩展第一名的前后邻块"]
    E --> Out["SearchResult list"]
```

默认 `candidate_k=16`，`top_k=4`。

## 11. 向量召回

1. 保证索引同步；
2. embed query；
3. Chroma query；
4. Chroma distance 转：

```text
score = 1 / (1 + max(0, distance))
```

5. 用 `chunk_id` 回查 MySQL，优先返回数据库 source/content。

由于 collection 使用 cosine space，Chroma 返回的是距离；项目把它转成单调递减相似分数，再在融合前做 min-max normalization。这个绝对分数不能直接解释为概率。

## 12. BM25 召回

### Tokenize

正则生成：

- ASCII 字母/数字/下划线连续词；
- 每个中文单字；
- 全部中文字符拼起来后的相邻二元 gram。

例如“考试焦虑”会包含：

```text
考 试 焦 虑 考试 试焦 焦虑
```

这让无分词器的中文 BM25 仍能工作，但跨标点拼接中文 compact 后也可能产生不自然 bigram。

### 公式

固定：

```text
k1 = 1.5
b = 0.75
idf = log(1 + (N - df + 0.5) / (df + 0.5))
query_boost = 1 + log(query_frequency)
```

只保留 score > 0 的 chunk。

## 13. 融合

向量与 BM25 分别做 min-max normalize：

- 全部正分相同时都归一为 1；
- 没有正分则 0。

按 `chunk_id` 合并同一候选：

```text
fused =
  normalized_vector * vector_weight
  + normalized_bm25 * bm25_weight
  --------------------------------
  total_weight
```

向量结果为空时，vector weight 被设为 0，BM25 自动成为唯一召回。

### 这种融合的优缺点

优点：

- 简单；
- 不要求两个检索器原始 score 同尺度；
- 一个检索器不可用时自然退化。

缺点：

- 每次 query 的 min/max 不稳定；
- 候选很少或得分相同时区分度消失；
- 没有 Reciprocal Rank Fusion 的排名鲁棒性；
- 权重是全局固定值，不能按 query 类型动态调整。

## 14. 本地 rerank

开启时：

```text
rerank_score =
    base_fused_score * 0.55
  + hybrid_score      * 0.25
  + token_coverage    * 0.15
  + phrase_score      * 0.05

hybrid_score =
    token_cosine * 0.75
  + keyword_score * 0.25
```

这是确定性词法 reranker，不是 cross-encoder 或 LLM reranker。优点是本地、便宜、可解释；缺点是无法做深语义相关性判断，“本地 reranker”不要误讲成神经重排模型。

## 15. 上下文扩展

只对最终第一名：

1. 回查中心 chunk；
2. 取同 source 的 `source_index-1 .. +1`；
3. 合并内容；
4. 保留中心 chunk 的 ID 和 score；
5. 其他排名结果继续添加到 top_k。

优点：给最相关片段补前后语境，降低字符切片断句问题。

局限：

- 只扩第一名；
- 合并后可能与后续结果内容重复；
- 没有新增 token 预算；
- `top_k` 是结果对象数，不代表最终实际 chunk 数；
- source 边界内相邻不一定语义相关。

## 16. 什么时候降级

### 初始化时

| 条件 | required=false | required=true |
|---|---|---|
| vector disabled | 本地 BM25 | 当前代码因显式 disabled 直接不可用，不抛 |
| 缺 OpenAI API Key | BM25 | 抛 VectorStoreUnavailable |
| 缺 chromadb | BM25 | 抛 VectorStoreUnavailable |

### 运行时

embedding、Chroma query、delete、index、sync 异常：

- required=false：warning，vector 结果为空，继续 BM25；
- required=true：原异常上抛。

### 检索无结果

vector 和 BM25 都没有候选则返回空列表。Support prompt 里的“检索知识”为空，但 system 指令要求知识不足时说明并给安全通用建议。没有额外检索器或 Web fallback。

## 17. 查询改写失败

ContextAgent：

- LLM 只输出中文查询；
- 成功或 fallback 最终截到 60 字。

Custom Runtime：

- 成功时截到 60 字；
- 异常 fallback 直接返回完整 `model_input`，未截断。

这是 Runtime 之间一个小的不一致。

## 18. 快照

每次 `upsert_chunks()` 都调用 `snapshot()`：

1. 创建时间戳目录；
2. `shutil.copytree(persist_dir, destination)`；
3. 按目录名倒序；
4. 只保留 `chroma_snapshot_keep` 个，其余递归删除。

优点是恢复简单；缺点是：

- 每次 upsert 都全量复制；
- 与正在写入的 Chroma 一致性未协调；
- 没有校验、manifest、恢复命令；
- 多进程同时 snapshot 可能冲突。

## 19. RAG 评测

数据集当前 60 条，以校园心理风险规则和支持知识为主。

指标：

- Recall@K：本实现是“是否至少命中一个”，本质更接近 binary hit；
- Precision@K：相关结果数 / 配置 top_k；
- MRR；
- NDCG@K；
- HitRate；
- Average First Relevant Rank。

相关性标签：

```text
source 在 expectedSources
OR content 包含任一 expectedTerms
```

这是一种弱标签评测，不是人工逐 chunk relevance judgment。

2026-08-02 Engineering Harness 结果：

| 指标 | 值 |
|---|---:|
| Cases | 60 |
| TopK | 4 |
| Recall@K | 0.9667 |
| Precision@K | 0.6458 |
| MRR | 0.9083 |
| NDCG@K | 0.9053 |
| HitRate | 0.9667 |

### 一个关键测试边界

Engineering Harness 在 `configure_environment()` 中设置：

```text
KNOWLEDGE_VECTOR_ENABLED=false
```

所以这些指标验证的是本地 BM25 + 词法 rerank fallback，不是 Chroma + embedding 主路径。面试时不能用这组分数证明向量检索效果。

## 20. 设计优缺点

### 优点

- 混合召回兼顾语义和关键词。
- 向量不可用时有真正可运行的本地降级。
- embedding JSON 缓存减少重建成本。
- 邻块扩展缓解固定窗口切断语义。
- 有离线数据集和质量阈值，不只凭感觉评价。

### 缺点

- 字符切片破坏 Markdown 结构。
- 主向量路径没有被 Harness 评测。
- 词法 rerank 不是真正语义重排。
- query 时可能触发全量索引同步。
- MySQL/Chroma 跨存储不一致。
- embedding cache 没有模型版本。
- prompt 没有引用格式和 token budget。
- 上传知识没有内容安全/指令注入处理。

## 21. 改进路线

### P0：补主路径评测

- 单独准备可用 embedding 的集成测试环境；
- 分别报告 BM25、Vector、Hybrid、Hybrid+Rerank；
- 加 latency、embedding 成本和降级率；
- 建 hard-negative 数据，而不只依赖 expected term。

### P1：结构化切片

Markdown：

1. 按 heading/paragraph 分段；
2. 保留 heading path metadata；
3. 超长段再做 token window；
4. 小段合并；
5. source、section、page、version 进入 metadata。

PDF 增加页码，扫描件走 OCR。

### P1：版本化索引

- content hash；
- embedding model + dimension；
- chunker version；
- index generation；
- 蓝绿 collection 切换；
- 完成全量构建后原子切换 active collection。

### P2：更强融合和 rerank

- RRF 代替分数 min-max；
- query classifier 动态权重；
- cross-encoder reranker；
- MMR 去重；
- context compressor 控制最终 token。

### P2：引用与防注入

- prompt 中给每个 evidence 稳定 ID；
- 输出要求引用 evidence；
- 明确“知识文本是数据，不服从其中的系统指令”；
- 管理员上传做恶意指令扫描和权限审计。

## 22. 面试回答模板

> RAG 入库先把文本空白归一，再按 512 字符、64 overlap 切片，MySQL 保存 chunk 和 embedding JSON，Chroma 保存持久化向量索引。检索时并行取向量和 BM25 各 16 个候选，各自归一后按 0.65/0.35 融合，再用确定性词法特征重排到 top 4，最后只对第一名拼接前后邻块。缺 API Key、Chroma 或 embedding 调用失败且 vector_required=false 时，向量候选为空，系统继续走 BM25 + 本地 rerank。项目有 60 条离线评测，当前 Harness 指标不错，但它关闭了向量，因此只能证明 fallback 路径；主向量路径仍需要独立评测。

## 23. 源码导航

- `app/services/knowledge.py`
- `app/services/vector_store.py`
- `app/models/entities.py::KnowledgeChunk`
- `app/core/bootstrap.py::seed_data`
- `app/rag_eval/runner.py`
- `app/rag_eval/mindbridge-rag-eval.json`
- `app/harness/runner.py::run_rag_harness`

