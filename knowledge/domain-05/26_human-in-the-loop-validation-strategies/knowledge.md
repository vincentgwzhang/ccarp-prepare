# 26 · Human-in-the-loop 验证策略设计

> 学习序号 26 · 清单 #26 · Domain 5: Governance, Safety & Risk Management（14%）。核查日期：2026-10-02。
> 考试依据：[Exam_Guide.md](../../../Exam_Guide.md) 第 6 节原文 “Apply human-in-the-loop validation strategies”。本课关注人应在何时、凭什么、以何种权限介入，以及怎样验证这套机制；#24 已讲整体 guardrails，#25 已讲风险分类，#27 再处理法规映射。

## 1. HITL 不是“界面上有个确认按钮”

**Human-in-the-loop（HITL）** 是把具备适当能力与责任的人，放到 AI 系统的关键决策或执行路径中，使其能审查证据、补充判断、批准、拒绝、修改或升级处理。

一个有效的 HITL gate 必须同时回答：

1. **Trigger**：什么风险或不确定性触发人工介入？
2. **Reviewer**：谁具备权限、专业知识与独立性？
3. **Review package**：人能看到哪些原始证据、拟议动作和影响？
4. **Decision**：可做 allow、deny、edit、request evidence、escalate 中的哪些动作？
5. **Binding**：批准是否绑定到确切 action、resource、arguments、版本和期限？
6. **Execution**：批准后由谁执行，执行前是否重新校验状态？
7. **Audit and learning**：如何记录、评价并回流为控制和 eval？

如果 reviewer 只能看到模型生成的一句“此操作安全”，或者确认后 agent 可以修改参数，人工只是 rubber stamp，不是可靠控制。

## 2. Human-in、on、out of the loop

| 模式 | 人的位置 | 适用场景 | 主要代价/风险 |
|---|---|---|---|
| Human-in-the-loop | 动作执行前必须由人决定或补充输入 | 高影响、不可逆、需要同意或专家判断 | 延迟、成本、队列容量、审批疲劳 |
| Human-on-the-loop | 系统通常自动运行，人监控并可暂停/覆盖 | 低到中风险、可逆、监控信号充分 | 人可能来不及发现，监控盲区 |
| Human-out-of-the-loop | 无逐项人工介入，靠 policy/eval/monitoring | 低风险、确定性验证充分、易恢复、高吞吐 | 未知 failure 可能直接触达用户 |

还可组合：低风险自动执行，中风险 exception review，高风险 mandatory pre-approval；已自动执行的低风险案例再做抽样复核。考试中不要默认“所有输出都人工审批”最安全：无法承载的控制会造成积压、绕过和形式化点击。

## 3. 什么时候必须或适合人工介入

### 3.1 现实后果与不可逆性

优先在**执行前**介入：

- 付款、退款、签约、接受条款或代表用户表示 consent；
- 删除、发布、发送消息、修改共享/生产系统；
- 医疗、法律、金融、人事等高影响建议或决定；
- 访问或披露敏感数据；
- 难以撤销、blast radius 大或补偿成本高的动作。

Anthropic 的 computer-use 文档明确建议：可能产生有意义现实后果、需要 affirmative consent 的决定应由人确认；涉及完美精度或敏感信息的任务不应在缺少人工监督时使用。

### 3.2 不确定性、歧义与异常

- 必要信息缺失、来源矛盾或 grounded evidence 不足；
- model/classifier confidence 落在灰区；
- 输入超出已评估分布，出现新语言、新工具或新任务；
- policy 无法确定 allow/deny；
- 检测到 prompt injection、权限变化或异常执行路径；
- tool 返回冲突、partial failure 或无法确认最终状态；
- 用户要求超过 agent 授权边界。

不要只用模型自报 confidence 决定是否升级。更可靠的 trigger 结合 deterministic rules、action risk、数据敏感度、evidence coverage、模型/分类器信号和运行异常。

### 3.3 人类有独特判断价值

