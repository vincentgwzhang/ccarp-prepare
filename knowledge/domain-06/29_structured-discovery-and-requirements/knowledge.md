# 29 · 结构化发现与需求收集框架

## Guide 对齐

- **Domain**：Domain 6 · Stakeholder Communication & Lifecycle Management（14%）
- **Exam Guide 要求**：`Conduct structured discovery and requirement gathering`
- **本课目标**：把“做一个 Claude 助手”之类的方案表述，转化为可追溯、可测试、可用于架构决策的业务、任务、数据、工具、风险与非功能需求。
- **范围边界**：本课聚焦 discovery 与 requirement baseline。#30 处理架构决策和 trade-off 的沟通，#31 处理持续反馈与 SLA 对齐，#32 处理正式架构文档和实施指导，#33 处理完整生命周期。

结构化发现不是把干系人说的话原样记进 backlog，也不是为预选方案寻找理由。它要收集证据、暴露冲突与未知项，并形成一份所有关键角色都能验证的 **requirements contract**。

Anthropic 建议从满足任务的最简单方案开始，仅在确有需要时增加复杂度：很多场景用单次 LLM 调用加 retrieval 或 examples 已足够；well-defined task 通常更适合可预测的 workflow，而需要模型动态决策的开放任务才可能需要 agent。这个判断必须来自发现结果，而不是来自“想做 agent”的预设。

---

## 1. 为什么 LLM discovery 不能照搬普通 CRUD 需求访谈

传统系统常把输入、规则和输出当作确定性 contract；LLM 系统还必须面对：

1. **概率性输出**：同一输入可能产生不同但都合理的回答，也可能出现少量严重失败。
2. **上下文依赖**：质量不仅取决于模型，还取决于 prompt、检索内容、工具结果、会话状态与权限。
3. **能力与风险耦合**：为 agent 增加写工具会提升完成能力，也扩大误操作、越权和 prompt injection 的 blast radius。
4. **自然语言标准含糊**：“准确”“有帮助”“像专家”若没有 rubric、population 和 threshold，无法验收。
5. **持续变化**：业务知识、数据、工具、模型和攻击方式都会变化；requirements 必须包含监控、回退和再评估条件。

因此，发现阶段既要问“系统做什么”，也要问：**依据什么做、允许做到哪里、错了怎么办、怎样证明达标、谁承担剩余风险**。

---

## 2. Discovery 的核心产物：需求契约

一份可进入设计阶段的 discovery brief 至少应覆盖下表。

| 维度 | 必须回答的问题 | 典型证据 |
|---|---|---|
| Problem / outcome | 当前痛点和业务结果是什么？若不建设会怎样？ | 基线数据、真实案例、流程耗时 |
| Stakeholders | 谁使用、审批、运营、提供数据，谁会受到影响？ | stakeholder map、decision rights |
| Workflow | 触发、步骤、决策、异常、交接和结束状态是什么？ | as-is / to-be 流程、案例 walkthrough |
| Task contract | 输入、输出、允许依据、质量标准和失败语义是什么？ | use case、acceptance criteria |
| Scope / anti-scope | 明确做什么、不做什么、禁止什么？ | in/out/prohibited use 清单 |
| Data / context | 数据从哪来、谁是权威、是否新鲜、谁可访问？ | data inventory、lineage、ACL、retention |
| Tools / autonomy | 可读写哪些系统？哪些动作需批准、幂等或回滚？ | tool/action matrix、approval policy |
| NFR | latency、availability、throughput、cost、security 等门槛？ | SLI/SLO 草案、容量与威胁假设 |
| Evaluation | 怎样定义成功和失败？用什么代表性样本验证？ | baseline、eval suite、release gates |
| Delivery constraints | 时间、预算、平台、集成与运维能力有哪些限制？ | dependency / constraint log |
| Governance | 谁接受风险、处理 incident、批准变更？ | owner、RACI/decision rights |
| Unknowns | 哪些是假设，怎样、何时、由谁验证？ | assumption / risk / open-question log |

这里的重点不是文档数量，而是 **traceability**：业务目标应能追溯到 requirement，requirement 应能追溯到 architecture decision、test 和 owner。

---

