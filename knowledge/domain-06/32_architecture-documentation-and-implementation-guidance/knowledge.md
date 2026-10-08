# 32 · 架构文档标准与实施指导交付

## Guide 对齐

- **Domain**：Domain 6 · Stakeholder Communication & Lifecycle Management（14%）
- **Exam Guide 要求**：`Document architectures and provide implementation guidance`
- **本课目标**：把已确认的业务目标、架构决策和风险控制，转化为不同受众可以理解、工程团队可以实施、测试团队可以验证、运营团队可以接管的交付物。
- **范围边界**：本课回答“交付什么文档、怎样使其可实施”。#30 已覆盖决策与 trade-off 的沟通；#33 才覆盖 discovery、design、handoff、monitoring、iteration 的完整生命周期治理。

架构文档不是一张漂亮图，也不是代码的长篇复述。合格的交付物必须让接手者回答四个问题：

1. **为什么建**：目标、范围、约束和质量属性是什么？
2. **建什么**：系统由哪些 building blocks、接口、数据流和信任边界组成？
3. **怎样建和验证**：实施顺序、契约、测试门槛、发布与回滚条件是什么？
4. **怎样运行和改变**：谁负责、怎样观察、故障如何降级、什么变化触发复审？

---

## 1. 文档的价值：降低跨角色的信息损失

同一份架构会被不同受众以不同问题阅读：

| 受众 | 最关心的问题 | 适合的交付物 |
|---|---|---|
| Executive / sponsor | 价值、范围、成本、风险、主要取舍 | 一页概览、关键 KPI/风险、roadmap |
| Product / domain | 用户流程、成功标准、人工责任、限制 | use case、runtime scenario、acceptance criteria |
| Developer | 边界、接口、状态、错误语义、代码位置 | container/component view、contract、reference flow |
| Data / ML / AI | 数据来源、prompt/context、eval、版本与质量门槛 | AI artifact registry、evaluation contract |
| Security / legal | 数据流、trust boundary、权限、retention、控制证据 | threat/data-flow view、control mapping |
| SRE / operations | 部署、依赖、SLI/SLO、告警、降级和恢复 | deployment view、runbook、dashboard/alert link |
| QA / evaluator | 什么算正确，如何复现与判定 | test strategy、eval dataset、grader/threshold |

“把所有内容塞进一个 100 页 PDF”并不等于满足所有受众。应维护一个一致的 architecture model，再为不同任务提供合适的入口和视图。

---

## 2. 最小但完整的架构文档集合

不要求机械套用某个模板，但至少应覆盖以下信息。

### 2.1 Introduction、scope 与 constraints

- problem statement、目标用户和业务 outcome；
- intended use / anti-scope；
- functional scope 与明确不做什么；
- 关键 quality goals：quality、latency、availability、cost、safety、security；
- 法规、数据、区域、技术、预算、期限和组织约束；
- assumptions、open questions、术语表与 owner。

### 2.2 Context view：系统边界和外部关系

Context view 应显示：

- 用户和角色；
- 本系统的责任边界；
- 外部 identity provider、source systems、vector store、Claude platform、tools、human review 和 downstream systems；
- 每条关系交换什么信息；
- 哪些边界跨 tenant、网络、组织或信任域。

C4 的 system context diagram 适合回答“我们正在描述哪个系统，它与谁交互”。

### 2.3 Building-block view：静态分解和责任

每个重要 building block 至少描述：

- 名称、单一主要责任和 owner；
- inbound/outbound interfaces；
- 输入、输出、状态与依赖；
- quality/security characteristics；
- source repository / module / deployment artifact；
- 相关需求、风险和 open issues。

只对 **重要、复杂、风险高、易变化或出人意料** 的部分继续下钻。arc42 明确建议 relevance over completeness；正常、简单、标准化组件无需画到 class 级别。

### 2.4 Runtime view：关键场景怎样真正运行

静态图无法解释 agent loop 或 failure handling。选择少量 architecture-significant scenarios：

