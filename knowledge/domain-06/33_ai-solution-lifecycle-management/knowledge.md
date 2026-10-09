# 33 · AI 方案生命周期模型：发现 → 设计 → 交接 → 监控 → 迭代

## Guide 对齐

- **Domain**：Domain 6 · Stakeholder Communication & Lifecycle Management（14%）
- **Exam Guide 要求**：`Support lifecycle phases (discovery, design, handoff, monitoring, iteration)`
- **本课目标**：把 AI 方案视为持续演化的产品与受治理的 socio-technical system，为每个阶段定义决策、证据、owner、进入/退出条件以及反馈路径。
- **范围边界**：#29 深入 structured discovery，#30 讲架构决策沟通，#31 讲反馈闭环与 SLA，#32 讲文档和实施指导；本课不重复各专题，而是把它们串成可运行、可交接、可监控、可迭代、也可停止的生命周期。

生命周期不是“上线即结束”的 waterfall。更准确的模型是一组带反馈边的状态和决策门槛：

```text
Discovery ⇄ Design / Prototype ⇄ Validation
                                ↓
                         Handoff / Release
                                ↓
                       Monitor / Operate
                                ↓
                             Iterate
                                └────────→ 重新进入 discovery / design / validation

Governance、risk、evidence、ownership、version lineage 贯穿所有阶段。
```

不同组织可以合并或重命名阶段。考试判断重点不是背固定流程，而是识别：当前处于什么阶段、缺少什么证据、谁有权决策、下一步应进入哪个受控环节。

---

## 1. 为什么 Claude / LLM 方案需要生命周期治理

传统软件也需要 lifecycle management，但 LLM 系统的行为还受以下可变因素共同决定：

- model 与 provider behavior；
- system prompt、examples 与生成参数；
- context、RAG corpus、chunk/index 与权限过滤；
- tool schema、实现、授权和 side effect；
- orchestration、memory、state 与 human checkpoint；
- user population、输入分布和攻击方式；
- policy、业务流程、法规与风险容忍度。

所以“代码没有变化”并不代表系统行为没有变化。知识库刷新、tool 权限扩大、用户群改变或 model alias 指向新版本，都可能让原有评估失效。生命周期治理要持续维护一条证据链：

```text
Business outcome / risk
  → requirement / architecture decision
  → implementation artifact versions
  → evaluation and release evidence
  → production telemetry and feedback
  → next decision / change / retirement
```

NIST AI RMF 将 Govern、Map、Measure、Manage 描述为彼此作用、可按需要应用的 functions，而非必须按顺序打勾的 checklist。Governance 是跨阶段能力，风险管理也贯穿整个 AI lifecycle。这与考试中的阶段模型并不冲突：阶段帮助组织工作，治理帮助每个阶段做出可追溯决策。

---

## 2. Discovery：先证明问题值得解决

Discovery 的输出不是“决定使用哪个 Claude model”，而是可验证的问题和约束。

### 2.1 必须澄清的内容

- **Problem / outcome**：谁在什么 workflow 中遇到什么问题？改善什么业务结果？
- **Intended use / anti-scope**：系统允许做什么，明确不允许做什么？
- **Stakeholders**：用户、受影响群体、sponsor、domain expert、security、legal、operations、reviewer 和最终 accountable owner。
- **Current baseline**：当前质量、周期、成本、人工步骤和已知失败率；没有 baseline 就无法证明价值。
- **Task characteristics**：任务是否开放、可分解、需要外部知识、是否有客观答案、错误是否可逆。
- **Data / context / tools**：source of truth、权限、freshness、敏感性、tool side effects 和系统依赖。
- **Autonomy / human role**：Claude 建议、起草、执行可逆动作，还是触发高影响不可逆动作？
- **Quality attributes**：accuracy、latency、availability、cost、safety、security、privacy、explainability。
- **Success / guardrail metrics**：什么结果证明值得继续？什么 failure 必须阻断发布？
- **Feasibility / risk**：技术、数据、组织、法律、运营和 adoption 风险。

### 2.2 Discovery exit gate

进入 design 前至少需要：

1. 有 owner 的 problem statement 和 target users；
2. 可测 success criteria、guardrails 和 baseline；
3. 主要数据、tool、human 和 trust boundaries 已识别；
4. 高风险假设与最小验证计划已列出；
5. 明确 `continue / pivot / stop` 的判断标准。