## 3. 从 stakeholder map 开始，而不是只访谈 sponsor

Sponsor 能说明投资目标，却未必知道每天的异常路径、数据限制和受影响群体。至少识别：

- **Value owner / sponsor**：定义业务结果与投资边界；
- **Primary users**：直接提供输入、接收输出或执行建议；
- **Operators / support**：处理失败、告警、升级和恢复；
- **Domain experts**：定义正确性、例外和严重性；
- **Data / system owners**：确认 source of truth、质量、ACL 与集成约束；
- **Security、privacy、legal、compliance**：定义不可妥协的边界和证据；
- **Platform / SRE / FinOps**：确认可靠性、容量、成本和可观测性；
- **Secondary / indirect stakeholders**：其数据被处理或其权益受到系统决定影响、但未直接操作系统的人；
- **Implementation team**：验证可行性，但不能代替业务角色定义价值与风险。

对每类角色记录四件事：`interaction`、`impact`、`information contributed`、`decision authority`。这样可以避免“开了多人会议”却没人有权确认需求的假共识。

NIST 建议同时理解系统 intended purpose、部署情境、法律/社会预期，并通过 PRD、user story、UX research、systems engineering 等方法收集 stakeholder requirements。受影响但不是用户的人也必须进入发现范围。

---

## 4. 访谈真实工作，而不是让人设计 AI

高质量 discovery 问的是事实和约束：

1. “请带我走一遍最近五个真实案例。”
2. “哪个案例最难？为什么？最后由谁判断？”
3. “输入缺失、来源冲突或系统不可用时，现在怎么处理？”
4. “误报和漏报哪一个代价更高？最坏后果是什么？”
5. “什么动作可撤销？什么动作必须在执行前人工批准？”
6. “答案依据发生冲突时，哪个来源有最终权威？”
7. “高峰负载、截止时间、审计和保留要求是什么？”
8. “哪些例子看似相同，实际上必须得到不同结果？”

弱问题是“你想要聊天机器人还是 agent？”这会让访谈过早锁定实现。更好的问题是“用户要完成什么任务、当前在哪个 decision point 失败、系统需多大自由度”。

优先收集 **artifacts and observations**：历史 ticket、表单、政策、错误记录、人工检查表、升级记录、SLA 报表。口头表述与真实工作冲突时，不应直接选择其中一个，而应记录差异并由 owner 裁决。

---

## 5. 用任务链拆解需求

将每个 use case 画成：

```text
Trigger → Inputs → Decisions → Actions → Outcome → Feedback
```

逐段追问：

- **Trigger**：谁、何时、为什么启动？是否允许自动触发？
- **Inputs**：必需字段、来源、权限、新鲜度、缺失行为是什么？
- **Decisions**：规则明确还是需要语义判断？不确定性如何表达？
- **Actions**：只生成建议、起草内容，还是会写入外部系统？
- **Outcome**：以文本看起来正确为准，还是以外部状态真正改变为准？
- **Feedback**：谁纠错？纠错回到数据、prompt、tool 还是 policy？

必须同时走查 happy path、边界情况、对抗输入、外部依赖失败、权限不足和人工拒绝。Agent evaluation 中尤其要区分 **transcript** 与 **outcome**：agent 说“退款已完成”不等于后台真的存在合法退款记录。

---

## 6. 写出可验收的 task contract

可用如下模板把模糊描述变成测试对象：

```text
在 [context / precondition] 下，
[actor] 使用 [authorized data and tools] 完成 [task]，
输出满足 [content / schema / evidence contract]，
在 [latency / cost / availability constraints] 内，
当 [uncertainty / risk / dependency failure] 时执行 [abstain / escalate / retry / rollback]，
并通过 [test population, metric, threshold, owner] 验收。
```

示例转换：

| 模糊要求 | 可验证方向 |
|---|---|
| “回答要准确” | 定义 claim、权威来源、groundedness/correctness rubric、关键 slice 和门槛 |
| “响应要快” | 明确 TTFT 或端到端完成时间、p95、负载和 tool path |
| “必须安全” | 定义 threat model、数据分类、授权边界、禁止动作、攻击成功率和 incident gate |
| “减少客服工作量” | 建立当前 handle time / escalation baseline，测 accepted resolution 与返工 |
| “自动处理退款” | 定义资格规则、金额边界、审批条件、幂等键、审计和回滚 |