- 正常请求：输入 → retrieval → Claude → tool → validation → output；
- sensitive tool action 的 authorization / HITL；
- indirect prompt injection 被发现后的处理；
- tool timeout、rate limit、partial completion、retry 与 idempotency；
- hallucination / low confidence 时 abstain 或 escalation；
- index refresh、prompt/model rollout 与 rollback；
- feedback 如何关联 trace 并进入 regression eval。

可使用 sequence diagram、activity diagram、state machine 或编号步骤。重点是组件、消息、状态变化、控制点和异常路径，而不是图形工具。

### 2.5 Deployment view：软件怎样映射到运行环境

需要说明：

- dev/test/staging/prod 及环境差异；
- region、network zone、runtime、queue、database、vector store 与 secrets boundary；
- building block 到 infrastructure 的映射；
- availability、scaling、backup、DR 与 data residency；
- ingress/egress、private/public path 和 external provider dependency；
- release unit、configuration source 和 rollback unit。

对于分布式 Claude 系统，只画逻辑组件却不画部署和网络边界，会遗漏大量安全、延迟和故障风险。

### 2.6 Cross-cutting concepts

集中描述跨多个组件的一致规则：

- identity propagation、authentication、authorization；
- data classification、encryption、retention、redaction；
- error taxonomy、retry、timeout、circuit breaker、idempotency；
- logging、trace correlation、metrics、sampling；
- configuration、feature flag、versioning、deployment；
- guardrails、HITL、incident management；
- prompt/context/tool/eval artifact governance。

不要在每个组件下复制同一规则。arc42 将 cross-cutting concepts 用于维持 architecture 的 conceptual integrity，并建议只记录系统真正需要的主题。

### 2.7 Decisions、quality requirements、risks

- 通过 ADR 链接重要决策、备选方案、证据和 consequences；
- 把“快速、准确、安全”改写成可测 quality scenarios；
- 维护按优先级排序的 risk / technical-debt register，并记录 mitigation、owner 和 review trigger。

一个 quality scenario 可使用：`source → stimulus → environment → affected artifact → response → response measure`。这样文档才能生成测试和验收条件。

---

## 3. Claude / LLM 系统必须额外记录的契约

传统应用文档只写 `App → LLM API` 远远不够。Claude 系统的行为同时由 model、prompt、context、tools、data、orchestrator 和 controls 决定。

### 3.1 Model 与 invocation contract

- 选择模型的任务依据，而不是只写营销名称；
- 使用固定 model ID 还是 alias，以及升级/回滚政策；
- Messages/API version、关键参数和 feature dependencies；
- routing、fallback、timeout、retry 与 budget；
- token、latency、cost 和 quality 的验收边界。

具体 model ID、参数和限制属于当前实现细节，必须链接到受版本控制的配置，而不是在多处手工复制。

### 3.2 Prompt contract

- system prompt、模板和 few-shot examples 的 version / owner；
- 变量来源、required/optional、validation 和 escaping；
- instruction hierarchy、untrusted content delimiter；
- expected output / schema、abstention 与 escalation；
- 关联 eval suite、批准状态和 rollback version。

文档不应复制完整 secret-bearing prompt；应链接到 version-controlled artifact，并记录其责任和契约。

### 3.3 Context / RAG contract

- source of truth、ingestion owner、freshness 与 deletion；
- chunking、embedding、index/version compatibility；
- retrieval / reranking / filtering path；
- tenant/ACL enforcement 在哪个确定性层执行；
- citation / provenance 规则；
- empty、stale、conflicting 或 unauthorized context 的处理。

### 3.4 Tool contract

Anthropic 的 tool definition 包括 `name`、详细 `description` 和 `input_schema`；Claude 是否正确选择工具高度依赖描述和 schema。架构交付还必须补充：

- business purpose、允许/禁止使用场景；
- caller identity、required scope、resource-level authorization；
- input/output schema、semantic validation、error taxonomy；
- side effects、idempotency、timeout、retry 与 compensation；
- confirmation / HITL、audit 和 data sensitivity；
- mock / sandbox 与 contract tests。

Schema-valid 只代表结构有效，不代表业务授权、语义或安全正确。

### 3.5 Agent / workflow contract

