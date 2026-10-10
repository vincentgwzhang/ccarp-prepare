# 34 · Claude 模型选型权衡

## Guide 对齐

- **Domain**：Domain 2 · Claude Models, Prompting & Context Engineering（13%）
- **Exam Guide 要求**：`Select appropriate Claude models based on trade-offs`
- **本课目标**：从业务任务、质量门槛、延迟、成本、功能、平台和生命周期约束出发，用真实 workload 的评估证据选择 Claude model，并设计升级、降级、路由与迁移策略。
- **范围边界**：本课讲 model selection。#22 已深入 token、latency 与 cost-performance 优化；#35–#40 将分别深入 system prompt、prompt techniques、context、caching、modular prompts 和 Skills。

Exam Guide 考查的不是记住某天的 model 排名、价格或 context window，而是能否回答：

> 在给定任务、风险和运行约束下，哪个候选模型以可接受的总成本满足质量与安全门槛？证据是什么？变化后怎样复验和回滚？

---

## 1. 模型选型是约束满足，不是排行榜比赛

“选择最强模型”通常不是完整的 architecture decision。生产选型至少包含三层：

1. **Hard constraints**：必须满足，否则淘汰。
2. **Optimization objectives**：在合格候选中优化。
3. **Operational policy**：怎样固定、路由、监控、升级和回滚。

### 1.1 Hard constraints

- task-specific quality / accuracy / safety threshold；
- 所需 input modality、tool use、structured output、thinking 或其他 capability；
- provider / region / platform availability；
- context 与 output capacity；
- security、privacy、data residency、compliance 和 commercial terms；
- latency SLO、throughput、rate/capacity requirement；
- model lifecycle 必须覆盖计划运行和迁移周期。

### 1.2 Optimization objectives

- cost per successful task；
- p50 / p95 / p99 end-to-end latency 与 time to first token；
- task success、edge-case success、human-review burden；
- maintainability、migration effort 和 operational simplicity；
- failure impact、retry / escalation rate、downstream rework。

只比较单价会漏掉失败重试、更多 agent turns、长输出、人工返工和事故成本；只比较 benchmark score 又会漏掉目标 workload、prompt、tools、RAG 和风险切片。

---

## 2. 当前模型产品线：只作为实现示例

截至 **2026-10-10**，Anthropic 官方 Models overview 与 Choosing a model 页面将当前主要选择描述为以下角色。此表是产品现状，不是永久考试事实：

| 当前产品角色 | 官方定位摘要 | 适合作为起始候选的场景 |
|---|---|---|
| Haiku line | 最低延迟和价格、面向高吞吐任务 | classification、extraction、routing、实时且结果可校验的任务 |
| Sonnet line | speed 与 intelligence 的平衡 | 日常 coding、分析、内容、enterprise workflow、tool use |
| Opus line | 长时间 agentic coding 与复杂 knowledge work | 大型重构、复杂工程、长任务和较高 autonomy |
| Fable line | demanding reasoning 与 long-horizon agentic work | 当高 effort Opus 在本地 eval 中仍不足的高难任务 |

不要从名称推出绝对结论：

- 更高 tier 不保证在每个 workload 上更准确或更便宜；
- 更低 tier 不代表不安全，也不自动适合所有简单任务；
- model family、版本、effort、prompt、tools 和 orchestration 共同决定结果；
- 新 model 发布后，角色、价格和 feature support 可能改变。

官方当前指南提供两种合理的起点：

- **Efficiency-first**：从快速、低成本候选开始，用 eval 证明是否已经满足门槛；只有在明确 capability gap 时升级。
- **Capability-first**：复杂、高价值、高风险或高 autonomy 任务先从更强候选建立质量上界，再通过降低 effort、优化 workflow 或降级 model 找效率边界。

两者没有通用胜负。问题的风险、失败可检出性和质量门槛决定从哪端搜索。

---

## 3. Model selection contract

在跑 benchmark 前先写一页 selection contract，避免看到结果后移动标准。