- 目标本身需要价值判断、同意、同理心或语境权衡；
- SME 才能判断专业正确性；
- 需要 separation of duties 或双人批准；
- 需要处理例外、申诉和 contested decision；
- 模型 grader 必须以 human expert calibration。

若问题能被确定性规则可靠验证，例如金额是否超过硬上限、订单是否属于用户，就应先用代码强制执行，而不是浪费 reviewer 注意力。

## 4. 四种常见验证模式

### 4.1 Pre-execution approval

Agent 先提出动作，系统暂停，人批准后才执行。适合高影响或需要 consent 的副作用。

批准对象应是不可歧义的 proposal：

```text
actor + action + target + exact arguments + evidence + expected effect
+ policy/risk reason + expiry + proposal hash/version
```

批准后任何关键参数变化，都应使原批准失效并重新审查，避免 **time-of-check/time-of-use（TOCTOU）** 和“批准 A、执行 B”。

### 4.2 Exception-based review

低风险正常路径自动处理，只有 policy gray zone、低置信、冲突证据或异常行为进入人工队列。它节省人力，但要求 gate 的 false negative 可接受，并持续抽样自动通过案例。

### 4.3 Post-hoc sampling and audit

对已完成、低风险且可逆的任务抽样复核，用于发现漂移、未知 failure、slice disparity 和 reviewer/calibration 问题。它是 detective/learning control，不能替代不可逆高风险动作的执行前批准。

抽样不应只有均匀随机；可组合：

- baseline random sample；
- 按风险、用户群、语言、模型/版本分层；
- 新版本、新工具和新场景加权；
- disagreement、近阈值和异常 trace 定向抽样。

### 4.4 Dual control / specialist escalation

高价值、高敏感或利益冲突场景可能要求第二位独立 reviewer、特定资质 SME 或合规/安全人员。不能让提出动作的人同时成为唯一批准者，也不能让模型生成的“自我批评”冒充独立审批。

## 5. Review package：让人看到可判断的信息

Reviewer 通常至少需要：

- 用户原始 intent 与可信身份/授权上下文；
- agent 拟执行的 action、目标对象和**确切参数**；
- 原始来源、retrieved passages、tool results 与 provenance；
- 模型结论与关键理由，但不能只给模型摘要；
- 不确定性、矛盾证据、缺失字段和触发 gate 的原因；
- 可能影响、可逆性、替代方案和 recovery plan；
- policy/rule ID、历史相关决定和必要的业务状态；
- 清晰的 allow、deny、edit、request-more-evidence、escalate 操作。

### 防止 automation bias

- 先显示原始证据和关键差异，再显示模型建议；
- 明确标注“AI-generated”，不把流畅文字当可靠性信号；
- 对高风险决定要求 reviewer 主动确认关键事实，而非一键同意；
- 显示反证、未知项和替代方案；
- 定期插入已知测试案例，衡量 reviewer 是否认真判断；
- 允许安全地 disagree，并避免用 approval speed 作为唯一绩效指标。

解释越长不一定越好。界面应把 reviewer 的有限注意力用于真正决定风险的证据。

## 6. 审批必须绑定执行，而不是只绑定文本

可靠流程如下：

```text
prepare proposal
  → deterministic validation
  → human review
  → signed/bound approval
  → revalidate current state
  → execute idempotently
  → record outcome
```

关键设计：

- 审批记录绑定 user/reviewer、action、resource、arguments、policy version、proposal hash 与 expiry；
- reviewer 只能批准自己有权限批准的范围；
- 执行前重新验证 AuthZ、对象状态、金额、版本和资源锁；
- 参数变化、审批过期或状态变化必须重新审查；
- 使用 idempotency key、transaction 或 compensation 防止重复执行；
- deny 不能被 agent 自行重试绕过；
- 审批 UI 与执行 service 使用同一 canonical representation，避免显示内容与执行 payload 不一致；
- log 包含 proposal、decision、reviewer、reason、执行结果，但遵循 data minimization。