发现阶段允许结论是“不使用 LLM”“只使用 deterministic workflow”或“先修复数据质量”。为了证明已经投入而强行进入 prototype，是典型 sunk-cost failure。

---

## 3. Design 与 Prototype：用最小方案验证最大不确定性

Design 把 discovery contract 转成可测试的 architecture hypothesis。应选择 **最简单且足以满足约束** 的模式，而非默认构建 fully autonomous agent。

### 3.1 设计决策

- workflow、agentic 或 augmented LLM；
- model selection criteria、routing 和 fallback；
- prompt/context/RAG/tool contracts；
- identity、authorization、data flow 和 trust boundaries；
- state、memory、idempotency、timeout、retry 和 compensation；
- guardrail、HITL、abstention 和 escalation；
- evaluation dataset、graders、threshold 和 critical slices；
- telemetry、version correlation、rollout 与 rollback。

### 3.2 Prototype 的正确作用

Prototype 是为了降低不确定性，例如：

- Claude 能否从代表性材料中提取所需信息？
- retrieval 能否保留 tenant ACL？
- tool contract 是否让模型稳定选择正确参数？
- human reviewer 是否能在给定证据下可靠审批？
- 端到端 latency 和 unit economics 是否有可行区间？

一个精心挑选的 demo 只证明“这几个例子能运行”，不证明 production readiness。Prototype 常刻意省略 HA、完整 authorization、observability、red-team、data deletion 和 rollback；这些欠账必须显式记录，不能在演示成功后悄悄变成生产设计。

### 3.3 Design exit gate

- 关键 ADR 记录 context、options、trade-offs、evidence 和 residual risk；
- threat/data-flow、human responsibility 和 failure behavior 已设计；
- evaluation plan 覆盖 normal、boundary、adversarial 与 critical slices；
- prototype 已回答最高风险假设，剩余未知项有 owner；
- production hardening、handoff 和 operations 工作已纳入计划。

---

## 4. Validation 与 production-readiness gate

虽然 Guide 用五个核心阶段表达 lifecycle，但 design 到 handoff 之间通常需要明确的 validation gate。它不是额外考试范围，而是防止未验证 prototype 直接交接的必要控制。

### 4.1 证据组合

| 维度 | 代表性证据 |
|---|---|
| Functional / quality | representative eval、slice analysis、failure taxonomy、human review |
| Safety / security | abuse cases、direct/indirect prompt injection、authorization negative tests、red team |
| Reliability | timeout、rate limit、partial failure、retry、idempotency、fallback、load test |
| Privacy / compliance | data inventory、purpose/access/retention、control mapping、approval evidence |
| Operations | SLI/SLO、dashboard、alerts、runbook、on-call、incident drill |
| Business / human | workflow acceptance、training、override/escalation、baseline comparison |
| Release | immutable/versioned candidate、canary plan、rollback trigger 与 rollback unit |

LLM evaluation 不是一次性的 pass/fail 仪式。Anthropic 的 agent evaluation 实践强调，eval 可以在上线前暴露问题，而且其价值会随着系统生命周期中的持续使用而累积；单一 automated score 也不能替代 human、production 和 deterministic checks 的组合。

### 4.2 Release candidate 必须可重现

生产 readiness 结论要绑定完整 version bundle，而不只是 application commit：

```text
application / orchestrator version
model ID or governed alias policy
system prompt + template + examples version
tool schema + implementation + permission policy version
RAG corpus snapshot + parser/chunk/index/embedding configuration
guardrail / policy / classifier version
evaluation dataset + grader + threshold version
infrastructure / deployment configuration
```

否则“同一个测试通过的系统”可能无法重建，也无法解释生产 degradation 来自哪次变化。

---

## 5. Handoff 与 Release：责任被接受，而不只是文件被发送

Handoff 是责任、知识、权限和证据的显式转移。把 repository 链接发给 operations 不等于完成交接。

### 5.1 Handoff package

