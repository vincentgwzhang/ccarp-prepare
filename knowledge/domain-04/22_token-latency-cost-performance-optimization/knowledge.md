# 22 · Token、Latency 与 Cost–Performance Trade-off 优化

## Guide 对齐与学习边界

- **Domain**：4 · Evaluation, Testing & Optimization（16%）
- **Exam Guide 明确要求**：`Optimize token usage, latency, and cost-performance trade-offs`。
- **本课目标**：在质量、安全与可靠性门槛下，定位端到端成本/延迟瓶颈，选择 prompt/context、caching、model/routing、tool orchestration、streaming 或 batch 等合适杠杆。
- **Guide 题型提示**：Sample 2 明确考察“稳定大前缀 + 短动态输入”应把稳定内容放前、动态内容放后并启用 prompt caching，而不是盲目截断或无条件换最小模型。
- **边界**：本课聚焦优化决策；下一课才系统讲持续 logging/observability。当前价格、型号、API 字段和 feature availability 不是考试固定知识。

核心原则：

> 优化目标不是“token 越少越好”，而是以最低的 **cost per successful task**，在约定的质量、safety/security、可靠性和 latency SLO 下完成任务。

---

## 1. 先定义优化约束，而不是先删 Token

任何优化前先固定 success criteria：

- **质量**：task success、groundedness、precision/recall、关键 slice；
- **安全**：越权、泄漏、unsafe action 等 hard gates；
- **延迟**：TTFT、端到端 p50/p95/p99、deadline violation；
- **成本**：每次 request、每次 completed task、每个成功 outcome；
- **可靠性**：error、timeout、retry、fallback、tool success；
- **吞吐**：requests/tasks per minute 与并发需求。

Anthropic 的 latency 指南建议先建立能正确工作的 prompt，再做 latency reduction；过早施加长度或模型限制，可能让团队根本看不到任务应有的质量上限。

### 三种经常冲突的目标

1. **Token efficiency**：输入/输出更少；
2. **Responsiveness**：用户更快看到或拿到结果；
3. **Capability/quality**：复杂任务正确完成。

例如删掉 policy context 会减少 tokens，却可能增加错误、人工升级和重复请求，最终使每个成功任务更贵。

---

## 2. 建立端到端 Token、Latency、Cost 模型

### Token 账本

一次 LLM 工作流可能包含：

- system prompt、policy、few-shot examples；
- conversation history / memory；
- retrieved chunks、files、images/PDF；
- tool definitions 与 tool results；
- generated output / thinking；
- retry、fallback、grader 和子 Agent 的额外调用。

不能只统计最终一次 Messages call。Agent 任务应汇总整个 trace 的 input/output/cache/tool/retry usage。

### 成本模型（概念）

```text
LLM cost
  = ordinary input tokens × input rate
  + output tokens × output rate
  + cache writes × write rate
  + cache reads × read rate
  + feature/tool-specific charges

Cost per successful task
  = (LLM + retrieval + tool + infrastructure + review cost)
    / successful business outcomes
```

具体费率会变化，考试更重要的是识别变量和分母。便宜但频繁失败、重试或升级人工的路径，可能比一次高质量调用更贵。

### Latency budget

```text
End-to-end latency
  = client/network/gateway
  + queue/rate-limit wait
  + input processing → TTFT
  + output generation
  + tool/retrieval round trips
  + orchestration/retries
  + validation/post-processing
```

- **TTFT (Time to First Token)**：用户多久看到第一个 token；
- **Time to completion**：完整结果何时可用；
- **Task latency**：包含所有 tools、retries、human approval 的业务完成时间。

只优化模型生成，可能对一个主要耗时在慢数据库 tool 的 agent 没有明显效果。

---

## 3. 先测量：按 Trace、Route 与 Slice 分解

至少记录：

| 维度 | 指标 |
|---|---|
| Request | input/output/cache tokens、model/config、duration、error |
| User-perceived | TTFT、stream interruption、time to usable answer |
| Agent | turns、tool calls、serial/parallel time、retry/fallback |
| Retrieval | query latency、chunks/tokens、relevance/freshness |
| Outcome | task success、quality/safety gates、人工升级 |
| Cost | request cost、trace cost、cost per successful task |
| Slice | intent、语言、complexity、tenant、context size、route |

