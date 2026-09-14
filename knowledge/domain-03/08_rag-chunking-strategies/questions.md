# RAG 分块（Chunking）策略：练习

> 学习序号 08 · 清单 #8 · Domain 3（19%） · 2026-09-14；7 道原创场景题，非官方真题。结构和 rationale 风格参考 [Exam_Guide.md](../../../Exam_Guide.md) 第 8 节；资料见 [knowledge.md](knowledge.md)。

## 1. 丢失文档语境

### Question

金融知识库按固定长度切分。一个被频繁漏检的 chunk 只有“该公司本季度收入同比增长 8%”，公司名和季度只出现在父标题。查询会明确包含公司名和季度。最佳改造是什么？（选择 1 项）

### Options

- A. 删除父标题，让 chunk 更短。
- B. 为 chunk 保留 heading/document metadata，并评估将简短、chunk-specific 的公司与季度语境加入索引文本。
- C. 把所有公司所有季度合并成一个巨大 chunk。
- D. 仅增大 Claude 的输出 token 上限。

### Correct Answer

B。

### Explanation

B 修复了检索单元缺少实体和时间的问题，并可通过 contextual enrichment让索引文本自包含。A 进一步丢失语境。C 会混合大量主题、稀释检索信号。D 影响生成上限，不能让检索器看到缺失的公司与季度。

## 2. 法律条款与例外

### Question

合规手册的一条主规则与紧随其后的例外被固定窗口切到不同 chunk。系统经常只召回主规则，给出过度绝对的答案。哪项设计最合理？（选择 1 项）

### Options

- A. 按条款/section 结构切分并保留定义和例外；若 section 过长，可检索较小 child 后扩展到包含完整约束的 parent。
- B. 将每个单词单独作为 chunk，以最大化精确度。
- C. 保持现状，让 Claude 猜测是否存在例外。
- D. 删除例外条款，减少索引大小。

### Correct Answer

A。

### Explanation

A 同时利用精确 child 匹配和完整 parent 语境，符合规则与例外不可被随意割裂的约束。B 会摧毁语义。C 没有提供必要证据。D 改变权威内容并造成错误答案。

## 3. Overlap 过高

### Question

团队把相邻 chunk overlap 提高到 90%。离线检索的 top-10 经常是同一段的近似副本，其他支持事实反而进不了 Claude context。最佳下一步是什么？（选择 1 项）

### Options

- A. 将 overlap 提高到 100%，确保没有边界损失。
- B. 根据边界类失败重新评估较小 overlap，并按 parent/source position 去重或合并相邻命中。
- C. 取消所有 evaluation，只看 index 条目数。
- D. 把重复片段当作十份独立证据，提高答案置信度。

### Correct Answer

B。

### Explanation

B 针对 overlap 的真实目的——修补边界——做实测，并防止重复结果挤占 context。A 会让重复更严重。C 无法判断召回和答案质量。D 将复制的数据错误解释为相互独立的证据。

## 4. 表格分块

### Question

产品价格表被逐行抽取后切分，命中的数据行只有“Enterprise | 25 | annual”，没有列名、币种或表标题。哪两项改造最合适？（选择 2 项）

### Options

- A. 在每个表格片段保留表名、列名、单位和行/分组关系。
- B. 验证 layout/parser 是否正确恢复表格，再评估按逻辑行组切分。
- C. 仅将数字 25 重复一百次来增加相似度。
- D. 删除 provenance，避免 Claude看到来源。
- E. 无条件按 PDF 页边界切分，即使一张表跨页。

### Correct Answer

A、B。

### Explanation

A 让数据可解释，B 先排除解析损坏并采用数据结构边界。C 增加噪声而不补语义。D 破坏引用和追溯。E 会再次切断跨页表格；页码是来源信息，不天然等于检索边界。

## 5. 一个策略处理所有文档

### Question

企业知识库包含 Markdown 手册、Java 源码、合同、表格和会议 transcript。团队建议全部统一为每 1,000 字符切一刀以简化运维。最佳架构判断是什么？（选择 1 项）

### Options

- A. 采用统一固定窗口，因为所有数据都有相同语义结构。
- B. 保留固定窗口作为 baseline/兜底，但按文档类型选择 heading、函数/类、条款、表格和 speaker/turn 等边界，并用共同评估集验证。
- C. 不做 chunking，将全部企业文档作为单个 index item。
- D. 只根据文件扩展名决定答案是否正确。

### Correct Answer

B。

### Explanation

B 兼顾简单 baseline 与不同数据的天然结构，且要求以评估验证复杂度是否值得。A 的前提错误。C 会造成主题稀释和不可扩展。D 的文件类型只能辅助解析策略，不能证明检索或答案质量。

## 6. 如何选择 chunk 参数

### Question

架构师需要在两个 chunking 方案中选型。方案 X 的 recall@10 较高，但返回大量重复/无关文本，最终 grounded correctness 更低且每查询 token 更多；方案 Y 的 recall@10 略低，却满足业务正确率、延迟和成本目标。哪项选择最合理？（选择 1 项）

### Options

- A. 必须选 X，因为单一 retrieval recall 指标永远优先于业务结果。
- B. 在代表性数据上联合比较 retrieval、噪声/重复、端到端正确率、引用、延迟和成本；按题设当前证据选择满足整体约束的 Y。
- C. 将两个方案的 chunk size 取算术平均，无需重建 index 或重测。
- D. 只比较 index 文件大小。

### Correct Answer

B。

### Explanation

B 把 chunking 视为端到端架构决策。X 的高 recall 没有转化为可接受的最终结果；Y 在题设下满足业务约束。A 用单指标替代目标。C 的平均参数没有实证意义且变更后必须重测。D 忽略质量和线上表现。

## 7. 检索 chunk 与引用粒度

### Question

应用检索到一个较大的 parent section，其中含三个独立事实。产品要求 Claude 的引用尽量精确到各事实，而不是每次引用整个 section。使用 Claude Search Results/Citations 时，哪项处理最合适？（选择 1 项）

### Options

- A. 将送给 Claude 的证据组织为较小、聚焦的逻辑 text/content blocks，并保留共同 source/title；同时分别评估检索边界与引用边界。
- B. 删除 source 和 title，使引用更短。
- C. 假设 vector index 的 parent chunk 必须永远等于 citation 的最小 block。
- D. 把三个事实拼成一个不可分割 block，然后要求 API 引用其中任意字符范围而不核查产品行为。

### Correct Answer

A。

### Explanation

A 符合 Claude 官方建议：聚焦的逻辑 blocks 能提供更细引用边界；检索单元与传给模型的引用单元可以不同。B 破坏来源归属。C 混淆两个架构层。D 依赖未验证的引用粒度，也无法满足题设的精确引用目标。