- control path 由 code 还是 model 决定；
- state、memory、checkpoint 和 resumability；
- maximum steps / time / cost；
- termination、success、partial success 和 stuck-loop detection；
- delegation contract、shared artifact 与 result aggregation；
- permission boundary 和 irreversible-action gate。

### 3.6 Evaluation 与 safety contract

- success criteria、dataset / slices、grader 与 threshold；
- deterministic、LLM、human 和 production signals 的组合；
- safety/security hard gates 与允许 trade-off 的指标；
- release candidate 对应的 prompt/model/tool/index versions；
- regression、canary、rollback 和 post-release observation。

Anthropic 的 evaluation 指南强调先定义明确、可测的 success criteria，再选择测试方法。因此 implementation guide 不能只写“完成开发后测试效果”。

---

## 4. 图不是文档：每张图都要可解释和可核对

C4 提供从 system context、container、component 到 code 的层级抽象，并补充 dynamic 与 deployment diagrams；它是 notation-independent、tooling-independent 的。考试判断重点不是“必须用 C4”，而是选取与受众问题匹配的视图。

一张可用的架构图至少需要：

- 明确标题、scope、view level 和日期/version；
- legend 与一致术语；
- 每个元素的 type、名称、责任；
- 每条箭头的方向、关系、protocol / data；
- trust / tenant / network boundary；
- 与正文、接口契约和 ADR 的链接；
- 明确标出 current state、target state 或 migration state，不能混在一张图里不说明。

### 常见错误

- 只有厂商 logo，没有组件责任和数据流；
- 箭头无标签，无法判断调用方向与传输内容；
- logical、runtime、deployment 三种含义混在一张图；
- 图中名称与代码、dashboard、runbook 不一致；
- 每个 class 都画出来，关键边界反而不可见；
- 只画 happy path，不画 authorization、failure、fallback 和 human gate。

---

## 5. 从 architecture 到 implementation guidance

架构师不能只交付“目标图”，然后让开发者自行猜测。实施指导应把目标状态拆成安全、可验证的增量。

### 5.1 定义 implementation units

每个 unit / work package 包含：

```text
Objective / user outcome:
In scope / out of scope:
Affected building blocks:
Interfaces and data contracts:
Prompt/context/tool artifacts:
Security and privacy controls:
Acceptance criteria / eval slice:
Observability required:
Dependencies and owner:
Rollout / rollback:
Evidence produced:
```

优先交付 end-to-end vertical slice：即使能力有限，也贯通真实输入、Claude 调用、最小 tool/data path、验证、telemetry 和人工处理。只分别完成“prompt 层”“backend 层”却长期无法端到端验证，会推迟发现契约错误。

### 5.2 排定实现顺序

通常先处理：

1. 风险最高、未知最多的 assumption spike；
2. identity/data/trust boundary 和不可逆 control；
3. 最小 end-to-end walking skeleton；
4. interface/tool/prompt contracts 与 automated checks；
5. quality、safety、observability gates；
6. scale、performance 和 operational hardening；
7. staged rollout、handoff 与 evidence closure。

具体顺序由依赖和风险决定，不是固定 waterfall。

### 5.3 给出 reference implementation，而不是复制粘贴陷阱

可提供：

- 一条 canonical request / tool-use loop；
- schema、error handling 和 trace propagation 示例；
- golden-path config；
- contract / eval test fixture；
- example dashboard 与 alert；
- local sandbox / mock tool。

同时说明示例省略了什么、哪些字段必须由环境提供、哪些值不能进入 source control。示例若绕过 auth、timeout 或 validation，开发者很可能把“demo shortcut”带入生产。

### 5.4 明确 Definition of Done

一个 Claude feature 的完成条件可能包括：

- code、prompt、tool schema、index config 均有版本；
- functional / contract / eval / security tests 达到门槛；
- critical slices 和 negative cases 通过；
- telemetry、dashboard、alert 和 trace 可用；
- runbook、on-call owner、fallback 和 rollback 已演练；
- data/privacy/security review evidence 完整；
- 文档与实际部署一致，handoff 接受者完成 walkthrough。

“代码 merge”不是 architecture implementation 的完成标准。

---

## 6. Traceability：让要求、组件、控制和证据可追踪