```text
Task / user population:
Input distribution and critical slices:
Required capabilities / platform:
Quality and safety hard gates:
Latency / throughput SLO:
Budget and cost metric:
Human review / fallback:
Candidate models and effort settings:
Evaluation dataset and graders:
Decision rule:
Rollout, monitoring and rollback:
Review / deprecation trigger:
```

### 3.1 Task contract

不要以“客服”“coding”“总结”作为完整任务定义。需要描述：

- input size、language、modality、ambiguity 和 distribution；
- output schema、factuality、citation、tone 和 completeness；
- 是否调用工具、是否多轮、是否有真实 side effect；
- success、partial success、abstention 和 failure 的判定；
- error 是否可检测、可逆、可由人复核；
- average case 与 hardest / highest-risk tail。

同一个应用可能包含不同任务：intent routing、retrieval query generation、answer drafting、policy judgment、tool execution。它们不必使用同一个模型。

### 3.2 Hard gate 与 preference

把条件分开：

| 类型 | 示例 | 决策方式 |
|---|---|---|
| Hard quality gate | 高风险 slice recall ≥ 目标值 | 未达到即淘汰 |
| Hard safety/security gate | 不得产生未授权 tool action | 未达到即阻断 |
| Hard operational gate | p95 在 SLO 内、平台/region 可用 | 未达到即淘汰或改架构 |
| Preference | 更低 cost、较快 median latency | 合格候选之间优化 |

不能用低成本抵消安全硬门槛，也不能让总体平均 accuracy 掩盖严重 slice failure。

---

## 4. 用真实 workload 建立证据

Anthropic 官方选型指南把 use-case-specific benchmark / evaluation set 视为最重要步骤，并要求使用实际 prompts 和 data 比较 response quality、accuracy 与 edge cases。

### 4.1 候选实验应保持可比

对每个候选记录：

- exact model ID、platform、region 和测试日期；
- prompt / system / examples version；
- tool schemas、RAG/index、orchestrator 和 guardrails version；
- effort / thinking、output cap、timeout、retry 等配置；
- dataset、slices、grader、重复次数和统计不确定性；
- token usage、latency、task success、failure / escalation reason。

若同时改变 model、prompt、tools 和 dataset，就无法把结果归因给 model。迁移时允许为新 model 做必要 prompt adaptation，但应保留：

1. **controlled comparison**：尽量只换 model，识别行为差异；
2. **best-achievable comparison**：各候选使用合理优化后的配置，比较最终可达结果。

### 4.2 不只看总体均值

至少分别看：

- normal / frequent cases；
- ambiguous、long、multilingual 和 adversarial cases；
- high-risk / irreversible action cases；
- tool/retrieval failure 与 empty/conflicting context；
- hardest 10% 或其他业务 tail；
- cold / warm cache、interactive / batch 等运行模式。

低价模型在普通请求上可能与高 tier 无差异，但在 tail 上失败后产生 retry、escalation 和损害；也可能相反——高 tier 在任务已饱和时增加成本却没有质量收益。

---

## 5. 正确比较成本：cost per successful task

基础公式：

```text
Expected cost per successful task
  = (model + cache + tool + retrieval + infrastructure + human-review cost)
    / successful task count
    + expected downstream failure cost
```

更实用的单任务视角：

```text
Expected total cost
  = first-attempt cost
  + P(retry) × retry cost
  + P(escalation) × escalation/review cost
  + P(failure) × remediation / business impact
```

为什么 per-token price 可能误导：

- 强模型可能用更少 turns、searches 和 retries 完成任务；
- 便宜模型可能对大多数任务足够，因此高 tier premium 没有价值；
- output token 通常比 input token 贵，但模型可能生成长度不同；
- cache hit、batch、effort 和 context strategy 会改变实际账单；
- failed task 仍收费，还可能触发第二次调用与人工处理。

官方 cost/intelligence 指南也明确建议用 **cost per completed task** 比较候选，并声明官方 benchmark 结果只具方向性，必须在自己的 workload 上测量。

