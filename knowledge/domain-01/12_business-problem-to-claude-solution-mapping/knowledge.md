# 业务问题 → Claude 解决方案的映射框架

> **学习序号：12**  
> **Exam Guide 对应范围：Domain 1 — Solution Design & Architecture（17%）**  
> **具体目标：Translate business problems into Claude-based AI solutions**

## 1. 这道考点究竟在考什么

架构师的起点不是“哪里可以放一个 Claude”，而是把一个业务问题转写成可验证的系统任务，并判断 Claude 是否应该参与、参与哪一段、被允许做什么，以及什么结果才算成功。

本知识点聚焦于**从业务问题到 solution envelope（方案边界）**的映射。它不深入比较 workflow、agentic system、augmented LLM，也不展开端到端组件图；这些分别属于后续知识点 #14 和 #13。

考试中更可能出现的形式是：给出业务目标、数据、风险与 SLA，要求选择最合适的方案或指出设计缺口。最佳答案通常不会是“能力最强、自动化最多”，而是**满足目标且复杂度、权限和风险都受约束的最小方案**。

## 2. 为什么不能直接从技术能力出发

“Claude 能总结、调用工具、写代码”只是 capability；它没有回答：

- 谁遇到了什么问题，当前 baseline 是什么？
- 哪一步需要处理模糊的自然语言，哪一步必须严格确定？
- 错误会导致轻微返工，还是资金、权限、合规后果？
- Claude 的输出是建议、草稿、结构化数据，还是会触发真实动作？
- 怎样证明方案比现状更好？

从技术能力出发常导致三类误配：

1. **把确定性规则交给概率模型**：例如让模型计算税费或决定访问权限。
2. **把辅助任务升级为高权限自治**：例如只需起草客服回复，却开放退款和账户修改权限。
3. **只优化模型指标，不优化业务结果**：摘要“看起来很好”，但没有缩短处理时间或减少返工。

Anthropic 的架构建议是从尽可能简单的方案开始，只有在任务确实需要时才增加复杂度；agentic system 往往以更高 latency 和 cost 换取任务表现或灵活性。

## 3. 一张可复用的 Problem-to-Solution Mapping Canvas

在画架构图之前，先完成以下映射。每一栏都必须有业务或工程证据，而不是产品口号。

| 维度 | 必答问题 | 对方案的影响 |
|---|---|---|
| 业务结果 | 要改善什么 KPI？当前 baseline 是什么？ | 判断是否值得做，并定义验收门槛 |
| 用户与流程 | 谁发起、谁消费、谁承担错误后果？ | 决定交互、审批和 ownership |
| 任务单位 | 单次输入是什么，期望输出/动作是什么？ | 把模糊目标变成可评估任务 |
| 数据与真相源 | 数据来自哪里、是否最新、是否有 ACL、哪套系统是 source of truth？ | 决定是否需要 retrieval/tool，以及授权边界 |
| 不确定性 | 输入是否非结构化、意图是否开放、规则是否稳定？ | 判断 Claude 与确定性代码的分工 |
| 输出契约 | 自由文本、分类、JSON、建议还是副作用动作？ | 决定 schema、验证器和执行接口 |
| 错误成本 | 错误是否可逆？影响金额、健康、权限或合规吗？ | 决定 autonomy、guardrail、HITL 和 fallback |
| 非功能约束 | volume、latency、availability、cost budget、数据驻留要求是什么？ | 约束模型调用和系统形态 |
| 成功标准 | 如何离线评估、试点验收和线上监测？ | 防止凭 demo 主观判断 |
| 运营闭环 | 谁处理异常、修正数据、更新政策并批准扩大权限？ | 确保方案可持续运行 |

### 3.1 先定义 task，而不是先定义 model

“改善客服体验”不可直接设计。可以拆成可评估的任务：

- 判断工单意图与优先级；
- 从获授权的知识库检索相关政策；
- 生成带出处的回复草稿；
- 识别信息不足并请求补充；
- 在资格服务确认后，提交待审批的退款请求。

这一步只是问题映射，不等于已经选择 workflow 或 agent。架构模式要等任务边界、风险和成功标准明确后再选。

## 4. 判断 Claude 是否适合参与

### 4.1 通常适合 Claude 的任务特征

- 输入或输出以自然语言、文档、图像或代码等非结构化信息为主；
- 需要语义理解、归纳、分类、抽取、改写、解释或生成；
- 合法答案可能不止一个，但可以用 rubric、参考答案或人工判断评价；
- 任务需要结合上下文做柔性判断，并允许明确的 abstain/fallback；
- 可通过 retrieval 或 tools 提供受控的外部知识与动作能力。

