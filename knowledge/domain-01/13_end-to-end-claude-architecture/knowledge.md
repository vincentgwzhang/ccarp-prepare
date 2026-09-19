# 端到端 Claude 架构：Input → Processing → Output → Feedback Loops

> **学习序号：13**  
> **Exam Guide 对应范围：Domain 1 — Solution Design & Architecture（17%）**  
> **具体目标：Design end-to-end architectures (input → processing → output → feedback loops)**

## 1. 考点边界

本考点要求你能把一个 Claude-based solution 设计成**完整、可运行、可改进的系统**，而不是只画一个 `Application → Claude API` 箭头。

一套完整架构至少要回答：

1. 输入从哪里来，进入模型前如何认证、规范化和约束？
2. 处理阶段如何组装 prompt/context、访问权威数据、调用 Claude 与工具？
3. 输出如何校验、授权、持久化并交付给用户或下游系统？
4. 失败时如何 retry、fallback、补偿或转人工？
5. 线上反馈怎样进入评估和版本迭代，而不直接污染生产行为？

本课讨论这些组件怎样形成端到端链路；下一课 #14 才专门比较 workflow、agentic 与 augmented LLM 的模式选型。

## 2. 参考架构

```text
Channels / Events / Batch
          │
          ▼
[Input & Trust Boundary]
 authn/authz · schema · size · rate limit · content controls
          │
          ▼
[Application Orchestrator] ───────► [State / Idempotency / Queue]
 task state · policy · timeout              │
          │                                  │
          ├────────► [Context Layer] ◄───────┘
          │          retrieval · ACL · history · source of truth
          │
          ├────────► [Claude Gateway]
          │          prompt/version · model config · budget
          │                     │
          │                     ▼
          └────────────── [Claude API]
                                │ tool request / response
                                ▼
                     [Tool Execution Boundary]
                     allowlist · authz · validation · audit
                                │
                                ▼
                     [Output Validation & Policy]
                     schema · grounding · business rules · HITL
                                │
                                ▼
                     [Delivery / Action / Persistence]
                                │
             ┌──────────────────┴──────────────────┐
             ▼                                     ▼
      Runtime signals                       User/business outcomes
 logs · traces · cost · errors        corrections · acceptance · incidents
             └──────────────────┬──────────────────┘
                                ▼
                     [Eval & Improvement Loop]
 datasets · regression tests · review · versioned rollout
                                │
                         approved change only
                                └────────► prompts/context/tools/policy
```

这不是 Exam Guide 强制规定的唯一图，而是一种帮助检查遗漏的 reference architecture。组件可合并或拆分，但它们承担的责任不能消失。

## 3. Input：入口不仅是“用户输入框”

输入可能来自同步 API、聊天 UI、上传文档、消息队列、定时 batch 或另一个 agent。进入处理层前应建立 trust boundary。

### 3.1 Input contract

定义并验证：

- 调用者身份、tenant、role 与 resource scope；
- 请求 schema、media type、字符编码、大小和必填字段；
- conversation/task ID、correlation ID 与幂等键；
- 数据分类、同意、保留期限和允许用途；
- 请求 deadline、优先级、同步或异步交付方式。

### 3.2 不可信内容与可信指令分离

用户文本、检索文档和网页内容都是 untrusted data；它们不应获得与 system policy 相同的权限。输入检查不能保证消灭 prompt injection，因此还必须在 tool execution 和 output/action 阶段再次授权。**不能把输入清洗当作唯一安全边界。**

### 3.3 何时拒绝、排队或降级

- schema/身份不合法：拒绝，不调用模型；
- 超过同步 SLA 的长任务：进入 durable queue，返回 task ID；
- 缺少关键事实：请求补充信息或转人工；
- 重复请求：用 idempotency state 返回已有结果，避免重复副作用；
- 高峰或依赖故障：按业务优先级限流、延迟或使用事先验证的 fallback。

## 4. Processing：把概率推理包在确定性控制中

### 4.1 Application orchestrator 是责任中心

无论内部采用单次调用、固定 workflow 还是 agent loop，application/orchestrator 都应负责：

