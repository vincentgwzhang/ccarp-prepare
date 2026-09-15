# RAG 流水线设计：索引与向量库策略

> 学习序号 09 · 清单 #9 · Domain 3: Integration（19%）。核查日期：2026-09-15。
> 考试依据：[Exam_Guide.md](../../../Exam_Guide.md) 第 6 节原文 “Design a RAG pipeline with appropriate chunking and indexing strategies”，并参考第 8 节 Sample 3：文档刷新后出现 confident-but-wrong，应优先排查 retrieval/indexing、stale chunks 或 mismatched embeddings。本课承接 [08 Chunking](../08_rag-chunking-strategies/knowledge.md)，聚焦 index/vector store 的架构选择；semantic、keyword、hybrid retrieval 的查询策略留给清单 #10。

## 1. 先分清三个东西

- **Embedding**：把 query 或 chunk 转成数值向量，使语义相近的内容在向量空间中接近。
- **Index**：为某种检索方式建立的数据结构。向量 index 支持 nearest-neighbor search；lexical inverted index 支持关键词/BM25；metadata index 支持 tenant、ACL、时间、类型等过滤。
- **Vector store / database**：保存 vector、chunk、metadata，并提供写入、查询、过滤、复制、备份和运维能力的系统。它可能是专用向量数据库、带 vector 扩展的关系数据库，或同时支持 lexical/vector 的搜索平台。

“用了 vector database”并不代表 RAG 已设计完成。仍要确定 embedding contract、distance metric、exact/approximate index、过滤时机、更新一致性、租户隔离、版本迁移和质量验证。

```text
Source of truth
  → parse + chunk
  → enrich metadata / ACL
  → embed with versioned model + preprocessing
  → write chunk + vector + versions
  → build/refresh index
  → publish index version / alias

Query
  → authorize + normalize
  → query embedding with compatible contract
  → metadata/ACL filter + nearest-neighbor search
  → deduplicate/rerank
  → authorized evidence → Claude
```

## 2. Embedding contract：索引与查询必须在同一空间

向量不是可随意互换的通用坐标。一个生产索引应记录至少：

- embedding provider/model/version；
- vector dimension、数值类型与 normalization；
- query/document 是否有不同 embedding mode 或前缀；
- distance/similarity function（cosine、dot product、Euclidean 等）；
- chunking/parser/contextualization version；
- source document、chunk、内容 hash 与有效时间。

Query vector 必须使用与 indexed document vector 兼容的 embedding contract。换模型、dimension、preprocessing 或度量时，通常需要新 index / collection 并重新 embedding；不能把新旧 vector 混到同一空间后假设 score 仍可比较。

