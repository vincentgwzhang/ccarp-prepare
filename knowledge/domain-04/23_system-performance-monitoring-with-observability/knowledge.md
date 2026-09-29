# 23 · 基于日志和可观测性工具的系统性能监控

> 学习序号 23 · 清单 #23 · Domain 4: Evaluation, Testing & Optimization（16%）。核查日期：2026-09-29。
> 考试依据：[Exam_Guide.md](../../../Exam_Guide.md) 第 6 节原文 “Monitor system performance using logging and observability tools”。本课关注生产运行中如何发现 performance、quality 与 cost regression，并把证据反馈到评估和优化；第 07 课侧重 Domain 3 的规模化可观测性架构选型。

## 1. 监控对象是完整任务，不只是一次模型调用

Claude 应用的用户任务通常经过 gateway、authentication、retrieval、Claude、多轮 tool calls、validation 和 persistence。单次 API 返回 HTTP 200，只能说明一次传输/调用成功，不能证明：

- 用户任务完成且答案正确；
- 检索到了相关且新鲜的证据；
- 工具产生了预期业务结果且没有重复副作用；
- latency、token 与 cost 满足 SLO；
- 输出没有越权、泄露或安全问题。

因此主关联键应是 `task_id` / `interaction_id`，一次任务下可以有多个 API attempts 和 tool spans。监控的目标是从“用户结果异常”下钻到“哪一阶段、哪一版本、哪个 slice 造成异常”，并让故障样本进入 regression eval。

```text
task / interaction
  └─ trace
      ├─ input + auth
      ├─ retrieval / rerank
      ├─ Claude request(s)
      ├─ tool execution(s)
      └─ validation + business outcome
```

## 2. Metrics、logs、traces：同一事实的不同视角

| 信号 | 最适合回答 | Claude 系统示例 | 不适合单独承担 |
|---|---|---|---|
| Metrics | 是否出现趋势、SLO breach 或异常分布 | task success、p95/p99、429/5xx、tokens、cost per success、tool failure | 还原某次请求的完整因果链 |
| Structured logs/events | 某个离散事件发生了什么 | error class、stop reason、policy decision、deploy/index/prompt version | 高效计算所有请求的尾延迟分布 |
| Distributed traces | 时间花在哪里、跨组件的因果关系 | gateway → retrieval → Claude → tool → validator | 全流量长期保存详细正文 |

OpenTelemetry 将 traces、metrics 与 logs 作为可关联的 telemetry signals。它提供 instrumentation、context propagation 与 export 机制，但不会自动定义 `task_success`、groundedness 或安全违规等业务语义；这些仍需应用团队设计。

生产日志应有稳定 schema，而不是“能输出 JSON 就算 structured”。至少记录时间、service、environment、task/trace/span ID、event type、版本、结果类别、耗时和安全摘要。日志正文默认不包含完整 prompt、response、RAG chunk、token、credentials 或原始 PII。

## 3. 一套可操作的分层指标

### 3.1 用户结果与 SLO

从用户视角定义 SLI，而不是从某个组件方便采集的指标出发：

- end-to-end task success / completion rate；
- p50、p95、p99 latency，以及 TTFT 与 time-to-complete；
- grounded/citation correctness、严重事实错误率；
- human escalation、用户纠正、abandonment；
- policy violation、unsafe action、data leakage；
- `cost per successful task`，而非只看 cost per API request。

平均值会掩盖长尾。若 p50 正常而 p99 恶化，应按 route、intent、context size、tool path、retry count、tenant tier、release 和 region 分解，再用 exemplar trace 找出慢链路。

### 3.2 平台与资源信号

- traffic：任务数、并发、tokens/minute、route mix；
- errors：4xx、429、5xx、529、timeout、stream 中途 error、最终失败；
- latency：gateway、queue、retrieval、Claude、各 tool、validation 的 histogram；
- saturation：queue depth、worker concurrency、connection pool、rate-limit headroom；
- retries/fallbacks：每次 attempt、backoff、fallback path 与最终 outcome。

Anthropic 官方错误文档指出 transient failures 可被 SDK 自动重试，且 streaming 在 HTTP 200 之后仍可能出现 error event。因此不能只统计客户端显式发起的请求或初始 status code；否则 retry、额外 token/cost 和最终失败会被隐藏。

### 3.3 LLM、RAG 与 agent 专属信号

- input/output/cache token、调用轮数、stop reason、refusal、truncation；
- cache hit、route/model/config/prompt version；
- retrieval empty-result、filtered-result、index/corpus version、抽样 relevance/freshness；
- tool success、latency、schema/auth failure、duplicate/compensation、loop depth；
- validator outcome、citation coverage、groundedness sample、human review result；
- Usage & Cost 数据与应用侧按任务记录的 cost estimate/业务 outcome 对账。