- 加载版本化 policy、prompt 和配置；
- 获取对当前 principal 可见的 context；
- 管理 deadline、budget、重试次数与停止条件；
- 保存业务状态，而不是假设 Claude API 保存会话；
- 解析响应状态，决定继续、调用工具、fallback 或结束；
- 记录可关联、经脱敏的 telemetry。

Anthropic 当前 Messages API 文档明确说明 API 是 stateless：应用在多轮场景中负责传入所需历史。因此，生产架构必须明确 conversation state 的 owner、压缩/保留策略以及并发更新规则。

### 4.2 Context assembly

每次调用只组装完成当前任务所需的 context：

- system instructions 与业务 policy；
- 当前用户请求和必要会话状态；
- 经 ACL 过滤的检索结果；
- 工具定义及其权限范围；
- 必要的 examples、output schema 和 provenance metadata。

context 是“提供给模型的信息”，不是 source of truth。账户余额、退款资格等仍应由权威服务确认。

### 4.3 Claude gateway

对 Claude 的调用最好经过统一 gateway/adapter，以集中管理：

- 模型与 prompt/config version；
- timeout、并发、预算和重试策略；
- request ID、token/usage 与 latency 记录；
- provider/API errors 与 application errors 的分类；
- feature rollout、fallback 和 kill switch。

不要盲目 retry 所有失败。只对可安全重试的 transient failure 使用 bounded retry + backoff；有副作用的下游工具还需要 idempotency key。格式错误、权限错误或业务拒绝通常必须修正请求或进入 fallback，而不是无限重试。

### 4.4 正确解释模型响应

HTTP 成功只表示请求被处理，不代表业务任务完成。应用还要检查：

- 内容 block 类型；
- `stop_reason`：自然结束、tool request、截断、拒绝等需要不同处理；
- output schema 和业务约束；
- 是否包含足够 evidence/provenance；
- 是否需要继续一轮、执行工具或转人工。

Anthropic 的当前文档要求应用依据 `stop_reason` 决定使用响应、继续、执行工具或 fallback。具体枚举可能演进，实施时应以最新 schema 为准。

## 5. Tool execution 是独立的安全与事务边界

Claude 产生 tool request 并不等于工具已经执行，也不等于动作已获授权。执行层应完成：

1. 检查 tool 是否在当前任务 allowlist；
2. 以实际 principal/tenant 重新做 object-level authorization；
3. 校验参数 schema、业务前置条件和资源版本；
4. 对高影响动作请求 approval；
5. 用 idempotency key 执行并记录审计；
6. 将最小必要、明确标识成功或失败的 tool result 返回 orchestrator。

对于资金、账户或外部通知等副作用，应明确 transaction boundary、重复提交行为和 compensation。模型无法替代 database transaction、optimistic locking 或 saga 的一致性职责。

## 6. Output：生成结果必须经过交付契约

### 6.1 三层验证

| 层 | 关注点 | 例子 |
|---|---|---|
| Syntactic | 结构是否可解析 | JSON/schema、必填字段、类型 |
| Semantic | 内容是否符合任务要求 | 摘要是否覆盖关键条款、引用是否支持结论 |
| Business/Policy | 是否允许交付或执行 | 金额上限、权限、合规、人工审批 |

schema-valid 不代表事实正确，更不代表业务允许。反过来，自由文本也不应直接驱动交易系统。

### 6.2 输出去向决定控制强度

- **展示给用户的建议**：标注局限、证据和下一步；
- **写入内部草稿**：保留来源、版本和 reviewer；
- **写入 system of record**：要求更强校验、幂等和审计；
- **触发外部副作用**：需要授权、审批/政策门、补偿与确认；
- **进入另一个模型**：明确数据/指令边界，防止错误和注入级联。

### 6.3 失败不是单一状态

端到端结果至少应区分：成功、部分成功、需要补充信息、业务拒绝、模型拒绝、依赖失败、超时、待审批和人工接管。把所有情况都转换成“抱歉，请重试”会破坏可恢复性和可观测性。