- current architecture、critical runtime/failure scenarios 与 ADR；
- deployment topology、dependencies 和 external-provider assumptions；
- prompt/context/tool/eval contracts 与 version registry；
- SLI/SLO、dashboards、alerts、trace/query 方法；
- runbook、escalation path、incident classification 与 communication template；
- known limitations、accepted/residual risks 和禁止使用范围；
- rollout、feature flag、rollback、fallback 与 data recovery；
- training、human review instructions 和 access provisioning；
- RACI / owner、support window、acceptance checklist 与 sign-off。

### 5.2 Release 策略

优先使用受控暴露面：internal pilot、shadow、canary、small cohort、feature flag 或 progressive rollout。每一步都应有：

- target population 和时间窗口；
- release criteria 与 hard guardrail；
- telemetry / feedback capture；
- pause、rollback 和 escalation trigger；
- 有权继续、暂停或回退的人。

CI/CD、artifact versioning、infrastructure as code 和自动化 checks 能缩短安全迭代周期；但自动化发布不能替代风险审查，尤其是 autonomy、data boundary 或 irreversible tool action 发生变化时。

### 5.3 Handoff exit gate

只有当 receiving owner 明确接受、所需访问已验证、监控和告警可用、runbook 被演练、rollback 可执行、known risks 已签收，handoff 才算完成。架构师可以继续提供支持，但不能成为未声明的永久单点 on-call。

---

## 6. Monitoring：同时观察系统、模型行为和业务后果

Production 是新的证据来源，不是 lifecycle 终点。

### 6.1 四层信号

| 层次 | 示例 |
|---|---|
| Technical / operational | availability、error、rate limit、timeout、p50/p95/p99 latency、queue、tool failure |
| Quality / safety / security | task success、groundedness、abstention、policy violation、injection signal、unauthorized action |
| Cost / efficiency | tokens、tool/API spend、cost per successful task、review load、rework |
| Business / human | completion time、adoption、escalation、override、complaint、harm、workflow outcome |

总体平均值可能掩盖关键 failure。应按 user cohort、language、task type、risk class、data source、model/prompt/tool/index version 等 slices 分析，并遵守 privacy 和 cardinality 限制。

### 6.2 可行动的反馈记录

Feedback 应尽可能关联：

- trace / request / conversation identifier；
- user/task/risk slice；
- version bundle；
- relevant input、retrieved context、tool calls、output 与 human action；
- severity、reproducibility、owner 和 disposition。

不是所有 production signal 都应立即改 prompt。先区分 incident、systemic defect、data drift、retrieval freshness、tool/API change、training gap 或孤立偏好。严重 safety/security/privacy failure 先 containment 和 escalation，再进入普通 iteration。

---

## 7. Iteration：把证据转化为受控变化

一个健康的 iteration loop：

```text
Signal → Triage → Hypothesis → Eval/update → Candidate change
       → Regression + risk review → Staged rollout → Post-release observation
```

### 7.1 先修测试，再修系统

当真实 failure 暴露 coverage gap，应先把它（经去敏和适当变体）加入 regression dataset，再修改 prompt、retrieval、tool 或 orchestration。否则修复可能只对单例有效，下次又会回归。

### 7.2 根据变化影响选择复验范围

并非所有变化都需要完全相同的审批，但“看起来很小”不能成为跳过验证的理由。

| 变化 | 常见影响 | 至少考虑的复验 |
|---|---|---|
| Prompt / examples | behavior、format、refusal、tool choice | relevant eval slices、schema、安全回归 |
| Model / alias | quality、latency、cost、behavior | representative full regression、capacity、rollback |
| Tool schema / permission | action selection、side effect、authorization | contract/negative tests、HITL、audit、compensation |
| RAG source / parser / index | freshness、recall、ACL、citation | ingestion/index checks、retrieval slices、tenant isolation |
| User population / use case | distribution、language、harm profile | new cohort data、human factors、risk reassessment |
| Policy / regulation | allowed behavior、evidence、retention | control mapping、approval、data/process changes |

风险越高、影响面越大、变化越难回滚，门槛越严格。一次只改变一个因素有助于归因；若必须成组变化，则应记录 bundle 并设计能区分主要原因的实验。

### 7.3 生命周期决策不是永远 “continue”

每轮 review 都可以得出：

- **continue**：指标和风险在可接受范围内；
- **improve**：已知问题可通过受控迭代修复；
- **pivot**：原假设、workflow 或 architecture 不成立；
- **pause**：证据不足或风险暂不可控；
- **retire**：价值消失、替代方案更优、合规/安全/成本不可接受。

