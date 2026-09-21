# 多 Agent 系统设计与编排策略

> **学习序号：15**
>
> **Exam Guide 对应范围：Domain 1 — Solution Design & Architecture（17%）**
>
> **具体目标：Design multi-agent systems and orchestration strategies**

## 1. 最重要的结论

多 Agent 不是“多调用几次模型”，而是让多个具有独立角色、上下文或工具边界的 Agent 共同完成一个目标。它真正解决的是：**单个 Agent 的上下文、专业能力或并行吞吐不足以经济地覆盖任务**。

考试中的稳健判断顺序是：

1. 单个 augmented LLM 是否足够？
2. 固定 workflow 是否足够？
3. 单 Agent 动态循环是否足够？
4. 只有在任务能安全地分工、并行或专业化时，才采用多 Agent。

多 Agent 的收益来自合理分工，而不是 Agent 数量。若子任务彼此强依赖、共享状态频繁变化，额外 Agent 反而会增加协调成本、延迟、冲突和失败面。

## 2. 范围与相邻知识点

本课关注 **system topology、角色边界、任务委派、调度、结果汇聚、状态与失败治理**。

- Domain 3 的“Agent-to-Agent 通信模式”回答消息如何传输、关联和恢复；本课回答系统由谁协调、如何分工以及如何形成最终结果。
- 下一课“复杂问题的拆解技术”会系统讲如何拆问题；本课只讲拆分结果如何成为可执行的 delegation contract。
- 并行调用多个无状态模型不一定构成多 Agent；关键是各执行者是否拥有独立任务、上下文/状态和一定程度的行动循环。

## 3. 什么时候值得使用多 Agent

### 3.1 强信号

- **可独立并行的广度型任务**：例如分别研究多个市场、审查互不依赖的模块或检索不同来源。
- **专业化**：不同角色需要不同 system prompt、tools、data access 或评价标准。
- **上下文隔离**：每个工作流只需局部信息，分开 context 能减少无关信息干扰。
- **动态工作量**：协调者只有看到中间结果后，才能决定还需调用哪类专家。
- **单个上下文不足**：复杂研究需要多条探索路径，最终只汇总证据和结论。

Anthropic 的 multi-agent research 实践采用 lead agent 规划并并行委派 subagents，再由 lead 综合结果。官方同时指出，这种模式更适合 breadth-first、可独立探索的任务；对必须共享大量上下文或子任务之间依赖很强的任务并不理想。

### 3.2 反信号

- 流程短、固定且能够稳定编码为 workflow；
- 每一步严格依赖上一步的完整输出，几乎不能并行；
- 多个角色必须持续修改同一份共享状态；
- latency/cost 预算紧，单 Agent 已满足质量目标；
- 任务包含不可逆高风险动作，却没有确定性授权和人工门控；
- 没有证据表明多 Agent 改善业务 outcome，只是为了采用新架构。

## 4. 常见拓扑与选型

### 4.1 Orchestrator–workers / Supervisor–specialists

```text
                         ┌─ Researcher A ─┐
Input → Orchestrator ────┼─ Researcher B ─┼→ validate → synthesize → Output
                         └─ Reviewer ─────┘
```

中央 orchestrator 负责理解目标、分派任务、控制预算、收集结果和最终综合；workers 在隔离上下文中完成有限职责。这是最容易治理的通用起点。

适用于任务数量或路径无法预先完全确定，但各子任务能够清楚界定的场景。缺点是 orchestrator 可能成为瓶颈或单点故障，并且“差的委派”会被所有 worker 放大。

### 4.2 Router–specialists

Router 根据输入选择一个或少数专家，例如 legal、security、billing。它适合类别相对稳定且大多不需要跨专家综合的任务。与 orchestrator–workers 相比，router 通常只决定去哪里，orchestrator 还负责计划、协调和合并多份结果。

### 4.3 Hierarchical orchestration

顶层协调者把大型目标交给领域 lead，再由 lead 管理局部 workers。它可扩大组织规模并隔离上下文，但会增加层级延迟、信息压缩损失和责任追踪难度。只有一层已不足、边界又自然分区时才值得采用。

### 4.4 Peer-to-peer / decentralized

Agents 彼此协商、转交或共同维护计划，没有单一主管。它在开放协作研究中可能灵活，但冲突解决、终止、审计和全局预算最难控制。企业高风险工作通常需要外部 coordinator、shared ledger 或确定性 policy layer，不能只依赖 Agent 自治协商。

### 4.5 Evaluator / critic as a role

独立 reviewer 可按 rubric 检查 worker 结果并要求修订，减少 producer 的自我确认偏差。但 reviewer 也会出错；必须有清楚标准、有限轮数和升级路径。不能把“多 Agent 投票”当作事实正确性的天然保证。

## 5. Orchestrator 的职责

一个可靠 orchestrator 至少承担六类职责：

