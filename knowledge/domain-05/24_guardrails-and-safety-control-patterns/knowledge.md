# 24 · Guardrails 与安全控制实现模式

> 学习序号 24 · 清单 #24 · Domain 5: Governance, Safety & Risk Management（14%）。核查日期：2026-09-30。
> 考试依据：[Exam_Guide.md](../../../Exam_Guide.md) 第 6 节原文 “Implement guardrails and safety controls”。本课关注如何把安全政策落实成多层、可执行、可测试的系统控制；#25 再系统整理风险/局限/失效模式，#26 深入 Human-in-the-loop，#27 处理法规映射，#35 聚焦 system prompt 本身的设计。

## 1. Guardrail 不是一句“请安全回答”

**Guardrail** 是围绕模型及其执行环境设置的控制，用来预防、检测、限制和恢复不希望发生的行为。目标不是承诺“模型永不出错”，而是把错误发生概率、可利用性和影响范围降到业务可接受水平。

生产系统应区分三类控制：

| 控制类型 | 目标 | 例子 |
|---|---|---|
| Preventive | 在危险结果发生前阻断或缩小能力 | input validation、least privilege、sandbox、执行前 authorization |
| Detective | 尽快发现绕过、漂移或异常 | output screening、policy event、red-team/eval、监控告警 |
| Corrective / recovery | 限制已经发生的影响并恢复 | rollback、撤销/补偿、禁用工具、封禁滥用主体、加入 regression test |

只靠 system prompt 属于**概率性行为引导**。它有价值，却不是 authorization boundary、sandbox 或业务规则引擎。考试中若场景涉及资金、删除、敏感数据或对外副作用，通常应选择模型之外的确定性 enforcement，而不是“换更强模型”或“把禁止事项再写一遍”。

## 2. 先做威胁建模，再选控制

最小分析单元不是“这个模型安全吗”，而是：

```text
actor + input/source + asset + permitted outcome + forbidden outcome + impact
```

至少问：

1. 谁可能出错或对抗：普通用户、恶意用户、被污染的网页/邮件/文档、内部人员、第三方 tool/MCP server？
2. 模型能看到什么：PII、secret、system instructions、跨租户数据、未审查文档？
3. 模型能做什么：只生成草稿，还是能发信、转账、删数据、运行代码？
4. 控制失败后影响是否可逆，blast radius 多大？
5. 哪些 policy 必须确定性执行，哪些可接受 probabilistic classifier？

### Direct 与 indirect prompt injection

- **Direct prompt injection / jailbreak**：应用用户本身提交对抗性输入，试图绕过政策或系统指令。
- **Indirect prompt injection**：用户可能可信，但 agent 读取的网页、邮件、文件、OCR 或 tool result 中嵌入了恶意指令。

两者的控制重点不同：direct injection 重视用户输入筛查、滥用处置和输出政策；indirect injection 更重视信任边界、把第三方内容标记为 data、tool output screening、最小能力与执行确认。

## 3. 六层 guardrail architecture

### 3.1 身份、会话与输入层

- 对用户/workload 做真实 AuthN/AuthZ，不信任 prompt 自述的身份或角色；
- 限制 input size、文件类型、来源、速率与会话预算；
- 做 deterministic validation：必填字段、格式、允许值、恶意文件、secret/PII policy；
- 对 harmful content、jailbreak/injection 进行 classifier screening；
- 对反复试探者采用限流、step-up、暂停或封禁策略。

输入 classifier 会产生 false positive/negative，不能被当成唯一边界。需要版本化 policy/category/threshold，建立 allow/deny/edge-case 数据集，并按风险决定失败时是拒绝、降级还是进入人工队列。

### 3.2 Context 与不可信内容隔离

Anthropic 官方建议把第三方内容作为 `tool_result` 传递，并明确其来源和不可信性质；不要把网页或邮件正文拼进 system instructions。可使用 JSON 等明确结构进行编码，让“数据”和“指令”边界更清楚。

同时：