Usage & Cost Admin API 提供组织级历史 usage/cost 数据，适合趋势、对账、容量和告警。它不具备应用的完整业务上下文，不能取代 task-level trace、用户结果或实时 incident telemetry；具体可用性、权限和字段依接入面而异。

## 4. 版本和 slice 是定位回归的关键

每个 task 至少关联下列**版本标识或 hash**：应用 release、prompt/template、model/config、tool schema、retriever/reranker、index/corpus、guardrail/evaluator。再保留有限且有意义的 slice：intent、language、risk tier、route、context-size bucket、customer tier。

例：总 task success 只下降 0.4%，但“西班牙语 + 退款 + 新 prompt version”下降 18%。没有版本和 slice，总量 dashboard 只会显示轻微噪声；有它们才能形成可检验假设并快速 rollback。

### Cardinality 纪律

Metric label 的每个唯一组合通常需要单独聚合状态。不要把 `request_id`、`task_id`、原始 user ID、prompt 或任意 URL 放进 labels。它们属于 logs/traces；metric labels 应使用受控集合，如 service、environment、release、route、error class、language bucket。高基数会导致内存、存储和查询成本失控，还可能让重要序列进入 overflow。

## 5. 告警：从用户影响到可行动根因

有效告警同时满足：有用户或风险意义、有明确 owner、有 runbook、有足够持续时间/样本、能采取行动。

建议分层：

1. **Page**：用户可见 task success/尾延迟持续违反 SLO；高严重度安全事件；副作用错误。
2. **Investigate soon**：error-budget burn、quality slice drift、cost per success 异常、fallback/retry 持续升高。
3. **Ticket/capacity planning**：token 或 rate-limit headroom 趋势、长期 queue growth、低 cache hit。

不要对每个瞬时 429 单独 page。应结合持续窗口、最终失败率、queue/retry 和用户影响；429 可能来自常规限流或流量快速增长等不同条件。类似地，5xx 计数需要与自动重试后的最终 outcome 一起看。

发布时记录 deploy/change marker，并使用 canary cohort。告警触发后应能回答：何时开始、影响哪些 slices、是否与版本/索引/流量变化一致、是否需要 rollback 或降级。

## 6. 质量漂移不能靠传统 APM 自动判断

线上 monitoring 有真实流量覆盖优势，但 Anthropic 官方 eval 实践也明确指出其 signals noisy 且通常缺少 grading ground truth。应组合：

- deterministic checks：schema、citation presence、授权、tool outcome；
- 抽样的 calibrated LLM grader；
- 高风险或主观任务的 systematic human review；
- weekly/manual transcript review 发现未知 failure mode；
- 用户 feedback，但承认其稀疏、自选择和偏向严重问题；
- production sample 的脱敏 replay / shadow eval；
- 将事故和负面样本加入版本化 regression dataset。

线上 proxy（如 response length、latency、thumbs-up）不是 ground truth。grader 也要按人类标注校准并监控 drift，不能成为未经验证的单点裁判。

## 7. Dashboard 与 incident 闭环

一个实用 dashboard 由上到下逐层下钻：

1. **Outcome**：task success、quality/safety、human escalation、cost per success；
2. **Service health**：traffic、errors、tail latency、saturation、SLO/error budget；
3. **Claude/API**：calls、tokens、cache、429/5xx/529、retry、stop outcomes；
4. **RAG/tools**：retrieval quality proxy、index version、tool latency/failure、loop depth；
5. **Versions/slices**：release、prompt、model/config、corpus、route、intent/language/risk。

Incident workflow：

1. Detect：SLO、quality/safety 或 cost anomaly；
2. Scope：确定开始时间、影响面和异常 slice；
3. Trace：抽取 error/high-latency exemplars，还原 critical path；
4. Correlate：检查 deploy、prompt/model/tool/index/config 变化；
5. Mitigate：rollback、route/fallback、限流、暂停危险 action；
6. Learn：postmortem，补 instrumentation/runbook；
7. Prevent：将 failure 变成 regression case，验证修复后渐进 rollout。

## 8. Sampling、隐私与 telemetry 自身可靠性

- metrics 通常全量聚合；正常 traces 可采样，但 error、高延迟、新 release 和安全事件要提高保留率；
- 纯 head/random sampling 可能丢掉稀有故障，tail sampling 更能按最终结果保留，但成本和复杂度更高；
- prompt/response content 采集采用 data minimization、redaction/tokenization、细粒度访问控制、审计和 retention/deletion policy；secret 永不记录；
- 可观测性 pipeline 也要监控 dropped spans/logs、export failure、queue saturation 和时钟问题；
- 普通 telemetry backend 故障通常不应拖垮 inference path；若某类审计记录是合规前置，则 fail-open/fail-closed 必须由风险策略明确决定。

