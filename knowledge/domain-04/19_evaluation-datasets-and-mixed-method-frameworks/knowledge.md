# 19 · Evaluation dataset 与 Mixed-method Test Framework

## Guide 对齐与学习边界

- **Domain**：4 · Evaluation, Testing & Optimization（16%）
- **Exam Guide 明确要求**：`Design evaluation datasets and test frameworks using mixed methodologies`。
- **本课目标**：设计能代表真实任务与风险的 evaluation dataset，并用 code、LLM、human、生产信号等互补方法构建可持续的 test framework。
- **边界**：上一课已定义 accuracy、latency、cost、safety、security 等指标；本课回答“用什么 cases、在什么环境、由谁/什么 grader、怎样持续运行”。下一课才深入 A/B testing 与 iterative improvement。

这里的核心不是选择某个评估产品，而是建立可信的 evidence pipeline：

> **requirements / failures / production patterns → versioned tasks → isolated trials → traces & outcomes → complementary graders → slice-level results → release decision → new regression cases**

---

## 1. Evaluation dataset 不是一堆 prompts

一个完整 test case 至少包含：

| 字段 | 作用 |
|---|---|
| `task_id` / version | 可追踪、可复现、可比较 |
| input / initial state | 用户输入、上下文、环境和前置状态 |
| task contract | 明确目标、允许/禁止行为与约束 |
| reference / expected outcome | 一个已知可行的答案或最终环境状态 |
| graders / assertions | 如何判定正确、部分成功或失败 |
| tags / slices | intent、语言、风险、难度、tool path 等 |
| provenance | 来源：需求、生产样本、incident、synthetic 等 |
| corpus/tool snapshot | RAG 文档版本、工具 schema、sandbox 初始状态 |
| privacy / retention metadata | 是否脱敏、访问范围、保留期限 |

对于 agent，最终文本不是全部结果。Anthropic 将 **transcript/trace** 定义为一次 trial 的完整交互记录，将 **outcome** 定义为最后环境状态。Agent 可能声称“退款成功”，真正要检查的是订单或退款记录是否存在且金额、权限都正确。

---

## 2. 数据从哪里来

### 2.1 需求与业务流程

把 product requirements、acceptance criteria、SOP 和风险控制转成 cases。它们定义“系统本来应做什么”，适合作为初始 capability suite。

### 2.2 人工测试与专家示例

将开发者、产品人员、domain expert 上线前反复手测的场景固化。人工已在检查的行为往往是最具价值的首批 test cases。

### 2.3 生产流量、support ticket 与 incident

从真实分布中抽样并脱敏；把投诉、bug、失败 trace 和安全事件转成 regression cases。每次修复若没有留下测试，未来很容易重犯。

### 2.4 Synthetic / generated cases

用于扩充 paraphrase、边界值、少数语言、罕见组合和 adversarial variants。Synthetic data 适合增加覆盖，但不能单独代表真实分布，因为生成器的偏差、措辞和盲点会被带入数据集。

### 2.5 Red-team 与 threat-driven cases

从 threat model 推导 direct/indirect prompt injection、data exfiltration、越权 tool call、恶意文档与多轮诱导。普通 happy-path 样本无法代替这些测试。

初始 suite 不必等待“完美大数据集”。Anthropic 2026 年 agent eval 实践建议可从少量真实失败和手工检查开始，再随系统成熟扩大。关键是 case 有信息量、能定位、能重复，而不是单纯追求数量。

---

## 3. Coverage matrix：代表流量，也代表风险

随机抽样通常只代表高频 happy path。更稳健的 dataset 同时覆盖：

- **Intent / workflow**：查询、修改、审批、拒绝、升级；
- **Difficulty**：正常、模糊、缺信息、冲突信息、多约束；
- **Input shape**：短/长 context、噪声、不同格式与语言；
- **Risk tier**：普通、财务、隐私、安全、不可逆动作；
- **Tool path**：无需工具、单工具、多工具、工具失败/超时；
- **User segment**：地区、语言、权限角色、tenant；
- **Positive + negative cases**：某行为应发生，也应覆盖不该发生的情况；
- **Temporal/corpus state**：新旧政策、文档刷新、索引更新。

