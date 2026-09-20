# 架构模式选型：Workflow vs Agent vs Augmented LLM

> **学习序号：14**  
> **Exam Guide 对应范围：Domain 1 — Solution Design & Architecture（17%）**  
> **具体目标：Select appropriate architectural patterns (workflow, agentic, augmented LLM)**

## 1. 最重要的结论

选型的核心不是“有没有调用 Claude”或“有没有 tools”，而是：

> **任务下一步由谁决定？**

- **Augmented LLM**：一次或少量模型调用获得 retrieval、tools、memory 等能力；应用仍拥有清晰的请求边界。
- **Workflow**：应用代码预先定义主要控制路径，Claude 在指定步骤内完成语义任务。
- **Agent**：Claude 根据目标、当前状态与 environment feedback 动态决定下一步和工具使用，直到完成、失败或触发停止条件。

考试通常奖励满足需求的**最小充分复杂度**。能够用一次 augmented LLM 调用解决，不要为了“先进”升级成 workflow；固定路径可以解决，不要无理由升级为 autonomous agent。

## 2. 术语校准：三者不是完全平行的盒子

Anthropic 的 `Building effective agents` 给出了两层分类：

1. **Augmented LLM 是基础 building block**：LLM 加上 retrieval、tools、memory 等 augmentation。
2. **Workflow 与 Agent 是控制方式**：workflow 通过预定义代码路径编排 LLM/tools；agent 让 LLM 动态控制过程和工具使用。

因此 augmented LLM 可以被放进 workflow，也可以成为 agent loop 的“brain”。三者更准确的关系是：

```text
                 Augmented LLM
         (model + retrieval/tools/memory)
                    /       \
                   /         \
      code controls path   model controls path
             │                    │
         Workflow               Agent
```

Exam Guide 把 `workflow, agentic, augmented LLM` 并排列作选型目标。答题时应理解它们的实际组合关系，同时把题目中的 “agentic” 解释为**模型主导动态控制的 agent pattern**，除非题干另有定义。

## 3. Augmented LLM

### 3.1 是什么

在基础模型调用旁增加一种或多种能力：

- **Retrieval**：提供企业文档、政策或搜索结果；
- **Tools**：读取系统状态、进行计算或执行受控动作；
- **Memory/state**：提供任务所需的持久信息；
- **Examples/instructions**：提供任务契约与行为约束。

典型形态是应用准备 context，调用 Claude，然后验证输出；即使 Claude在单次回合中请求一个工具，也不自动意味着系统已成为 agent。

### 3.2 适用场景

- 任务边界清楚，单次或少量调用可完成；
- 主要难点是理解/生成，而不是动态规划多步路径；
- retrieval 或一个受限工具即可补足事实；
- latency、cost 和可预测性优先；
- 人类或下游程序接管后续步骤。

例子：基于 ACL-filtered policy 文档回答问题；从发票抽取 schema；根据工单内容生成回复草稿。

### 3.3 不适用信号

- 任务明确包含多个有依赖的阶段和中间检查；
- 单次调用无法可靠处理长链条，失败难以定位；
- 下一步取决于中间结果，需要动态探索；
- 任务需要长时运行、暂停、恢复和多次工具交互。

### 3.4 Trade-off

优点是最简单、时延和成本较低、易评估；局限是处理复杂多步任务时容易把过多职责塞进一次 prompt，导致 instruction conflict、长上下文和不可诊断失败。

## 4. Workflow

### 4.1 是什么

Workflow 由 application/orchestrator 控制主流程：步骤、分支、重试、审批和停止条件主要写在代码或配置中。Claude 负责某些节点的分类、抽取、生成或评价。

```text
Input → classify → [route A | route B] → validate → approve → output
          Claude        code             code/HITL
```

即使某一步由 Claude 选择类别，**允许的类别和后续路径仍由系统预先定义**，因此仍是 workflow。

### 4.2 适用场景

- 业务路径能够枚举或稳定分解；
- 合规、审计或 SLA 要求可预测执行；
- 每一步有明确 input/output contract；
- 高影响动作需要确定性 gate 或 human approval；
- 希望独立评估、重试和替换每个步骤。