## 9. 常见 failure modes 与考试判断

| Failure mode | 为什么错 | 更好的选择 |
|---|---|---|
| 只看 average latency | 掩盖 p95/p99 和关键 slice | histogram + tail percentile + trace exemplar |
| HTTP 200 当 task success | 忽略语义、工具和业务失败 | 分开 transport、component、task outcome |
| `request_id` 做 metric label | 制造高 cardinality | 放 log/trace，metric 使用受控维度 |
| 只监控 Claude 调用 | 漏掉 retrieval、queue、tool、validation | 端到端 task trace |
| 每个 429 都 page | 告警风暴且缺少用户影响 | 窗口、SLO、最终 outcome、headroom 联合判断 |
| 只靠用户反馈评质量 | 稀疏且有选择偏差 | monitoring + grader + transcript/human review + eval |
| 只看 cost/request | 便宜但失败的路径可能更贵 | cost per successful task |
| 无 deploy/version marker | 无法关联回归与变更 | 所有关键配置版本化并写入 telemetry |
| 全量记录 prompt/tool result | 泄露和合规风险 | metadata-first、最小化、脱敏、授权与保留期 |
| Dashboard 没有 owner/runbook | 只能观察，不能响应 | alert → owner → action → regression test |

## 10. 考场决策框架

遇到 monitoring 场景题，依次判断：

1. 用户可见的 task outcome/SLO 是什么？
2. 问题是趋势检测、单次事件还是跨服务因果？分别选 metric、log/event 或 trace。
3. 是否覆盖 retrieval、tools、validation、retry 和 business outcome？
4. 是否看 tail、关键 slices 和版本，而非均值/总量？
5. 质量信号有没有 ground truth 或经过校准的多方法证据？
6. 告警是否可行动，并关联 owner、runbook、rollback？
7. telemetry 是否最小化敏感数据并控制 cardinality/sampling？
8. 是否把生产失败反馈为 eval/regression case？

通常最佳答案是：用端到端可关联的 signals 定位用户影响，以分位数和关键 slice 发现 regression，结合多种质量评估方法，并形成 rollback 与 regression-test 闭环。

---

## 资料来源与核查日期

核查日期：**2026-09-29**。

1. Anthropic Claude Platform Docs, [API errors](https://platform.claude.com/docs/en/api/errors)：错误类型、request ID、SDK transient retry 与 streaming 中途错误。
2. Anthropic Claude Platform Docs, [Rate limits](https://platform.claude.com/docs/en/api/rate-limits)：限流、headroom 与 Console monitoring；具体 limits 不作为考试记忆数字。
3. Anthropic Claude Platform Docs, [Usage and Cost API](https://platform.claude.com/docs/en/manage-claude/usage-cost-api)：组织级历史 usage/cost tracking、reconciliation、optimization 与 alerting。
4. Anthropic Engineering, [Demystifying evals for AI agents](https://www.anthropic.com/engineering/demystifying-evals-for-ai-agents)：production monitoring、user feedback、A/B、transcript review 与 human evaluation 的互补关系。
5. OpenTelemetry, [Observability primer](https://opentelemetry.io/docs/concepts/observability-primer/)：telemetry signals、SLI/SLO 与 distributed tracing。
6. OpenTelemetry, [Metrics](https://opentelemetry.io/docs/concepts/signals/metrics/) 与 [Logs](https://opentelemetry.io/docs/concepts/signals/logs/)：聚合、cardinality、稳定 schema 及 log/trace correlation。

### Guide 要求 vs 当前产品实现

- **Guide 要求**是会使用 logging 与 observability tools 持续监控系统 performance；考试重点是信号选择、端到端关联、诊断/告警与反馈闭环。
- Anthropic request/error、rate-limit、usage/cost 字段和 SDK retry 属当前实现；OpenTelemetry 是实现选择，不是 Guide 指定的唯一工具。
- 与 #07 的区别：#07 解决规模化 instrumentation/collection 架构选型；本课使用这些信号检测 performance/quality/cost drift，并驱动 Domain 4 的 evaluation 与 optimization。

### Needs verification

实施前需复核当前 Anthropic error/usage/rate-limit 字段、SDK retry 默认值、Admin API 凭证与平台可用性、各接入面的 telemetry、模型和价格；同时根据业务基线重新确定 SLI/SLO、alert windows、quality threshold、sampling、retention 和 grader calibration。上述可变细节不用于确定练习题答案。

下一步：[questions.md](questions.md)。