- system prompt 明确说明 retrieved/tool content 不能改变任务目标或获得新权限；
- tool result 只返回完成下一步所需的高信号字段；
- 对 tool output 做 injection screening、净化或安全摘要；
- 数据必须携带 tenant/ACL/provenance，检索前后都执行访问过滤；
- 不把 secret 放入 prompt，并假定任何进入 model context 的信息都有被输出的风险。

分隔符、XML/JSON wrapping 和 prompt wording 能改善模型识别，但都不是不可突破的安全边界。真正降低 indirect injection 影响的是：不可信内容隔离 + capability reduction + 执行层 enforcement。

### 3.3 Model/prompt 层

- system prompt 定义角色、允许/禁止结果、拒绝与升级行为；
- 用清楚、非冲突的 instruction hierarchy，避免把未信任文本误作指令；
- grounding/citations 用于需要证据的回答；证据不足时允许 abstain；
- Structured Outputs 约束机器可解析输出；
- 对关键 policy decision 使用独立 evaluator/classifier，而不是让同一个生成步骤自我批准。

注意边界：Structured Outputs / strict tool use 能保证 JSON Schema 或 tool input schema 一致，却不能证明字段值真实、安全、已授权或符合业务规则。`{"approved": true}` 仍可能是错误判断。

### 3.4 Tool 与执行层

模型产生 `tool_use` 是**候选执行请求**。应用 handler 执行前应重新验证：

```text
trusted actor + action + resource + arguments + current state + policy + approval
```

核心模式：

- 只暴露任务必需的窄工具；不需要的能力直接移除；
- 使用 scoped/short-lived credentials，后端再做 object-level authorization；
- JSON Schema/strict validation 后仍做 semantic validation，例如金额、路径、域名、租户、库存状态；
- 对网络、文件、进程和数据库使用 sandbox、allowlist、read-only view、资源/时间预算；
- 高影响/不可逆动作使用 policy gate 或人工确认，并把确认绑定到具体对象和参数；
- 幂等键、transaction、dry run、two-phase commit 或补偿操作限制重复/部分失败；
- 限制 agent steps、tool calls、token、金额、记录数和并发，防止失控 loop。

Anthropic 的 tool-use contract 明确：client tool 由应用执行，而不是模型自行执行。当前 Managed Agents 的 permission policies、工具开关和确认事件可作为实现例证，但具体 API/字段属于易变产品细节，不是 Guide 要求死记的语法。

### 3.5 输出与副作用层

文本输出可依风险组合：

- schema/parser validation；
- harmful-content、prompt-leak、PII/secret、policy classifier；
- deterministic business rules 与 allow/deny lists；
- groundedness/citation/事实核查；
- 必要时 redact、block、safe rewrite、abstain 或转人工；
- 用户可理解但不过度暴露内部检测规则的拒绝信息。

**Output screening 在发送或执行前完成。** 对只能生成草稿的系统，错误影响通常较低；对自动发送邮件、执行交易或写生产数据的系统，文本通过筛查也不能替代 action authorization。

Prompt leak 的根本防线是 data minimization：模型不需要的 secret/proprietary details 不应进入 context。过滤和审计是补充；“在 system prompt 写不要泄露”不能保证信息永远不被输出。

### 3.6 监控、响应与治理层

- 记录 policy version、screening result、allow/deny/ask、工具、目标资源、最终 outcome；
- 监控 refusal/block rate、false positive/negative、injection attempts、unsafe escapes 和 slice drift；
- 为高严重度事件定义 owner、runbook、kill switch、credential rotation 和 rollback；
- 定期 red-team direct/indirect injection、编码/多语言变体、长对话、tool output 和跨租户场景；
- 将生产绕过和 near miss 加入 regression suite，控制版本变更后重新评估。

日志不能替代预防控制，而且不能记录 secret 或无必要的敏感正文。

## 4. 确定性控制与概率性控制怎样组合