---

## 6. 延迟不是一个数字

模型选型应拆解：

- queue / admission latency；
- time to first token（TTFT）；
- output generation time；
- tool / retrieval round trips；
- retry、fallback、human approval；
- total task completion time。

Streaming 可以改善 perceived latency，却不一定缩短 task completion；更强模型如果少走几个 agent rounds，单次生成较慢但端到端可能更快。对 synchronous UI 看 TTFT 和 p95；对离线评估或 backfill，看 throughput 与 cost；对 agent 看完成任务的 wall-clock time，而不是一次 API call。

---

## 7. Effort 与 model tier 是两个不同旋钮

当前部分 Claude models 支持 `effort`，用于在同一 model 内调节思考、tool use/self-verification 的深度与成本/延迟。官方当前指南建议先从该 model 的默认 effort 出发，再根据 eval 上下调整，并指出调 effort 往往比直接换 model 更合适。

架构判断：

- quality 已达标、成本或延迟过高：先 sweep effort；
- 当前 model ceiling 不足：再提升 tier；
- 高 tier 的 low effort 可能优于低 tier 的 high effort，也可能不优；必须实测；
- 对某些简单任务，高 effort 只增加无用推理和工具轮次；
- effort availability、取值和默认值是产品细节，实施前核查。

不要把 model name 当作完整配置。决策单位通常是：

```text
(model ID, effort/thinking config, prompt, tools, context, budgets, platform)
```

---

## 8. 单模型、路由和多模型架构

### 8.1 单模型

优点：实现、观测、评估和迁移简单；行为一致。若一个较经济的候选对整个 workload 达标，优先单模型通常更可维护。

### 8.2 Rule-based routing

按明确元数据分流，例如语言、任务类型、风险等级、input size、是否需要 vision/tool。优点是可解释、易审计；缺点是规则需要维护，不能准确预测所有难度。

### 8.3 Classifier / confidence routing

用 classifier 或低成本 model 估计任务类型/难度，再选执行模型。必须评估 router 自身错误，尤其不能让不可信用户通过 prompt 自报“低风险”绕过强控制。

### 8.4 Escalation / fallback

- 低成本 executor 处理主流量；
- deterministic verifier、tool result、confidence signal 或 failure rule 触发更强 advisor / retry；
- 高风险动作仍使用独立 authorization / HITL，不能只靠更大模型。

### 8.5 Orchestrator-worker

高能力 orchestrator 规划和汇总，经济型 workers 执行大量独立子任务。适合可并行分解的长任务，但增加调用、状态、错误传播和 observability 复杂度。

多模型策略只有在 **端到端** 结果优于单模型 baseline 时才成立。路由模型、双重调用、缓存破坏、延迟和维护成本都必须计入。

---

## 9. Model ID、版本稳定性与生命周期

### 9.1 不要凭名称猜 pinning 语义

Anthropic 当前文档说明：

- 每个 model ID 对应固定的 model version；
- 4.6 generation 及之后的 dateless ID 是 canonical pinned snapshot，不是自动漂移到最新版的 alias；
- 更早 generation 某些短 alias 会指向该 minor version 的最新 dated snapshot；
- model weights 固定并不代表 serving infrastructure 永远不变，router、safety classifier 等基础设施变化仍可能产生细微行为差异。

因此生产系统要保存 exact model ID、request/config、prompt/tool/index versions 和 trace。不要仅在文档中写“Sonnet”或“最新 Claude”。

### 9.2 Lifecycle state

官方区分：

- **Active**：支持且推荐；
- **Legacy**：不再更新，未来可能 deprecated；
- **Deprecated**：仍能调用，但不再推荐，已有 retirement date / replacement；
- **Retired**：不可再用，请求失败。

Partner-operated platforms 的 lifecycle 和日期可能与 Anthropic-operated platform 不同。模型选择必须包含：

