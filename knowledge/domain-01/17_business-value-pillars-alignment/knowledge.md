# 17 · 业务价值支柱对齐框架

## Guide 对齐与学习边界

- **Domain**：1 · Solution Design & Architecture（17%）
- **Exam Guide 明确要求**：`Align solutions to business value pillars (efficiency, transformation, productivity, cost, performance SLAs)`。
- **本课目标**：把业务目标转成可衡量的成功条件，再让架构决策、约束和验证方法都能追溯到这些条件。
- **边界**：本课讨论“为什么选择某种架构、怎样证明它创造价值”。Domain 4 会进一步展开评估数据集、测试方法、A/B test 和持续优化；这里不替代完整 evaluation framework。

考试中的关键不是背五个名词，而是识别：题目真正优化哪个价值支柱？什么指标能证明它？哪些 guardrail 不能被牺牲？哪种架构在约束下最合适？

---

## 一条可审计的价值链

Claude 方案不应从“选哪个模型”开始，而应建立以下 traceability chain：

> **Business outcome → KPI / target → AI capability → architecture lever → guardrail / counter-metric → measurement & owner**

例如，客服部门的目标不是“部署一个 agent”，而可能是：

| 层次 | 示例 |
|---|---|
| Business outcome | 在不降低解决质量的前提下缩短客户等待时间 |
| KPI / target | 90 天内将 p95 首次响应时间降低 30%，重开率不升高 |
| AI capability | 分类、检索政策、起草回复、建议下一步动作 |
| Architecture lever | workflow + RAG；高风险动作保留 human approval |
| Guardrail | 答案依据性、升级准确率、隐私、重开率 |
| Measurement & owner | 工单系统与评估集；客服运营负责人签收 |

这条链能够防止 solution-first thinking：先买工具或构建 agent，最后才寻找用例和价值解释。

每个 KPI 至少要明确：**baseline、target、time horizon、population/segment、data source、owner**。没有基线的“提高 20%”没有意义；只看总体平均值还可能掩盖高风险客户、语言或业务线的退化。

---

## 五类价值支柱

### 1. Efficiency：以更少浪费完成同一结果

Efficiency 关注流程摩擦、等待、返工和资源浪费。典型指标包括：

- cycle time、handling time、queue age；
- automation/straight-through processing rate；
- handoff 次数、rework rate、重复录入量；
- 从输入到**可接受结果**的时间，而不是只测模型生成时间。

架构上可考虑固定 workflow、并行处理、检索复用、缓存、批处理或有边界的自动化。但“更快地产生错误结果”不算效率提升。因此应同时观察质量、返工、投诉、风险事件等 counter-metrics。

### 2. Productivity：单位人员或团队产出的有效价值

Productivity 关注组织能力，而不只是单个步骤更快。可使用：

- 每位员工每小时完成且被接受的案件数；
- 在相同人员规模下解决的复杂任务量；
- 从机械工作转移到高价值判断的时间；
- adoption、active use、accepted suggestion rate。

“生成了多少文本”“调用了多少次模型”通常是 activity metric，不是生产力。若员工必须花更多时间验证和返工，模型吞吐量增加也可能让实际生产力下降。

**Efficiency 与 Productivity 的区别**：前者侧重流程做得更省，后者侧重人或组织产生更多有价值的结果。二者常相关，但不能互换。

### 3. Cost：看 unit economics 与 total cost of ownership

只比较 token 单价会漏掉大量真实成本。TCO 至少要考虑：

- model inference、retrieval、tools、storage 与基础设施；
- 设计、集成、evaluation、observability 和维护；
- human review、升级处理、返工与错误成本；
- incident、security、compliance 与 opportunity cost。

比“每次 API 调用成本”更有业务意义的指标是：

```text
cost_per_successful_outcome =
  (inference + tools + infrastructure + human_review + rework + operations)
  / accepted_successful_outcomes
```

当便宜方案导致更多失败、重试或人工复核时，它可能具有更差的 unit economics。ROI 可表达为 `(可归因收益 - TCO) / TCO`，但必须公开假设、时间范围和归因方法。

对于无需即时响应的大批量工作，可以评估异步 batch；对于稳定且重复的 prompt/context 前缀，可以评估 prompt caching。它们是实现杠杆，不是所有场景的默认答案，且当前产品价格、限制和支持范围会变化。

