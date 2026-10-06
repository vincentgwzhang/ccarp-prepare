# 30 · 架构决策与权衡的沟通方法（ADR 等）

## Guide 对齐

- **Domain**：Domain 6 · Stakeholder Communication & Lifecycle Management（14%）
- **Exam Guide 要求**：`Communicate architectural decisions and trade-offs`
- **本课目标**：把 Claude 系统的重要架构选择写成可复核、可追溯、能被不同干系人正确执行的决策，而不是只留下结论或会议记忆。
- **范围边界**：本课聚焦一次架构决策的 context、options、evidence、trade-offs、decision 和 consequences。#31 才深入持续反馈与 SLA 对齐，#32 才深入完整架构文档和实施指导，#33 才讨论端到端生命周期。

架构沟通的价值不在于“证明架构师是对的”，而在于让团队知道：**为什么现在必须决定、哪些约束不可违反、评估过哪些真实选项、选择获得了什么、付出了什么、何时应重新决策**。

---

## 1. 决策、设计与沟通是三件不同的事

- **Requirement**：系统必须满足什么结果与约束；
- **Architecture decision**：面对这些 constraints，选择哪一种方向；
- **Detailed design**：组件、接口、数据流和实现如何落地；
- **ADR（Architecture Decision Record）**：记录一个架构上重要的决定及其理由、备选项和后果；
- **Runbook / implementation guide**：怎样部署、操作、诊断和恢复。

ADR 不是完整设计文档，也不是会议纪要。会议纪要记录“谁说了什么”，ADR 记录“团队基于什么 context 做出了什么决定”。AWS 强调 ADR 最小应包含 context、decision 与 consequences；Microsoft 进一步建议记录被排除的 alternatives、trade-offs、confidence 和状态。

### 哪些决策值得 ADR

满足下列任一条件通常就值得记录：

- 影响系统结构、接口、依赖或关键 quality attributes；
- 对 security、privacy、reliability、cost、latency 或 operability 有长期影响；
- 难以撤销、迁移成本高或会造成组织锁定；
- 多个团队需要共同遵循；
- 未来很可能有人重新问“为什么不采用另一方案”；
- 基于重要但可能变化的假设或低置信度证据。

局部变量命名或易于撤销的小实现选择不必写成 ADR。否则 decision log 会被噪声淹没。

---

## 2. 一个可用 ADR 的最小结构

| 字段 | 应回答的问题 |
|---|---|
| ID / Title | 一眼看出要决定什么，避免使用“AI architecture”之类空标题 |
| Status | Proposed、Accepted、Rejected、Superseded 等 |
| Date / Owner / Deciders | 谁推动、谁有决定权、何时生效？ |
| Decision question | 需要做出的单一选择是什么？ |
| Context | 当前问题、业务目标、baseline 和触发条件是什么？ |
| Drivers / constraints | 哪些指标、法规、期限、预算和组织能力决定优先级？ |
| Assumptions / unknowns | 哪些尚未证实？置信度多高？ |
| Options considered | 至少列出真实可行选项，包括维持现状；不要制造 strawman |
| Evidence | eval、benchmark、threat model、成本或 spike 结果是什么？ |
| Decision | 具体选择及适用范围是什么？ |
| Consequences | 正面、负面与中性后果；谁承担成本？ |
| Risks / mitigations | 哪些 residual risk 被接受，哪些控制必须落地？ |
| Validation / review | 用什么验收？出现什么信号时复审？ |
| Links | requirements、diagram、risk、eval、implementation tasks 的链接 |

推荐标题使用 decision statement，例如：

> ADR-014：客服退款采用确定性 workflow，Claude 仅负责解释与信息抽取

而不是：

> ADR-014：客服 AI

---

## 3. Trade-off 的核心：把“获得什么、牺牲什么”说完整

架构通常不存在脱离 context 的“最佳方案”。Trade-off communication 至少包含：

1. **Decision drivers**：真正影响选择的维度；
2. **Direction**：某方案改善哪个维度；
3. **Price**：因此恶化、增加或失去什么；
4. **Boundary**：收益在哪些 workload / risk class 下成立；
5. **Evidence**：结论来自实际 eval，还是尚待验证的假设；
6. **Owner**：谁接受后果和 residual risk；
7. **Revisit trigger**：何时原来的平衡不再成立。

可用一句话强迫自己暴露代价：

> 我们选择 **X**，因为在 **C 情境**中它使 **A** 达到门槛；代价是 **B**，由 **M** 控制。当 **T** 发生时重新评估。