1. **Plan**：确定工作包、依赖关系、优先级和是否可并行。
2. **Delegate**：为 worker 提供明确契约，而非一句模糊的“去研究一下”。
3. **Schedule**：限制并发、预算和 deadline；支持 cancel、timeout 和 backpressure。
4. **Observe**：跟踪 task ID、agent role、状态、费用、工具调用与 artifacts。
5. **Validate**：检查 schema、证据、权限边界、完成条件和冲突。
6. **Synthesize**：去重、解决矛盾、标明不确定性，并形成面向最终目标的答案。

Orchestrator 不应只拼接 worker 文本。它需要把局部结果映射回全局 acceptance criteria。

## 6. Delegation contract：多 Agent 的关键接口

可执行的委派应包含：

| 字段 | 目的 |
|---|---|
| Objective | worker 要解决的具体问题 |
| Scope / boundaries | 包含什么、明确排除什么，避免重叠和越权 |
| Input snapshot | 使用哪个版本的数据及必要背景 |
| Allowed tools / authority | 可读写哪些资源，哪些动作必须审批 |
| Output schema | 结构、粒度、证据/引用和错误表达方式 |
| Budget / deadline | token、调用、时间或费用上限 |
| Completion criteria | 何时算完成、何时返回 partial / blocked |
| Provenance | 结论对应的来源、artifact 和处理版本 |

这类似后端系统中的 API contract。契约缺失会产生 scope overlap、重复搜索、格式不兼容和“worker 看似成功但无法综合”的问题。

## 7. 并行、同步与异步

### 7.1 哪些任务可以并行

只有不存在数据依赖，或能基于同一 immutable snapshot 执行的子任务，才适合 fan-out。总时延大致受 critical path 和最慢必要 worker 影响，而不是简单等于所有 worker 时间之和。

### 7.2 同步编排

协调者等待必要 workers 完成后继续。优点是控制流直观、状态较少；缺点是容易被 straggler 阻塞。适合短任务和必须一次返回完整结果的请求。

### 7.3 异步编排

任务进入 queue，workers 独立执行并写入 durable artifacts，协调者按事件汇聚。它适合长任务和弹性扩展，但必须显式设计：

- task correlation 与幂等 key；
- stale result / duplicate delivery 处理；
- cancellation 与 deadline 传播；
- partial completion 和重新分配；
- 状态版本、一致性及恢复策略。

“改成异步”不会自动提高可靠性，只是把等待问题变成分布式状态问题。

## 8. Context、state 与 artifact 设计

### 8.1 Context isolation

每个 Agent 只接收完成职责所需的 context，能降低 distraction、能力膨胀和 prompt injection 的横向传播。但隔离也意味着 worker 不知道其他 worker 的发现，因此必须由协调层维护共享任务视图。

### 8.2 避免隐式共享可变内存

多个 Agent 同时覆盖同一 memory 容易产生 lost update 和不可重现结果。更安全的做法是：

- 使用 immutable input snapshot 或带 version 的 state；
- 让 workers 产生 append-only artifacts / proposed changes；
- 由单一 owner 或确定性事务层合并写入；
- 保存 task、source、agent/config version 和时间戳。

### 8.3 Artifact 优于层层转述

大型研究结果、patch、测试报告应保存在外部 artifact store，消息中传引用和摘要。若每层 Agent 都重新摘要上一层内容，会形成 “telephone game”，逐层丢失限定条件与证据。

## 9. 汇聚、冲突与事实判断

多个 Agent 给出相同答案不等于答案正确，它们可能受同一错误来源或 prompt 偏差影响。可靠综合应：

1. 保留每个结论的 provenance；
2. 按 source authority、freshness 和 task rubric 评分；
3. 对矛盾点做 targeted re-query 或交给独立 adjudicator；
4. 无法解决时明确报告 disagreement / uncertainty；
5. 不用简单 majority vote 替代事实验证。

对于结构化结果，先做 schema validation 和 deterministic checks，再让 Claude 执行语义综合。

## 10. Failure containment 与恢复

| Failure mode | 更稳健的处理 |
|---|---|
| Worker 超时/崩溃 | 保留已完成 artifacts；有界重试；必要时重新分配或返回 partial |
| 重复工作 | scope partition、coverage map、dedup key |
| 无限增殖 Agent | 最大深度/并发/总 worker 数和预算 |
| 无限研究或 reviewer 循环 | 明确 completion criteria、最大轮数和 stop condition |
| Worker 返回不可用格式 | schema validation，允许一次定向修复而非全局重跑 |
| Orchestrator 错误分解 | checkpoint、plan review、根据 early signal 重新规划 |
| 一个错误污染最终答案 | provenance、独立验证、置信度与 fail-safe 输出 |
| 状态变更部分成功 | idempotency、补偿动作、事务边界或 human escalation |

