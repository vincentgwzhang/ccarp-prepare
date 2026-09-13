# 大规模 Claude 系统的可观测性设计

> 学习序号 07 · 清单 #7 · Domain 3: Integration（19%）。核查日期：2026-09-13。
> 考试依据：[Exam_Guide.md](../../../Exam_Guide.md) 第 6 节原文 “Analyze observability challenges and select monitoring strategies at scale”。本课关注跨 Claude、检索、工具和业务服务的规模化可观测性选型；Domain 4 的“用 logging 和 observability tools 监控系统性能”更偏持续评估与优化，二者有交集但不是重复条目。

## 1. 可观测性要回答什么

**Monitoring** 用预设指标和告警回答“已知问题是否发生”；**observability** 让团队从系统产生的外部信号推断内部状态，也能调查事先未写好告警的故障。生产 Claude 系统不是一次模型调用，而是一条可能包含鉴权、检索、rerank、Claude、多轮工具、审批和异步任务的执行链。只看 API 成功率或平均延迟，无法解释“HTTP 200 但答案错了”“模型很快但工具超时”“重试让用户被扣款两次”。

一个可用设计至少回答四类问题：

1. **可靠性**：请求、任务及关键工具是否成功？超时、重试、拒答和限流发生在哪里？
2. **性能与容量**：端到端 p50/p95/p99、TTFT、各阶段耗时、队列深度、并发和 rate-limit headroom 如何？
3. **质量与安全**：检索是否命中正确证据，回答是否 grounded，工具参数/授权是否正确，guardrail 或人工审批是否触发？
4. **成本与用量**：输入/输出/cache token、模型/租户/功能路径的用量及单位成功任务成本如何？

前两类主要是传统运行信号；后两类是 LLM 系统不可缺少的**语义信号**。可观测性本身不能证明答案正确，必须把 trace 与线上抽样评价、业务结果和安全事件关联。

## 2. Logs、metrics、traces 各司其职

| 信号 | 最适合回答 | Claude 场景 | 规模化注意点 |
|---|---|---|---|
| Metrics | 趋势、SLO、告警 | 成功率、p95、429/5xx、token、tool failure、grounded-answer rate | 标签必须低基数；不能把 request ID、原始用户 ID 放进 metric label |
| Logs / events | 某次发生了什么 | 错误类型、stop reason、工具决策、策略版本、检索文档 ID | 结构化、可关联、分级保留；敏感内容先脱敏或不采集 |
| Distributed traces | 时间花在哪里、跨服务因果链如何 | gateway → retrieval → Claude → tool → approval | 传播 trace context；高流量需采样，错误/高延迟链路优先保留 |