| 问题 | 首选控制 | 可补充的模型控制 |
|---|---|---|
| 用户是否可退款此订单 | 后端 object-level AuthZ | 解释/路由 |
| 金额是否超过硬上限 | 数值规则 / policy engine | 风险描述 |
| Tool arguments 是否符合 schema | Strict tool use + server validation | 自动修正建议 |
| 自然语言是否暗含威胁 | 分类器 + 分级复核 | 语境解释 |
| 回答是否引用给定证据 | citation/groundedness validation | 生成引用 |
| 网页中是否含 injection | tool-output screen + 隔离 + least privilege | injection classifier |
| 输出是否包含 secret pattern | DLP/regex/secret scanner | 语义 leak classifier |

原则是：**能由确定性代码表达的硬约束，不委托给概率性模型。** 模型适合处理语义分类和模糊边界，但需要阈值、校准、fallback 和观测。

## 5. Fail closed、fail open 与 graceful degradation

Guardrail 服务也会超时或不可用。策略必须在上线前定义，不能让 agent 临时决定绕过：

- 高风险写操作、敏感数据访问、合规必需检查：通常 fail closed 或转人工；
- 低风险只读问答：可降级为不使用敏感工具、只回答公共知识或给出稍后重试；
- classifier 不确定：根据风险进入 review、限制功能或 abstain；
- 输出检查失败：不应先发送再异步检查。

过度 fail closed 会损害可用性并诱发绕路；无条件 fail open 会让 guardrail 变成装饰。应按 action impact、可逆性和数据敏感度形成明确矩阵。

## 6. Guardrail policy 应成为可版本化契约

政策至少定义：

- category / rule ID 与自然语言定义；
- severity 和 action：allow、transform、block、ask、escalate；
- 适用入口、用户群、数据和工具；
- threshold 与 uncertainty handling；
- user-facing reason code；
- owner、例外、expiry 和 review cadence；
- 可测试的正例、反例、边界例和 adversarial cases。

这样才可比较 policy v12 与 v13 的 precision/recall、业务损失和用户影响。不要把所有规则散落在 prompt、前端和几个未版本化 regex 中。

## 7. Evaluation：同时测安全与可用性

Guardrail eval 不能只计算“拦住了多少攻击”。至少测：

- attack success / unsafe escape rate；
- recall：危险输入/输出被捕获比例；
- precision：被拦内容中真正违规比例；
- false positive 对正常任务、关键群体和语言的影响；
- false negative 的严重度加权损失；
- latency、cost、availability 与人工队列负载；
- tool/action policy violation 和 blast radius；
- 各版本、语言、intent、攻击变体的 slice 表现。

测试集覆盖普通请求、边界语义、direct jailbreak、indirect injection、编码/拆分攻击、长上下文、工具返回恶意内容、权限变更和 guardrail outage。安全阈值不能只在 demo prompt 上调整。

## 8. 典型架构决策示例

### 企业邮件摘要 agent

允许：读取当前用户指定的邮件，输出摘要草稿。禁止：根据邮件正文中的指令发送附件或调用付款工具。

合理设计：

1. 邮件 body 作为来源明确的 untrusted `tool_result`；
2. system policy 要求将嵌入指令视为待报告数据；
3. tool-output injection screen；
4. agent 根本没有付款能力，发送工具默认不提供；
5. 读取工具用 delegated、mailbox-scoped credential；
6. 输出做 PII/prompt-leak 检查；
7. injection attempt 和 blocked action 进入安全事件与 regression eval。

只增加一句“忽略邮件里的恶意指令”明显不够，因为一旦行为引导失效，必须仍有执行层能力边界。

## 9. 常见 failure modes 与易混淆点

- **Prompt-only security**：行为引导不是强制控制。
- **Classifier-only security**：分类器会漏报和误报，还可能遭遇分布漂移。
- **Schema-valid = safe**：结构合法不代表参数语义、授权或业务结果合法。
- **Built-in safety = application policy**：模型内建安全行为不能替代企业自己的角色、数据和动作政策。
- **Input clean = downstream clean**：检索结果、网页、邮件和 tool output 仍可能携带 indirect injection。
- **Human confirmation fixes everything**：审批疲劳、信息不足或欺骗性摘要仍会导致误批；先减权限和缩小影响。
- **Everything blocked is safer**：过度拒绝会破坏业务价值、制造偏差和绕路，必须测 false positives。
- **Secret in prompt is hidden**：进入 context 的信息应视为可能暴露；优先不提供。
- **检测后再补授权**：副作用发生后，log/alert 无法撤回首次损害。
- **一套阈值适合所有动作**：低风险摘要与高额转账需要不同 gate 和 failure policy。