例子：客服工单分类 → 按类别检索 → 生成草稿 → policy validation → 人工审批；合同 intake → 字段抽取 → 规则校验 → 异常队列。

### 4.3 常见 workflow 形态

Anthropic 官方文章列出的 patterns 可用于识别题干：

| Pattern | 控制结构 | 合适条件 | 主要代价 |
|---|---|---|---|
| Prompt chaining | 固定顺序分步 | 能清晰拆成连续子任务 | 多次调用增加 latency |
| Routing | 先分类再走预定义分支 | 输入类别稳定且可准确判断 | 错路由影响后续结果 |
| Parallelization | 独立任务并行或多路评审 | 子任务独立、需要速度/多视角 | 聚合、成本和一致性 |
| Orchestrator–workers | 模型动态提出子任务，框架负责委派/汇总 | 子任务数量/内容难预知 | 更难评估与控制 |
| Evaluator–optimizer | 生成与评价按固定循环迭代 | 有清晰 rubric，反馈能改善结果 | 循环成本与停止条件 |

`orchestrator–workers` 有动态成分，但整体仍可作为规定了“分解 → worker → 汇总”骨架的 workflow。架构并非只有二元开关，而是一条 control-flexibility spectrum；判断时看**谁控制主要路径和允许动作**。

### 4.4 不适用信号

- 任务路径无法事先枚举，强行建分支会爆炸；
- 每个请求所需子任务差异极大；
- 环境反馈会不断改变计划；
- 维护大量 edge-case branches 的成本高于受控 agent loop。

### 4.5 Trade-off

Workflow 提供较强 predictability、auditability 和 step-level observability，也更容易插入确定性检查；代价是开发维护编排逻辑，面对开放长尾任务时可能僵化。

## 5. Agent（Agentic / Autonomous Pattern）

### 5.1 是什么

Agent 接收目标后，根据当前上下文与环境反馈反复进行：

```text
observe → decide/plan → act with tool → observe result → revise/stop
```

Claude 不只是某个固定步骤的处理器，而是动态决定需要哪些步骤、顺序和工具。Agent 仍需要 harness/orchestrator 来管理 session、tools、预算、安全和停止条件；“自主”不等于没有控制面。

### 5.2 适用场景

- 任务开放，所需步骤或数量无法可靠预先列举；
- 需要在环境中探索、执行、验证并根据结果修正；
- 任务成功可以被环境反馈验证，例如 tests、compiler、查询结果；
- 允许更高 latency/cost，并有 sandbox、权限和人工 checkpoint；
- 业务价值足以抵偿 agent harness 的运营复杂度。

例子：跨多个未知文件定位 bug、修改代码、运行测试再修复；研究任务中动态决定搜索路径、补充证据并综合结果。

### 5.3 不适用信号

- 路径已知且稳定，用 workflow 更直接；
- 任务是一次分类、摘要或抽取；
- 错误不可逆且无法在执行前验证/审批；
- 缺少 objective success criteria 或可靠 environment feedback；
- 极低 latency、严格 cost ceiling 或强确定性要求；
- 工具权限不能安全收窄或没有 sandbox。

### 5.4 Agent harness 的责任

生产 agent 至少需要：

- durable session/task state；
- tool allowlist、credential isolation 与执行时授权；
- iteration、time、token/cost budget；
- checkpoint、steering、interrupt 与 human escalation；
- tool result/error 的清晰反馈；
- sandbox 和副作用隔离；
- tracing、audit、evaluation 与 rollback/kill switch。

Anthropic 2026 年的 Managed Agents 架构把 agent 抽象为 session、harness、sandbox/tool environment 等可分离组件，说明 agent 不是“模型加一个 while loop”这么简单。该产品的当前具体状态不是 Guide 的固定必背要求，但其架构原则可用于理解 production agent。

### 5.5 Trade-off