### 4.2 不应由 Claude 单独负责的任务

- 精确计算、余额更新、权限判定、库存扣减等确定性事务；
- 只需简单查表、正则或固定业务规则即可可靠完成的步骤；
- 任何错误都不可接受且没有验证、审批或补偿机制的高影响动作；
- 极低延迟路径，而模型调用无法满足既定 SLA；
- 没有合法数据来源、无法获得代表性评估集，或没有业务 owner 的场景。

这里的结论通常不是“完全不用 Claude”，而是把职责拆开：Claude 负责理解和建议，确定性服务负责规则、授权、计算和 transaction commit。

## 5. 从问题形态映射到 solution envelope

| 业务问题形态 | Claude 的合理角色 | 必要的非模型控制 | 不宜直接升级为 |
|---|---|---|---|
| 大量文本需分类/抽取 | 输出受 schema 约束的候选结果 | schema validation、业务规则校验、低置信度队列 | 任意工具调用 agent |
| 员工需要基于内部资料答疑 | 基于获授权内容生成回答或摘要 | retrieval ACL、出处、无证据时拒答 | 把模型记忆当 source of truth |
| 专家写作耗时 | 生成草稿、比较方案、解释材料 | 人工 review、版本记录、敏感信息控制 | 未审阅直接对外发布 |
| 开放式客服对话 | 理解意图、检索信息、提出下一步 | 身份校验、政策服务、审批、审计 | 默认开放退款/账户修改权限 |
| 多步且路径可预定义 | 作为某些步骤的语义处理器 | 编排、检查点、重试、幂等 | 因“多步”就自动选 autonomous agent |
| 路径无法预先列举且需要探索 | 作为受边界约束的决策组件 | 工具白名单、预算、停止条件、人工升级 | 无边界长期自主运行 |

注意：上表只确定“Claude 在系统中的角色和边界”。具体选择单次调用、workflow 还是 agent，要在后续架构模式题中进一步判断。

## 6. Autonomy 必须随风险而变化

可把方案按影响力分为五级：

1. **Assist**：提供摘要、解释或草稿，用户决定是否采用。
2. **Recommend**：提出结构化建议，同时显示证据和不确定性。
3. **Prepare action**：生成待执行命令/工单，但必须审批。
4. **Bounded action**：只在明确政策、金额、对象和频率限制内自动执行。
5. **High-impact autonomy**：跨系统进行高影响或难以逆转的决策与动作。

随着错误后果、不可逆性和攻击面增加，应提高验证、审批、审计、权限隔离和 fallback 强度。Claude 输出“退款 100 欧元”不等于它拥有退款授权；**capability、recommendation 与 authority 是三件不同的事**。

## 7. 成功标准：从“效果好”到可验收

Anthropic 官方评估文档强调 success criteria 应 specific、measurable、achievable、relevant，并通常需要多维度评价。架构师应把业务 KPI 与 AI/system metrics 连接起来。

示例：不要写“客服回答更好”，而应定义：

- **业务结果**：平均处理时间相对 baseline 降低，且升级率不恶化；
- **任务质量**：意图分类、事实一致性、政策引用、完整性达到既定门槛；
- **风险**：越权动作数为零；高风险场景全部进入审批；
- **运营**：p95 latency、单位完成任务成本、fallback rate 在预算内；
- **体验**：用户/坐席满意度与修改率达到目标；
- **边缘情况**：缺失数据、矛盾政策、提示注入、超长输入有明确处理结果。

指标门槛必须由真实 baseline、风险容忍度和试验确定，不能照抄某篇文档的数字。测试集应覆盖实际任务分布和 edge cases；上线前离线评估，试点阶段再观察真实业务指标。

## 8. 示例：退款客服场景的完整映射

### 8.1 原始需求

“用 Claude 自动处理退款，降低客服成本。”

这句话缺少数据、权限、错误后果与验收标准，不能直接导出架构。

### 8.2 业务映射

- **目标**：缩短退款相关工单处理时间，同时不增加错误退款和投诉；
- **输入**：用户消息、订单号、身份验证状态；
- **真相源**：订单服务、当前退款政策，而非 Claude 的参数记忆；
- **模糊部分**：理解用户意图、从对话提取原因、解释政策；
- **确定部分**：身份校验、资格、金额上限、支付状态、实际退款 transaction；
- **风险**：资金损失、越权访问、错误承诺、prompt injection；
- **输出契约**：意图、订单引用、政策证据、建议动作、缺失字段；
- **初始 autonomy**：Claude 起草回复和退款建议；规则服务判断资格；坐席审批；
- **验收**：质量、越权率、处理时间、单位成本、人工修改率均达到门槛。