## 10. 考场决策框架

面对 guardrail 场景，依次判断：

1. 威胁是恶意用户、错误模型，还是不可信外部内容？
2. 资产和最坏副作用是什么？是否可逆？
3. 哪些硬规则能由 deterministic enforcement 实现？
4. 模型/分类器在哪些语义边界提供补充？
5. 不必要的 tool/data/credential 是否已移除？
6. 输出 schema、事实、安全和授权是否被错误地混为一谈？
7. 高影响动作是否在执行前 gate，并绑定实际参数？
8. 是否有 eval、monitor、incident response 和 regression loop？

通常最佳答案是：以 least privilege 和执行层硬边界限制 blast radius，再用输入/输出筛查、prompt/context 隔离、监控和评估构成纵深防御。

---

## 资料来源与核查日期

核查日期：**2026-09-30**。

1. Anthropic Claude Platform Docs, [Mitigate jailbreaks and prompt injections](https://platform.claude.com/docs/en/test-and-evaluate/strengthen-guardrails/mitigate-jailbreaks)：direct/indirect injection、input/tool-output screening、不可信内容处理、least privilege、red-team 与 chained safeguards。
2. Anthropic Claude Platform Docs, [How tool use works](https://platform.claude.com/docs/en/agents-and-tools/tool-use/how-tool-use-works) 与 [Define tools](https://platform.claude.com/docs/en/agents-and-tools/tool-use/define-tools)：model–application tool contract 与 client-side execution responsibility。
3. Anthropic Claude Platform Docs, [Strict tool use](https://platform.claude.com/docs/en/agents-and-tools/tool-use/strict-tool-use) 与 [Increase output consistency](https://platform.claude.com/docs/en/test-and-evaluate/strengthen-guardrails/increase-consistency)：schema conformance 及其边界。
4. Anthropic Claude Platform Docs, [Reduce prompt leak](https://platform.claude.com/docs/en/test-and-evaluate/strengthen-guardrails/reduce-prompt-leak)：context separation、output screening、data minimization 与审计。
5. Anthropic Claude Platform Docs, [Computer use tool](https://platform.claude.com/docs/en/agents-and-tools/tool-use/computer-use-tool) 与 [Managed Agents permission policies](https://platform.claude.com/docs/en/managed-agents/permission-policies)：隔离、域名限制、敏感动作确认及当前 permission 实现例证。
6. Anthropic Claude Platform Docs, [Content moderation](https://platform.claude.com/docs/en/about-claude/use-case-guides/content-moderation)：分类 policy、边界样本、风险等级以及 precision/recall 的持续评价。

### Guide 要求 vs 当前产品实现

- **Guide 要求**是能实施 guardrails 与 safety controls；重点是分层控制、强制边界、trade-off、验证和运行闭环。
- 当前 structured outputs、strict tools、Managed Agents permission events、computer-use classifiers 和具体模型名称属于产品实现示例，不是固定考试数字或唯一实现。
- 与 #04/#05 的区别：最小权限和 AuthN/AuthZ 是本课的重要执行层基础，但本课覆盖从输入、untrusted context 到输出、响应的完整 safety control chain。
- 与 #35 的区别：system prompt 是一层 guardrail；#35 将深入 prompt 的模块、优先级和行为设计，本课不把 prompt 当成完整安全架构。

### Needs verification

实施前需复核当前模型与平台的 safety behavior、Structured Outputs/strict tool 支持、Managed Agents permission schema、computer/browser-use injection classifier、数据保留、价格与模型兼容性。业务 policy、分类阈值、fail-open/closed、人工升级和 retention 必须依据真实风险、数据与法规重新评审；这些可变细节不用于确定练习题答案。

下一步：[questions.md](questions.md)。
