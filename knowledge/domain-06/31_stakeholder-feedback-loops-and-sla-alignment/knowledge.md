# 31 · 干系人反馈闭环与 SLA 对齐管理

## Guide 对齐

- **Domain**：Domain 6 · Stakeholder Communication & Lifecycle Management（14%）
- **Exam Guide 要求**：`Manage stakeholder feedback loops and expectation alignment (including SLAs)`
- **本课目标**：把用户、业务、工程、运营、安全与合规的反馈，转化为可追踪的改进或明确的不处理决定；同时用 SLI/SLO/SLA 把“系统应该怎样表现”和“未达标时怎么办”说清楚。
- **范围边界**：本课聚焦 production feedback、expectation contract 与 service-level governance。#32 才深入完整架构文档和实施指导，#33 才覆盖 discovery 到 iteration 的完整生命周期。

真正的 feedback loop 不是“收集了 thumbs-down”，而是：

```text
Signal → Context → Triage → Decision → Change → Validation → Release → Communication → Monitoring
```

任何一段缺失都会形成“反馈黑洞”：有数据但没有 owner、有修复但没有 regression test、有发布但没有向受影响者说明结果。

---

## 1. 为什么 Claude 系统特别需要多源反馈闭环

LLM 系统具有非确定性、开放式输出和复杂上下文依赖。单一 dashboard 或满意度按钮无法回答系统是否健康：

- 用户可能觉得结果“差”，实际原因可能是 retrieval、tool、prompt、权限、UI 或产品预期；
- offline eval 覆盖已知任务，但可能漏掉真实分布、长尾语言和新型攻击；
- 生产指标能发现 error / latency，却未必知道答案是否正确；
- 用户反馈可揭示新问题，但通常稀疏、自选择且受期望影响；
- 专家 review 能判断高风险质量，却昂贵且吞吐有限；
- aggregate metric 可能掩盖只影响某个 model、prompt、tenant、语言或时间窗口的退化。

Anthropic 将 automated eval、production monitoring、A/B testing、user feedback、人工 transcript review 和 systematic human evaluation 视为互补信号，而非互相替代。一次“大家感觉变差”的报告既不能被直接忽略，也不能未经验证就触发全局回滚。

---

## 2. 先建立反馈来源地图

| 来源 | 能回答什么 | 主要盲区 |
|---|---|---|
| Explicit user feedback | 用户认为哪个交互有帮助、错误或不符合预期 | self-selection、原因模糊、可能被 UI 诱导 |
| Implicit behavior | abandon、retry、edit、copy、escalate、task completion | 行为不等于原因，需避免过度推断 |
| Support / complaint | 可复现的业务影响、语言与 expectation gap | 高频抱怨不一定代表总体分布 |
| SME / human review | domain correctness、严重性、policy nuance | 慢、贵、grader disagreement |
| Offline eval | 回归、边界案例和版本间可重复比较 | 与真实流量不匹配会产生虚假信心 |
| Production telemetry | latency、error、tool、cost、path、drift | 缺少答案 ground truth；隐私限制 |
| Incident / security report | 严重 failure、blast radius、控制缺口 | 低频但不可用平均数稀释 |
| Business outcome | resolution、conversion、handle time、rework | 有滞后，且受外部因素影响 |
| Compliance / audit | control evidence 与违反义务的风险 | 不直接说明一般用户体验 |

反馈策略必须同时覆盖 **体验、任务结果、服务可靠性、质量、安全与业务价值**。不要把“thumbs-up 高”直接等同于事实正确，也不要把 API 200 等同于任务完成。

---

## 3. Capture：反馈必须能关联完整交互上下文

一个反馈事件至少应能关联：

- `feedback_id`、时间与反馈渠道；
- `trace_id / task_id`，以及允许保留的 end-to-end trace；
- use case、tenant / user segment、语言和风险级别；
- model、prompt、policy、retrieval index、tool 与 application 版本；
- 输入/输出或经过脱敏的引用；
- retrieved documents、tool calls、latency、token / cost 和最终 outcome；
- 用户给出的 category、free text、severity 与期望结果；
- consent、retention、访问控制和数据分类；
- triage status、owner、decision 与 resolution link。

AWS 的 production feedback 指南强调，反馈只有与生成它的 interaction 和完整 trace 关联时才可用于可靠诊断；`trace_id` 能把 prompt、检索、模型、tool 和 latency 串起来。

### 隐私边界

“为了改进模型而全量保存聊天”不是默认合理方案。应根据用途最小化字段、脱敏、限制访问和 retention；高风险内容与普通产品反馈可能需要不同存储。反馈数据本身也可能包含 prompt injection、个人数据或敏感业务信息，不能未经审核直接进入 prompt、RAG 或训练集。

---

