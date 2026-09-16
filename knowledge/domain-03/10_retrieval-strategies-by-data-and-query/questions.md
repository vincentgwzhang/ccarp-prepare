# 练习题：按数据形态与查询模式选择检索策略

> 学习序号 10 · Domain 3: Integration。以下均为原创练习题，并非官方真题。题型与判断方式参考 `Exam_Guide.md` Sample Questions：从场景约束选择最佳架构，而不是背产品参数。

## Question 1（单选）

### Question

一个运维助手需要从 200 万条 runbook 和 incident 文档中查找解决方案。工程师最常粘贴类似 `PAY-ERR-0417` 的精确错误码，但文档中还有大量自然语言的原因与修复描述。当前只用 semantic vector search，经常返回含义相近却对应其他错误码的文档。哪项改动最合理？

### Options

- **A.** 增大 Claude 的 context window，把更多 vector 结果全部发送给模型。
- **B.** 移除错误码后再生成 query embedding，避免罕见 token 干扰。
- **C.** 并行运行错误码感知的 lexical search 与 semantic search，合并候选并评估 fusion/reranking。
- **D.** 提高生成 temperature，让 Claude 探索更多可能原因。

### Correct Answer

**C**

### Explanation

精确错误码是 lexical 的强信号，自然语言原因则适合 semantic；hybrid candidate generation 能保留两者，再通过 fusion/reranking排序。A 增加噪声和 token，并未修复召回；B 恰好删除最有判别力的约束；D 影响生成随机性，不解决 retrieval 错配。

---

## Question 2（单选）

### Question

HR 政策库使用正式术语 “post-employment access revocation”。员工通常问：“人离职以后，多久会关掉所有系统权限？”文档没有出现用户的原句。数据量中等，没有精确编号要求。哪种 candidate retrieval 最能直接解决这一主要 mismatch？

### Options

- **A.** 仅按整句 exact phrase match。
- **B.** Semantic retrieval，并用代表真实释义问题的数据验证 recall。
- **C.** 按文档创建时间倒序返回最新的五篇。
- **D.** 把所有政策标题做字母排序后取前五篇。

### Correct Answer

**B**

### Explanation

这是典型的“含义相同、措辞不同”，semantic retrieval 正面处理 vocabulary mismatch。A 很可能零命中；C 可作为 freshness 条件，却不能确定主题相关性；D 与 query 无关。若实际 query 同时包含政策编号，再评估 hybrid，但题干没有该约束。

---

## Question 3（单选）

### Question

团队构建 hybrid search，将 BM25 score 与 cosine similarity 直接相加后排序。离线测试中，一次 analyzer 更新便让 lexical 结果几乎支配所有排名。最稳健的首要修正是什么？

### Options

- **A.** 将两个 raw score 除以固定常数后继续相加，不需要评估集。
- **B.** 使用 rank-based fusion（如 RRF），或先基于 judgments 校准/归一化分数再加权。
- **C.** 删除 lexical 路径，因为 vector score 永远更可信。
- **D.** 只增加 Claude 的 max output tokens。

### Correct Answer

**B**

### Explanation

两个检索器的 raw score 不在同一尺度，不能假定可直接相加。Rank-based fusion 避开分数可比假设；若 score margin 有价值，则应在评估数据上归一化、校准和调权。A 的固定常数缺少数据依据且对 drift 脆弱；C 丢掉精确匹配能力；D 不改变检索排名。

---

## Question 4（单选）

### Question

财务助手需要回答：“客户 C-482 在过去 30 天已结算退款总额是多少？”权威交易库可通过受控、只读 API 按客户、状态和日期查询。知识库中也存有数月前的账单说明文本。最佳架构是什么？

### Options

- **A.** 对账单说明做 semantic top-K，让 Claude 从相似片段估算总额。
- **B.** 把整个交易表导出为 prose，每天做一次 embedding。
- **C.** 用授权后的 API/structured query 获取并聚合实时权威数据；需要时另行检索政策解释。
- **D.** 只做 BM25，因为客户 ID 是精确字符串。

### Correct Answer

**C**

### Explanation

该问题要求按状态/日期过滤并计算权威、当前的数值，属于 structured retrieval/aggregation。C 同时保持授权与 provenance；政策说明可由另一检索路径补充。A 可能使用过时或不完整片段；B 牺牲 freshness 和确定性；D 虽能匹配客户 ID，却不能可靠执行聚合。

---

## Question 5（多选，选择 **3** 项）

### Question

一个多租户支持系统采用 lexical + semantic hybrid retrieval。安全要求是任何未授权 ticket 均不得进入 reranker 或 Claude。为提高质量并保持边界，哪三项设计最合适？

### Options

- **A.** 从可信会话身份生成 tenant/ACL filter，并在两路 candidate retrieval 中强制执行。
- **B.** 对两路合规候选取 union，去重后再 fusion/rerank。
- **C.** 先检索全库并交给 reranker，再从 final top-K 删除其他租户结果。
- **D.** 分别测 candidate recall、最终排序质量和 negative authorization cases。
- **E.** 只保留同时出现在 lexical 与 semantic top-N 的 intersection。

### Correct Answer

**A、B、D**

### Explanation

A 在敏感内容进入后续组件前执行安全边界；B 保留两种方法的互补 recall；D 同时验证质量和“越权结果为零”。C 已让敏感数据进入 reranker，违反题干约束；E 会丢掉只被一种方法正确召回的证据，通常不适合作为 hybrid candidate policy。

---

## Question 6（单选）

### Question

知识助手加入高质量 reranker 后，precision 上升，但 p95 延迟超过 SLA。分析发现简单的 exact-ID query 原本通过 lexical lookup 已能稳定得到唯一结果。哪项调整最能保留收益并降低不必要延迟？

### Options

- **A.** 所有 query 继续 rerank，但把 final K 增大一倍。
- **B.** 对 exact-ID/高置信度确定性命中绕过 reranker；仅对歧义、hybrid 或低置信度 query 启用，并分别评估。
- **C.** 删除 lexical lookup，让所有 query 统一走 semantic + reranker。
- **D.** 降低 Claude temperature。

### Correct Answer

**B**

### Explanation

Query routing 可以让不需要第二阶段判断的精确查询走低延迟路径，把 reranking成本用于真正受益的 query。A 会增加工作量与 context 噪声；C 破坏 exact-ID 的最佳路径；D 只影响生成，不能消除 reranker 延迟。

---

## Question 7（单选）

### Question

一次 hybrid retrieval 实验使总体 nDCG 提高 8%，团队准备直接上线。分片结果却显示：自然语言 FAQ 显著改善，而包含 SKU 的查询 recall@10 从 96% 降到 71%；SKU 查询占高价值售后请求的 25%。下一步最佳做法是什么？

### Options

- **A.** 直接上线，因为总体指标已经提高。
- **B.** 删除 SKU query，避免它影响实验结果。
- **C.** 保留分片评价，修正 SKU analyzer/routing/fusion，并以业务权重、端到端正确性、延迟和成本重新验收。
- **D.** 只增加生成阶段的 few-shot examples。

### Correct Answer

**C**

### Explanation

总体平均值掩盖了关键、高价值 query slice 的严重退化。应定位 exact-token 路径、routing 或 fusion 问题，并联合业务影响与端到端指标重新决策。A 会把已知高影响缺陷上线；B 操纵评估分布；D 不能找回 retrieval 已漏掉的 SKU 证据。