Anthropic 当前 Embeddings 文档说明 text embeddings 用于语义相似度，也明确 Anthropic **目前不提供自有 embedding model**，并建议按 dataset/domain、推理性能和 customization 评估供应商。[Claude Platform: Embeddings](https://platform.claude.com/docs/en/build-with-claude/embeddings) 这属于截至核查日的产品事实，不是 Guide 要求背诵的永久结论；具体供应商、型号、dimension 和价格都可能变化。

选择 embedding 时必须用自己的语言、领域术语、短/长 query 和 hard negatives 测试。公共 leaderboard 只能筛选候选，不能替代业务 recall 和端到端答案评价。

## 3. Exact search 与 ANN

### Exact nearest neighbor

Exact search 计算查询与所有候选 vector（或过滤后的候选集）的距离，能得到该度量下的真实 top-K。它概念简单、可作为 recall ground truth；当 corpus 小、过滤后候选很少、QPS 低或正确性优先时可能足够。代价通常随候选规模增长，难以在大规模高 QPS 下保持低尾延迟。

### Approximate nearest neighbor（ANN）

ANN 用索引减少需要比较的候选，以部分 recall 换取速度和规模能力。常见族包括 graph-based HNSW 与 inverted-file IVF；具体实现各异：

- HNSW 通常有较好的 speed–recall trade-off，但建索引更慢、内存占用更高，写入/删除和维护成本要验证。
- IVF 将 vector 分区/聚类，只探测部分候选区域；构建/内存可能更经济，但训练数据代表性和 probe 参数会影响 recall。

pgvector 的官方文档可作为一个具体实现例证：默认 exact search；增加 approximate index 会以 recall 换速度；其 HNSW 与 IVFFlat 在查询、构建和内存方面有不同权衡。[pgvector Indexing](https://github.com/pgvector/pgvector#indexing) 这些不是所有向量数据库都完全相同的行为，更不要求考试背参数默认值。

关键原则：**ANN 的“查询成功”不等于召回正确**。应在代表性 query 上将 ANN 结果与 exact ground truth 比较 recall@K，同时测 p95/p99、吞吐、内存、build/update time；不要只看平均 latency。

## 4. Metadata filtering 是正确性与安全边界

RAG 常需先限定 tenant、ACL、region、language、document type、valid-from/to、product/version。过滤不仅提升相关性，也可能是防止跨租户数据进入 Claude context 的强制安全控制。

需要问清 vector store 如何组合 filter 与 ANN：

- **Pre-filter**：先建立允许的候选集，再做向量检索；权限语义清晰，但极高选择性过滤可能需要专门 partition/index 策略。
- **Post-filter**：先从 ANN 取候选，再过滤；若 top-K 大量被剔除，最终可能少于 K 条或漏掉过滤集合内真正最近的结果。简单地“多取一些”只能缓解，不能在所有分布下保证 recall。
- **Integrated/iterative filtering**：检索过程中继续探索直到得到足够合规结果或达到预算；质量更好但延迟和实现复杂度增加。

pgvector 官方文档同样提醒 approximate index 与 filtering 组合可能返回不足结果，并给出 iterative scan、partial index、partition 等实现选择。[pgvector Filtering](https://github.com/pgvector/pgvector#filtering)

权限过滤必须来自可信身份/政策层，不能让模型自己生成 `tenant_id` 后再相信它。无权内容最好从候选阶段就排除；日志、cache 和 reranker 也要维持同一权限边界。

## 5. 如何选择存储形态

| 方案 | 适合 | 优势 | 主要代价/风险 |
|---|---|---|---|
| 关系数据库 + vector 能力 | 数据已在关系库、规模/QPS 可控、强 metadata join/事务重要 | 少一个系统；业务数据、ACL 和 vector 可共同管理 | ANN 能力、横向扩展或搜索特性可能不如专用系统；需实测 |
| 搜索引擎 + vector | lexical/hybrid、过滤、聚合和全文搜索都是核心 | 一套平台组合倒排与向量检索 | 集群调优和 schema/index lifecycle 较复杂 |
| 专用/托管 vector database | vector 规模与 QPS 高、需要托管扩展或成熟 ANN | 向量检索、分片、复制和运维能力集中 | 新供应商/系统、成本、数据驻留、lock-in 与一致性语义需评估 |
| 内存/本地 index | 原型、单机、只读或边缘部署 | 简单、低网络延迟 | durability、多副本、并发更新和治理通常需自建 |

选型维度应来自需求：corpus/vector 数量及增长率、dimension、QPS/并发、p95/p99、目标 recall、写入/刷新频率、filter 选择性、hybrid/rerank 支持、multi-tenancy、数据驻留/加密/备份、可用性、团队能力、迁移性和总成本。

不要因“已有 PostgreSQL”就必然共库，也不要因“AI 项目”就必然购买专用 vector DB。低规模、复杂 ACL join 的系统与十亿级只读向量服务，合理答案可能完全不同。

## 6. Multi-tenancy 与分片

共享一个 ANN index 可提高资源利用率，但不同 tenant 的数据量会互相影响性能、recall 与 noisy-neighbor 风险；错误过滤更会造成数据泄露。常见边界：

- shared collection + 强制 tenant filter：租户多且多数很小，成本低，但必须验证 filtered ANN recall 和访问控制；
- partition/namespace per tenant group：在共享基础设施上缩小候选和故障域；管理复杂度增加；
- separate collection/index/instance：高价值、大租户、监管隔离或定制 embedding；隔离强但成本和运维对象多。

Shard key 要匹配流量和过滤模式，避免按随机 chunk ID 分片后每次 query fan-out 到所有 shard。热点 tenant、跨 shard top-K 合并、replica lag、region routing 和 rebalancing 都应纳入延迟/一致性测试。

## 7. Freshness、更新和一致性

Source of truth 与 vector index 是派生数据。可靠 pipeline 应支持幂等 upsert、删除传播、失败重试、dead-letter/reconciliation，以及下面的版本字段：

```text
document_id, document_version, chunk_id, content_hash,
parser_version, chunker_version, embedding_version, index_version,
created_at, valid_from, valid_to, tenant_id, acl_tags
```

文档 refresh 后必须回答：旧 chunks 是否全部删除？新 index 是否完整？读流量何时切换？失败能否回滚？常见安全迁移是 **blue/green index**：离线构建新版本 → 完整性/抽样/recall/end-to-end eval → 原子切 alias/read pointer → 监控 → 延迟删除旧版。它占用双份资源但避免半新半旧。

逐条 in-place 更新成本较低、freshness 好，但在 parser/chunker/embedding 大版本迁移中可能产生混合空间、重复或空窗。若业务允许 eventual consistency，要明确定义 freshness SLO；“写入 source 成功”不等于“已可检索”。高风险撤回或权限变更应有比普通内容更新更严格的传播与验证机制。

Guide Sample 3 正是这类诊断：文档刷新后突然 confident-but-wrong，而模型和延迟未变，首先调查 stale chunks、漏删、失败 re-index、embedding mismatch 或 read alias 指向错误版本，而不是先怀疑 Claude 权重静默变化。

## 8. Index 质量与运行指标

离线与上线至少观测：

- retrieval recall@K、precision@K、MRR/nDCG（按任务选择）；
- ANN 相对 exact search 的 recall；
- zero-result/不足 K、ACL filter 后候选数、重复率；
- query p50/p95/p99、timeout、QPS、cache hit；
- ingestion lag、failed embeds、dimension mismatch、upsert/delete backlog；
- index build time、内存/磁盘、replica lag、shard skew；
- index/version 分组的 grounded answer correctness、citation 与单位成功任务成本。

评估集要覆盖领域术语、多语言、时间/版本过滤、小租户、高选择性 ACL、hard negatives 和文档更新。索引调参不能只在无 filter 的随机查询上完成。对安全过滤，除了质量测试还要做 negative authorization tests，确认未授权 chunk 永不进入 reranker、cache 或 Claude。

## 9. 容易混淆的概念

- **Vector store ≠ source of truth**：vector/index 可重建；原文、权限与版本应有权威来源。
- **Similarity score ≠ probability**：不同模型、度量、query 的 score 不必可直接比较；阈值要校准。
- **Embedding quality ≠ index quality**：好 embedding 也可能被低 recall ANN、错误 metric、过滤或 stale index 破坏。
- **Indexing ≠ retrieval policy**：index 提供可查结构；top-K、query rewrite、hybrid fusion 和 reranking 是在线策略，详见下一课。
- **Index version ≠ document version**：一次 index 可包含许多 document versions；两者都要记录。
- **Backup ≠ 可重建计划**：即使可从 source 重建，也要评估重建时间是否满足 RTO；备份也需验证恢复。

## 10. 典型 failure modes

- 用新 embedding model 生成 query，却查询旧模型建立的 vectors；
- dimension 相同便误认为 embedding 空间兼容；
- approximate index 调优只看 latency，不测 recall；
- post-filter 后不足 K，却把它误判为“知识库没有答案”；
- tenant/ACL 仅写在 prompt，没有在 retrieval 强制执行；
- re-index 失败后仍切换 alias，或旧 chunks 未删除造成冲突；
- 删除 source 后 vector、cache、lexical index 中仍有残留；
- 单一大租户形成热点，拖慢共享 shard；
- 只保存 vector 不保存 provenance/version，无法审计或重建；
- 把某个厂商默认 HNSW 参数当作所有数据的最佳配置。

## 11. 考试决策顺序

1. 明确数据规模、更新率、query/filter、隔离、安全、SLO 和团队约束。
2. 定义 versioned embedding/index contract，保证 query 与 document vector 兼容。
3. 小规模或高精度基线先考虑 exact；规模/延迟需要时选 ANN，并实测 recall–latency–resource 曲线。
4. 把 metadata/ACL filtering 当正确性和安全需求，验证与 ANN 的交互。
5. 根据 join/hybrid/规模/运维/合规决定关系库、搜索引擎或专用 vector DB。
6. 设计幂等 ingestion、删除传播、blue/green re-index、回滚与 freshness SLO。
7. 用 retrieval、端到端质量、安全和运行指标共同验收。

## 12. 范围、来源与待核查

`list.md` 将 Guide 的 RAG pipeline 目标拆为 chunking、indexing 与 retrieval strategy。本课 #9 的粒度合理，无需调整；只在必要处预告 #10，不提前完成该知识点。

资料核查于 2026-09-15：[Claude Platform Embeddings](https://platform.claude.com/docs/en/build-with-claude/embeddings)、[Anthropic Contextual Retrieval](https://www.anthropic.com/engineering/contextual-retrieval)、[pgvector Indexing / Filtering](https://github.com/pgvector/pgvector#indexing)。后者仅用于说明 exact/ANN/filter 的具体实现权衡，不代表 Anthropic 推荐该产品。

**Needs verification**：无影响核心结论或练习答案的未核实事实。实施前必须重新核查候选 embedding 与 vector store 的模型版本、dimension/metric、index/filter 语义、容量限制、数据驻留、安全认证、可用区/备份和价格；当前厂商型号与默认参数不作为考试答案依据。

下一步：[questions.md](questions.md)。