例如：

> 对高频、规则明确的客服分类使用较简单的模型和固定 routing，因为它在代表性 eval 上满足质量门槛并降低 p95 与成功任务成本；代价是复杂边界案例能力不足，因此由 confidence/rule gate 升级到更强路径。当业务分布或质量 slice 低于阈值时复审。

注意：具体模型、价格和阈值会变化；稳定的决定是 selection policy、evidence 和 guardrails，而不是把当前产品名称写成永恒事实。

---

## 4. 用 decision matrix 辅助，不让分数代替判断

典型 Claude 架构决策可按下列维度比较：

| 维度 | 典型问题 | 证据 |
|---|---|---|
| Task quality | 是否满足准确性、groundedness、completion 与 edge cases？ | 代表性 eval、人工校准 |
| Safety / security | 是否守住授权、数据和工具动作边界？ | threat model、red-team、policy tests |
| Latency | p95/p99、TTFT 与完整任务耗时怎样？ | 目标负载 benchmark |
| Cost | 每个 accepted successful task 的端到端成本？ | token/tool/retry/human review 数据 |
| Reliability | timeout、fallback、dependency failure 怎样处理？ | failure injection、SLO 数据 |
| Operability | 能否观测、调试、回滚和处理 incident？ | telemetry / runbook review |
| Complexity | 组件、状态、耦合和认知负担增加多少？ | dependency / ownership map |
| Reversibility | 更换模型、检索或平台的迁移成本？ | interface boundary、exit plan |
| Compliance | 数据区域、retention、审计与责任边界？ | control/evidence mapping |

可以使用 weighted matrix，但必须谨慎：

- 权重来自已确认的 requirements，不是架构师个人偏好；
- hard guardrail 不能被总分抵消，例如严重数据泄露不能靠低成本“加分”覆盖；
- 保留原始测量和不确定性，不能只展示一个精确到小数点后的总分；
- 先淘汰不满足硬约束的方案，再比较 Pareto trade-off；
- sensitivity check：小幅改变权重就翻转结论，说明决策脆弱，应补充证据或明确价值判断。

---

## 5. Claude 系统中常见的决策类型

### 5.1 Workflow vs agent

Anthropic 区分：workflow 由预定义代码路径编排，agent 由模型动态决定过程与工具使用。Agent 的灵活性通常以 latency、cost、调试难度和更大行动风险为代价；规则清晰的任务通常优先使用更可预测的 workflow。

ADR 应写清：任务变异性、可预先编码程度、允许 autonomy、最大 steps/budget、stop condition、权限与 HITL，而不只写“使用 agent”。

### 5.2 Model / routing strategy

稳定决策不是“永远使用某型号”，而是：

- 哪些任务 slice 需要怎样的 capability；
- 哪些 quality / safety thresholds 是硬门槛；
- latency、cost 与 quality 如何权衡；
- 何时 route、escalate、fallback 或拒绝；
- 如何用实际 prompts、data 和 edge cases 重测。

Anthropic 当前的 model-selection 指南本身也把选择描述为 capability、speed、cost 之间的平衡，并要求使用 use-case-specific benchmark 与实际 prompts/data 比较。型号、参数、价格属于可变实现细节。

### 5.3 RAG / context strategy

需要沟通的不只是“采用向量库”，而是：

- authoritative sources、freshness 和 ACL；
- lexical / semantic / hybrid 与 reranking 的收益和成本；
- top-K、context size 对 recall、latency 和 distraction 的影响；
- 无证据时 abstain 还是允许 general knowledge；
- reindex、删除和 rollback 策略。

### 5.4 Tool autonomy

“支持退款工具”并非完整决定。ADR 应包含 read/draft/write/irreversible 分类、credential boundary、server-side validation、approval、idempotency、audit 与 recovery。增加 autonomy 获得吞吐和覆盖率，也可能增加越权与不可逆副作用。

### 5.5 Platform / integration

比较 direct API、cloud provider、MCP、自建 connector 或托管 agent 时，应讨论 portability、identity、network boundary、data handling、observability、operational ownership 和 exit cost，而非只比较 feature checklist。

---

## 6. Evidence：把意见变成可复核判断

架构沟通中的证据层级可以是：

1. **Business requirement / risk tolerance**：为何某维度重要；
2. **Representative eval**：实际 prompts、data、工具和边界案例；
3. **Operational benchmark**：目标并发、p95、retry、成功任务成本；
4. **Security / safety evidence**：threat model、red-team、权限与 failure tests；
5. **Prototype / spike**：验证未知假设；
6. **Official product evidence**：能力、限制、生命周期和数据条款；
7. **Expert judgment**：可以使用，但要标记假设、confidence 和验证计划。