### 4. Performance SLA：把服务表现变成可验证承诺

性能不是一句“响应要快”，而是包含负载条件和测量口径的服务契约：

- latency：p50/p95/p99、time to first token、end-to-end completion；
- availability、error/timeout rate、throughput、queue time；
- freshness、recovery time、capacity under peak load；
- 必要时加入最低 quality/safety floor。

常用层次：

- **SLI**：实际测量值，例如“过去 30 天请求的 p95 端到端时延”；
- **SLO**：团队希望达到的内部目标；
- **SLA**：面向客户或业务方的承诺，通常还定义违约后果。

平均时延会掩盖 tail latency；仅测 Claude API 时延则会遗漏排队、retrieval、tool execution、validation 和网络。Streaming 能改善 perceived responsiveness，但不必然缩短完整任务时间。架构还应考虑 capacity、backpressure、timeouts、retries、fallback 和 graceful degradation。

### 5. Transformation：实现以前做不到的新能力或经营模式

Transformation 不等于把现有人工步骤电子化。它通常意味着：

- 新产品、新客户体验或新收入来源；
- 新的覆盖范围，例如让小众语言或长尾请求获得服务；
- 过去因规模、知识或时间限制无法完成的工作；
- 更短的产品/服务 time-to-market。

可观察新能力启用率、用户采用率、覆盖率、转化/收入、风险调整后的业务价值，而不只看节省工时。由于转型型项目不确定性更高，宜使用 staged investment：先验证关键假设与风险，再逐步扩大 autonomy、用户群和投入。

**Transformation 与自动化的区别**：自动化通常改善已有流程；转型改变可提供的价值或流程本身。一个项目可以同时服务多个支柱，但应说明主支柱和优先级。

---

## 多目标设计：价值不是单一最大化问题

真实架构经常面对冲突：

- 更强的验证提高质量，却增加 latency 与 cost；
- 更多 autonomy 提高覆盖和效率，却扩大风险半径；
- 更大的 context 可能改善信息覆盖，却提高成本和响应时间；
- 更便宜的配置可能增加人工复核和返工。

因此应使用“**主要目标 + 硬性门槛 + 次要优化项**”表达决策。例如：

1. 必须满足 privacy、security 和 grounded-answer quality floor；
2. 在满足门槛的方案中，选择 p95 latency 最低者；
3. 若两者接近，再比较 cost per successful resolution。

这是 constrained optimization，而不是把所有指标压成一个不透明分数。考试题中若某选项只最大化速度或成本、却破坏明确质量/合规约束，通常不是最佳答案。

### Balanced scorecard 示例

| 角色 | 指标 | 类型 |
|---|---|---|
| 主要价值 | p95 工单解决周期 | Efficiency |
| 结果质量 | 首次解决率、重开率 | Guardrail |
| 人员价值 | 每人完成且接受的案件数 | Productivity |
| 经济性 | cost per successful resolution | Cost |
| 服务契约 | availability、p95 end-to-end latency | SLA |
| 风险 | 隐私事件、错误升级率 | Hard guardrail |

指标不宜无限增加。优先保留能改变 go/no-go 或架构决策的指标，并为每个指标指定 owner。

---

## 从价值支柱反推架构

| 业务约束或价值重点 | 常见架构方向 | 需要防范 |
|---|---|---|
| 稳定、重复、步骤明确；强调效率 | Deterministic workflow，模型只处理需要语义判断的节点 | 把低变异规则问题过度 agentic 化 |
| 开放式任务、路径不可预知；强调新能力 | 有边界的 agent + tools + checkpoints | 无限 loop、权限过大、成本失控 |
| 高频查询、需要企业事实依据 | RAG + source/ACL controls + evaluation | 检索失败、陈旧数据、越权内容 |
| 严格交互 SLA | 简化 critical path、容量规划、超时/降级、必要时 streaming | 只优化模型响应而忽略端到端 tail latency |
| 非实时高吞吐任务 | Async queue / batch、幂等与结果回收 | 用交互式同步架构承担离线任务 |
| 高风险决策 | Human-in-the-loop、审批 gate、audit trail | 把“有人参与”当成无条件安全保证 |
| 成本敏感但任务分层明显 | Routing、分层验证、缓存/复用 | 低成本路径破坏质量门槛 |

