# 复杂问题的拆解技术

> **学习序号：16**
>
> **Exam Guide 对应范围：Domain 1 — Solution Design & Architecture（17%）**
>
> **具体目标：Apply decomposition techniques for complex problem solving**

## 1. 最重要的结论

复杂问题拆解不是把长 prompt 随意切成几段，而是把一个难以直接验证的目标转换为一组：

- 边界明确的 work units；
- 可表达的 dependency；
- 可验证的 intermediate artifacts；
- 可控制的风险与预算；
- 最终能够无损 recomposition 的结果。

好的拆解让每一步都更简单、可观察、可重试；坏的拆解只把不确定性从一个大任务搬到多个含糊的小任务中。

答题时先判断任务结构，再选择拆解方式：

1. 步骤固定且有依赖：**sequential decomposition / prompt chaining**；
2. 子任务独立：**parallel sectioning**；
3. 输入类别不同：**routing**；
4. 子任务无法预先确定：**dynamic decomposition / orchestrator-workers**；
5. 结果可按清晰 rubric 反复改进：**evaluator-optimizer loop**。

## 2. 范围与相邻知识点

本课回答“**把问题切成什么、按什么关系执行、如何证明每块完成**”。

- 第 14 课回答 workflow、agent 或 augmented LLM 的总体模式选型；
- 第 15 课回答拆出的工作如何在多 Agent 系统中委派与编排；
- 本课的核心在 task structure，即使最终只使用一个 Claude 调用或传统代码，也仍需要正确拆解；
- Domain 2 的 prompt engineering 会深入 zero-shot、few-shot 等提示技巧，本课不把“要求模型展示 chain-of-thought”当成架构拆解方法。

架构层面的 decomposition 应外化为 plan、task graph、contracts、artifacts 和 checks，不依赖读取模型的隐藏推理过程。

## 3. 从 outcome 反推，而不是从动作清单开始

### 3.1 先定义完成状态

在拆步骤前先回答：

- 最终 deliverable 是什么？
- 谁消费它，格式和时效要求是什么？
- 哪些事实必须有 authoritative source？
- 哪些 safety、security、privacy 和合规约束不可破坏？
- 哪些 acceptance criteria 能由代码、专家或 rubric 验证？

如果“完成”的定义含糊，拆出的每个 subtask 即使局部成功，整体也可能失败。Anthropic 的评估文档强调 success criteria 要 specific、measurable、achievable、relevant；这些标准同样应向下映射到关键 work units。

### 3.2 把约束当作一等输入

不要把权限、预算、deadline、数据新鲜度和人工审批留到最后补丁式处理。它们会改变 task boundary：例如“生成付款建议”和“执行付款”必须分成不同工作包，后者需要确定性授权与 human gate。

## 4. 建立 task graph，而不是只有 checklist

Checklist 只说明有哪些步骤；task graph 还说明它们的关系。

```text
                 ┌─ retrieve policy ─┐
Input → classify ┤                    ├→ reconcile → draft → validate → approve
                 └─ retrieve account ┘
```

常见 edge 类型：

- **Sequential dependency**：B 必须使用 A 的输出；
- **Parallel independence**：B、C 使用同一 snapshot 且互不修改共享状态；
- **Conditional branch**：分类结果决定进入哪条路径；
- **Iteration**：evaluation 不通过时回到 generation，但必须有上限；
- **Join**：多个结果满足约定后才能 synthesis；
- **Human checkpoint**：信息不足、高风险或不可逆动作前暂停。

关键不是最大化并行，而是找出真正的 critical path。把存在数据依赖的任务强行并行，会产生 stale assumptions 和返工。

## 5. 五种常用拆解维度

### 5.1 按阶段（stage / lifecycle）

把任务拆成理解、检索、分析、生成、验证、批准和交付等阶段。适合输入输出自然衔接的固定流程。

例：合同处理可拆为 `ingest → extract clauses → compare policy → identify risk → draft advice → legal review`。

### 5.2 按对象或分区（object / partition）

按文档、服务、客户、地区、文件或数据分片。只要分区边界独立，就可 parallel sectioning。

例：迁移 20 个相互独立的微服务，可按 service 拆；但 shared library 和 schema change 应单独作为 dependency 处理。

### 5.3 按关注点（concern / perspective）

让不同 work units 分别分析 correctness、security、performance、cost 或 compliance，最后按统一 rubric 合并。它能减少一次调用中多个目标争夺 attention。

但关注点可能交叉，例如 security fix 影响 latency，因此 synthesis 必须检查 cross-cutting trade-off，不能只拼接报告。

### 5.4 按输入类别（routing）

先分类，再进入 specialized path。适合类别稳定、可高置信识别且所需 tools/policies 不同的场景。

分类本身也要有：unknown/ambiguous 路径、置信阈值、误路由检测和升级机制。

### 5.5 按风险（risk-first）

优先隔离不可逆动作、敏感数据、外部副作用与高不确定步骤。先用 read-only analysis 形成 proposal，再由受控组件或人批准执行。

Risk-first decomposition 的价值不是提高文本质量，而是缩小 blast radius。

