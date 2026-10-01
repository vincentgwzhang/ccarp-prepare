# 25 · LLM 系统的风险、局限性与失效模式分类学

> 学习序号 25 · 清单 #25 · Domain 5: Governance, Safety & Risk Management（14%）。核查日期：2026-10-01。
> 考试依据：[Exam_Guide.md](../../../Exam_Guide.md) 第 6 节原文 “Identify risks, limitations, and failure modes of LLM systems”。本课关注识别、归因、排序和记录风险；#24 已讲控制实现，#26 将深入 Human-in-the-loop，#27 处理法规，#28 处理偏见、公平与透明度。

## 1. 三个词不能混用

| 概念 | 问的问题 | 例子 |
|---|---|---|
| Limitation（局限性） | 系统或模型天然不能可靠保证什么？ | 生成具有概率性；长上下文中的信息不一定被同等利用；模型可能在证据不足时仍给出流畅回答 |
| Failure mode（失效模式） | 系统具体怎样偏离预期？ | 引用不存在的条款、调用错工具、重复退款、把 A 租户数据返回给 B 租户 |
| Risk（风险） | 某失效发生的可能性与影响是什么？ | 错误医疗摘要被直接用于治疗，造成高严重度且难以逆转的伤害 |

局限性并不自动等于事故，失效模式也不等于根因。比如“给出过期退款政策”是可观察的 failure mode；根因可能是索引未刷新、检索召回错误、上下文被截断，也可能是模型无依据补全。风险评估还必须加入场景、暴露量、影响和既有控制。

一个实用链条是：

```text
asset / objective
  → hazard or threat
  → trigger / contributing causes
  → failure mode
  → local effect
  → business or human impact
  → controls
  → residual risk
```

不要把所有错误都写成“hallucination”。那会隐藏真正可修复的 indexing、authorization、parser、retry 或业务规则问题。

## 2. 从系统边界开始，而不是从模型清单开始

风险识别至少要画出：

1. **Actors**：终端用户、管理员、恶意用户、内部人员、第三方内容提供者、tool/MCP 服务；
2. **Assets**：正确决策、敏感数据、资金、生产环境、企业声誉、用户权益；
3. **Trust boundaries**：用户输入、retrieval corpus、tool result、模型输出、执行服务、人工审批；
4. **Capabilities**：模型能看什么、建议什么、调用什么、最终能改变什么；
5. **Success and forbidden outcomes**：不仅写正常路径，还写不可接受结果；
6. **Lifecycle**：设计、数据摄取、评估、部署、运行、更新和退役。

相同模型用于“内部草稿助手”和“自动放款 agent”时，基础局限可能相同，风险却完全不同。后者有更高 autonomy、更敏感资产、更大 blast radius 和更低可逆性。

## 3. 分层 taxonomy：风险来自完整系统

### 3.1 模型与认识能力（model / epistemic）

- **Hallucination / fabrication**：输出与事实或给定 context 不一致，却可能语言流畅、置信感很强；
- **Uncertainty miscalibration**：缺少信息时不 abstain，或表达的自信与正确率不匹配；
- **Knowledge boundary**：训练知识可能缺失、过期或不含企业私有事实；
- **Reasoning / planning limitation**：多约束、长链条或需要精确状态跟踪的任务中漏条件、跳步骤；
- **Non-determinism and sensitivity**：输入措辞、顺序、采样或上下文细节改变，可能得到不同结果；
- **Over-refusal / under-refusal**：正常任务被错误拒绝，或危险任务未被正确限制；
- **Slice disparity**：语言、领域、地区、表达方式或用户群之间表现不均。

Anthropic 官方明确指出，高级模型仍可能生成与事实或上下文不一致的内容；允许表达“不知道”、用原文和引用 grounding 能降低风险，但不能消除它。因此“升级模型后不再需要验证”通常不是好答案。

### 3.2 Prompt 与 context

- 指令含糊、冲突、遗漏优先级或把数据误当指令；
- direct jailbreak / prompt injection 改变预期行为；
- indirect injection 藏在网页、邮件、文档或 tool result 中；
- prompt/system instructions 或敏感 context 泄漏；
- 上下文过长导致 focus 降低，关键信息被淹没；
- compaction/summarization 丢失关键约束、未决事项或来源；
- 输出格式不稳定、parser 将自然语言误解成结构化动作。