表中的数值门槛必须由业务风险、基线和测试结果确定，不能从通用文章复制。Anthropic 的评估文档强调先定义明确、可衡量、与任务相关的 success criteria，再建立代表真实和边界情况的 evaluation。

---

## 7. 特别检查数据、工具和 autonomy

### 7.1 Data / context contract

对每个来源记录：owner、authority、freshness、coverage、classification、ACL、retention、residency、lineage 和 conflict policy。

“接入知识库”不是完整要求。必须知道：过期页面是否仍可检索？不同政策冲突时谁优先？用户是否只能检索自己有权看的段落？删除或撤销授权后索引多久同步？

### 7.2 Tool / action contract

对每项能力记录：

- read / draft / write / irreversible；
- resource scope 与调用者身份；
- 参数约束、schema 与业务校验；
- idempotency、timeout、retry 与 duplicate side effect；
- approval point 和批准时展示的 exact payload；
- audit trail、compensation / rollback；
- tool result 是否来自不可信内容。

发现阶段无需确定完整实现，但必须揭示风险等级。读取产品目录和直接发起付款不能共享同一 autonomy assumption。

---

## 8. 先有 requirements，再选择复杂度

发现完成后按最小充分复杂度依次判断：

1. 不用 AI，规则、搜索或 UI 改进能否解决？
2. 单次 augmented LLM（prompt + context/examples）是否足够？
3. 是否需要由代码固定编排的 workflow？
4. 只有在步骤无法预先编码、需要模型自主选择路径且收益覆盖额外成本/风险时，才考虑 agent。

若任务规则清晰、动作高风险、审计严格，workflow 往往比开放 agent 更适合。若任务探索性强、路径随中间结果变化，agent 可能有价值，但需明确预算、stop condition、权限和 supervision。

这是 architecture **输入**，不是本课替代 #30 的 architecture decision record。后者应记录候选方案、trade-off 与最终决定。

---

## 9. Definition of Ready：何时可以结束 discovery

进入方案设计或 pilot 前，至少确认：

- problem、目标 outcome 和当前 baseline 已被 owner 接受；
- primary、secondary、indirect stakeholders 与 decision rights 已识别；
- 代表性 use cases、edge cases、anti-scope 和 prohibited uses 已记录；
- authoritative data、ACL、freshness 与冲突处理已明确；
- 工具动作、autonomy level、人工 gate 和失败恢复已分类；
- success metrics、hard guardrails、eval population 与证据来源可执行；
- NFR、预算、集成、合规和运营约束有 owner；
- 假设、风险、依赖、未知项及其验证计划透明；
- requirement IDs 能链接到后续设计、测试和验收。

Discovery 不要求所有未知项都已消失，但要求未知项没有被伪装成事实。重大未决项可通过 time-boxed spike、prototype 或 pilot 验证，并写明 go/no-go 条件。

---

## 10. 典型 failure modes

1. **Solution-first discovery**：先决定“做 agent”，再反向证明需要它。
2. **Sponsor-only discovery**：忽略一线 operator、数据 owner 与受影响非用户。
3. **Happy-path bias**：没有缺失输入、冲突来源、越权、依赖失败和人工拒绝场景。
4. **No baseline**：无法判断系统是否真的改善业务。
5. **Adjective requirements**：只写准确、快速、安全，没有 population、metric 和 threshold。
6. **Model = system**：把端到端成功率误写成模型能力，忽略 retrieval、tool、policy 和 UI。
7. **Implicit authority**：没有 source of truth、ACL、delegation 和 write boundary。
8. **Impossible promise**：在开放生成任务上承诺“100% 正确”，却没有 abstain / escalate。
9. **Hidden action risk**：把“帮助处理”理解为可以执行不可逆写操作。
10. **No owner**：需求冲突、风险接受和上线门槛无人裁决。
11. **Premature product binding**：把当前模型名、context limit 或 Beta 功能写成长期业务需求。
12. **Document dump**：产出大量文档，却没有 traceability、验收方法和决策权。