## 6. 固定拆解与动态拆解

### 6.1 Static decomposition

设计时就知道 subtasks、顺序和 contracts，由代码或 workflow engine 执行。

适用条件：

- 任务分布稳定；
- 路径可枚举；
- 合规和审计要求较高；
- 每一步可以独立测试。

Anthropic 将 prompt chaining 描述为把任务拆成固定序列，每个 LLM call 处理前一步输出，并可在中间加入 programmatic gates。它通过增加 latency，换取每一步更简单、更准确和更容易验证。

### 6.2 Dynamic decomposition

运行时由 Claude 根据具体输入、环境反馈和已发现信息生成或修订 plan。适合事先不知道要查哪些来源、修改哪些文件或需要多少步骤的任务。

动态不代表无限自由。系统仍需规定：

- 允许的目标范围和 tools；
- 最大 step/worker/token/time budget；
- task contract 与 artifact schema；
- plan review、checkpoint 和 stop conditions；
- 何时请求人类判断。

### 6.3 Hybrid decomposition

生产系统常用外层静态、内层动态：

```text
deterministic intake
    → dynamic investigate/plan
    → deterministic validation
    → human approval
    → controlled execution
```

它把不确定推理放在可控边界内，同时让高影响阶段保持可预测。

## 7. Granularity：切多大才合适

### 7.1 过粗（under-decomposition）

信号包括：

- 一个 work unit 同时承担检索、判断、写作和执行；
- 输入 context 太大且大量无关；
- 失败后只能全量重跑；
- 无法定位是事实、推理、格式还是工具调用失败；
- completion 只能写成“做好这个复杂任务”。

### 7.2 过细（over-decomposition）

信号包括：

- 每个步骤没有独立价值，只是机械转述；
- handoff 数量和 serialization 成本超过推理收益；
- 多次摘要造成信息损失；
- validation 与 orchestration 成本激增；
- subtask 需要反复读取几乎相同的完整 context。

### 7.3 Goldilocks test

一个 work unit 通常应满足：

1. 只有一个主要 objective；
2. 所需 context 可以清楚界定；
3. 输出是下游可消费的 artifact；
4. 有独立 completion test；
5. 失败可局部重试或安全降级；
6. 不需要与其他并行任务持续共享可变状态。

不能满足这些条件时，应重新划分或合并。

## 8. Subtask contract 与 artifact

每个 subtask 至少定义：

| Contract element | 说明 |
|---|---|
| Objective | 本步骤解决的单一问题 |
| Inputs | 数据、版本、来源和前置结果 |
| Constraints | 权限、预算、deadline、不可做事项 |
| Output artifact | JSON、decision record、patch、report、evidence set 等 |
| Invariants | 跨步骤必须保持的规则，如 ACL、schema、事实来源 |
| Acceptance test | code check、rubric、专家审核或环境验证 |
| Failure semantics | retryable、blocked、partial、unsafe 或需升级 |

尽量传递 structured artifact，而不是自然语言“我大概做了什么”的转述。Artifact 让数据 lineage、重试、回放和审计成为可能。

## 9. 中间检查与验证

拆解的主要收益之一是能在错误扩散前阻断它。

### 9.1 Gate 放在哪里

- 高 fan-out 前：先验证 plan，避免错误并行放大；
- 事实进入分析前：验证 source、freshness 和 access scope；
- 多结果 join 时：检查 completeness、schema 和冲突；
- 不可逆动作前：确定性 policy + human approval；
- 最终交付前：端到端 acceptance test。

### 9.2 Validator 要匹配 artifact

- 确定性规则能判断时优先 code-based check；
- 需要语义判断时使用明确 rubric 的 LLM evaluator；
- 高风险或模糊判断由 domain expert；
- 环境行为必须通过真实 tool/test 反馈验证，不能只问生成者“你完成了吗”。

Evaluator-optimizer 适用于有清楚评价标准、反馈确实能带来可测改进的任务。应设置最大轮数和最小 improvement threshold，避免无限自我修订。

## 10. 长任务：按可交付增量拆解

长时间任务容易出现“一次做太多”、跨 context 丢失进度和过早宣布完成。Anthropic 的 long-running agent 实验采用 feature list、progress file、git history、一次处理一个 feature、保持 clean state 和端到端测试来缓解这些问题。

可迁移为通用做法：

1. 把总体目标展开为可验证 backlog；
2. 标记 dependency、priority 和 status；
3. 每个 session/iteration 只完成一个 bounded increment；
4. 结束前保存 artifact、decision、remaining work 和验证证据；
5. 下一轮先恢复状态并运行 baseline check；
6. 只有所有必要 acceptance criteria 通过，才声明总体完成。

这不是要求考试记住某个文件名，而是理解 **persistent progress + incremental clean handoff** 的架构原则。

## 11. Context allocation 与拆解

分解后不应把全部原始 context 复制到每一步。应为每个 work unit 提供最小充分的 high-signal context，并保留按需取数能力。