## 4. Triage：频率、严重性和证据要分开

推荐 triage 至少记录：

1. **Impact / severity**：是否造成安全、权限、财务、合规或重大业务后果？
2. **Frequency / reach**：出现多少次、影响哪些 slice？
3. **Confidence**：是否可复现？证据是 single report 还是多信号一致？
4. **Urgency**：继续运行会扩大损害吗？
5. **Ownership**：属于 prompt、retrieval、tool、model、policy、UX、platform 还是 expectation？
6. **Action class**：contain、rollback、investigate、add eval、product change、documentation、no action；
7. **Reporter expectation**：是否需要 acknowledgement、状态更新或正式 incident communication？

优先级不能只用 `frequency × votes`。一次跨租户泄露可能比一千次语气偏好更紧急；同时，少数 loud users 也不应未经 population 分析就定义全体需求。

### 建议的响应分层

- **Critical**：security/privacy/irreversible harm——立即 containment、on-call/escalation、审计保全；
- **High**：核心任务或关键客户严重退化——快速复现、release gate / rollback 判断；
- **Normal**：可验证的质量、UX 或覆盖缺口——进入有 owner 的 backlog 与 eval；
- **Observation**：证据不足——继续采样并说明验证条件；
- **Expectation gap**：系统行为符合设计但沟通不足——更新产品承诺、UI、培训或文档，而不是盲目改模型。

这些等级和时限必须由组织定义；教材不提供通用 incident SLA。

---

## 5. Close：反馈要变成“证据 → 决定 → 验证”

闭环处理可以使用以下模板：

```text
Signal:
Affected population / versions:
Observed outcome and expected outcome:
Severity / frequency / confidence:
Root cause or current hypothesis:
Decision and owner:
Containment / change:
Regression test or eval added:
Release and rollback criteria:
Reporter / stakeholder communication:
Post-release monitoring result:
```

### 将生产失败回流为 eval

一个被确认、去敏且有代表性的 failure case 应：

1. 固化输入、环境、source / tool state 和 expected outcome；
2. 写入合适的 regression slice，而不是随意混入训练数据；
3. 修复后验证该 case 以及邻近场景，避免 overfit；
4. 运行已有 suite，防止改善一个维度却损害另一个；
5. 通过 canary / A/B / staged rollout 验证真实结果；
6. 上线后继续观察原问题是否消失、是否出现新分群退化。

Anthropic 的 agent eval 实践指出，没有 eval 时团队只能等待投诉、手工复现并猜测是否回归；完整视图必须结合生产监控和真实用户信号。反馈不应只生成 ticket，还应提高未来变更的可验证性。

---

## 6. Expectation alignment：对齐的是服务契约，不是宣传口号

上线前和持续运营中，应让干系人理解：

- **Intended use / anti-scope**：系统适合和不适合做什么；
- **Quality boundary**：什么叫成功、允许怎样的不确定性、何时 abstain；
- **Authority boundary**：Claude 提供建议、draft，还是能执行外部动作；
- **Human responsibility**：哪些结果必须 review，谁是最终 decision maker；
- **Data boundary**：使用哪些 source、freshness、ACL 与 retention；
- **Service level**：availability、latency、throughput、quality 等怎样测；
- **Failure behavior**：degrade、fallback、queue、retry、escalate 还是停止；
- **Support / incident**：如何报告、多久 acknowledgement、谁负责状态更新；
- **Change communication**：model、prompt、policy、tool 或 scope 变化如何通知；
- **Shared responsibility**：供应商、平台团队、应用团队、业务 owner 各负责什么。

Expectation alignment 不是承诺“Claude 永不出错”。成熟的承诺会说明概率性质量、known limitations、人工升级和补救机制。

---

## 7. SLI、SLO、SLA：必须严格区分

| 概念 | 定义 | 示例 |
|---|---|---|
| **SLI** | 对用户关心的某个 service level 的定量测量 | `good responses / eligible responses`、端到端 p95 latency |
| **SLO** | 用 SLI 表达的目标值或范围，通常带时间窗口 | 过去 28 天，99.5% eligible requests 在 5 秒内产生首个有用结果 |
| **SLA** | 与用户/客户之间的协议，包含未达到承诺后的明确后果 | 未达到月度 availability 承诺时提供 service credit / escalation |

判断一个声明是不是 SLA 的简单问题是：**未达到时明确发生什么？** 如果没有合同或约定后果，通常只是 SLO 或目标。

Google SRE 强调 SLI 是 carefully defined quantitative measure，SLO 是其目标，SLA 则包含 missed target 的后果。即使没有对外 SLA，内部仍应定义 SLI/SLO 管理服务。

---

## 8. 一个完整 SLO 的写法

不要只写 `availability ≥ 99.9%`。至少定义：

