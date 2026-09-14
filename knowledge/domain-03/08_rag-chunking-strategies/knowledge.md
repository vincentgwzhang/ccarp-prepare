# RAG 流水线设计：分块（Chunking）策略

> 学习序号 08 · 清单 #8 · Domain 3: Integration（19%）。核查日期：2026-09-14。
> 考试依据：[Exam_Guide.md](../../../Exam_Guide.md) 第 6 节原文 “Design a RAG pipeline with appropriate chunking and indexing strategies”。本课只深入 chunking；index、vector store 与混合检索分别留给清单 #9、#10。考试重点是根据数据和查询约束作架构选择，而不是背一个固定 token 数。

## 1. Chunk 是检索与证据的基本单元

RAG 在离线阶段把文档转换为可检索单元；在线阶段根据 query 找到相关 chunk，将其作为证据交给 Claude。Chunking 同时决定：

- 检索器“看到”的语义单位；
- 一个命中结果携带多少必要上下文和多少噪声；
- index 条目数量、存储与预处理成本；
- top-K 结果占用多少 context budget；
- 引用能否精确指向支持结论的片段。

因此 chunking 不是单纯的字符串切片。理想 chunk 应尽量满足：**语义完整、能独立理解、边界稳定、保留来源与权限 metadata、大小适合检索模型和生成阶段预算**。这些目标互相冲突，需要用真实查询评估。

```text
source documents
  → parse / normalize / preserve layout
  → structure-aware boundaries
  → enforce size limits
  → optional overlap or contextual enrichment
  → attach provenance + version + ACL metadata
  → embed / index
  → retrieve → deduplicate / rerank → Claude
```

## 2. Chunk 太小与太大的权衡

### 较小 chunk

优点是主题更集中、相似度信号较纯、引用更精确；缺点是容易切断定义与例外、主语与指代、表头与数据行、函数与依赖，index 条目和重复 metadata 也更多。

### 较大 chunk

优点是保留更多局部上下文，跨段落问题可能一次命中即可回答；缺点是多个主题混在一起会稀释相关性，返回更多无关文字，占用生成 context，并降低引用粒度。