Anthropic 的 Token Counting 能在发送前估算结构化 request 的 input tokens，帮助管理成本、rate limits、routing 和 prompt length；最终实际 usage 仍应以 response/usage reporting 为准。Usage & Cost API 可用于组织级历史 usage/cost 分析和 reconciliation，但其 credential、平台支持与字段属于当前实现。

不要只看平均数：long-context、复杂 tool path 和 retry-heavy slice 往往决定 p95/p99 与尾部成本。

---

## 4. 输入 Token 优化：减少低信号，不删除必要证据

### 4.1 Prompt 去冗余

- 删除重复规则、礼貌性模板和语义相同的 examples；
- 使用清晰、直接的 instruction 和少量 canonical examples；
- 将稳定 policy/source 维护为单一版本，避免在多处重复注入；
- 用结构化字段表达约束，但不要为了“压缩”变成模型难以理解的密文。

“短”不是目标，“最小充分 context”才是目标。每次删除都必须重跑 eval，特别检查 edge cases 与 safety rules。

### 4.2 Retrieval 代替 Monolithic Context

对于大型知识库，先检索与当前 query 相关的高信号 chunks，而不是把整个 corpus 放进 prompt。优化点包括 query routing、chunking、metadata/ACL filter、top-K、reranking 和去重。

Trade-off：top-K 太大增加 cost、TTFT 和注意力竞争；太小则丢证据。应以 retrieval recall、answer quality 与 end-to-end latency 联合调优，而不是只看 token 数。

### 4.3 Conversation 与 Tool Context 管理

- 对历史对话做高保真 summarization/compaction，保留决定、未解决事项、权限与 source references；
- 清除已不需要的巨大 raw tool results，保留可追踪摘要/标识；
- 按需加载 tools/skills，而不是每次暴露全部 definitions；
- 将确定性 state 放在外部 store，需要时检索，不要求模型永久“记住”。

Compaction 会引入信息损失和摘要错误，必须测试 long-session continuity，必要时保留 authoritative state 而非递归摘要。

---

## 5. 输出 Token 优化：控制任务，而不是粗暴截断

- 明确期望格式、audience、段落/句子数量和必要字段；
- 先给 decision/answer，再按需提供 explanation；
- machine-to-machine 场景只返回 downstream 必需的 structured fields；
- 避免让模型重述整个 input 或 tool result；
- 在可交互产品中，将 detail 作为用户按需展开的第二步。

`max_tokens` 是 hard cap，不是“期望长度”。设得过低可能截断 JSON、citation、tool argument 或关键结论。应配合 stop/completion detection、schema validation，并在截断时安全续写或重试；具体 stop 字段须按当前 API 核查。

输出 tokens 往往同时影响生成时间和成本，但过短会降低质量并触发更多 follow-up，需看整个 session/task。

---

## 6. Prompt Caching：重用稳定前缀

适合：大量请求共享较大的 system prompt、policy、tool definitions、reference document 或增长中的稳定 conversation prefix。

### 设计原则

```text
[stable tools / system / policy / examples / shared document]
                         ↓ cacheable prefix
[dynamic user message / current retrieved context / volatile timestamp]
```

- 稳定内容放前，动态内容放后；
- 避免把 request ID、当前时间或随机顺序插进稳定前缀；
- prompt/tool ordering 与序列化保持一致；
- 观察 cache write/read tokens、hit ratio、eviction/TTL 和 invalidation；
- 低复用、低流量或持续变化的 prefix，cache write 成本可能不值得。

Prompt caching 可以降低重复前缀的处理成本和 latency，但**不等于缩短 context**：模型仍需要在完整 context 上推理，过多低相关内容仍可能损害注意力与质量。Caching 解决“重复计算/计费”，context engineering 解决“给模型哪些信息”。

Exam Guide Sample 2 的答案正是缓存稳定 8,000-token 前缀，而不是截断所需 policy 或盲目 downsizing。

---

## 7. Model、Effort 与 Routing

### 选择满足门槛的最低成本路径

在真实 prompts/data 上比较候选配置的：