## 7. 两类 Feedback Loop

### 7.1 Runtime control loop：单个任务内闭环

运行时反馈帮助当前任务决定下一步：

```text
Claude output/tool request
        ↓
validator / tool / environment returns ground truth
        ↓
orchestrator decides: continue | retry | repair | approve | fallback | stop
```

例如代码 agent 运行测试后根据失败结果修复；客服系统发现订单不存在后请求用户核对，而不是继续编造。循环必须有最大迭代、deadline、cost budget 和停止条件，避免 runaway loop 与复合错误。

### 7.2 Improvement loop：跨请求的系统改进闭环

线上数据不应直接、自动改写 production prompt。安全路径是：

1. 收集显式反馈、人工修正、task outcome、错误和性能指标；
2. 脱敏、采样、标注，并分析反馈偏差；
3. 将代表性案例加入 versioned eval dataset；
4. 修改 prompt、context、tool 或 policy；
5. 跑 regression/safety eval，并由 owner review；
6. canary/A-B rollout，监控后再扩大；
7. 发生退化时 rollback。

“用户点了赞”只是一个 noisy signal：可能受速度、措辞或选择偏差影响，不一定证明事实正确。应将 user feedback、业务 outcome、专家 rubric 和 operational metrics 组合使用。

## 8. 状态与一致性设计

Claude-based system 常同时包含四种状态：

| 状态 | 例子 | 设计重点 |
|---|---|---|
| Conversation state | 历史消息、摘要、用户偏好 | owner、版本、保留和并发写入 |
| Task state | 当前步骤、deadline、审批、重试次数 | durable state machine、恢复点 |
| Business state | 订单、工单、余额、权限 | system of record、transaction、授权 |
| Learning state | eval case、标签、prompt/config version | 数据治理、可复现、审批与回滚 |

常见错误是把四种状态都塞进聊天历史。聊天历史不是可靠的事务日志，也不是业务数据库；反馈数据也不应未经审核直接进入生产 prompt。

## 9. 同步与异步路径

### 同步适合

- 交互式问答或短任务；
- 能在调用者 deadline 内完成；
- 失败可立即反馈，且不依赖长时间外部任务。

### 异步适合

- 文档批处理、长时分析、多步骤外部依赖；
- 需要 durable retry、暂停审批或断点恢复；
- 调用者可以用 task ID 查询或接收 callback/event。

异步设计必须补上 queue、状态机、幂等、结果存储、取消、超时和通知；仅把 HTTP timeout 调大不是可靠的长任务架构。

## 10. Failure modes 与设计回应

| Failure mode | 表现 | 设计回应 |
|---|---|---|
| Context contamination | 未授权/不相关数据进入 prompt | ACL before retrieval、provenance、最小上下文 |
| Truncated output | 响应可解析但不完整 | 检查 stop reason、完整性和业务字段 |
| Duplicate side effect | retry 导致重复退款/发信 | idempotency key、状态检查、outbox/transaction |
| Stale business state | 模型依据旧订单状态行动 | 执行前从 source of truth 重新验证 |
| Runaway loop | 工具失败后反复调用 | iteration/deadline/cost limit、circuit breaker |
| Feedback poisoning | 恶意评价自动改变行为 | 隔离反馈、审核、eval gate、版本化发布 |
| Partial success hidden | 部分工具成功却整体报错 | per-step state、补偿、明确 partial result |
| Untraceable incident | 无法关联模型与下游调用 | correlation/request/task IDs、版本和审计事件 |
| Silent quality regression | API 正常但答案变差 | representative eval、outcome monitoring、rollback |

## 11. 场景：合同审查助手

需求：法务上传合同，系统提取关键条款、对照公司 playbook，生成风险清单与修改建议；高风险判断由律师确认。

### Input

- 用户身份和 matter ACL；
- 文件类型、大小、恶意文件检查；
- matter ID、document version 与 idempotency key；
- 数据保留和允许用途。

### Processing