不存在通用“最佳 512/800/1000 tokens”。选择应依赖：文档结构、典型答案跨度、query 粒度、embedding/token 限制、top-K、reranking、Claude 输入预算和可接受成本。Anthropic 的 Contextual Retrieval 实验曾使用特定 chunk 配置，但那是 2024 年特定数据与组件下的实验，不是 Exam Guide 要求或普适默认值。[Introducing Contextual Retrieval](https://www.anthropic.com/engineering/contextual-retrieval)

一个实用起点是：先以天然语义边界切分，再对超长单元递归拆分；然后用评估集调大小和 overlap，而不是先选数字再强迫所有数据适配。

## 3. 常见策略与使用条件

| 策略 | 方法 | 适合 | 主要风险 |
|---|---|---|---|
| Fixed-size | 按字符/token 固定窗口切分 | 无结构文本的快速 baseline、大规模流式数据 | 在句子、条款、代码中间断开 |
| Recursive separators | 依次尝试章节、段落、句子、词等边界，直到满足上限 | 通用 prose，需兼顾结构与大小 | 规则依赖解析质量；语义边界仍可能不完整 |
| Structure-aware | 按标题、章节、条款、列表、表格、函数或类切分 | Markdown、HTML、法律文本、API 文档、代码 | 单元大小不均；超长章节仍需二次拆分 |
| Semantic | 根据相邻句段的语义变化决定边界 | 结构弱、主题转换明显的长文本 | 预处理更贵，边界受模型/阈值影响，结果需评估 |
| Parent–child / small-to-big | 检索小 child，返回或扩展到较大 parent | 需要精确匹配又需上下文完整的场景 | 去重和预算控制更复杂，parent 可能引入噪声 |
| Contextualized chunks | 为 chunk 添加来自整篇文档的简短、chunk-specific context 后再索引 | chunk 含“该公司”“本季度”等孤立指代 | 离线成本与版本管理增加；生成 context 可能错误，必须验证 |

策略可以组合。例如先按 Markdown heading 切 section，再递归切超长 section，并为每个 child 保存 heading path 与 parent ID。不要把“semantic chunking”当成必然优于结构规则；高度规范的法律条款或代码往往已有更可靠边界。

## 4. Overlap：修补边界，不是越多越好

相邻 chunk 适度重叠可提高跨边界事实被完整召回的概率，例如定义在上一段末尾、限制条件在下一段开头。但 overlap 会：

- 增加 index/storage 与 embedding 工作量；
- 让 top-K 返回多个近似重复 chunk，挤掉其他证据；
- 重复占用 Claude context，甚至让重复内容被错误地当作多份独立证据；
- 使评价与引用去重更复杂。

如果问题源于结构丢失，优先修复 parser、按标题/条款切分或使用 parent-child，而不是无限加 overlap。需要 overlap 时，记录 parent/chunk ID 和字符位置，在线去重或合并相邻命中，并通过数据决定窗口大小。

**Overlap 与 Contextual Retrieval 不同**：前者复制邻近原文；后者为每个 chunk 添加从整篇文档得到的定位说明。Anthropic 展示的典型问题是一个 chunk 只写“该公司的收入增长 3%”，却缺少公司和季度；其方法在索引前添加 chunk-specific context，使其可被正确检索。[Introducing Contextual Retrieval](https://www.anthropic.com/engineering/contextual-retrieval)

## 5. 按数据形态设计边界

### Prose / 知识库文章

保留标题层级，优先按 section/paragraph/sentence。将 `heading_path`、document title、version、language 作为 metadata；孤立代词或省略主语的段落需要父标题或 contextual enrichment。

### 法律、政策与操作手册

保持条款编号、定义、例外和适用范围。不要把 “except/unless/不得” 与主规则分开。可用小 child 检索具体条款，同时把包含定义和例外的 parent section 交给生成阶段。

### 代码与 API 文档

以函数、类、接口、端点或配置 section 为边界，保留 signature、所属模块、语言和版本。固定字符切分可能把函数体与名称、注释或 error contract 分开。跨文件依赖应通过 metadata/图或后续扩展获取，不应把整个 repository 塞进单个 chunk。

### 表格

保留表名、列名、单位和行/分组关系；每个拆分片段重复必要表头。不要把一行从其列定义中剥离。对复杂表格，应先验证 parser 是否正确恢复结构；错误抽取不是调整 chunk size 能解决的。

### 对话与 transcript

按 turn 或话题切分，保留 speaker、时间和 conversation/thread ID；问答对通常应绑定。仅按固定 token 切可能把问题与答案分开。

### PDF / 扫描件

页眉页脚、双栏、脚注与 OCR 错误会污染边界。先做 layout-aware parsing 和质量检查，再 chunk。页码是 provenance，不一定就是最佳检索单元；跨页表格和段落不能盲目按页切。

## 6. Metadata、权限与版本是 chunk contract 的一部分

每个 chunk 至少应可追溯到稳定 `document_id`、`chunk_id`、source location、document/chunk version 和 parent；视业务加入 title/heading path、时间有效性、语言、数据类型、tenant/ACL tags。用途包括过滤、去重、引用、增量重建和删除。

**权限不能交给 Claude 判断**。检索前或检索过程中必须以可信身份执行 metadata/ACL filtering，防止未授权 chunk 进入模型 context。复制 overlap 或生成 contextual text 时也必须继承相同权限和保留策略。

文档更新要有明确策略：同一逻辑 chunk 的稳定 ID 是否保留、旧版本何时移除、embedding/index 是否原子切换。若新旧 index 混用，可能同时返回相互冲突的政策。Chunking 规则或 parser 变更通常意味着需要版本化并重新构建相关 index。

## 7. 检索 chunk 与 Claude 引用 block 不完全相同

离线 **retrieval chunk** 是索引/召回单元；传给 Claude 的 `search_result` content block 或 citation document block 是生成与引用单元。二者可以一一对应，也可以将相邻 child 合并为一个 result，或在一个 result 内放多个逻辑 text blocks。

Claude 官方 Search Results 文档指出：较小、聚焦的 text blocks 能提供更细引用边界，长内容应拆为逻辑 blocks；同时只应返回最相关结果以避免 context overflow。[Search results](https://platform.claude.com/docs/en/build-with-claude/search-results)

Claude Citations 文档还区分：plain text/PDF 会按产品规则进一步处理，而 custom content documents 使用提供的 blocks、不会再额外 chunk，可控制引用粒度。[Citations](https://platform.claude.com/docs/en/build-with-claude/citations)

这些是截至核查日的 Claude API 产品行为，不是 Guide 要求背诵的 RAG 算法。架构题要先判断“哪层的 chunk”：索引召回边界、送入模型的证据边界，还是 citation 最小单元。

## 8. 如何用 eval 选择，而不是凭直觉

构建覆盖真实 query 分布的评估集，并为每个问题标注所需 document/chunk 或 supporting facts。至少比较：

- **retrieval recall@K**：需要的证据是否进入候选/最终 context；
- **precision/noise**：返回中有多少真正相关，重复 chunk 占多少；
- **end-to-end grounded correctness** 与拒答行为；
- citation correctness/granularity；
- 每查询返回 token、延迟、index 大小和预处理成本；
- 按文档类型、短/长答案、跨 section 问题、精确标识查询分片分析。

保持其余组件一致，对 fixed-size baseline、structure-aware、不同大小/overlap、parent-child 或 contextualized variants 做对照。只调 recall@K 可能得到大量噪声；只看最终答案可能掩盖 Claude 偶然用常识答对但证据未召回。Chunking、retrieval top-K 与 reranking相互影响，最终选择要做端到端验证。

## 9. 典型 failure modes

- **孤儿 chunk**：出现“该方案/上一季度/如下”，但缺少实体、时间或标题。
- **切断约束**：主规则命中，例外条款落在另一 chunk 且未召回。
- **重复淹没**：高 overlap 让 top-K 都是同一段的副本。
- **主题稀释**：整个长章节作为一个 chunk，目标事实只占很小部分。
- **解析先坏了**：双栏 PDF、表格或 OCR 顺序错误，却错误归因于 embedding。
- **metadata 丢失**：无法引用、删除、过滤权限或区分新旧版本。
- **一个策略处理所有数据**：代码、表格、政策和 transcript 被同一字符窗口粗切。
- **用厂商实验数字作默认值**：忽略自己的 query、数据与下游组件。

## 10. 考试决策速记

1. 先识别数据天然结构和典型问题需要的证据跨度。
2. 优先保留语义/结构边界；超长单元再递归拆分。
3. 小 chunk 提升精确度但可能丢上下文；大 chunk 保上下文但可能稀释主题。
4. 用 overlap、parent-child 或 contextual enrichment解决不同类型的边界问题，不能互相盲目替代。
5. 保留 provenance、version、ACL；检索阶段强制授权。
6. 用 retrieval + end-to-end eval 联合选择，数字来自实测而非背诵。

## 11. 范围、来源与待核查

`list.md` 把 Guide 的 RAG pipeline 目标拆为 chunking、indexing、retrieval 三课，本课粒度合理，无需修正。资料核查于 2026-09-14：[Anthropic Contextual Retrieval](https://www.anthropic.com/engineering/contextual-retrieval)、[Claude Search Results](https://platform.claude.com/docs/en/build-with-claude/search-results)、[Claude Citations](https://platform.claude.com/docs/en/build-with-claude/citations)。Anthropic 2024 文章中的具体 chunk size、top-K、模型、成本和效果数字只代表其当时实验，未用于本课题目答案。

**Needs verification**：无影响核心概念或练习答案的未核实事实。实施前应重新核查所选 parser、embedding/index 服务的 token 限制和切分语义，以及 Claude API 当前 `search_result`/Citations 的支持范围与 schema；不要用本文确定生产 chunk 数字。

下一步：[questions.md](questions.md)。