架构选型必须连接到量化假设。例如，不应写“使用 caching 降成本”，而应写“稳定前缀占输入的主要部分；在 representative workload 上测量 cache hit、cost per accepted task 与 p95 latency，未命中时仍满足 SLA”。

---

## Pilot 与 go/no-go

一个价值导向的 pilot 应按以下顺序设计：

1. **记录 baseline**：当前流程的时间、成本、质量、风险和分群表现。
2. **定义 hypothesis**：哪种 Claude capability 通过什么机制改善哪个支柱。
3. **设定 target 与 guardrails**：包括 go/no-go threshold。
4. **限定范围**：选择代表性流量和风险可控的用户群；保留 rollback。
5. **同时测离线与在线结果**：离线 eval 验证已知案例，线上指标验证真实 adoption、workflow 和运营影响。
6. **按结果迭代**：扩大、重构、降低 autonomy 或停止，而不是因为已投入就继续。

注意选择偏差：只测简单请求会夸大自动化率；只测愿意采用的早期用户会夸大生产力；只算模型账单会夸大成本收益。

---

## 典型 failure modes

1. **Solution-first**：先决定“上 agent”，再找业务目标。
2. **Vanity metrics**：以生成量、调用量、demo 成功率替代可接受业务结果。
3. **No baseline / no owner**：无法证明改善，也没人对指标负责。
4. **Average-only SLA**：平均值好看，但关键用户遭遇严重 tail latency。
5. **Cost per request**：忽略成功率、复核、重试、返工和运营成本。
6. **Local optimization**：模型节点更快，但端到端流程因验证或人工队列更慢。
7. **Unbalanced autonomy**：为提高效率扩大权限，却没有 approval、audit 和 blast-radius controls。
8. **Transformation without adoption**：新能力技术上可用，但没有 workflow integration、培训或激励。
9. **Metric gaming**：压缩 handling time 导致草率关闭工单和重开率上升。
10. **Volatile implementation facts as strategy**：把某一时点的价格、模型速度或 API 限制当成长期架构原则。

---

## 考试决策模板

遇到场景题时依次判断：

1. 题目主要价值支柱是什么？是否还有次要支柱？
2. 最能代表**被接受业务结果**的 KPI 是什么，而不是 activity metric？
3. baseline、目标、分群、时间范围和 owner 是否明确？
4. 哪些 quality/safety/security/compliance 条件是硬门槛？
5. 哪个架构最简单地满足约束？是否过度 agentic？
6. 成本是否按 successful outcome 和 TCO 计算？
7. SLA 是否为端到端、分位数指标，并说明负载条件？
8. 如何 pilot、观测、rollback，并形成 improvement loop？

---

## Guide 要求与当前实现细节

Guide 要求掌握的是价值支柱对齐与架构判断，不要求记忆某个 Claude 型号、价格、固定延迟或产品限制。当前 Anthropic 文档可用于理解实现杠杆：

- 官方 evaluation 指南要求先定义 success criteria，再构建评估；criteria 应具体、可衡量，并可覆盖 fidelity、consistency、relevance、privacy、latency 与 price 等维度。
- latency 优化应先建立 baseline，并区分 time to first token 与端到端体验；不应在质量门槛未建立前盲目优化速度。
- prompt caching、batch processing、Usage & Cost API 可以帮助实现特定成本和运营目标，但其字段、支持范围、价格和限制会变化。

### Needs verification

实施前重新核查：模型与 API 能力、模型速度、价格、rate/usage limits、batch 和 caching 的支持范围、Usage & Cost API 字段与权限、区域与数据处理条款。生产 SLA 必须用目标部署环境和代表性负载实测，不能用文档示例或供应商平均值代替。

---

## 核查来源（2026-09-23）

- Anthropic， [Define success criteria and build evaluations](https://platform.claude.com/docs/en/test-and-evaluate/develop-tests)
- Anthropic， [Reducing latency](https://platform.claude.com/docs/en/test-and-evaluate/strengthen-guardrails/reduce-latency)
- Anthropic， [Prompt caching](https://platform.claude.com/docs/en/build-with-claude/prompt-caching)
- Anthropic， [Batch processing](https://platform.claude.com/docs/en/build-with-claude/batch-processing)
- Anthropic， [Usage and Cost API](https://platform.claude.com/docs/en/manage-claude/usage-cost-api)