- task success 与关键 edge cases；
- TTFT、completion、p95/p99；
- input/output/retry cost；
- safety/security 与 tool-use reliability。

不是所有任务都需要同一配置：

- 简单分类、提取或路由 → 低成本/低延迟路径；
- 复杂推理、模糊决策或高风险 case → 更强配置；
- 低置信度、validation failure 或 policy trigger → escalation。

Routing 本身也要评估 precision/recall。错误地把难题送去便宜路径会增加 failure 和 retry；把全部请求送去昂贵路径则失去优化价值。

### 不要用当前产品默认值当架构原理

模型名称、effort/thinking 行为和速度模式会变化。稳定原则是：用 eval 找到 Pareto frontier，并按任务难度与 SLA 路由；具体支持范围实施前核查。

---

## 8. 减少 Round Trips 与关键路径

### 并行独立工作

天气、库存和用户偏好等互不依赖的 calls 可以并行；“先查 order_id，再用它退款”有数据依赖，必须串行。并行化降低 wall-clock latency，但不必然减少 token 或总成本，且会提高并发、rate-limit 与部分失败处理复杂度。

### 把确定性逻辑留给 Code

- schema validation、排序、过滤、算术、权限检查由程序完成；
- 不要让模型反复调用工具完成简单循环或重算确定性结果；
- 合并过细、chatty 的 tool API，但避免创建权限过大的万能工具；
- 对副作用使用 idempotency key，避免 retry 重复执行。

### Retry 与 Fallback

只对 transient、可安全重试的失败采用有界 retry/backoff；记录每次 attempt 的成本。Semantic failure 不应原样重试多次。Retry storm 会同时恶化 tail latency、cost 与 capacity。

---

## 9. Streaming 与 Batch：解决不同问题

| 方法 | 适合 | 改善 | 不解决 |
|---|---|---|---|
| Streaming | 交互式长回答 | 用户更早看到结果，改善 perceived responsiveness | 通常不减少 token、最终完成时间或总成本 |
| Async Batch | 夜间分类、离线 eval、批量 enrichment | 当前产品可降低单位处理成本、适合高吞吐非紧急任务 | 不适合低延迟交互；结果非即时、需处理异步失败 |

Anthropic 当前文档说明 batch 请求彼此独立、结果异步返回且可能需要较长处理窗口；当前定价提供折扣，但比例属于可变产品信息，不能当作永恒架构常数。

选择标准是 deadline：用户在等待答案时优先同步/streaming；能等待小时级处理的 workload 才考虑 batch。

---

## 10. 优化决策矩阵

| 症状 | 优先调查/杠杆 | 常见误区 |
|---|---|---|
| 重复大静态前缀，动态尾部很短 | Prefix ordering + prompt caching | 截断必要 policy |
| 输入充满不相关文档 | Retrieval、reranking、去重、context pruning | Cache 后继续塞全部文档 |
| 输出冗长 | 明确 answer contract、sentence/field limit | 只用过低 `max_tokens` 截断 |
| 用户等待感强，完整时间尚可 | Streaming、先返回 status/partial result | 误称总 latency/cost 已下降 |
| 多个独立 tools 串行 | 并行执行与 partial-failure policy | 并行有依赖或副作用的 calls |
| 夜间批处理成本高 | Async batch + shared-prefix strategy | 用 batch 服务实时 chat |
| 简单与复杂请求混合 | Eval-driven model/config routing | 所有请求都换最小模型 |
| p95/p99 高但 p50 正常 | 看慢 tool、retry、queue、long-context slices | 只优化平均 token 数 |
| 单次调用便宜但任务常失败 | 优化 cost per successful task | 只看 cost per request |

---

## 11. 安全的迭代流程

1. 定义 quality/safety guardrails 与 latency/cost SLO；
2. 建立 request → trace → outcome 的 baseline；
3. 找最大贡献项：input、output、model、tools、retry、queue 或 infra；
4. 一次选择一个 lever，并预测影响；
5. 在代表性 eval 与关键 slices 上比较 quality、latency、cost；
6. Canary/A/B 后逐步 rollout；
7. 监控 regression、cache hit、route mix、retry 与 cost per success；
8. 记录决策和 rollback 条件。

