# 按数据形态与查询模式选择检索策略

> 学习序号 10 · 清单 #10 · Domain 3: Integration（19%）。核查日期：2026-09-16。
> 考试依据：[Exam_Guide.md](../../../Exam_Guide.md) 第 6 节原文 “Apply retrieval strategies matched to data shape and query pattern”。本课承接 [08 Chunking](../08_rag-chunking-strategies/knowledge.md) 与 [09 Index / Vector Store](../09_rag-index-and-vector-store-strategies/knowledge.md)，聚焦在线 retrieval policy：该查什么、用哪种检索器、怎样合并与重排。考试重点是根据场景约束选择策略，而不是默认所有数据都做 vector search。

## 1. Retrieval 不是“对向量库做一次 top-K”

一个可控的 retrieval pipeline 通常分为四层：

```text
user query + trusted identity
  → query analysis / routing / filters
  → candidate generation
       lexical | semantic | structured query | graph | hybrid
  → fusion + deduplication + reranking
  → evidence sufficiency / authorization / context budget checks
  → selected evidence + provenance → Claude
```

- **Candidate generation** 追求高 recall：需要的证据不要过早丢失。
- **Fusion / reranking** 追求高 precision：把最能回答问题的证据排到 context 前部。
- **Context assembly** 受 token budget、证据完整性、引用和权限约束。
- **Generation** 只能利用被检索到的证据；Claude 无法可靠修复 retrieval 已经漏掉的关键事实。

选择策略前，先识别两个维度：

1. **数据形态**：自然语言、代码、日志、表格/交易数据、关系、时间序列、图片/OCR，还是多种混合；
2. **查询模式**：精确定位、概念/意图、带过滤条件、聚合计算、关系遍历、跨文档综合，还是需要最新状态。

## 2. Lexical retrieval：精确词形是信号时优先

Lexical retrieval 使用 term、倒排索引、BM25 等方式匹配 query 与文档中的词。它特别适合：

- SKU、错误码、ticket ID、类名、API endpoint、法规编号；
- 人名、产品名、缩写和领域稀有词；
- 用户粘贴的一段报错或原文短语；
- 必须区分 `ERR_AUTH_41` 与 `ERR_AUTH_14` 的场景。

优势是 exact token 信号强、可解释、通常不需要 query embedding。局限是同义词、释义、跨语言和用户表达变化会造成 vocabulary mismatch；分词器、stemming、stop words 对代码、CJK、多语言或带符号标识符也可能处理不当。

因此“关键词搜索落后、semantic 一定更好”是错误结论。若答案取决于一个罕见标识符，lexical 往往是最直接的检索器。可以通过字段级权重、phrase match、同义词表与语言适配增强它，但这些规则也需版本化和评估。

## 3. Semantic retrieval：含义一致而措辞不同时优先

Semantic / dense retrieval 把 query 与文档编码到兼容的 embedding 空间，通过相似度寻找语义接近的内容。它特别适合：

- FAQ 中用户问题与文档措辞不同；
- 概念、意图、主题或自然语言描述；
- “员工离职后多久删除账户”与“offboarding access revocation period”一类释义匹配；
- 文档中存在大量同义表达，而用户不知道正式术语。

它的局限包括：

- 相近语义不代表包含所需精确事实；
- 型号、数字、否定词、版本或相似 identifier 可能区分不足；
- embedding 对语言、领域和 query 长度的表现必须用真实数据验证；
- similarity score 不是“答案正确概率”，也不能跨模型或检索器直接比较。

Semantic retrieval 仍需 metadata/ACL filter、版本控制和 provenance。它解决的是匹配方式，不会自动解决 freshness、authorization 或结构化计算。

## 4. Hybrid retrieval：互补召回，不是无条件默认

Hybrid retrieval 通常并行运行 lexical 与 semantic search，合并候选后再排序：

```text
query
 ├─ lexical top-N  ─┐
 └─ semantic top-N ─┼→ union → fusion → optional reranker → final top-K
                    ┘
```

它适合 query 分布同时包含精确标识符与自然语言意图，或同一 query 同时需要两类信号。例如：“`ERR-4297` 为什么在批量退款时发生？”错误码需要 lexical，原因描述可能需要 semantic。