---

## 11. 场景示例：客服退款助手

模糊诉求：“用 Claude 自动处理退款，减少客服成本。”

结构化发现后可能得到：

- **Outcome**：缩短合格退款的处理时间，同时不提高错误退款和重复退款率；
- **Inputs**：已认证用户、订单 ID、支付状态、商品类别与现行退款政策；
- **Authority**：订单系统是交易状态 source of truth，版本化政策库是资格依据；
- **Task**：解释政策、收集缺失信息、判断候选资格、生成或执行结构化退款请求；
- **Autonomy**：低金额且满足确定性规则的请求可自动提交；边界/异常/高金额转人工——具体边界由业务 owner 和风险评估确定；
- **Controls**：服务端重新校验资格与金额，用户范围授权，idempotency key，执行前展示 payload，完整审计；
- **Failure semantics**：来源冲突或状态不明时不得猜测，明确说明并升级；
- **Evaluation**：按政策版本、商品、语言、边界案例切片，测资格判断、实际后台 outcome、误退款、漏退款、p95、cost per accepted resolution；
- **Anti-scope**：不改变政策、不接受聊天文本中的凭证、不绕过支付系统授权。

这些需求仍未强制指定 agent。若资格规则明确且动作高风险，确定性 workflow + Claude 解释/信息抽取可能是更稳妥的候选方案。

---

## 12. 考试决策模板

遇到 discovery / requirements 场景题，按顺序判断：

1. 题目给的是业务 problem，还是未经验证的 solution request？
2. 是否有真实案例、baseline、affected stakeholders 与 decision owner？
3. task 的 trigger、input、decision、action、outcome、feedback 是否完整？
4. data authority、freshness、ACL 与 tool write boundary 是否明确？
5. success 是否可量测，并覆盖 edge cases 与高风险 slices？
6. error-cost asymmetry 是否决定 abstain、HITL 或 hard guardrail？
7. 复杂度是否由需求推出，而非默认选择 agent？
8. 假设和未知项是否有验证计划，而不是被当作事实？

通常最佳答案会先澄清业务结果和约束、收集代表性证据并定义验收；直接选模型、写 prompt 或承诺全面自动化往往过早。

---

## Guide 要求与当前实现细节

Guide 要求掌握结构化 discovery 和 requirement gathering 的方法，不要求背诵某个产品模板。当前 Claude model、API、Managed Agents、tool schema、context window、价格或平台支持是后续实现选择，不能替代稳定的业务与风险需求。

### Needs verification

实际实施前需重新核查：当前 Claude 模型与 API 能力、tool/agent 产品状态、数据处理条款、区域支持、限制、计费和平台差异。所有性能门槛、风险容忍度、autonomy boundary 和 SLA 必须由目标组织、实际数据与代表性负载验证；本课示例不是通用默认值。

---

## 核查来源（2026-10-05）

- Anthropic, [Building effective agents](https://www.anthropic.com/engineering/building-effective-agents)：最小充分复杂度、workflow 与 agent 的控制路径差异。
- Anthropic Claude Docs, [Define success criteria and build evaluations](https://platform.claude.com/docs/en/test-and-evaluate/develop-tests)：可量测 success criteria、代表性与 edge-case evaluation。
- Anthropic, [Demystifying evals for AI agents](https://www.anthropic.com/engineering/demystifying-evals-for-ai-agents)：task、trial、trace、outcome 和多轮工具系统的评价边界。
- NIST AI RMF Playbook, [Map](https://airc.nist.gov/airmf-resources/playbook/map/)：intended purpose、部署情境、stakeholder requirements、假设与限制的文档化。
- AWS Well-Architected Generative AI Lens, [Generative AI lifecycle — Scoping](https://docs.aws.amazon.com/wellarchitected/latest/generative-ai-lens/generative-ai-lifecycle.html)：业务问题、可行性、成本、风险和成功指标的早期范围定义。
- AWS Responsible AI Lens, [Identify downstream stakeholders](https://docs.aws.amazon.com/wellarchitected/latest/responsible-ai-lens/raiuc02-bp01.html)：primary、secondary、indirect 与 vulnerable stakeholders 的识别和持续复核。