---

## 8. Phase gates：证据化决策，不是审批剧场

| Gate | 核心问题 | 典型决策 |
|---|---|---|
| Discovery gate | 问题、价值、风险和可行性是否值得投资？ | stop / explore / design |
| Design gate | architecture 和验证计划能否控制关键风险？ | redesign / prototype / build |
| Production-readiness gate | release candidate 是否达到 quality、safety、security、operations 门槛？ | block / pilot / release |
| Scale gate | pilot evidence 是否支持扩大用户和权限？ | hold / expand / rollback |
| Lifecycle review | 价值、风险和 assumptions 是否仍成立？ | continue / improve / pivot / retire |

一个 gate 至少记录：decision owner、inputs/evidence、criteria、decision、exceptions、residual risk、actions 和 review date。不要使用“stakeholders 看过了”“demo 很好”作为 gate evidence。

---

## 9. Retirement / Decommission：被忽略但必要的终态

虽然 Guide 的核心措辞以 iteration 收尾，负责任的 lifecycle 必须包含退出路径：

- 停止新流量并沟通替代流程；
- 撤销 API/tool credentials、service accounts 和 elevated permissions；
- 移除 routes、scheduled jobs、webhooks、agents 与 stale integrations；
- 按 retention/legal hold 处理 prompts、traces、indexes、feedback 和 backups；
- 保存必要 decision/evaluation/audit evidence；
- 更新 architecture、CMDB、runbook、owner 和 support status；
- 对用户影响、残余风险和下游依赖执行 closure review。

“不再维护但仍可访问”的 agent 往往比明确退役更危险，因为它保留权限，却失去 owner、监控和更新。

---

## 10. 责任模型：每个阶段都要有 accountable owner

| 角色 | 生命周期责任示例 |
|---|---|
| Business / product owner | outcome、scope、priority、risk acceptance、continue/retire |
| Architect | end-to-end design、trade-offs、lifecycle gates、traceability |
| Domain expert | ground truth、edge cases、human workflow、acceptance |
| Engineering / AI | implementation、artifact versions、tests、technical operation |
| Security / privacy / legal | risk/control review、required evidence、incident escalation |
| Evaluation / QA | dataset、graders、coverage、independent release evidence |
| SRE / operations | SLO、telemetry、on-call、runbook、capacity、rollback |
| Human reviewers / users | calibrated review、feedback、override/escalation behavior |

RACI 不能代替实际能力与权限。被标为 Responsible 的人必须拿到 dashboard、runbook、access、training 和决策通道。

---

## 11. 贯穿案例：退款客服 Agent

一家电商希望 Claude Agent 自动处理退款。

1. **Discovery**：区分查询、建议和实际退款；发现高金额退款不可逆且有欺诈风险；baseline 是人工处理时间和错误率。
2. **Design**：低风险查询使用 workflow；退款建议由 Claude 生成；后端执行确定性 eligibility 与 amount checks；高金额进入 HITL。
3. **Validation**：用正常、边界、欺诈、prompt injection、多租户和 tool timeout cases 评估；验证金额/订单 ID 在审批与执行间不可被替换。
4. **Handoff / Release**：仅对内部客服 canary；交付 version bundle、dashboard、review instructions、fraud escalation 和 rollback。
5. **Monitoring**：观察 successful resolution、错误退款、人工 override、用户投诉、tool error、latency 和 cost per resolved case，并按地区/语言/金额 slice。
6. **Iteration**：生产中发现新型 partial-shipment case；先加入 regression set，再修改 eligibility tool 与 prompt，复验后 staged rollout。
7. **Retirement / pivot**：若自动退款风险持续不可控，则保留 suggestion-only 模式并撤销 Agent 的退款执行权限。

关键判断：最优设计不是“尽可能自动化”，而是在业务价值、证据和风险容忍度下选择合适 autonomy，并允许在生命周期中升降权限。

---

## 12. 典型 failure modes