- inventory：哪些服务、prompt、batch job 和 fallback 在用哪个 ID；
- deprecation watch 与 accountable owner；
- replacement eval / prompt audit；
- shadow/canary、rollback 与 cutover；
- 在 retirement 之前完成迁移，不把 deadline 当测试开始日期。

---

## 10. Context window、knowledge 与 feature support

### 10.1 Context capacity 不是 context quality

候选支持足够 context 只是 hard constraint。把所有内容塞满 window 可能增加：

- irrelevant/noisy context；
- retrieval conflict 与 instruction injection；
- TTFT、成本与 cache pressure；
- “信息存在但未被正确使用”的风险。

正确问题是“模型能否在目标 context distribution 上可靠找到、遵循并引用需要的信息”，而不是“最大窗口有多大”。#37 将深入 context management。

### 10.2 Knowledge cutoff 不是 freshness solution

任务需要当前事实时，应使用受控 retrieval/tool 和 provenance；不能因为某候选的 knowledge cutoff 更新，就省略数据管道。模型知识也不能作为权限、业务规则或账户状态的 source of truth。

### 10.3 Feature / platform matrix

选型必须在目标平台核对：

- model availability 与 exact ID；
- tool / vision / thinking / structured output 等 capability；
- context、output、rate/capacity limits；
- region、data handling、compliance 与 network integration；
- pricing、cache/batch 支持和 lifecycle schedule。

Claude API、Amazon Bedrock、Google Cloud、Microsoft Foundry 或其他运行面不能假设完全等价。

---

## 11. 一个可执行的选型流程

### Step 1：分解 workload

把应用拆为可独立评估的 task classes，而不是给整个产品贴一个 model。

### Step 2：写 hard gates 与 business weights

先写 quality/safety/latency/platform 门槛，再定义 cost、speed、maintainability 的权重。

### Step 3：选择少量合理候选

包括不同 tier 和 effort；排除不支持 required capability/platform 的型号。

### Step 4：建立 representative eval

使用真实 prompts、data、tools、RAG、normal/tail/risk slices；同时测质量、成本、延迟和人工负担。

### Step 5：找 Pareto frontier

淘汰同时更贵、更慢、质量还更差的候选。对 remaining candidates 使用明确 decision rule，而不是凭感觉。

### Step 6：做 architecture-level optimization

尝试 prompt/context/tool 改善、effort sweep、routing 或 escalation；与简单单模型 baseline 比较。

### Step 7：staged rollout

绑定版本、shadow/canary，设置 guardrails、SLO、rollback 与 owner。

### Step 8：持续复审

当 model、prompt、tool、data distribution、价格、platform、risk 或 lifecycle state 改变时，重新评估。

---

## 12. Java / Backend 架构示例

假设电商客服包含三类请求：

1. 订单意图分类：高吞吐、结果可用规则校验；
2. 基于订单与政策生成回复：需 grounded、工具调用和低延迟；
3. 复杂争议处理：长上下文、跨工具推理、高业务风险。

可设计为：

```text
API Gateway
  → deterministic auth / tenant context
  → task classifier + risk rules
      → fast tier: classification/extraction
      → balanced tier: normal grounded response
      → high-capability tier: complex case analysis
  → deterministic policy / tool authorization
  → human review for high-impact action
  → telemetry with route, model/config version and outcome
```

Java service 中不要把 model ID 散落在业务代码。使用 version-controlled policy：

```java
record ModelRoute(
    String taskClass,
    String modelId,
    String effort,
    Duration timeout,
    int maxAttempts,
    String fallbackRoute,
    String evalPolicyVersion
) {}
```

路由配置必须与 eval evidence、deployment version 和 rollback policy 关联。Fallback 也要经过验证；若主模型因安全/授权 hard gate 失败，不能简单切换到更宽松的模型继续执行。

---

## 13. 典型 failure modes