推荐维护轻量 traceability matrix：

| Requirement / risk | Architecture element | Control / implementation | Verification | Evidence / owner |
|---|---|---|---|---|
| 不得跨租户检索 | Retrieval gateway | server-side tenant filter | negative ACL test | CI report / Data team |
| 高额退款需批准 | Tool executor | policy check + HITL | unauthorized / stale approval tests | audit event / Payments |
| p95 首个有用结果目标 | Orchestrator + retrieval | timeout / parallel path | load test + prod SLI | dashboard / Platform |

这个矩阵可发现三类缺口：

- requirement 没有 architecture owner；
- control 存在但没有验证；
- test 通过却无法生成审计或运营证据。

不要追求每行代码到需求的重型追踪；优先覆盖高风险、跨团队和 architecture-significant 项。

---

## 7. Handoff package 与责任转移

一次有效 handoff 至少包括：

- architecture overview 与 view index；
- ADR、接口/数据/tool contracts；
- prompt/context/eval artifact locations 和版本规则；
- threat/risk/control mapping；
- implementation backlog、依赖、milestone 与 acceptance gates；
- environment/configuration/deployment guidance；
- runbook、SLI/SLO、dashboard、alert、incident/escalation；
- known limitations、technical debt、open decisions；
- owner / approver / on-call / change authority；
- walkthrough、Q&A、readiness checklist 和正式 acceptance。

上传文档链接不等于 knowledge transfer。应让接手团队通过 scenario walkthrough 或 tabletop exercise 证明：他们能部署、观察、处理典型失败并安全回滚。

---

## 8. 保持文档可信：docs-as-code 与 drift control

文档一旦与系统分离，就会迅速失去权威性。可采用：

- architecture、diagram source、prompt、schema 和 ADR 与代码共同 version control；
- PR template 要求勾选受影响文档、eval 和 runbook；
- contract/schema lint、broken-link、diagram build 和 config reference checks；
- 通过代码或 deployment metadata 自动生成易漂移的 inventory；
- 每页记录 owner、last reviewed、status 和 review trigger；
- model、prompt、tool、index、policy 变更时自动链接对应 evidence；
- incident / postmortem 后回写 runtime view、risk 和 runbook。

“自动生成”并不保证语义正确。生成式 AI 可帮助起草或对比文档，但 owner 必须核对系统边界、隐含假设、部署现实和安全控制，不能把 hallucinated architecture 当作事实。

### 不应写入一般架构文档的内容

- API key、token、private key、生产凭证；
- 可直接利用的未修复漏洞细节（应进入受控系统）；
- 无必要的个人数据、完整生产 prompt/trace；
- 会迅速过期却没有自动来源的手工 inventory；
- 未标记为假设的推测。

文档的访问控制应匹配内容敏感度，但团队仍需能找到完成职责所需的信息。

---

## 9. 场景示例：企业政策问答 Agent 的交付

仅交付下面一张图是不够的：

```text
Web UI → Backend → Claude → Vector DB
```

合格交付还会说明：

1. Context view：员工、IdP、HR policy source、Claude、support escalation；
2. Container view：gateway、orchestrator、retriever、tool executor、feedback/eval service；
3. Runtime views：正常 grounded answer、ACL denial、stale source、low-confidence escalation；
4. Deployment view：region、network、secrets、index、queue 与环境映射；
5. Contracts：prompt variables、retrieval filters、citation output、tool schema/error；
6. Controls：tenant ACL 在 retrieval 前确定性执行，tool scope 不由模型决定；
7. Quality scenarios：groundedness、p95、freshness、data leakage hard gate；
8. Implementation plan：先做单部门 read-only vertical slice，再扩数据源和动作能力；
9. Acceptance：eval slices、negative ACL tests、load test、canary、rollback；
10. Operations：trace、dashboard、feedback path、runbook 与 owners。

这使团队能够从“看懂概念”走到“按同一约束实现并证明正确”。

---

## 10. 典型 failure modes