当前 Managed Agents 的 permission policy 可在 tool call 前暂停并接收 allow/deny confirmation，是实现例证。它不自动覆盖应用自定义工具：custom tool 是否执行仍由应用决定。因此 Guide 考点是架构原则，不是背某个 API event 名称。

## 7. Queue、SLA 与 failure policy

HITL 是一个需要容量规划的服务，而不是无限资源。

至少设计：

- 基于 severity、deadline、客户影响的 priority；
- reviewer skills、region、language 与 authorization routing；
- review SLO、队列深度、age、abandonment 和 escalation；
- proposal expiry 与 stale-state revalidation；
- reviewer 不可用、意见分歧和超时的 fallback；
- workload caps、轮班、休息与质量抽查；
- incident surge 下的降级模式。

### 超时不能默认批准

- 高影响写操作：通常 fail closed、等待、转更高级 reviewer 或取消；
- 中风险且可逆：可转降级路径，例如只生成 draft、不自动发送；
- 低风险只读：可按预定义 policy 自动处理并加强抽样；
- 紧急场景：使用事先批准的 break-glass 流程、明确范围、双重记录和事后审查。

“审批超时则自动 allow”会把排队压力直接变成安全绕过。

## 8. Reviewer 的权限、能力与独立性

选择 reviewer 依据的不只是职级：

- 是否理解业务与证据；
- 是否有权限代表相关主体作出决定；
- 是否独立于请求者和系统开发者；
- 是否接受 policy、bias、security 与工具使用培训；
- 是否能识别模型不确定性和 prompt injection；
- 是否有清晰责任、升级路径与合理工作量。

典型角色分离：

| 角色 | 职责 |
|---|---|
| Requestor / user | 提供 intent、必要信息和 consent |
| Agent | 准备 proposal、证据与不确定性 |
| Policy service | 确定 hard rules、风险等级与 gate |
| Reviewer / SME | 独立判断、修改、拒绝或升级 |
| Executor | 校验 approval 与当前状态后执行 |
| Auditor / risk owner | 评估控制效果与 residual risk |

高风险场景不应让 agent 自己选择一个宽松 reviewer，或让未验证的 prompt 指定“经理已批准”。

## 9. 怎样评价 HITL 是否真的有效

### Decision quality

- approval/denial correctness，必要时与 gold/SME adjudication 比较；
- reviewer disagreement、inter-rater agreement；
- model recommendation 被 overturn/edit/escalate 的比例与原因；
- 人工捕获的严重错误、near miss 与漏放；
- 按风险、语言、群体、reviewer、模型版本的 slice 结果。

### Operational quality

- time-to-decision、queue age、SLO miss；
- review volume、automation rate 与 unit cost；
- reviewer workload、fatigue proxy、abandonment；
- stale/expired approvals、重复审批和 bypass attempts；
- downstream incident rate 与 recovery time。

### Gate quality

- 应升级案例的 recall；
- 被升级案例中确实需要人工判断的 precision；
- false negative 的严重度；
- false positive 对时延、用户体验和人工容量的影响。

Anthropic 对 agent eval 的建议是组合 code-based、model-based 与 human graders，并以人类专家校准 LLM judge。人工判断质量也需测量；“有人看过”不是正确性的证明。

## 10. 把人工决定用于改进，但不要盲目训练

记录结构化 reason code、修改前后差异、证据和最终 outcome，可用于：

- 增补 regression tests 与 adversarial cases；
- 调整 deterministic policy、routing 和 UI；
- 校准 classifier/LLM grader；
- 识别高频可自动化的低风险判断；
- 发现新的风险类别与专业知识缺口。

但 reviewer decision 可能不一致、有偏见或误判。不能把每个批准结果直接当作“正确标签”在线训练；先做 quality sampling、adjudication、去除敏感信息、版本管理和 drift 检查。

## 11. 常见 failure modes