```text
Population / scope:
Good event:
Valid event / denominator:
SLI data source:
Target:
Rolling or calendar window:
Exclusions:
Segmentation:
Owner:
Alert / action policy:
Review cadence:
```

### 示例

> 对已认证员工在生产环境提交的、输入有效且不属于计划维护窗口的政策问答请求，以 gateway 接收至 UI 获得首个可读结果为边界；过去滚动 28 天中，99% 在 4 秒内完成该阶段。按地区和 retrieval path 分群；由 Platform owner 负责。当 error-budget burn 超过约定阈值时暂停非必要 latency 变更并调查。

数值只是示范结构，不是推荐默认值。

### 分母与 service boundary

- 客户取消请求算失败还是排除？
- 无效输入、拒绝请求、上游超时如何分类？
- 测 server latency 还是用户端到端体验？
- streaming 的 TTFT 达标，但最终任务未完成是否算成功？
- tool 执行成功但业务 outcome 未写入是否算 good event？

分母不清，团队就会在事故发生后争论指标，而不是解决问题。

---

## 9. LLM 系统的服务目标不只 availability

传统 operational SLI 仍然必要：

- availability / successful request rate；
- TTFT、time to first useful result、end-to-end p95/p99；
- timeout、rate-limit、retry、queue wait；
- tool / retrieval dependency success；
- throughput、cost per successful task。

但 LLM 服务还需要 quality / risk objectives：

- task completion / accepted resolution；
- grounded claim、citation support、schema adherence；
- correct abstention / escalation；
- unauthorized action、data leakage、severe harmful output；
- slice-specific quality 与 drift。

质量信号常无法对每次请求实时获得 ground truth，可采用经过校准的抽样 review、delayed outcome、deterministic checks、LLM grader 与人工 adjudication。必须标明 grader、sample、confidence 和 delay。

### 不要把所有指标都写进 SLA

SLA 应限于双方理解、可客观量测、责任边界明确且能支持后果的承诺。内部 SLO 可以比外部 SLA 更严格，为检测和恢复留出 buffer。开放式生成的“100% 准确”既不现实，也难以客观判定；更合理的是定义适用任务、hard safety constraints、抽样质量门槛与人工升级。

---

## 10. Error budget：把可靠性与变更速度变成共同规则

若 SLO 为 99.9% good events，则 error budget 是 0.1% non-good events（在同一明确定义的窗口和分母内）。它的作用不是允许团队随意失败，而是事先约定：

- budget 健康时，团队可按正常策略发布和实验；
- burn 过快时，提高 review、减慢 rollout 或停止非必要变更；
- budget 耗尽时，按 policy 优先处理可靠性；
- 重大单次事件即使未耗尽总体 budget，也可能触发 postmortem 或 hard gate。

Google SRE 的示例 policy 将 error budget 作为 feature velocity 与 reliability 的共同控制机制，而非惩罚。具体 burn rate、冻结条件与例外必须由组织和 executive backing 事先确认。

### LLM 风险不能全部“预算化”

不要让严重 security、privacy、compliance 或不可逆伤害被总体质量 budget 抵消。这些通常应是独立的 hard guardrail / incident trigger。Error budget 适合可接受的 service-level miss，不是购买高严重性违规的额度。

---

## 11. 治理结构：谁听、谁判断、谁行动、谁被告知

为每类反馈和服务目标定义：

- **Signal owner**：维护采集质量与 dashboard；
- **Triage owner**：分类、去重、确定 severity；
- **Domain adjudicator**：判断 expected outcome；
- **Decision owner / DRI**：决定修复、拒绝、回滚或接受风险；
- **Implementation owner**：完成 change 与 tests；
- **Service owner**：对 SLO/error budget 负责；
- **Communication owner**：向用户、客户和管理层更新；
- **Escalation path**：当 product、SRE、security 或 business 不一致时由谁裁决。

### Cadence 必须匹配信号

- Critical incident：实时；
- operational health 与 error-budget burn：持续监控并按告警响应；
- feedback triage：固定、可预期节奏；
- quality / risk review：按变更和风险安排；
- business / stakeholder review：按决策周期回顾趋势、承诺和 trade-offs。

频率没有通用答案，但“所有反馈等季度会议再看”显然无法管理安全事件；“每个 thumbs-down 都立刻改 prompt”则会制造振荡和回归。

---

## 12. 防止反馈驱动的错误优化