合理的第一版不是“给 agent 所有客服工具”，而是 read-only retrieval + 结构化建议 + 确定性 eligibility service + human approval。只有当数据证明低风险类别稳定可靠时，才考虑对小额、可逆、可审计动作做 bounded automation。

## 9. Trade-off 与典型 failure modes

### 9.1 主要 trade-off

- **自动化率 vs 风险**：更多自动动作可降低人工成本，但扩大错误和攻击后果。
- **灵活性 vs 可预测性**：模型能处理长尾表达，但输出和路径更难穷举。
- **质量 vs latency/cost**：更多上下文、检查或调用可能提高质量，也增加时延和成本。
- **通用性 vs 可评估性**：任务越开放，覆盖性的 gold set 和确定性验收越困难。
- **个性化 vs 隐私**：更多用户上下文可能改善结果，也扩大数据最小化和访问控制压力。

### 9.2 常见 failure modes

1. **Solution-first**：先买平台、选模型，再寻找问题。
2. **Scope collapse**：把“辅助坐席”偷换成“自动处理所有工单”。
3. **Source-of-truth confusion**：让 Claude 生成政策或账户事实，而不是查询权威系统。
4. **Authority leakage**：工具可执行范围大于业务需求或用户权限。
5. **Happy-path eval**：只用干净样例，没有测试缺失、冲突、恶意和长尾输入。
6. **Proxy metric trap**：只看回答得分，不看处理时长、人工修改、事故和单位成本。
7. **No fallback owner**：模型拒答或工具失败后无人接管。
8. **Premature complexity**：简单 retrieval + 单次调用足够，却直接建设多 Agent 系统。

## 10. 易混淆概念

| 概念 A | 概念 B | 区别 |
|---|---|---|
| Business outcome | Model metric | 前者衡量业务是否改善；后者只衡量系统某个能力 |
| Claude capability | Execution authority | 能提出或生成动作，不代表有权执行动作 |
| Context | Source of truth | Context 是提供给模型的信息；真相源仍由权威系统定义 |
| Demo | Production solution | Demo 证明可能性；生产方案还需权限、SLA、评估、运营和治理 |
| Automation | Agent | 自动化可由确定性代码或固定流程完成，不必由 agent 自主决定路径 |
| Confidence | Correctness | 模型自报信心不能替代事实核验、规则校验或评估数据 |

## 11. 考试中的决策顺序

面对 scenario-based question，可按下面顺序排除干扰项：

1. 明确业务目标、用户和 baseline；
2. 划分模糊语义任务与确定性任务；
3. 找到 source of truth 和数据访问边界；
4. 根据错误后果确定 autonomy 与 HITL；
5. 选择满足目标的最小 solution envelope；
6. 检查质量、安全、latency、cost 与业务 KPI 是否可测；
7. 确认 fallback、审计和运营 owner。

若选项只是“换更强模型”“增加更多工具”“完全自动化”，却没有修复题干中的数据、权限、验证或目标问题，通常不是最佳答案。

## 12. 资料范围与时效说明

核查日期：**2026-09-18**。

- 最高优先级考试范围：[Claude Certified Architect – Professional Exam Guide v1.0](../../../Exam_Guide.md)。本课对应 Domain 1 的首个 objective。
- Anthropic Engineering：[Building Effective AI Agents](https://www.anthropic.com/engineering/building-effective-agents)。用于核查“从最简单可行方案开始”、复杂度的 cost/latency 权衡、workflow/agent 边界以及清晰 success criteria 与 human oversight 的重要性。
- Claude Platform Docs：[Define success criteria and build evaluations](https://platform.claude.com/docs/en/test-and-evaluate/develop-tests)。用于核查 success criteria、multidimensional evaluation、真实任务分布与 edge cases。

### Guide 要求与当前产品实现的区分

Exam Guide 要求掌握的是把业务问题映射为 Claude-based solution 的**架构判断能力**，并未要求背诵某个当前模型、价格、context window、SDK 或 API 字段。本课没有使用这些易变信息来确定结论或题目答案。

**Needs verification：** 实施具体方案时，仍需重新核查所选模型/API 的可用能力、限制、区域支持、数据处理条款、价格和 latency；这些不是本课的固定考试结论。