Agent 提供最大 flexibility，适合未知路径与长尾任务；代价是更多模型/工具调用、更高 latency/cost、更大攻击面、compounding errors，以及更困难的测试和可复现性。

## 6. 一张选型矩阵

| 判断维度 | Augmented LLM | Workflow | Agent |
|---|---|---|---|
| 路径 | 一步或短边界 | 主要路径预定义 | 路径动态发现 |
| 控制者 | 应用发起/收口 | 代码/编排器 | 模型在 harness 边界内 |
| 工具 | 无或少量受限工具 | 每步允许工具明确 | 模型动态选择允许工具 |
| 状态 | 请求/短会话 | 明确 step state | 长时 session + environment state |
| 可预测性 | 高 | 高至中 | 中至低 |
| 适应长尾 | 低至中 | 中 | 高 |
| latency/cost | 通常最低 | 随步骤增加 | 通常最高且方差大 |
| 评估 | 单任务输出 | step + end-to-end | trajectory + outcome + safety |
| 典型失败 | 一次 prompt 过载 | 路由/步骤传播错误 | 循环、越权、复合错误 |

表格表达的是相对趋势，不是固定数值。一个设计良好的 workflow 也可能比一个低预算 agent 更昂贵；必须用实际 workload 测量。

## 7. 选型决策算法

### Step 1：先尝试最小方案

问：“一次 Claude 调用，加必要 retrieval/tool 和 output validation，能否在质量、latency、cost、安全目标内完成？”

- 能：选 augmented LLM。
- 不能：继续判断失败原因，而不是直接选 agent。

### Step 2：任务能否稳定分解

- 步骤/分支可定义，中间结果可校验：选 workflow。
- 路径不能预知、依赖环境反馈探索：agent 才成为候选。

### Step 3：Agent 是否具备安全前提

检查：

- 有无明确目标和 stopping condition？
- 环境能否给出可验证 feedback？
- tools 能否最小权限和 sandbox？
- 可否限制 time/iteration/cost？
- 高影响动作能否审批或回滚？
- 是否有 trajectory evaluation 和人工接管？

任一关键前提缺失，都应降低 autonomy，改为 workflow 或 human-led augmented LLM。

### Step 4：用评估证明复杂度有价值

比较 baseline 与候选方案的 task success、质量、安全、latency、cost、人工工作量和业务 outcome。只有复杂方案带来可测量的净收益才升级。

## 8. Hybrid 是常态，但控制边界必须清晰

真实系统常组合模式：

- Workflow 先验证身份与分类，只有开放式疑难任务进入 agent；
- Agent 提出计划，但资金动作经过 deterministic workflow + human approval；
- Workflow 的每个节点都可能使用不同 augmented LLM；
- Agent 将结构化结果交回 workflow，后者负责发布、审计和通知。

Hybrid 不是“什么都用一点”，而是根据风险划分 control boundary：让模型在需要 flexibility 的局部拥有决策权，让代码在授权、事务和 policy gate 处保持确定性。

## 9. 容易混淆的概念

| 误解 | 正确认识 |
|---|---|
| 使用 tools 就是 Agent | 固定代码可调用工具；单次 tool request 也可能只是 augmented LLM |
| 多次 Claude 调用就是 Agent | 固定 prompt chain 是 workflow，调用次数不决定 agency |
| RAG 就是 Agent | RAG 通常只是 augmentation；是否 agent 取决于控制路径 |
| Agent 没有 orchestrator | Agent 需要 harness 管理工具、状态、预算和安全 |
| Workflow 完全没有模型决策 | 模型可分类、评价或动态分解，但系统仍可规定主骨架和边界 |
| Human-in-the-loop 就不是 Agent | Agent 可以在 checkpoint 请求人工判断；HITL 不改变主要控制权定义 |
| Agent 一定比 Workflow 准确 | 取决于任务、工具和评估；长轨迹还可能累积错误 |
| Agentic workflow 是精确术语 | 行业用法混乱；答题先看题目怎样定义控制权 |

## 10. 典型 failure modes

### Augmented LLM