- **Approval theater**：只显示模型结论，不显示原始证据和真实参数。
- **Approve A, execute B**：审批未绑定 payload，参数之后被修改。
- **Reviewer lacks authority/expertise**：看似人工复核，实际无权或无能力判断。
- **Automation bias**：人因输出流畅、默认选中或时间压力而照单全收。
- **Approval fatigue**：所有动作都弹窗，关键请求被淹没。
- **Timeout means allow**：队列故障成为绕过路径。
- **Post-hoc for irreversible harm**：事后抽样无法撤销首次损害。
- **Human as deterministic validator**：本可由代码验证的硬规则占用注意力并增加误差。
- **No escalation path**：reviewer 只有 allow/deny，证据不足时被迫猜测。
- **No revalidation**：审批后对象状态或权限已变化。
- **Feedback = truth**：未经校验的 reviewer decision 直接变成训练标签。
- **No capacity model**：上线后队列积压，业务转而绕过 HITL。

## 12. 考场决策框架

面对 HITL 场景，依次判断：

1. 动作是否有现实后果、需要 consent、不可逆或高 blast radius？
2. 哪些硬约束应先由 deterministic control 验证？
3. 人的独特判断价值是什么：专业知识、语境、价值判断还是授权？
4. 人应在执行前、异常时、监控中，还是事后抽样介入？
5. reviewer 是否看到原始证据、确切参数、不确定性与影响？
6. approval 是否绑定 action/resource/arguments/version/expiry？
7. 超时、拒绝、分歧、状态变化和 reviewer 不可用时怎样处理？
8. 怎样检测 automation bias、fatigue、漏升级和 reviewer drift？

通常最佳答案是：按风险选择最小但有效的人工介入，在高影响动作执行前提供可验证证据并绑定确切 payload，同时保留权限校验、审计、SLO、升级和持续评价；而不是对所有请求统一弹出一个“确认”。

---

## 资料来源与核查日期

核查日期：**2026-10-02**。

1. Anthropic Engineering, [Building Effective AI Agents](https://www.anthropic.com/engineering/building-effective-agents)：agent 在 checkpoint/blocker 暂停获取人工反馈、使用环境 ground truth 与停止条件。
2. Anthropic Claude Platform Docs, [Computer use tool — Security considerations](https://platform.claude.com/docs/en/agents-and-tools/tool-use/computer-use-tool)：现实后果、affirmative consent、敏感任务的人工确认与监督，以及逐个 tool block 执行前检查。
3. Anthropic Claude Platform Docs, [Prompting best practices — Balancing autonomy and safety](https://platform.claude.com/docs/en/build-with-claude/prompt-engineering/claude-prompting-best-practices)：难以逆转、影响共享系统、破坏性或对外可见操作的确认边界。
4. Anthropic Claude Platform Docs, [Managed Agents permission policies](https://platform.claude.com/docs/en/managed-agents/permission-policies)：tool call 的 allow/ask/deny、暂停/恢复流程，以及 custom tool 由应用负责执行的当前实现。
5. Anthropic Engineering, [Demystifying evals for AI agents](https://www.anthropic.com/engineering/demystifying-evals-for-ai-agents)：code/model/human grader 的互补性、人工成本与专家校准。

### Guide 要求 vs 当前产品实现

- **Guide 要求**是能应用 HITL validation strategy：选择介入点、reviewer、证据、审批绑定、运行机制和评价方法。
- 当前 Managed Agents permission policy、event/stop-reason 字段、computer-use classifier 与具体模型行为是产品实现例证，不是固定考试语法。
- 与 #24 的区别：HITL 是多层控制中的一类；本课深入人的决策质量、交互与运行队列，不重复完整 guardrail architecture。
- 与 #25 的关系：risk、impact、detectability 和 reversibility 决定 human-in/on/out-of-loop 的配置。

### Needs verification

实施前需复核当前 Managed Agents、computer/browser use、permission policy、event schema、模型兼容性、平台差异、数据保留与价格。审批阈值、review SLO、人员资质、双人控制和证据保留期必须按具体业务、法规和组织 risk appetite 决定；这些可变项不用于确定练习题答案。

下一步：[questions.md](questions.md)。