| Failure mode | 为什么失败 | 更好的做法 |
|---|---|---|
| Solution-first discovery | 没有问题 baseline 和 kill criteria | 先定义 outcome、risk、evidence |
| PoC purgatory | demo 成功但 production gaps 无 owner | 明确 hardening backlog 和 readiness gate |
| Prototype 直接上线 | curated cases 不代表真实分布 | mixed evaluation、risk/ops validation、canary |
| Handoff by link | receiving team 未接受责任或没有权限 | package、training、drill、sign-off |
| 上线后只看 uptime | 行为、风险和业务损害不可见 | technical + quality + risk + outcome signals |
| 平均指标掩盖 harms | 关键 cohort failure 被稀释 | risk-based slices 与 worst-case review |
| 看到差评就改 prompt | 未归因，可能是 RAG/tool/data 问题 | trace/version triage、先补 regression case |
| 静默 model/index/tool change | 原评估证据与生产版本脱节 | version bundle、change trigger、revalidation |
| 永久 pilot | 没有 scale/stop decision owner | time-box、gate、continue/pivot/retire |
| 无退役流程 | orphaned agent 保留数据和权限 | decommission checklist 与 closure evidence |

---

## 13. 考试决策框架

面对 lifecycle 场景题，可按以下顺序排除干扰项：

1. **定位阶段**：是在发现、设计、验证、交接、监控、迭代还是退役？
2. **识别决策**：当前需要继续、阻断、回滚、升级权限、修复还是停止？
3. **寻找缺失证据**：问题/指标、architecture、eval、risk、operations、feedback、version 哪一类缺失？
4. **确认 owner 和 handoff**：谁负责、谁批准、谁运行、谁接受 residual risk？
5. **选择可逆且可验证的下一步**：prototype、canary、rollback、补测试、限制 autonomy 通常优于大爆炸上线。
6. **闭合反馈**：production failure 是否进入 regression 与重新验证，而非只做临时 prompt edit？

常见错误答案通常承诺绝对保证、以单次 demo 代替证据、把 system prompt 当作唯一控制、只优化单一指标、没有 owner/rollback，或把 production 当作终点。

---

## 14. Guide 要求与当前产品实现的区别

Guide 要求的是支持完整 lifecycle 的 architecture reasoning。具体 Claude model、API 字段、Managed Agents、evaluation platform、cloud deployment、日志字段、价格或 feature availability 都是会变化的实现细节。

### Needs verification

实际实施前必须重新核查：

- 当前 Claude model IDs、alias / deprecation policy、capability、limits 和 pricing；
- Anthropic API / SDK schema、tool use、batch、caching、observability 和 evaluation tooling；
- 部署平台的 region、data handling、retention、security、compliance 和 service limits；
- 第三方 RAG、vector store、grader、monitoring 与 CI/CD 产品的版本和契约。

这些变化项没有被用来确定本课练习题的正确答案。

---

## 15. 核查来源

核查日期：**2026-10-09**。

1. Anthropic Engineering — [Demystifying evals for AI agents](https://www.anthropic.com/engineering/demystifying-evals-for-ai-agents)
   - 用于核查 eval 在上线前发现失败、贯穿 agent lifecycle、组合 automated / human / production signals 的原则。
2. AWS Well-Architected Generative AI Lens — [Generative AI lifecycle](https://docs.aws.amazon.com/wellarchitected/latest/generative-ai-lens/generative-ai-lifecycle.html)
   - 用于核查 scoping、model selection、customization、integration、deployment、continuous improvement 以及阶段间的迭代关系。
3. AWS Responsible AI Lens — [Design principles](https://docs.aws.amazon.com/wellarchitected/latest/responsible-ai-lens/design-principles.html)
   - 用于核查 narrowly defined use case、release criteria 与 responsible-by-design 的生命周期原则。
4. NIST AI RMF — [Core](https://airc.nist.gov/airmf-resources/airmf/5-sec-core/)
   - 用于核查 Govern / Map / Measure / Manage 的迭代、跨阶段与非 checklist 特征。
5. NIST AI RMF Playbook — [Map](https://airc.nist.gov/airmf-resources/playbook/map/) 与 [Manage](https://airc.nist.gov/airmf-resources/playbook/manage/)
   - 用于核查 assumptions、TEVV continuum、production feedback、go/no-go、risk treatment、residual risk 和 decommission decisions。
6. AWS Generative AI Lens — [Automate lifecycle management](https://docs.aws.amazon.com/wellarchitected/latest/generative-ai-lens/genops04-bp01.html)
   - 用于核查 version control、CI/CD、rollback 与治理自动化。