两种权重需要明确区分：

1. **Traffic-weighted view**：估计整体生产体验；
2. **Risk-balanced view**：放大低频但高影响 case，作为 hard gate。

高风险 slice 不应因流量低而被平均掉。报告总体结果时必须同时展示关键 slice。

### 正反例成对

若只测试“该搜索时会不会搜索”，系统可能学成“任何问题都搜索”。同时要测试“不该搜索时是否直接回答”。同理：既测危险请求能否拒绝，也测无害请求是否被 over-refusal；既测需要 tool 时会调用，也测不应有副作用时不会调用。

---

## 4. 数据集角色与隔离

### Development set

开发者可以频繁查看和迭代，用于调 prompt、workflow、grader。因为团队会逐渐针对它优化，它不能单独证明泛化能力。

### Held-out validation / test set

限制查看与使用频率，用于阶段性、发布前的无偏验证。若每次调参都查看并针对失败修改，它实际上已经变成 development set，应补充新的 holdout。

### Regression suite

保存已经修复、必须持续正确的行为，目标通常接近全通过。它回答：“以前会做的事现在仍会吗？”

### Capability / challenge suite

包含尚未稳定解决、用于衡量能力上限和进步空间的 harder tasks。通过率低不一定阻塞发布，取决于当前产品承诺。高通过率后可以“毕业”为 regression case。

### Safety / security suite

针对威胁和高严重性 failure，可采用独立 hard gates；不能让普通质量分数抵消。

### Canary / production replay set

从近期生产分布中抽取并脱敏，用于发现 drift。需要防止同一用户会话、近重复样本或同一 source document 跨 split 泄漏，避免评估虚高。

---

## 5. Ground truth 与标注质量

### 任务必须可解且规格无歧义

好的 case 应让两位合格专家独立判断后得到一致的 pass/fail 结论。若专家自己无法完成，或 grader 检查了任务描述未说明的隐藏要求，测到的是坏题而非系统能力。

每个复杂 task 应有至少一个 **reference solution**：已知能够通过所有 graders 的输入到结果路径。它证明任务可解，也验证 harness/grader 没有配置错误。

### 开放式任务使用 rubric，不强求唯一答案

Rubric 应把维度拆开，例如 factual support、coverage、action correctness、communication，而不是一句“回答得好”。可定义：

- binary hard checks；
- ordinal scale 与每档 anchor；
- partial credit；
- critical failure override；
- `Unknown / insufficient evidence`，避免 grader 被迫猜测。

### Human annotation 治理

- 提供 annotation guideline 与示例；
- 使用 domain experts 处理法律、医疗、财务等专业内容；
- 抽样双人/多人标注，检查 inter-rater agreement；
- 对 disagreement 做 adjudication，并反向修订 task/rubric；
- 记录 label、rubric 和 source 的版本。

“Gold label”不是天然正确；它也需要审计、更新和 provenance。

---

## 6. Mixed methodologies：每种方法补另一种的盲区

### 6.1 Code-based grading

适合 exact/schema match、unit/integration test、数据库状态、权限、tool 参数、静态分析、latency/token 统计。

- 优点：快、便宜、客观、可复现，适合 CI；
- 缺点：容易 brittle，可能拒绝未预料但正确的路径，无法判断细腻语义。

优先 grade **outcome**，只在过程本身是安全/合规要求时才强制具体 tool sequence。例如必须先验证身份再退款属于过程约束；“必须按固定顺序浏览三个页面”若无业务必要，则可能过度约束。

### 6.2 LLM/model-based grading

适合开放式质量、groundedness、tone、completeness、pairwise comparison 和自然语言 assertions。