LLM 系统尤其不能用一次漂亮 demo 代替 evidence。Anthropic 指出 eval 可以明确预期行为、建立 latency/token/cost/error baseline，并减少团队对 edge cases 的不同理解。因此 ADR 中应链接 eval snapshot、dataset/version、环境和日期，而不是只写“测试效果很好”。

### 事实、假设与偏好的标记

- **Fact**：在明确版本、环境和数据上观察到；
- **Assumption**：尚未验证，但当前决策依赖它；
- **Preference**：组织愿意接受的价值取舍；
- **Constraint**：必须满足，违反即淘汰；
- **Unknown**：需要 spike 或 owner 决定。

把五者混写，是架构争论反复发生的常见原因。

---

## 7. 面向不同受众分层沟通

同一决定应保持同一事实基础，但表达深度不同。

| 受众 | 首先关心什么 | 合适表达 |
|---|---|---|
| Executive / sponsor | 业务结果、风险、成本、时间与残余不确定性 | 一页 decision brief、明确 ask |
| Product / domain | 用户流程、质量门槛、例外、人工升级 | scenario、acceptance / rejection examples |
| Security / legal | 数据流、授权、威胁、控制、证据和责任 | threat/control mapping、residual risk |
| Engineering | interfaces、state、failure modes、migration、test | ADR + diagrams + contracts |
| Operations / support | SLO、telemetry、fallback、incident、rollback | operational consequences、runbook links |

分层不是给不同人不同结论。Executive summary 省略实现细节，但不能隐藏负面 consequence、关键假设或不可接受风险。

---

## 8. 决策会议的高效结构

1. **Pre-read**：提前发送 Proposed ADR，而不是在会议里第一次展示 40 页 slides；
2. **Decision question**：一句话说明今天需要决定什么；
3. **Non-negotiables**：先确认 requirements 与 hard constraints；
4. **Options**：只讨论真实可行方案和维持现状；
5. **Recommendation**：说明 gain、cost、risk、confidence 与 evidence；
6. **Challenge assumptions**：争论事实和价值权重，而不是职位；
7. **Decision / dissent**：记录 accepted、rework 或 rejected 及理由；
8. **Commitments**：owner、actions、validation、rollback 与 review trigger；
9. **Publish**：更新状态并放入团队可访问的 decision log。

Consensus 不一定意味着所有人都偏好同一方案。关键是正确的 decider 在听取必要角色后作出决定，并让 dissent、accepted risk 与执行责任可见。

---

## 9. ADR 生命周期：不要改写历史

常见状态：

```text
Draft / Proposed → Accepted
                 ↘ Rejected
Accepted → Superseded by a new ADR
```

Accepted ADR 应作为 append-only history 保留。如果 assumptions、产品能力或 requirements 改变，应创建新 ADR，并链接 `supersedes / superseded by`，而不是静默改写旧决定。这样 incident review 才能知道团队在当时掌握什么证据。

建议设置复审触发器，而不只是固定日期：

- quality、安全或 SLA guardrail 连续失守；
- workload / data distribution 明显变化；
- 成本超过预算区间；
- provider feature、model、API 或支持状态发生 breaking change；
- 法规、数据分类、威胁模型改变；
- 原假设被证伪；
- migration benefit 已超过 switching cost。

AWS 与 Microsoft 都建议保留历史并用新 ADR supersede 已接受记录，这使决策的时间语境可追溯。

---

## 10. 场景示例：退款场景选择 workflow 而非开放 agent

### Decision question

客服退款应由开放 agent 自主判断并执行，还是由确定性 workflow 编排，Claude 仅负责意图识别、信息抽取和政策解释？

### Drivers

- 错误退款不可接受且必须可审计；
- 资格规则和审批链明确；
- 自然语言输入变化大；
- p95 与人工 handle time 有目标；
- 高金额和来源冲突必须人工批准。

### Options

1. 维持全人工；
2. 开放 agent 自主选择并调用退款工具；
3. 确定性 workflow + Claude 处理非结构化步骤 + 服务端校验/审批。

### Decision

选择 3。它保留 Claude 在语言理解上的价值，同时让资格、金额、授权、幂等和审批由确定性代码强制执行。

### Consequences