1. **Self-selection bias**：主动反馈者不代表所有用户；
2. **Vocal minority**：高频发声不能替代 impact / population 分析；
3. **Metric gaming**：为了减少 thumbs-down 而过度拒答或迎合；
4. **Expectation contamination**：UI 宣传过度导致满意度下降，根因未必是模型退化；
5. **Aggregate masking**：总体稳定但某版本、语言或 tool path 失败；
6. **Recency bias**：单次最新事件推翻长期证据；
7. **Automation bias**：reviewer 因系统建议而降低独立判断；
8. **Unsafe reuse**：把含敏感数据或恶意内容的反馈直接加入训练/上下文；
9. **No negative feedback**：用户已经流失或不知道如何报告，不等于系统健康；
10. **Closed ticket ≠ closed loop**：修复未验证、未发布或未观察效果。

Anthropic 的一次质量事件说明了 aggregate signal 的局限：不同变更影响不同 traffic slice 和时间窗口时，整体看起来只是模糊、波动的体验下降，内部测试也可能一开始无法复现。因此 feedback 必须与版本、slice、trace 和时间关联。

---

## 13. 场景示例：内部客服助手“变慢又变笨”

### 信号

多个客服反馈助手最近变慢、建议质量下降，但总体 API error rate 和平均 latency 正常。

### 正确闭环

1. 关联 feedback 与 trace，按 model/prompt/index/tool、地区、任务类型和时间切片；
2. 发现仅新 prompt + 长会话 + 某 tool path 的 p99、人工改写率退化；
3. SME review 确认不是单纯期望差异，并将失败案例去敏后加入 regression eval；
4. 先回滚该 slice，保留其他正常 traffic；
5. 修改 prompt / context strategy，在离线 suite 检查质量、安全、token、latency；
6. canary 发布并比较原 slice 的 accepted resolution 和 p99；
7. 更新 incident / stakeholder communication，说明影响范围、修复和后续监控；
8. 若 latency SLO / error-budget policy 被触发，按预先约定调整发布节奏。

错误做法包括：因平均值正常而否定用户、因投诉出现就全局更换模型、或只降低延迟却不验证质量回归。

---

## 14. 考试决策模板

遇到 feedback / SLA 场景题，依次判断：

1. signal 来自哪里，是否存在 bias 或缺少 ground truth？
2. 能否关联 trace、版本、slice、时间和业务 outcome？
3. severity、frequency、confidence 和 blast radius 是否分开？
4. 谁负责 triage、adjudication、decision、change 与 communication？
5. failure 是否进入 regression eval，并验证邻近场景与整体 suite？
6. SLI 的 numerator、denominator、boundary 和 data source 是否清楚？
7. SLO 是否含 target、window、owner 和 action policy？
8. SLA 是否真的包含未达标后果，且责任边界可控？
9. critical security/safety failure 是否独立于可消耗的 error budget？
10. reporter 和 stakeholders 是否收到 closure，post-release 是否继续监控？

最佳答案通常会把多源反馈与 trace/eval/owner 连接起来，并用用户导向、可操作的 SLO 管理预期。只收集反馈、只看平均 dashboard，或承诺 100% 准确，都不是完整闭环。

---

## Guide 要求与当前实现细节

Guide 要求掌握 stakeholder feedback loops、expectation alignment 和 SLA 管理。它不要求采用某个云厂商的 feedback service，也不要求背某组 SLO 数值。考试重点是信号互补、闭环责任、可测 service boundary、后果和沟通。

### Needs verification

实施前需重新核查当前 Claude model/API/tool/agent 行为、usage 与 telemetry 字段、rate limits、pricing、数据处理条款和平台支持。业务 SLO/SLA、error-budget policy、incident severity、反馈 retention、sampling、grader calibration 和响应时限必须由目标组织、合同、实际 workload 与风险共同确定。

---

## 核查来源（2026-10-07）

- Anthropic, [Demystifying evals for AI agents](https://www.anthropic.com/engineering/demystifying-evals-for-ai-agents)：eval、production monitoring、A/B、user feedback 和人工 review 的互补关系。
- Anthropic, [An update on recent Claude Code quality reports](https://www.anthropic.com/engineering/april-23-postmortem)：用户反馈、traffic slices、版本变更和 aggregate signal 难以复现的问题。
- Google SRE, [Service Level Objectives](https://sre.google/sre-book/service-level-objectives/)：SLI、SLO、SLA 的定义、用户导向指标和目标选择。
- Google SRE Workbook, [Example Error Budget Policy](https://sre.google/workbook/error-budget-policy/)：以 error budget 协调可靠性、发布节奏和升级责任。
- AWS Prescriptive Guidance, [Architecting the production feedback loops](https://docs.aws.amazon.com/prescriptive-guidance/latest/gen-ai-lifecycle-operational-excellence/prod-monitoring-feedback.html)：反馈与 trace 关联、集中 pipeline 和 HITL。
- AWS Well-Architected Generative AI Lens, [Generative AI lifecycle](https://docs.aws.amazon.com/wellarchitected/latest/generative-ai-lens/generative-ai-lifecycle.html)：监控、用户反馈和持续改进。