- 优点：灵活、可扩展、处理多个合理答案；
- 缺点：非确定、有偏差、成本高于规则，可能被输出措辞影响或被攻击。

需用清晰单维 rubric、结构化输出、盲化 variant 身份，并用专家样本校准。必要时使用多 judge 或重复判定，但不能把 consensus 当成绝对真相。

### 6.3 Human grading 与 transcript review

适合主观质量、高风险判断、发现未知 failure、校准 LLM graders 和审计错误归因。

- 系统性 human study 提供高质量参考，但慢且昂贵；
- 定期 transcript sampling 更轻量，可发现 grader bugs、合理的替代路径和新的 failure taxonomy。

### 6.4 Production monitoring、user feedback 与 A/B

- production monitoring 捕获真实分布、drift 与运营 failure；
- user feedback 提供真实例子，但 sparse、自选择且偏向严重问题；
- A/B testing 能测真实业务 outcome，但需流量、时间和安全边界，下一课详细展开。

**Mixed-method 并非每个 case 都使用全部方法**。应按风险与可验证性组合：能用 deterministic state check 就不要只用 LLM judge；主观质量可加 LLM rubric，并定期用 human 校准；未知生产 failure 再回流为离线 regression case。

---

## 7. Test framework 的参考架构

```text
Versioned Dataset Registry
  ├─ tasks + tags + provenance
  ├─ references + rubrics
  └─ environment/corpus snapshots
             │
             ▼
Evaluation Runner / Harness
  ├─ pin system variant and config
  ├─ create isolated clean environment
  ├─ run one or multiple trials
  └─ capture transcript, outcome, cost, latency
             │
             ▼
Grader Layer
  ├─ deterministic/code/state checks
  ├─ LLM rubric graders
  └─ human review/adjudication samples
             │
             ▼
Aggregator & Report
  ├─ overall + slices + uncertainty
  ├─ failure taxonomy + trace links
  ├─ baseline/variant comparison
  └─ hard gates + release recommendation
             │
             ▼
Failure → triage → fix → new regression task
```

### Harness 的关键性质

- **Reproducibility**：记录 model identifier、prompt、tools、parameters、corpus、grader 和 code version；
- **Isolation**：每个 trial 从干净状态开始，防止缓存、文件、数据库或 Git 历史泄漏；
- **Production fidelity**：agent harness、tool schema、权限和 error handling 尽量接近生产；
- **Observability**：保存 trace、tool calls、最终环境状态、token、cost、latency 与错误；
- **Concurrency control**：并行运行不共享会相互污染的状态；
- **Failure transparency**：能区分 agent failure、grader bug、task ambiguity 与 infrastructure flakiness。

---

## 8. 非确定性与多次 trials

一个 task 执行一次通过，不代表可靠。应根据业务要求重复试验：

- **pass@k**：k 次中至少一次成功的概率，适合允许多次尝试或候选方案的任务；
- **pass^k**：k 次全部成功的概率，适合用户期望每次都可靠的生产 agent；
- per-task success rate 与方差；
- 相同 failure 是否来自共享环境故障，而非独立模型行为。

不要自动重试并只保留最好结果，否则会把第一次失败和真实成本藏起来。Retry policy 本身是系统设计的一部分，必须固定并报告。

---

## 9. CI/CD 分层运行策略

| 层级 | 内容 | 典型触发 |
|---|---|---|
| Smoke | 少量核心 happy path + hard guardrails | 每个小改动 |
| Regression | 历史故障、承诺能力、关键 slices | 每个 PR / release candidate |
| Full capability | 大规模、较难、多 trial、成本较高 | 定期或重大变更 |
| Safety/security red-team | adversarial、多轮、越权和泄漏 | 发布前、权限/工具变更 |
| Production validation | telemetry、抽样 review、drift | 持续运行 |

发布 gate 可表达为：所有 critical assertions 必须通过；关键 regression slice 不得下降；capability 指标达到目标；latency/cost 不越界。对于随机结果，应比较置信区间或预定义容差，而不是看到一次小波动就下结论。