- 全局：goal、invariants、shared definitions、acceptance criteria；
- 局部：该 subtask 的 inputs、tools、examples 和历史；
- 外部状态：大型 artifacts、原始 evidence、进度与版本；
- handoff：结论、依据、未决问题和 artifact reference。

Anthropic 的 context engineering 指南强调以尽量小的高信号 token 集合支持结果，并用 structured notes、compaction 或 subagents 维持长任务连贯性。拆解不能成为 context duplication 的借口。

## 12. Recomposition：设计拆解时就考虑如何合并

如果不知道结果如何合并，就还没有完成拆解设计。需要预先定义：

- 哪些结果是 union、排序、汇总、选择或事务合并；
- 冲突按 source authority、版本、rubric 还是人工裁决；
- 是否要求 coverage map，如何发现漏项；
- 局部优化是否破坏全局 invariant；
- final synthesizer 需要哪些原始证据，而不只是摘要。

例如把架构评审拆为 security、performance、cost 三份分析后，必须有 trade-off join：最低成本方案可能不满足 security 或 SLA，不能把三份“各自最优”直接相加。

## 13. 典型 failure modes

| Failure mode | 表现 | 修正 |
|---|---|---|
| Goal drift | 子任务完成但偏离业务 outcome | 将全局 acceptance criteria 和 invariants 下传 |
| Missing coverage | 有些范围没有 owner | coverage map 与 join-time completeness check |
| Overlap | 多步骤重复检索/分析 | 互斥 scope、dedup key、清楚边界 |
| Wrong dependency | 读取旧结果或强行并行 | 显式 DAG、versioned inputs |
| Lossy handoff | 只传摘要，丢失证据和限定条件 | structured artifact + provenance |
| Premature completion | 局部通过就宣布整体完成 | backlog status + end-to-end acceptance test |
| Validation gap | 生成者自称成功，无环境证据 | 独立 check/tool feedback/HITL |
| Infinite refinement | evaluator 不断要求改写 | 最大轮数、阈值和 stop condition |
| Atomicity mismatch | 子任务重试导致重复副作用 | proposal/execution 分离、idempotency |
| Context duplication | 每步复制完整历史，成本和干扰上升 | global/local context 分层与按需检索 |

## 14. 场景化决策模板

面对考试场景，可以按顺序问：

1. 最终 outcome、consumer 和 measurable acceptance criteria 是什么？
2. 有哪些必须保持的 constraints / invariants？
3. 能否按 stage、object、concern、category 或 risk 划分？
4. 子任务之间是 sequential、parallel、conditional 还是 iterative？
5. 拆解能静态定义，还是必须根据运行时发现动态调整？
6. 每个 subtask 的 input/output contract 和 failure semantics 是什么？
7. 哪些中间 artifact 必须验证，gate 放在哪里？
8. 最终如何检查 coverage、解决冲突并 recombine？
9. 该复杂度是否经 evaluation 证明优于更简单方案？

## 15. 易混淆结论

- **Decomposition ≠ chain-of-thought**：前者是系统可观察的任务结构；后者是模型推理提示/行为问题。
- **Parallelization ≠ decomposition 的默认目标**：有依赖就应顺序执行。
- **更多步骤 ≠ 更可靠**：每次 handoff 都是新的失败点。
- **局部正确 ≠ 全局正确**：必须验证 coverage、invariants 和 recomposition。
- **动态计划 ≠ 无约束 Agent**：预算、权限、停止条件和验收仍由系统控制。
- **摘要 ≠ artifact**：下游若需要核查事实，应保留原始 evidence reference。

## 16. Guide 要求与当前实现的边界

Exam Guide 要求掌握 complex problem solving 的 decomposition 技术，重点是架构 reasoning：如何划分、排序、验证和重新组合。具体使用 workflow engine、Claude Agent SDK、Managed Agents 或自建服务，不是这一目标规定的唯一答案。

**Needs verification（实施前复核）**：若采用当前 Claude 产品的 compaction、memory、subagents 或 Managed Agents，需要重新核查模型支持、API schema、Beta 状态、限制、费用及数据处理条款。本课不以这些可变实现细节决定练习题答案。

## 17. 核查来源

核查日期：**2026-09-22**。

- [CCAR-P Exam Guide](../../../Exam_Guide.md)：考试范围、Domain 权重、目标和样题风格的最高优先级依据。
- [Anthropic Engineering — Building effective agents](https://www.anthropic.com/engineering/building-effective-agents)：prompt chaining、routing、parallelization、orchestrator-workers、evaluator-optimizer 与复杂度选型。
- [Anthropic Engineering — Effective context engineering for AI agents](https://www.anthropic.com/engineering/effective-context-engineering-for-ai-agents)：高信号 context、长任务 compaction、structured note-taking 与 subagent 分工。
- [Anthropic Engineering — Effective harnesses for long-running agents](https://www.anthropic.com/engineering/effective-harnesses-for-long-running-agents)：feature list、增量工作、持久进度、clean state 与端到端验证。
- [Claude Platform Docs — Define success criteria and build evaluations](https://platform.claude.com/docs/en/test-and-evaluate/develop-tests)：可测 success criteria、task-specific eval、code/human/LLM grading 的边界。