- **Positive**：可预测、易审计、降低错误工具动作风险；
- **Negative**：新增政策分支需要修改 workflow，无法像开放 agent 一样灵活探索；
- **Residual risk**：抽取错误仍可能引导错误候选路径；由服务端重新校验、人工 gate 和 regression eval 控制；
- **Review trigger**：如果大量合法案例因 workflow 缺少分支而升级，且 agent 方案在权限约束下通过质量/风险门槛，则重新评估。

这个 ADR 不是在宣称 workflow 永远优于 agent；它说明在当前 requirements、证据和风险边界下，为何它是更合适的选择。

---

## 11. 典型 failure modes

1. **Conclusion-only**：只有“使用 RAG/agent/某模型”，没有 context 和 rationale。
2. **Option theater**：备选项明显不可行，只为证明预设方案正确。
3. **Benefit-only pitch**：隐藏 latency、cost、锁定、运营或 residual risk。
4. **Feature checklist**：比较厂商功能，却不关联 workload 与 requirements。
5. **Demo-driven decision**：把少量成功 demo 当作 production evidence。
6. **Composite-score laundering**：用总分掩盖 security / compliance hard failure。
7. **False precision**：不可靠权重与估算产生看似科学的小数排名。
8. **No time context**：未记录 model/API/data/version 和决策日期。
9. **No reversibility**：没有 migration、fallback、rollback 或 exit strategy。
10. **No owner / decider**：会议达成“大家觉得可以”，却无人接受风险。
11. **Silent rewrite**：新决定覆盖旧 ADR，历史理由和审计链消失。
12. **ADR as design dump**：记录过长、decision 不清楚，团队无法快速使用。

---

## 12. 考试决策模板

遇到“如何沟通架构选择”类题目，依次检查：

1. 这是 architecturally significant decision 吗？
2. decision question、drivers、hard constraints 和 decider 是否明确？
3. options 是否真实，包括维持现状？
4. 是否使用目标 workload 的 eval / benchmark，而不是泛化宣传？
5. gain 与 cost 是否都明确，hard guardrail 是否独立？
6. consequences、residual risk、owner 和 mitigation 是否可见？
7. assumptions、confidence、版本与日期是否记录？
8. 是否有 validation、rollback 和 revisit trigger？
9. Accepted ADR 是否通过新记录 supersede，而非直接改写？

在答案选项中，优先选择“基于 requirements 和证据说明多方案 trade-off，并记录决定、后果和复审条件”的方案。只展示最佳情况、使用更强模型替代论证、或追求无异议会议，通常不是最佳答案。

---

## Guide 要求与当前实现细节

Guide 要求掌握架构决策与 trade-off 的沟通，不要求使用某一种 ADR 模板。ADR、decision matrix、diagram、benchmark 和 briefing 是沟通载体；核心是 decision quality、traceability、audience fit 和 lifecycle。

### Needs verification

实际决策前须重新核查当前 Claude model/API/tool/agent 能力、pricing、limits、数据条款、区域和平台支持，并在目标 prompts、data、tools、并发与风险环境重新 benchmark。教材中的产品示例和文档行为不是永久能力承诺，也不应用于确定未验证的生产答案。

---

## 核查来源（2026-10-06）

- Anthropic, [Building effective agents](https://www.anthropic.com/engineering/building-effective-agents)：workflow/agent 边界、最小充分复杂度及 latency/cost/performance trade-off。
- Anthropic Claude Docs, [Choosing the right model](https://platform.claude.com/docs/en/about-claude/models/choosing-a-model)：以 use-case benchmark 比较 capability、speed、cost，而不是抽象选择模型。
- Anthropic, [Demystifying evals for AI agents](https://www.anthropic.com/engineering/demystifying-evals-for-ai-agents)：用 eval 明确预期、建立基线并作为跨角色沟通信号。
- AWS Prescriptive Guidance, [Architectural decision record process](https://docs.aws.amazon.com/prescriptive-guidance/latest/architectural-decision-records/adr-process.html)：ADR 的 context/decision/consequences、owner、review 与 decision log。
- AWS Prescriptive Guidance, [Best practices for ADRs](https://docs.aws.amazon.com/prescriptive-guidance/latest/architectural-decision-records/best-practices.html)：分布式 ownership、历史保留、集中存储和定期 review。
- Microsoft Azure Well-Architected Framework, [Maintain an architecture decision record](https://learn.microsoft.com/en-us/azure/well-architected/architect-role/architecture-decision-record)：重要选项、trade-offs、confidence、append-only 与 supersede 机制。