“能放进 context window”不代表“会被稳定、准确地使用”。Context 是有限注意力预算，增加 token 可能带来递减收益；过度压缩又可能丢掉后来才显出重要性的细节。

### 3.3 数据、RAG 与 provenance

- 文档过期、索引刷新失败、chunk 边界破坏语义；
- semantic/keyword retrieval 与查询类型不匹配；
- top-K 返回无关、相互矛盾或缺少关键证据的片段；
- embedding/index migration 期间版本混用；
- corpus poisoning、恶意文档或不可信来源进入检索；
- tenant/ACL metadata 缺失或过滤顺序错误造成越权；
- citation 指向的来源并不支持实际 claim；
- 来源存在，但 answer 仍使用模型外部知识补全。

RAG failure 和 hallucination 的表象都可能是“答案错了”。应检查 query、retrieved evidence、版本、ACL、最终 prompt 和 citations，不能先假定需要重新训练模型。

### 3.4 Tool、Agent 与副作用

- 选择错误工具或生成语义错误的 arguments；
- schema 合法，但对象、金额、收件人或权限错误；
- capability bloat 让被攻陷的 agent 拥有不必要能力；
- loop 无停止条件，导致 runaway token、延迟和成本；
- 中间错误在多步/多 agent 链路中累积并被后续步骤当成事实；
- timeout/retry 对非幂等操作造成重复扣款、重复发信；
- partial failure 让外部系统与 agent state 不一致；
- delegation contract 不清造成遗漏、重复或矛盾结果；
- tool/MCP/server 或依赖被入侵，形成 supply-chain 风险。

Anthropic 指出，agent 的 autonomy 会增加成本和 compounding errors；开放式 loop 需要环境 ground truth、停止条件、sandbox、测试与 guardrails。即使每个组件单独“正常”，组件交互仍可能产生 emergent failure。

### 3.5 Security、privacy 与 abuse

- 数据泄露、cross-tenant exposure、secret/PII 进入输出或日志；
- AuthN/AuthZ 缺口、confused deputy、token audience/scope 错误；
- data exfiltration、越权工具调用、任意网络/文件/代码访问；
- prompt injection、jailbreak、model/tool output manipulation；
- denial of wallet、资源耗尽、批量滥用；
- 不安全第三方连接、插件、模型或数据供应链。

Security 关注对抗者对 confidentiality、integrity、availability 的破坏；safety 更广，包含无恶意攻击时仍会造成的人身、社会或业务伤害。两者有交集，但不能互相替代。

### 3.6 Safety、ethics 与 human factors

- 有害、危险或不适当内容；
- 偏见、差别错误率与机会不公平；
- 欺骗、操纵、冒充或过度拟人化；
- 用户不知道内容由 AI 生成或无法理解依据；
- **automation bias**：人类因输出流畅而过度信任；
- **approval fatigue**：审批者频繁点击确认，却未核对证据与真实参数；
- 责任边界不清，事故时无人拥有决策和修复责任。

#28 会深入伦理 AI；本课只要求能识别这些是系统风险类别，并知道不能用平均准确率把它们隐藏掉。

### 3.7 Evaluation、operations 与 business

- benchmark 与真实流量不匹配，造成 false confidence；
- test leakage、overfitting、grader bug 或 LLM judge 未校准；
- 平均分掩盖高风险 slice、长尾和严重失败；
- model/prompt/index/tool/policy 更新造成 regression 或 distribution drift；
- outage、rate limit、timeout、依赖故障和 retry storm；
- p95/p99 latency、cost per successful task 或人工队列超出 SLA；
- telemetry 缺失，无法重建事故路径；
- provider lock-in、弃用、区域/数据条款变化影响连续性。

Anthropic 的 eval 实践强调：自动 eval、生产监控、A/B、用户反馈、transcript review 和系统性人工评估各有盲区；没有任何单层能捕获全部问题。

## 4. 易混淆故障的快速分诊