最终选择通常位于 Pareto frontier：不存在另一个方案能在不损害任何目标的情况下同时改善 quality、latency 和 cost。架构师的职责是让业务看到这条 frontier 和约束，而不是承诺三个指标无限同时下降。

---

## 12. 典型 Failure Modes

1. **只看 token count**：忽略失败、retry、tool 和人工成本。
2. **盲目截断 context**：必要 policy/evidence 被删，质量或安全下降。
3. **Caching 等同 context reduction**：成本下降但注意力噪声仍在。
4. **无条件换最小模型**：能力不足导致 retry/escalation，反而更贵。
5. **用 streaming 宣称总延迟下降**：多数情况下只是更早显示 token。
6. **过低 `max_tokens`**：答案、JSON 或 tool call 被截断。
7. **并行化有依赖/副作用的 tools**：竞态、越权或重复写入。
8. **无限 retry**：放大 tail latency、cost 与 rate-limit pressure。
9. **用 batch 处理交互请求**：成本可能下降，SLA 却不满足。
10. **只测平均 latency**：关键 tenant/复杂路径的 p99 被隐藏。
11. **修改多个 lever 后才测量**：无法归因收益或 regression。
12. **硬编码当前价格/型号**：产品变化后架构假设失效。

---

## 13. 考试判断清单

遇到优化 scenario，依次问：

1. 业务 deadline、质量和 safety hard gates 是什么？
2. 瓶颈在 input processing、output generation、tool round trips、retry 还是 infra？
3. 静态大前缀是否可复用？动态内容是否破坏 cache prefix？
4. 是否有低相关 context 可通过 retrieval/pruning 删除，而不丢证据？
5. Streaming 是改善 TTFT/体验，还是实际减少完成时间？
6. 工作是否非实时、可 batch？
7. Model/routing 改动是否通过真实 eval，而非凭型号猜测？
8. 是否比较了 cost per successful task 和 p95/p99，而非单次均值？

优先选择：先测量、保留质量门槛、针对最大瓶颈使用匹配的杠杆，并用 eval/A-B 验证 trade-off 的方案。

---

## 资料来源与核查日期

核查日期：**2026-09-28**。

1. Anthropic Claude Platform Docs, [Reducing latency](https://platform.claude.com/docs/en/test-and-evaluate/strengthen-guardrails/reduce-latency)：TTFT、prompt/output length、model selection 与 streaming。
2. Anthropic Claude Platform Docs, [Pricing — Prompt caching](https://platform.claude.com/docs/en/about-claude/pricing#prompt-caching)：stable prefix 重用、cache writes/reads 与成本/latency 关系。
3. Anthropic Claude Platform Docs, [Token counting](https://platform.claude.com/docs/en/build-with-claude/token-counting)：发送前估算 input tokens，用于成本、rate-limit、routing 与长度管理。
4. Anthropic Claude Platform Docs, [Batch processing](https://platform.claude.com/docs/en/build-with-claude/batch-processing)：异步 batch 的适用场景、独立请求、限制与当前成本特性。
5. Anthropic Claude Platform Docs, [Usage and Cost API](https://platform.claude.com/docs/en/manage-claude/usage-cost-api)：组织级 usage/cost tracking、reconciliation 与 optimization analysis。

### Guide 要求 vs 当前产品实现

- **Guide 要求**是优化 token、latency 与 cost-performance trade-off；需要掌握架构推理、测量与验证方法。
- Prompt caching 是 Guide Sample 2 明确体现的机制；具体 cache TTL、费率、字段与支持型号属于当前产品实现。
- Batch 折扣、API/usage 字段、model/effort/speed 支持范围均可能变化，不是固定考试数字。

### Needs verification

实施前必须重新核查模型 identifiers/capabilities、tokenizer、pricing、cache TTL/eligibility/invalidation、Batch SLA/discount、Token Counting 与 Usage/Cost API 字段及权限、rate limits、streaming/stop behavior 和平台差异。任何 routing、context pruning 或 batch/caching 决策都应在目标 workload 上用真实 quality、p95/p99、cost per success 和 safety gates 重新验证；这些可变细节不用于确定练习题答案。