OpenTelemetry 提供 vendor-neutral 的 traces、metrics、logs 与 context propagation，但它是**采集和传输框架，不是自动生成业务语义**。应用仍需定义“任务成功”“正确引用”“越权拦截”等事件。[OpenTelemetry 概览](https://opentelemetry.io/)

## 3. 建立统一关联模型

建议把一次用户意图抽象为 `task` 或 `interaction`，而不是等同于单次 Claude API request：一个任务可能触发多轮模型调用与多个工具。每层使用不同标识并在事件中关联：

```text
business_task_id / session_id
  └─ trace_id
      ├─ auth span
      ├─ retrieval span → query type, top-k, index version, result document IDs
      ├─ Claude span → provider request-id, model/config version, usage, stop reason
      ├─ tool span → tool name, tool_use_id, result category, retry count
      └─ validation/approval span → policy/evaluator version, outcome
```

- `trace_id` 连接己方服务；Anthropic 的 `request-id` 用来定位某次 API 响应并在需要时协助支持调查。官方说明每个 API 响应都有唯一 `request-id` header，错误 body 也带 `request_id`。[Claude API errors](https://platform.claude.com/docs/en/api/errors)
- `tool_use_id` 可关联模型提出的调用、实际执行结果和后续模型回合。
- 异步队列会切断同步 parent-child 时间线；应传播 trace context，或用 span link 表示生产任务与消费任务的因果关系，而不是靠时间戳猜测。[OpenTelemetry traces](https://opentelemetry.io/docs/concepts/signals/traces/)
- 记录 prompt/template、工具 schema、检索索引、模型配置和应用 release 的**版本或 hash**，才能把回归与变更关联；不要把完整敏感内容当作标签。

## 4. 指标设计：从组件到用户结果

### 4.1 分层指标

- **平台/API 层**：按错误类别统计 4xx、429、5xx、529、timeout；记录重试次数、最终结果和 rate-limit response headers，而非把所有 429 当成相同根因。Anthropic 文档说明常规限流会返回 `retry-after`，而某些 spend-cap 情况行为不同；具体 header 集合可能变化，实施时查官方文档。[Rate limits](https://platform.claude.com/docs/en/api/rate-limits)
- **模型调用层**：调用耗时、TTFT、输入/输出/cache usage、stop reason、refusal、每次任务的模型回合数。
- **检索层**：查询耗时、空结果率、候选数、索引版本、过滤后数量、抽样 relevance/recall；“检索成功返回”不等于检索正确。
- **工具/agent 层**：每工具成功率和延迟、参数校验失败、权限拒绝、重复/补偿操作、多轮 loop 深度、预算或最大步数终止。
- **业务/质量层**：端到端任务完成率、grounded/citation correctness、升级人工率、用户纠正率、严重错误率、单位成功任务成本。

Anthropic 的 Usage & Cost API 可提供组织历史用量和成本的聚合视图，适合对账、趋势和告警；它不替代应用侧逐任务 trace、业务成功指标或实时故障诊断。[Usage and Cost API](https://platform.claude.com/docs/en/manage-claude/usage-cost-api)

### 4.2 SLO 和告警

先从用户结果定义 SLI/SLO，例如“已授权客服查询在 5 分钟窗口内，99% 于 3 秒内返回有依据的最终答案”。再分解为 Claude、retrieval、tool 等组件指标。告警应优先基于：

- 用户可见的端到端错误率/尾延迟持续违反 SLO；
- 安全或错误副作用等低频高严重度事件；
- 429/5xx、队列积压、检索空结果率等有行动手册的先行信号；
- 与基线或 release 相关的质量、token/任务、工具回合数异常。

避免为每个瞬时尖峰分页，也不要只看平均值。告警需要 owner、严重度、runbook、去重和 burn-rate/window 策略；dashboard 不等于告警策略。

## 5. Scale 下的核心权衡

### Cardinality

将 `request_id`、`session_id`、原始 prompt 或任意 tool argument 放进 metric label，会制造近乎每次请求一个 time series，推高内存、存储和查询成本。高基数信息放进 log/trace；metric 只按有限集合聚合，如服务、区域、稳定的 route、error class、模型配置族。Claude Code 的官方 OTel 文档也明确提供 metric cardinality 控制，并指出低基数通常性能与存储成本更好、但分析粒度下降。[Claude Code monitoring](https://code.claude.com/docs/en/monitoring-usage)

### Sampling

Metrics 通常全量聚合；logs 可按级别和事件类型采集；traces 在高流量下采样。纯随机 head sampling 成本低，却可能丢掉罕见错误。tail sampling 能依据完整链路保留错误、高延迟或新版本 trace，但需要缓冲状态，运维更复杂。应全量保留关键安全审计事件，并对错误/高延迟/特定 rollout 提高 trace 采样率，同时保留少量正常基线。[OpenTelemetry sampling](https://opentelemetry.io/docs/concepts/sampling/)

### 隐私与安全

Prompt、response、RAG chunk、工具参数可能包含 PII、凭证、源代码或客户机密。默认记录 metadata、版本、长度、类别和不可逆/受控标识；只有明确用途、合法基础、访问控制、保留期和脱敏措施时才采集内容。Secret 永不进入 telemetry。日志访问、导出和删除策略应与数据治理一致。Claude Code OTel 是一个具体产品实现：官方说明内容采集有开关，工具参数仍可能敏感，需要后端过滤/脱敏；不能据此假设自建 Claude API 应用已自动安全处理内容。[Claude Code monitoring](https://code.claude.com/docs/en/monitoring-usage)

### 成本与退化

Telemetry pipeline 也会丢数据、积压或反压业务。使用本地/sidecar 或 gateway collector 做批处理、重试、过滤与路由；为 collector 自身建立 dropped spans、export failures、queue saturation 等监控。可观测性故障原则上不应阻断普通推理请求，但合规必需的审计记录若不可写，系统可能需要 fail closed——这是风险策略决定，不是一刀切答案。

## 6. Architecture decision：如何选方案

1. 从关键用户旅程、风险和 SLO 出发，列出必须能回答的诊断问题。
2. 画出同步、异步和第三方边界；定义统一 trace/task context 和结构化事件 schema。
3. 在每个阶段记录输入/输出的**安全摘要**、版本、耗时、结果类别，而非无差别存正文。
4. 将技术健康、语义质量、安全和成本串到同一 task；用离线/线上 evaluator 补足仅靠 logs 无法判断的正确性。
5. 按流量估算 cardinality、采样、保留期和预算；确保错误、安全事件及 rollout 有足够诊断样本。
6. 用故障注入验证：检索返回旧 chunk、工具超时、Claude 429/5xx、异步消费者失败时，能否快速定位且不产生重复副作用。

一个低流量原型可先用结构化日志、基本 RED 指标和少量全量 trace；跨多个 region、tool 和异步 agent 的高流量平台，应采用标准 context propagation、collector tier、cardinality 预算及分层采样。不要因为“大规模”就默认记录所有 prompt，也不要因为成本高就只保留聚合指标而失去个案因果链。

## 7. 常见 failure modes 与易混淆点

- **HTTP 200 = 成功**：模型可能以 200 返回不 grounded 的答案，工具也可能在业务层失败。技术成功与任务成功必须分开。
- **日志越多越可观测**：无 schema、无 correlation ID、含敏感正文的大量文本日志通常更贵且更难查。
- **只监控 Claude API**：检索、工具、队列和授权往往才是瓶颈或错误源。
- **重试被隐藏**：只记录最终成功会掩盖供应商波动、额外成本和尾延迟；每次 attempt 与最终 task outcome 都要可见。
- **随机采样足够**：可能丢掉罕见越权或高延迟链路；需要风险导向保留策略。
- **可观测性 = evaluation**：telemetry 提供证据，evaluator/标注和业务反馈判断质量；二者要关联而非互相替代。
- **审计日志 = 调试 trace**：审计强调不可抵赖、访问/动作主体和保留要求；调试 trace 强调调用因果与性能，权限和留存策略通常不同。

## 8. 范围、来源与待核查

Guide 要求的是：能分析 scale 下的 observability 难题并选择 monitoring strategy。清单 #7 与其直接对齐，无需修改粒度。本课不要求记忆某家后端、固定 header 数量、实时价格或任何具体 telemetry 默认值。

资料核查于 2026-09-13：[Claude API errors](https://platform.claude.com/docs/en/api/errors)、[Rate limits](https://platform.claude.com/docs/en/api/rate-limits)、[Usage and Cost API](https://platform.claude.com/docs/en/manage-claude/usage-cost-api)、[Claude Code monitoring](https://code.claude.com/docs/en/monitoring-usage)、[OpenTelemetry](https://opentelemetry.io/)、[traces](https://opentelemetry.io/docs/concepts/signals/traces/) 与 [sampling](https://opentelemetry.io/docs/concepts/sampling/)。

**Needs verification**：无影响本课结论或练习答案的未核实事实。实施前须按所用 Claude 接入面（direct API、云平台或具体产品）重新核查可用 telemetry 字段、header、管理 API 权限、数据保留/区域要求及 OTel 功能成熟度；Claude Code 的现成功能仅作为官方实现例证。

下一步：[questions.md](questions.md)。