| 表象 | 先检查 | 不应直接断言 |
|---|---|---|
| 文档更新后仍答旧内容 | ingest/index version、query、retrieved chunks、cache | “模型知识过期，必须换模型” |
| JSON 可解析但退款错人 | identity、ownership、AuthZ、semantic validation | “Structured Output 已失败” |
| 相同请求偶尔得到不同答案 | prompt/context 差异、sampling、tool/data nondeterminism、trial 分布 | “线上一定出现代码竞态” |
| 引用存在但结论错误 | claim-evidence entailment、source quality、矛盾来源处理 | “有 citation 就不是 hallucination” |
| 安全事件只发生在网页搜索后 | retrieved content、tool trace、indirect injection、capability boundary | “可信用户发起了 jailbreak” |
| 平均成功率不变但投诉增加 | language/tenant/task/risk slice、严重度、流量 mix | “系统没有 regression” |
| 重试后发生两次写操作 | timeout 边界、idempotency、commit 状态、reconciliation | “模型重复输出而已” |

诊断时把 **symptom、failure mode、cause、impact** 分列。一个 failure mode 可以有多个根因，一个根因也可能触发多个影响。

## 5. 怎样把 taxonomy 变成 risk register

推荐每条至少记录：

| 字段 | 含义 |
|---|---|
| System/component | 风险所在组件与信任边界 |
| Asset / objective | 要保护的业务结果、数据或主体 |
| Scenario / trigger | 正常错误、边界情况还是 adversarial misuse |
| Failure mode | 可观察到的具体失效 |
| Cause / contributor | 可能根因与放大因素 |
| Impact | 对用户、业务、安全、合规和下游系统的影响 |
| Existing controls | preventive、detective、corrective controls |
| Detection signal | eval、log、metric、review 或 incident signal |
| Owner | 接受、降低、转移或停止该风险的人 |
| Residual risk | 控制之后仍剩余的风险及接受期限 |

### 排序维度

不要迷信一个乘法分数。至少共同考虑：

- likelihood / exposure frequency；
- severity / impact；
- detectability 与发现时延；
- reversibility；
- blast radius / affected scale；
- persistence；
- adversary exploitability；
- control strength 与 evidence；
- uncertainty：证据不足本身应被显式记录。

高影响、难发现、不可逆的低频风险，可能比高频但可自动恢复的小错误优先级更高。区分 **inherent risk**（控制前）和 **residual risk**（控制后），否则“已经有 guardrail”会被误当成“风险消失”。

## 6. 风险分析工作流

1. 定义业务目标、系统边界、actors、assets、数据与动作；
2. 沿 input → context/RAG → model → tool/action → output → operations 枚举失效；
3. 同时分析 accidental failure、misuse、abuse 与依赖故障；
4. 为每项建立 cause → failure → impact，而不是只贴一个类别标签；
5. 用代表性场景、adversarial case、历史事件和关键 slices 验证；
6. 根据影响、可检测性、可逆性和暴露量排序；
7. 指派 owner，选择 avoid/reduce/transfer/accept，并记录 controls；
8. 评估 residual risk，超过 appetite 的系统不得仅凭“模型通常表现良好”上线；
9. 让 production incident、near miss、用户反馈和 drift 回流到 risk register 与 regression eval。

风险清单不是一次性文档。模型、prompt、知识库、工具权限、业务流量或法规变化，都可能让原评分失效。

## 7. 典型案例：自动退款客服 Agent

**目标**：核验订单并在政策范围内退款。

可能风险链：

```text
恶意邮件/用户文本含 injection
  → agent 忽略原任务并选择退款工具
  → 参数 schema 合法，但订单不属于当前用户
  → 后端缺少 object-level AuthZ
  → 未授权退款，产生资金和审计损失
```

这里至少有四层问题：prompt injection 是 threat vector；错误 tool decision 是 failure mode；后端授权缺失是系统 vulnerability；资金损失是 impact。只把事件登记为“模型 hallucination”会让团队错修 prompt，而保留真正的越权漏洞。

另一个非对抗性链条是：tool timeout → client retry → 第一次调用实际已成功但响应丢失 → 第二次重复退款。其核心是 distributed-system failure 与非幂等副作用，不应归因于模型知识。

## 8. 常见错误判断