- 文档解析后保留页码/段落 provenance；
- 检索当前版本 playbook，并按业务单元过滤；
- orchestrator 组装任务上下文并调用 Claude；
- 对缺失页、解析失败和冲突条款显式处理；
- 记录 prompt/model/config 和 source versions。

### Output

- 使用结构化 finding：条款、证据、风险类别、建议、需人工判断项；
- schema validation + citation/evidence 检查；
- 律师确认后才写入 matter record 或生成对外 redline。

### Feedback

- 当前任务：证据不足则回到 retrieval/解析或请求人工；
- 长期改进：保存律师的接受/修改/驳回结果，经脱敏标注后进入 eval dataset；新版本通过 regression 和安全检查后再 canary 发布。

这个设计的关键不是多调用几次 Claude，而是每一阶段都有明确 contract、owner、failure handling 和闭环。

## 12. Trade-offs

- **更多 validation gates**：提高安全和可诊断性，也增加 latency 与实现成本；应按风险分级。
- **持久化更多状态**：支持恢复和审计，但增加隐私、保留和一致性负担。
- **同步体验**：实现简单、反馈即时，但不适合长任务和人工审批。
- **异步执行**：可靠且可恢复，但需要状态机、通知和幂等设计。
- **自动收集反馈**：数据量大，但信号噪声和偏差高；专家反馈质量高但昂贵。
- **集中 gateway**：便于治理、观测和切换，但可能成为 bottleneck 和 blast radius，需要高可用设计。

## 13. 考试判断清单

看到端到端架构题，依次检查：

1. 输入是否有 identity、schema、data classification 和 trust boundary？
2. source of truth、conversation/task/business state 各由谁维护？
3. context 是否经过 ACL，并且只包含任务所需信息？
4. orchestrator 是否处理 timeout、stop reason、retry、fallback 和 budget？
5. tool/action 是否在执行时重新授权并具备 idempotency？
6. 输出是否经过 syntactic、semantic、business/policy validation？
7. partial failure、人工接管和补偿是否明确？
8. telemetry 是否能关联 prompt/config、模型调用与业务 outcome？
9. 反馈是否先进入评估与审批，再发布到生产？
10. 是否能 rollback，且没有把模型当作数据库或授权主体？

## 14. 资料范围与时效说明

核查日期：**2026-09-19**。

- 最高优先级考试范围：[Claude Certified Architect – Professional Exam Guide v1.0](../../../Exam_Guide.md)。本课对应 Domain 1 的第二个 objective。
- Anthropic Engineering：[Building Effective AI Agents](https://www.anthropic.com/engineering/building-effective-agents)。用于核查 composable design、programmatic checks、environment feedback、停止条件、human oversight 和迭代改进原则。该文章已提示部分工具生态内容可能变化，本课只采用稳定的架构原则。
- Claude Platform Docs：[Using the Messages API](https://platform.claude.com/docs/en/build-with-claude/working-with-messages)。用于核查 Messages API 的 request/response 与 stateless conversation responsibility。
- Claude Platform Docs：[Stop reasons and fallback](https://platform.claude.com/docs/en/build-with-claude/handling-stop-reasons)。用于核查应用必须根据完成原因决定交付、继续、工具执行或 fallback。
- Claude Platform Docs：[Claude API errors](https://platform.claude.com/docs/en/api/errors)。用于核查错误分类、bounded retry、request ID 和长请求的架构考虑。
- Claude Platform Docs：[Define success criteria and build evaluations](https://platform.claude.com/docs/en/test-and-evaluate/develop-tests)。用于核查 multidimensional criteria、真实任务分布和 edge-case evaluation。

### Guide 要求与当前实现的区分

Guide 要求的是 `input → processing → output → feedback loops` 的端到端 architecture reasoning，并未指定必须采用某项当前 API 功能、某个模型或某个云组件。本文提到的当前 API 字段仅用于说明为什么 application layer 必须管理状态和完成分支，不将它们扩大为固定考试范围。

**Needs verification：** 实施前应重新核查最新 Messages/tool response schema、stop reason 集合、SDK retry 行为、模型/API 限制、价格、数据处理条款和平台可用性；练习题答案不依赖这些易变细节。