- 把复杂多步任务塞进单次 prompt；
- retrieval/context 不相关或未授权；
- schema-valid 但事实错误；
- 以 model memory 代替 source of truth。

### Workflow

- 分类错误导致 wrong-route；
- 中间错误被后续步骤放大；
- 固定分支无法覆盖长尾；
- retry 非幂等步骤导致重复副作用；
- 流程增长成难以维护的 branch maze。

### Agent

- 未设置 iteration/cost/deadline，形成 runaway loop；
- 工具过多或权限过宽；
- tool result 含恶意指令，导致 indirect prompt injection；
- 错误计划经多步累积；
- 缺乏客观反馈，agent 不知道何时完成；
- 长时状态不可恢复、不可审计或无法重放。

## 11. 场景推演

### 场景 A：内部政策问答

问题：员工提出问题，答案必须基于其有权查看的最新政策。

选择：**Augmented LLM**。应用按 ACL 检索文档，Claude 生成带来源回答，输出层验证证据。路径简单，无需自主规划。

### 场景 B：标准化供应商准入

问题：提取表单 → 检查必填项 → 风险分类 → 不同分支调查 → 合规审批。

选择：**Workflow**。步骤和审批可枚举，确定性控制与审计比开放探索更重要。

### 场景 C：大型代码库疑难缺陷

问题：事先不知道涉及哪些模块，需搜索、修改、编译、测试并根据失败迭代。

选择：**受约束 Agent**。路径不可预知且有 tests/compiler 提供 environment feedback；仍需 sandbox、工具范围、预算和 human review。

### 场景 D：退款客服

选择：**Hybrid**。普通问答使用 augmented LLM；已知退款路径使用 workflow；开放式调查可在 read-only sandbox 中使用 agent，但真实退款始终通过确定性资格检查和审批。

## 12. Architecture reasoning 模板

考试或设计评审时可按以下结构回答：

1. **任务结构**：路径是否可预定义，输入是否开放？
2. **控制权**：代码还是模型决定下一步？
3. **验证性**：中间与最终结果有无 objective feedback？
4. **风险**：错误是否可逆，工具权限和副作用是什么？
5. **约束**：latency、cost、SLA、合规与维护能力如何？
6. **选择**：说明为何最低复杂度模式足够，或为何必须升级。
7. **边界**：状态、预算、审批、fallback、observability 和 evaluation。
8. **升级证据**：以 eval 数据证明复杂度带来净收益。

## 13. 资料范围与时效说明

核查日期：**2026-09-20**。

- 最高优先级考试范围：[Claude Certified Architect – Professional Exam Guide v1.0](../../../Exam_Guide.md)。本课对应 Domain 1 的第三个 objective。
- Anthropic Engineering：[Building Effective AI Agents](https://www.anthropic.com/engineering/building-effective-agents)。用于核查 workflow/agent 的控制权区别、augmented LLM building block、五类 workflow、agent loop 和“先简单后复杂”原则。文章已提示部分工具生态自 2024 年后发生变化，本课只把其稳定 taxonomy 与 design reasoning 用于考试范围。
- Anthropic Engineering：[Scaling Managed Agents: Decoupling the brain from the hands](https://www.anthropic.com/engineering/managed-agents)。用于核查现代 production agent 中 session、harness、sandbox/tools 的责任分离与恢复/安全边界。
- Claude Platform Docs：[Claude Managed Agents overview](https://platform.claude.com/docs/en/managed-agents/overview)。用于核查 managed harness、long-running session 与 asynchronous agent workload 的当前产品定位。

### Guide 要求与当前实现的区分

Guide 要求的是模式选型和 trade-off 判断，不要求记忆 Managed Agents 的 beta header、具体 tools、模型名称、价格或 feature availability。Managed Agents 只用于展示现代 agent harness 的实现边界，不改变 Guide 的考试 taxonomy。

**Needs verification：** 若实际采用 Managed Agents、Agent SDK 或自建 Messages API loop，必须重新核查当时的产品状态、API schema、平台差异、数据保留/合规资格、模型支持、限制和价格；这些易变信息没有用于决定练习题答案。