Anthropic 的 Contextual Retrieval 研究组合了 Contextual Embeddings 与 Contextual BM25，并进一步测试 reranking；其特定实验显示组合能减少 retrieval failure。[Contextual Retrieval](https://www.anthropic.com/engineering/contextual-retrieval) 官方 Cookbook 也提供了从 baseline、contextual embeddings、BM25 hybrid 到 reranking 的示例。[Contextual Retrieval Cookbook](https://platform.claude.com/cookbook/capabilities-contextual-embeddings-guide) 这些结果基于特定数据集与实现，证明“值得评估”，不证明 hybrid 对每个 corpus 都必然获胜。

Hybrid 的代价是两套索引/查询路径、更多候选、fusion 调参与更高延迟。若 corpus 很小、query 只有精确 ID，或 structured query 已能确定性回答，引入 hybrid 可能只增加复杂度。

### Union，而不是盲目 intersection

Candidate stage 通常取 lexical 与 semantic 结果的 **union**，保留互补召回；若要求文档同时出现在两者 top-N 中，恰好只被一种方法找到的关键证据会被丢弃。之后再通过 fusion/reranking提升 precision。

## 5. Fusion：不同 score 不能直接裸加

Lexical score 与 vector similarity 的尺度、分布和含义不同，直接执行 `bm25_score + cosine_score` 通常没有稳健意义。常见 fusion 方法有：

- **Rank-based fusion（如 RRF）**：依据每个结果在各列表中的名次合并，不依赖原始 score 可比。优点是稳健、起步简单；缺点是丢失 score margin，并有 rank constant、候选窗口等参数。
- **Score normalization + weighted fusion**：先把各路 score 校准/归一化，再按权重组合。可表达某一路信号更重要，但对 score 分布、数据漂移和 query slice 敏感。
- **Learned ranking**：用带 judgment 的数据学习组合特征。潜在质量高，但训练、特征治理、解释与 drift 运维成本更高。

OpenSearch 官方文档以一个具体实现说明两种选择：normalization processor 保留分数差异后组合；RRF 按排名组合、忽略原始分数。[OpenSearch Hybrid Search](https://docs.opensearch.org/latest/vector-search/ai-search/hybrid-search/index/) 这用于解释通用权衡，并不代表 Exam Guide 要求掌握 OpenSearch 配置或推荐该产品。

重要安全原则：trusted tenant、ACL、region、有效期等硬条件应在各 candidate generator 中 pre-filter，或以等价方式保证未授权内容不进入候选、reranker、cache 和 Claude。不能先把越权内容交给模型，再要求它“不要使用”。

## 6. Reranking：用较贵判断处理较小候选集

两阶段检索常采用：

1. 廉价检索器召回较大的 candidate set；
2. reranker 根据 query–document 对重新排序；
3. 只把最终 top-K 证据交给 Claude。

Reranker 可改善细粒度相关性和结果次序，尤其适合 hybrid 后的异构候选。但它不能找回 candidate stage 已漏掉的文档，所以先保证 recall，再优化 reranking。代价包括额外延迟、算力/API 成本、候选长度限制，以及敏感文本离开既有安全边界的风险。

设计时分别设置 `candidate N` 与 `final K`，不要把“rerank 100 条”和“给 Claude 100 条”混为一谈。延迟敏感时可仅对低置信度/复杂 query 启用 reranker，或使用较小候选集；是否值得必须通过分 query slice 的端到端评价决定。

## 7. Structured、graph 与专用检索器：不要把数据库变成 prose

并非所有问题都应靠文本相似度回答：

| 数据/查询 | 首选模式 | 原因 |
|---|---|---|
| SKU、错误码、法规编号、精确短语 | lexical / exact field lookup | 标识符本身就是关键约束 |
| FAQ、政策释义、用户意图 | semantic，必要时 hybrid | 重点是含义而非相同词形 |
| 当前订单状态、账户余额、库存 | 授权后的 API/SQL/key lookup | 需要最新、确定性的结构化事实 |
| “过去 30 天按地区汇总退款额” | SQL/analytics query | 这是过滤与聚合，不是相似度问题 |
| “谁依赖这个服务、下游影响是什么” | graph/relationship traversal | 关系与路径是一等数据 |
| 混合知识库，既有产品代码又有自然语言 | hybrid + filter，必要时 rerank | lexical 与 semantic 信号互补 |
| 小而稳定且能安全放入 context 的知识集 | 直接 context 也可能更简单 | 没必要为 RAG 引入额外 failure surface |

常见架构是 **query routing**：分类 query 后调用适合的数据源，再将有 provenance 的结果交给 Claude。例如先从可信参数解析出 `customer_id`，通过授权 API 查询实时余额；同时从知识库检索余额规则。不能用“最相似的旧账单段落”代替实时余额。

Router 本身会误分类。应设计低置信度 fallback、允许多路并行、记录 route，并对 route accuracy 与最终任务成功率评估。高风险结构化查询要验证参数、使用 allowlist/read-only access，并保持最小权限。

## 8. Query transformation：修复表达，但可能改变意图

常见变换包括：

- normalization：大小写、空白、拼写、identifier 提取；
- rewrite：把对话中的“它”补成明确对象，保留原 query；
- expansion / multi-query：生成同义表达，提高 recall；
- decomposition：把多跳问题拆成多个可检索子问题，再组合证据；
- metadata extraction：提取时间、产品、语言或版本作为 filter。

变换会增加延迟和查询数量，也可能引入不存在的实体、删除否定或把用户意图改错。最佳实践是保留原 query，必要时同时检索原始与改写版本，限制允许推断的 filter，并在 sensitive 场景对实体/条件做验证。不要把 LLM rewrite 的 tenant 或权限字段视为可信身份。

## 9. Evidence sufficiency 与 abstention

取到 top-K 不等于有答案。系统要区分：

- 没有结果；
- 有相似结果，但不包含回答所需事实；
- 证据冲突或版本不一致；
- 证据存在但用户无权访问；
- query 需要结构化计算而当前只做了文本检索。

应定义基于评估校准的 evidence threshold / coverage check，并让 Claude 在证据不足时澄清或拒答，而不是凭常识补齐。不要把单个未经校准的 similarity cutoff 当成普适置信度；阈值应按 query 类型、语言和数据域评估。

## 10. 如何评价策略是否匹配

建立代表真实分布的 query set，并标注 supporting evidence 或 relevance judgments。至少按 query slice 分析：exact identifier、paraphrase、filter-heavy、multi-hop、多语言、freshness、no-answer 与 adversarial/ACL。

关注：

- candidate recall@N：关键证据是否被任一路召回；
- final precision/recall、MRR/nDCG 或 pass@K；
- reranker 前后关键证据排名变化；
- grounded answer correctness、citation accuracy、abstention；
- p95/p99 latency、每查询成本、返回 token；
- 未授权结果率必须为零；
- route accuracy，以及按 strategy/query slice 的 failure rate。

只看总体平均值可能掩盖严重退化：hybrid 总体提高，但 exact SKU query 变差；英文提高，但中文下降；普通查询提高，但 ACL 高选择性场景漏召回。OpenSearch 官方也明确指出 hybrid 的最佳配置依赖数据、用户行为和领域，没有 one-size-fits-all，应使用 query set 与 judgments 做实验。[Optimizing Hybrid Search](https://docs.opensearch.org/latest/search-plugins/search-relevance/optimize-hybrid-search/)

## 11. 典型 failure modes

- 所有 query 都只做 vector search，导致错误码、SKU 和版本号错配；
- 所有 query 都只做 BM25，用户换一种说法就召回失败；
- 取两路结果 intersection，丢掉仅被一路命中的关键证据；
- 将不同检索器的 raw score 直接相加；
- reranker candidate 太小，误以为它能找回未召回文档；
- 用 RAG 文本回答实时余额、库存或聚合问题；
- query rewrite 改掉否定词、实体或时间范围；
- ACL 仅在最终 top-K 后过滤，敏感内容已进入 reranker/cache；
- top-K 固定且无 sufficiency check，没证据也强行生成；
- 只看 aggregate retrieval score，不检查关键 query slices 和端到端答案。

## 12. 考试决策顺序

1. 识别数据形态、query intent、freshness、filter/权限与 SLA。
2. 精确词形/ID 用 lexical 或 deterministic lookup；释义/概念用 semantic；两者混合才考虑 hybrid。
3. 结构化事实、聚合和关系查询使用相应 API/SQL/graph，不强行向量化。
4. Hybrid 先做互补 candidate union，再用有依据的 fusion；不同 raw score 不直接相加。
5. 需要更高 precision 且预算允许时 rerank，但 candidate recall 是前提。
6. 强制 trusted pre-filter、provenance、freshness 与 evidence sufficiency。
7. 用真实 query slices 同时评价 retrieval、端到端正确性、安全、延迟与成本。

## 13. 范围、来源与待核查

`list.md` 将 Guide 的 RAG 相关目标拆为 chunking、indexing 与 retrieval strategy。本课 #10 直接覆盖 Guide 的独立 retrieval objective，粒度合理；没有把某个搜索产品或 Anthropic 实验参数扩大为考试要求，也未提前完成 #11 progressive discovery。

资料核查于 2026-09-16：[Anthropic Contextual Retrieval](https://www.anthropic.com/engineering/contextual-retrieval)、[Anthropic Contextual Retrieval Cookbook](https://platform.claude.com/cookbook/capabilities-contextual-embeddings-guide)、[OpenSearch Hybrid Search](https://docs.opensearch.org/latest/vector-search/ai-search/hybrid-search/index/)与 [Optimizing Hybrid Search](https://docs.opensearch.org/latest/search-plugins/search-relevance/optimize-hybrid-search/)。OpenSearch 仅作为官方技术实现例证，不代表 Anthropic 背书或考试指定技术。

**Needs verification**：无影响核心结论或练习答案的未核实事实。实施前须重新核查所选 lexical/vector engine 的 analyzer、score/fusion、filter、reranker、模型与 API 语义、数据驻留、限制和价格；Anthropic 2024 实验与 Cookbook 中的具体数据、模型、效果数字和延迟不应作为当前生产承诺。

下一步：[questions.md](questions.md)。