- **错误输出 = hallucination**：先检查数据、retrieval、tool、parser、state 和 policy。
- **平均准确率高 = 风险低**：严重错误、关键 slice 和不可逆动作不能被平均掉。
- **模型升级 = 风险关闭**：能力改善不等于消除不确定性、权限或集成缺陷。
- **Schema-valid = correct/safe/authorized**：schema 只解决结构契约。
- **有 citation = claim 有证据**：还需验证来源质量和 claim-evidence entailment。
- **可信用户 = 没有 injection**：用户读取的第三方内容仍可能携带 indirect injection。
- **加人工确认 = 风险解决**：automation bias、审批疲劳和误导性摘要仍会导致误批。
- **有 monitoring = 已预防**：监控可能在用户受损后才报警。
- **没有事故 = 没有风险**：也可能是暴露量低、检测不足或 near miss 未记录。
- **列出类别 = 完成治理**：没有 scenario、impact、detector、owner 和 residual decision 的清单不可执行。

## 9. 考场决策框架

看到风险识别题，依次问：

1. 这是模型局限、具体 failure mode，还是带场景与影响的 risk？
2. 系统边界、资产、actor 和可执行能力是什么？
3. 失效发生在哪一层，真正根因是否可能在另一层？
4. 是偶发错误、分布漂移、依赖故障，还是 adversarial attack？
5. 影响的 severity、detectability、reversibility 和 blast radius 如何？
6. 现有控制之后还剩什么 residual risk，谁有权接受？
7. 哪些证据能验证分类：trace、retrieved context、tool result、outcome、slice eval、人工复核？

最佳答案通常会先做**系统级归因和风险分层**，再决定控制；不会把每个问题都归因于模型，也不会用单一 prompt、单一平均指标或一次性 checklist 宣布安全。

---

## 资料来源与核查日期

核查日期：**2026-10-01**。

1. Anthropic Claude Platform Docs, [Reduce hallucinations](https://platform.claude.com/docs/en/test-and-evaluate/strengthen-guardrails/reduce-hallucinations)：hallucination 的表现、uncertainty、grounding/citation 及无法完全消除的边界。
2. Anthropic Claude Platform Docs, [Mitigate jailbreaks and prompt injections](https://platform.claude.com/docs/en/test-and-evaluate/strengthen-guardrails/mitigate-jailbreaks)：direct 与 indirect injection 的不同 threat model、不可信 tool content 与分层防护。
3. Anthropic Claude Platform Docs, [Increase output consistency](https://platform.claude.com/docs/en/test-and-evaluate/strengthen-guardrails/increase-consistency) 与 [Reduce prompt leak](https://platform.claude.com/docs/en/test-and-evaluate/strengthen-guardrails/reduce-prompt-leak)：输出不稳定、schema conformance、prompt leak 与过度复杂控制的 trade-off。
4. Anthropic Engineering, [Effective context engineering for AI agents](https://www.anthropic.com/engineering/effective-context-engineering-for-ai-agents)：context rot、有限注意力、compaction 丢失与长任务状态风险。
5. Anthropic Engineering, [Building Effective AI Agents](https://www.anthropic.com/engineering/building-effective-agents)：自主 agent 的 compounding errors、成本、环境反馈、停止条件与 sandbox 风险边界。
6. Anthropic Engineering, [Demystifying evals for AI agents](https://www.anthropic.com/engineering/demystifying-evals-for-ai-agents)：eval/grader 自身的 failure modes，以及 automated eval、production monitoring、用户反馈和人工复核的互补盲区。

### Guide 要求 vs 当前产品实现

- **Guide 要求**是能识别 LLM 系统的 risks、limitations 和 failure modes；重点是概念边界、系统分层、因果归因、影响排序与 residual risk。
- 当前模型名称、context/tool 特性、Structured Outputs 支持、平台 safety behavior 和具体 API 字段只是实现例证，不是固定考试数字。
- 与 #21 的区别：#21 聚焦 prompt failure / hallucination / model mismatch 的故障诊断；本课覆盖安全、数据、RAG、tool/action、人因、运营与业务在内的完整 risk taxonomy。
- 与 #24 的区别：#24 回答“怎样实施控制”；本课回答“可能怎样失败、为何失败、后果是什么、优先处理什么”。

### Needs verification

实施前需复核当前模型能力与 safety behavior、API/stop/schema 字段、context/tool/agent 限制、平台可用性、数据条款和价格；攻击手法与产品边界也会持续变化。风险阈值、严重度、risk appetite、owner 和 residual acceptance 必须结合组织、法规与真实业务数据确定，这些可变项不用于确定练习题答案。

下一步：[questions.md](questions.md)。