不要对所有失败盲目重试。若任务有外部副作用，必须区分“调用未完成”和“已执行但响应丢失”，否则重试可能重复扣款、发信或修改数据。

## 11. 安全与权限

- 每个角色仅配置必要 tools、MCP servers 和 data scope；不要让所有 workers 继承 coordinator 的全部能力。
- 凭证由受控执行层注入，不通过 Agent 消息传递。
- authorization 在资源/工具执行层强制实施，不能只依赖 prompt。
- Worker 输出属于不可信输入；可能含 prompt injection、恶意 tool output 或错误指令，进入协调者 context 前应过滤和标记来源。
- 高影响写操作应集中到少数受控执行角色，并经过 deterministic policy 与 human approval。
- 审计链应能回答：谁委派了什么、基于哪个版本、调用了哪些工具、产生了什么 artifact、谁批准了副作用。

当前 Managed Agents 文档展示了每个 Agent 可拥有自己的 prompt、tools、MCP servers 和 isolated thread，也展示了 coordinator roster、thread events、预算与 permission routing。这是 **当前产品实现示例**，不是 Exam Guide 要求背诵的 API 字段或限制。

## 12. Evaluation：评估系统，而不是只评估最后一句话

多 Agent 路径具有非确定性，不能要求每次 trajectory 完全相同；应同时评估 outcome 和过程是否合理：

- **Outcome**：任务完成率、正确性、覆盖率、证据质量、安全性；
- **Decomposition/delegation**：是否漏项、重叠、边界清楚；
- **Worker utility**：哪些调用对最终结果有贡献，哪些是无效工作；
- **Synthesis**：是否正确去重、处理冲突并保留 provenance；
- **Efficiency**：端到端 latency、critical path、token/cost、并发峰值；
- **Reliability**：worker 失败、超时、重复消息时能否降级恢复；
- **Security**：是否发生越权、敏感数据扩散或未经批准的副作用。

比较单 Agent 与多 Agent 时，要在同一数据集和相同业务 acceptance criteria 下评估增量价值。Anthropic 工程文章中的质量或 token 数字来自特定系统和实验，不能泛化为考试公式或默认容量规划依据。

## 13. 场景化决策模板

遇到题目时按以下顺序回答：

1. **任务性质**：广度型还是强依赖链？
2. **并行边界**：哪些工作基于独立输入，可同时执行？
3. **角色边界**：不同角色是否确实需要不同 context、tools 或权限？
4. **协调模式**：router、orchestrator–workers、hierarchy 还是固定 workflow？
5. **契约**：objective、scope、output、evidence、budget、completion criteria 是否清楚？
6. **状态模型**：共享 snapshot、artifacts、version 和写入 owner 如何设计？
7. **失败策略**：timeout、partial result、retry、cancel、stop condition 如何处理？
8. **安全与评价**：谁能执行副作用，如何审计，如何证明多 Agent 值得？

## 14. 易错结论

- “Agent 越多质量越高”——错误；协调开销与错误传播会快速增加。
- “并行一定更快”——错误；依赖、straggler、汇聚和限流决定 critical path。
- “多数投票能消除 hallucination”——错误；相关错误不会因投票消失。
- “独立 context 等于完全隔离”——错误；共享文件、凭证或工具仍可能形成侧向影响。
- “让 coordinator 拥有所有工具最方便”——通常违背 least privilege，也扩大 prompt injection blast radius。
- “worker 成功即系统成功”——错误；最终还要验证完整性、冲突、provenance 和全局目标。

## 15. Guide 要求与产品实现的边界

Exam Guide 要求能够设计 multi-agent system 并选择 orchestration strategy，重点是架构判断与 trade-off。Managed Agents、Agent SDK 或自建 queue/workflow engine 只是不同实现方式。

**Needs verification（实施前复核）**：Managed Agents 当前仍标注 Beta；其 API schema、beta header、model compatibility、roster/thread 上限、层级支持、预算语义、权限事件、地区支持、价格及数据处理条款可能变化。这些具体值不用于本课练习题的答案判断。

## 16. 核查来源

核查日期：**2026-09-21**。

- [CCAR-P Exam Guide](../../../Exam_Guide.md)：考试范围、Domain 权重、目标和样题风格的最高优先级依据。
- [Anthropic Engineering — How we built our multi-agent research system](https://www.anthropic.com/engineering/multi-agent-research-system)：orchestrator-worker、并行研究、delegation、evaluation 与生产 failure modes。
- [Anthropic Engineering — Building effective agents](https://www.anthropic.com/engineering/building-effective-agents)：orchestrator-workers、routing、parallelization 及“从简单方案开始”的设计原则。
- [Claude Platform Docs — Multiagent orchestration](https://platform.claude.com/docs/en/managed-agents/multiagent-orchestration)：当前 Managed Agents 的 coordinator、isolated threads、agent-scoped tools 和权限实现示例。