| Failure mode | 问题 | 修正 |
|---|---|---|
| “总用最强模型” | 成本/延迟浪费，未证明额外价值 | 按 task eval 与 hard gates 选型 |
| “总用最便宜模型” | 忽略失败、重试和下游成本 | 比较 cost per successful task |
| 只看公开 benchmark | 与真实 prompt/data/tools 分布不一致 | workload-specific eval |
| 只测平均 case | tail 和高风险 slice 被掩盖 | 单独测试 hard/risk slices |
| 同时改 model 和全部 prompt | 无法归因 | controlled + optimized 两阶段比较 |
| 认为大 context 自动更好 | 噪声、成本、延迟和 injection 增加 | relevance、retrieval、context eval |
| 用更强模型代替授权/HITL | capability 不是 deterministic control | 独立 policy enforcement |
| 多模型路由没有 baseline | 复杂度大于收益 | 与单模型端到端比较 |
| 写“latest model” | 不可重现、难回滚 | exact ID + version bundle |
| 到 retirement 前才迁移 | 无时间做 regression/canary | 提前 inventory、eval、cutover |

---

## 14. 考试决策框架

场景题中按以下顺序判断：

1. **任务与风险是什么？** 简单/可校验，还是复杂/长程/高影响？
2. **有哪些 hard constraints？** capability、quality、安全、延迟、平台、region、lifecycle。
3. **证据来自哪里？** 真实 prompts/data/tools 的 eval，而非型号名或营销描述。
4. **比较什么成本？** end-to-end cost per successful task，不只是 token 单价。
5. **是否需要 routing？** 先证明单模型不够，再引入多模型复杂度。
6. **怎样上线和变化？** exact ID、canary、monitor、rollback、deprecation migration。

常见最佳答案通常会“定义门槛 → 在代表性 workload 上比较 → 选择满足门槛的最低总成本方案 → staged rollout”。绝对化答案——永远 Opus、永远 Haiku、最大 context、最低单价——通常缺乏架构证据。

---

## 15. Guide 要求与当前实现细节

Guide 的稳定要求是 **based on trade-offs 选择合适 Claude model**。当前 product lineup、model IDs、默认 effort、feature support、prices、context/output limits、rate limits、platform availability 和 retirement dates 都可能变化。

### Needs verification

每次真实实施或考试前，应重新核查官方：

- Models overview / Choosing a model；
- exact model ID、capabilities、limits 和目标平台 availability；
- pricing、cache/batch/effort 支持；
- model lifecycle、deprecation 和 migration guide；
- model cards、data handling 与 platform-specific 条款。

本文练习题的正确答案依赖稳定的选型原则，不依赖 2026-10-10 的具体价格、型号排名或窗口数字。

---

## 16. 核查来源

核查日期：**2026-10-10**。

1. Anthropic — [Models overview](https://platform.claude.com/docs/en/models/overview)
   - 当前 lineup、相对角色、capabilities、platform IDs 和 Models API。
2. Anthropic — [Choosing the right model](https://platform.claude.com/docs/en/about-claude/models/choosing-a-model)
   - capability/speed/cost/effort 标准、efficiency-first/capability-first、selection matrix、真实 workload eval 与 multi-model strategy。
3. Anthropic — [Optimizing for cost and intelligence](https://platform.claude.com/docs/en/about-claude/models/optimizing-for-cost-and-intelligence)
   - cost per completed task、effort、tail workload、routing/orchestration，以及官方 benchmark 仅为方向性证据。
4. Anthropic — [Model IDs and versioning](https://platform.claude.com/docs/en/about-claude/models/model-ids-and-versions)
   - pinned IDs、旧 alias 与 dateless canonical ID 的区别，以及 serving infrastructure 变化边界。
5. Anthropic — [Model deprecations](https://platform.claude.com/docs/en/about-claude/model-deprecations)
   - Active / Legacy / Deprecated / Retired、迁移和 platform-specific lifecycle。
6. Anthropic — [Define success criteria and build evaluations](https://platform.claude.com/docs/en/test-and-evaluate/develop-tests)
   - task-specific、multidimensional success criteria、edge cases 和 held-out evaluation。