1. **Diagram-only delivery**：没有 contract、quality gate、owner 或 rollout；
2. **Document dump**：内容很多，却没有导航和受众入口；
3. **Happy-path bias**：缺少 error、security、HITL、degrade 与 rollback；
4. **LLM as a box**：未记录 prompt/context/tool/eval 的行为契约；
5. **Ambiguous arrows**：不标 protocol、data、direction 或 trust boundary；
6. **Target/current confusion**：目标架构被误认为已经上线；
7. **Implementation by implication**：目标图没有 work packages、依赖和验收；
8. **Unowned document**：无 owner、review date 或 change trigger；
9. **Copy-paste drift**：model/config/schema 在多处重复并相互矛盾；
10. **Secret leakage**：为“完整”而把凭证或敏感 trace 放进共享文档；
11. **Demo becomes production**：参考代码省略的控制没有被明确；
12. **No evidence chain**：需求、控制、test、dashboard 和审计证据无法对应。

---

## 11. 考试决策模板

遇到 architecture documentation / implementation guidance 场景题，依次判断：

1. 文档受众是谁，他们需要做什么决定或工作？
2. scope、constraints、assumptions 和 current/target state 是否清楚？
3. 是否需要 context、building-block、runtime、deployment 或 data/trust view？
4. prompt、context/RAG、tool、agent、eval 和 guardrail 契约是否可定位、可版本化？
5. quality requirement 是否具体可测，并能导出 acceptance test？
6. 是否记录 failure/degrade/rollback，而不仅是 happy path？
7. 实施是否拆为有 contract、owner、dependency 和 evidence 的 vertical increments？
8. requirement → element → control → verification → owner 是否可追踪？
9. handoff 是否包含 operations、security、known risks 和 ownership acceptance？
10. 文档如何随 code/config/incident 更新，并避免 secrets 与重复事实源？

最佳答案通常不是“增加更多图”，而是选择满足 stakeholder concern 的最小视图，并补齐可执行 contract、quality gate、owner、evidence 和 change mechanism。

---

## Guide 要求与当前实现细节

Guide 要求能记录架构并提供实施指导，但没有指定必须使用 C4、arc42、UML 或某个文档平台。C4 与 arc42 在本课中是组织视图和检查完整性的工具，不是考试强制格式。Claude 当前 API/tool/model 字段是实现细节；考试重点是如何把 AI-specific behavior、边界、控制与验证转化为可执行、可维护的交付物。

### Needs verification

实施前需重新核查当前 Claude model/API/tool/agent schema、feature status、limits、pricing、data handling、platform differences 和 versioning policy。组织还需自行确定文档分类、审批、retention、required views、handoff gates、RACI 和 evidence repository；示例中的组件、字段与实施顺序不是通用强制模板。

---

## 核查来源（2026-10-08）

- Anthropic, [Define tools](https://platform.claude.com/docs/en/agents-and-tools/tool-use/define-tools)：tool definition 的名称、描述、schema 与示例等当前接口契约。
- Anthropic, [Define success criteria and build evaluations](https://platform.claude.com/docs/en/test-and-evaluate/develop-tests)：先定义明确、可测 success criteria，再建立相应 evaluations。
- Anthropic, [Effective context engineering for AI agents](https://www.anthropic.com/engineering/effective-context-engineering-for-ai-agents)：agent context 包括 system instructions、tools、MCP、external data 与 message history，需要作为整体管理。
- C4 Model, [Official site](https://c4model.com/)：system context、container、component、code，以及 supporting dynamic/deployment diagrams 的层级视图。
- arc42, [Template overview](https://arc42.org/overview/)：可裁剪的架构文档结构，包括 scope、building blocks、runtime、deployment、cross-cutting concepts、decisions、quality 与 risks。
- arc42, [Building Block View](https://docs.arc42.org/section-5/)、[Runtime View](https://docs.arc42.org/section-6/)、[Deployment View](https://docs.arc42.org/section-7/)：静态分解、代表性运行场景和软件到基础设施映射。
- arc42, [Crosscutting Concepts](https://docs.arc42.org/section-8/)、[Quality Requirements](https://docs.arc42.org/section-10/)、[Risks and Technical Debt](https://docs.arc42.org/section-11/)：一致规则、可测质量场景与风险记录。