---

## 10. 数据集生命周期

Evaluation suite 是 versioned product asset：

1. 收集需求、真实失败和新风险；
2. 去重、脱敏、标注、审核；
3. 分配到 dev/holdout/regression/capability/security suites；
4. 运行并读 traces，确认失败“公平且可解释”；
5. 修正 broken task、grader bug 或环境噪声；
6. 监控 suite saturation 与生产 drift；
7. 增加更难或更新的 cases，同时保持历史 regression；
8. 指定 owner，记录变更理由和版本。

Public benchmark 可以提供参考，但不能替代业务 eval：训练污染、prompt format、harness 差异、错误 label 和任务不匹配都可能让分数失真。

---

## 11. 典型 failure modes

1. **Happy-path-only**：真实边界、工具失败与攻击完全没覆盖。
2. **Traffic-only sampling**：低频高风险 case 被忽略。
3. **One-sided dataset**：只测该触发的行为，导致 over-triggering。
4. **Holdout leakage**：反复查看并针对 test set 调参，评估失去独立性。
5. **Synthetic monoculture**：所有 cases 由同一模型/模板生成，语言和盲点过于一致。
6. **Ambiguous task**：专家都无法一致判断，却把 disagreement 算成模型错误。
7. **Brittle path grading**：强制无必要的中间步骤，惩罚正确的替代方案。
8. **Self-report as outcome**：agent 说“完成”就算成功，不查数据库/文件/业务状态。
9. **Shared-state contamination**：前一次 trial 的文件、缓存或数据帮助/破坏下一次。
10. **LLM judge without calibration**：把可扩展性误当可靠性。
11. **Retry cherry-picking**：只保留最好一次，隐藏不稳定与真实成本。
12. **Stale suite**：产品、数据和攻击方式变化，dataset 长期不维护。

---

## 12. 考试决策模板

遇到 dataset/framework 场景题时依次判断：

1. Cases 是否来自真实任务、历史 failure、边界和 threat model？
2. 是否同时覆盖正常/异常、该做/不该做、高频/高风险？
3. dev、holdout、regression、capability 是否隔离？
4. task 是否无歧义、可解，并有 reference solution？
5. 是否检查最终 outcome，而不是相信模型自述？
6. deterministic、LLM、human grader 是否各用在擅长的部分？
7. agent trial 是否环境隔离、接近生产并保存完整 trace？
8. 非确定性是否通过多 trial、方差或一致性要求处理？
9. 是否按关键 slices 报告，并让 critical safety/security failure 独立阻塞？
10. 生产 failure 是否回流为 regression case，suite 是否有人维护？

---

## Guide 要求与当前实现细节

Guide 要求掌握 evaluation dataset 与 mixed-method framework 的架构方法，不要求记忆某个评估平台、Claude 型号或产品字段。Anthropic 2026 年 agent eval 文章中的术语和实践可用于理解 task、trial、grader、trace、outcome、harness 与 suite，但具体框架、模型支持和 API 会变化。

### Needs verification

实施前重新核查：当前 Claude/API 的模型与参数、trace 可见字段、Agent SDK 或评估工具能力、价格、数据保留与隐私条款。任务数量、trial 次数、通过阈值、human review 比例和统计显著性必须按实际风险、流量和预算确定，不能照搬示例。

---

## 核查来源（2026-09-25）

- Anthropic，[Demystifying evals for AI agents](https://www.anthropic.com/engineering/demystifying-evals-for-ai-agents)（2026-01-09）
- Anthropic，[Define success criteria and build evaluations](https://platform.claude.com/docs/en/test-and-evaluate/develop-tests)
- Anthropic，[Challenges in evaluating AI systems](https://www.anthropic.com/research/evaluating-ai-systems)
- Anthropic，[Building Effective AI Agents](https://www.anthropic.com/engineering/building-effective-agents)
