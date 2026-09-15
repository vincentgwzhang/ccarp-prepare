# RAG 索引与向量库策略：练习

> 学习序号 09 · 清单 #9 · Domain 3（19%） · 2026-09-15；7 道原创场景题，非官方真题。题型和 rationale 参考 [Exam_Guide.md](../../../Exam_Guide.md) 第 8 节；资料见 [knowledge.md](knowledge.md)。

## 1. Embedding model 迁移

### Question

团队准备更换 embedding model。新旧模型恰好都输出 1,024 维 vector。为了零停机，工程师建议从今天起用新模型生成 query vector，但继续查询旧 vector index，等后台慢慢更新文档。最佳判断是什么？（选择 1 项）

### Options

- A. 可行；dimension 相同就证明两个向量空间完全兼容。
- B. 建立带新 embedding/chunking 版本的独立 index，完成回填和评估后原子切换读 alias，并保留回滚窗口。
- C. 将新旧 vector 按元素相加，结果自然兼容。
- D. 仅提高 Claude 输出 token 上限，embedding mismatch 会自行消失。

### Correct Answer

B。

### Explanation

B 保持 query/document embedding contract 一致，并允许完整验证与回滚。A 把相同维数误当成相同语义空间。C 没有数学或训练依据。D 改变生成上限，不能修复检索阶段的向量不兼容。

## 2. Exact 与 ANN 的选择

### Question

某内部政策库过滤到单个部门后通常只有 8,000 个 chunks，QPS 很低，但错误漏检代价高。团队尚未证明 ANN 是必要的。最佳起点是什么？（选择 1 项）

### Options

- A. 使用 exact nearest-neighbor search 建立质量基线并测量 SLA；只有规模/延迟需要时再引入 ANN，并对比 recall。
- B. 必须立即选择最复杂的 HNSW 配置，因为所有 RAG 都只能使用 ANN。
- C. 不记录 latency 或 recall，只要查询返回 HTTP 200 即可。
- D. 移除部门过滤，以增加候选数量。

### Correct Answer

A。

### Explanation

A 符合题设的小候选集、低 QPS 和高漏检成本，并让 exact 结果成为 ANN 评估基准。B 在未证实瓶颈时增加复杂度且可能损失 recall。C 无法证明正确性。D 扩大搜索空间并可能违反访问边界。

## 3. ANN 与高选择性过滤

### Question

共享 ANN index 先取全局 top-20，再按 tenant 和 ACL 做 post-filter。小租户经常只剩 0–1 条结果，尽管其授权数据中存在相关内容。哪两项最合理？（选择 2 项）

### Options

- A. 调查 vector store 的 pre-filter、integrated/iterative filtering、partition 或 namespace 能力。
- B. 在该租户的代表性查询和 filter 选择性下，对照 exact search 测 filtered recall 与结果数量。
- C. 让 Claude 忽略 ACL，直接使用其他租户的高分结果。
- D. 将空结果自动解释为知识库绝对没有答案。
- E. 只调整前端颜色，不改变或测量检索。

### Correct Answer

A、B。

### Explanation

A 针对 post-filter 候选耗尽选择合适架构，B 验证真正的 filtered recall。C 是数据泄露。D 把检索策略失败误判为事实不存在。E 不影响候选生成。

## 4. 文档刷新后错误答案

### Question

产品文档完成更新后，RAG 突然引用旧政策；Claude model/config 和 latency 未改变。哪项是最合理的第一组检查？（选择 1 项）

### Options

- A. 检查 ingestion/re-index job、删除传播、document/embedding/index versions，以及生产 read alias 当前指向。
- B. 假定 Claude 权重在后台静默改变，不检查数据链路。
- C. 立即把 temperature 调高，让模型创造更新内容。
- D. 删除所有版本和 provenance 字段以简化调查。

### Correct Answer

A。

### Explanation

A 直接匹配变更触发点，可发现 stale chunks、漏删、失败回填、embedding mismatch 或 alias 错误。B 与题设文档刷新相关性弱。C 会增加随机性且不能提供新政策证据。D 反而移除诊断所需信息，也呼应 Guide Sample 3 的判断逻辑。

## 5. Vector store 形态选择

### Question

一个中等规模系统已使用 PostgreSQL 保存文档 metadata 和复杂 ACL；QPS 可控，团队擅长 PostgreSQL，实测 vector 扩展满足 recall 和 p95。未来增长已有容量监控。最佳决策是什么？（选择 1 项）

### Options

- A. 可以先采用现有关系库的 vector 能力，记录容量边界与迁移触发条件；无需仅因项目是 AI 就增加专用数据库。
- B. 必须迁移到专用 vector database，因为关系数据库绝不支持向量检索。
- C. 把 ACL 从数据库移到 Claude system prompt，让 schema 更简单。
- D. 不做备份，因为 vector 永远不会损坏。

### Correct Answer

A。

### Explanation

A 根据已验证的性能、ACL join、团队能力与运维成本选型，并保留增长路径。B 是绝对化错误。C 将强制授权降级为模型指令。D 忽略恢复时间、配置和派生数据不可用风险。

## 6. 多租户隔离

### Question

平台有数万个小租户和两个受严格监管的大租户。小租户共享 collection 配合强制 tenant filter 已通过测试；两个大租户要求独立故障域、单独加密/备份策略和定制 embedding。最佳方案是什么？（选择 1 项）

### Options

- A. 所有租户无条件共享同一 index，不需要 tenant filter。
- B. 采用分层方案：普通小租户共享且强制过滤；受监管大租户使用独立 collection/index 或 instance，并分别版本化和监控。
- C. 为数万个小租户各建一台独立服务器，无需成本评估。
- D. 让大租户的 query 同时搜索其他租户，以提高 recall。

### Correct Answer

B。

### Explanation

B 让隔离强度匹配租户风险和定制需求，同时避免对所有小租户付出最高隔离成本。A、D 会造成跨租户泄露。C 可能可行但题设没有支持其巨大成本和运维负担，不能成为最佳默认答案。

## 7. ANN 优化的验收

### Question

一次 ANN 参数调整使平均查询时间下降 35%，但团队没有测 recall、p99、带 ACL filter 的请求或 grounded answer correctness。是否应立即全量上线？（选择 1 项）

### Options

- A. 应该；平均 latency 改善足以证明整体架构更优。
- B. 不应该；应按真实 query/filter 分布与 exact baseline 比较 recall@K，同时测尾延迟、资源和端到端答案质量，再渐进发布。
- C. 应该；ANN 的 approximate 只影响速度，不影响返回结果。
- D. 不需要测试，只要使用厂商默认参数就不会回归。

### Correct Answer

B。

### Explanation

B 覆盖 ANN 的核心 speed–recall trade-off、filter 交互及用户最终结果，并允许控制 rollout 风险。A 只看平均值会掩盖 p99 和漏检。C 否认 approximate search 可能改变结果。D 把通用默认值误当作特定数据与负载的证据。
