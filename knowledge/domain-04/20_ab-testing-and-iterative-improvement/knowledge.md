# 20 · LLM 系统的 A/B 测试与迭代改进

## Guide 对齐与学习边界

- **Domain**：4 · Evaluation, Testing & Optimization（16%）
- **Exam Guide 明确要求**：`Conduct A/B testing and iterative improvements`。
- **本课目标**：设计能回答因果问题的在线对照实验，并把实验结果、生产 trace 与离线 evaluation 连接成可审计的改进闭环。
- **前置知识**：#18 已定义 accuracy、latency、cost、safety、security 指标；#19 已设计 evaluation dataset 与 mixed-method test framework。
- **边界**：本课不展开下一知识点的 prompt failure、hallucination、model mismatch 故障分类；也不把任何特定实验平台的按钮或默认阈值当作 Guide 要求。

核心问题不是“B 的平均分是否比 A 高”，而是：

> 在其他条件可比、风险受控的情况下，把目标用户暴露给 B，是否**导致**预先定义的用户/业务结果产生有实际意义的改善，同时没有突破质量、安全、security、成本与延迟护栏？

---

## 1. A/B testing 在评估体系中的位置

Anthropic 的 agent eval 实践把 automated evals、production monitoring、A/B testing、user feedback、transcript review 与系统化人工评估视为互补证据。A/B testing 的独特价值是用真实流量比较变体，测量真实用户 outcome，并通过随机对照降低混杂因素影响；代价是需要时间和流量、只能测试已部署的变更，而且通常不能单独解释“为什么”。

因此合理顺序通常是：

1. **离线 eval**：在不影响用户的情况下验证已知能力、regression 与 safety/security hard gates；
2. **内部/dogfood 或 shadow**：检查集成、telemetry 与非用户可见行为；
3. **canary**：小比例验证系统稳定性与可回滚性；
4. **随机 A/B**：比较 live outcome，验证因果假设；
5. **逐步 rollout + 持续监控**：确认扩大流量后收益仍成立；
6. **trace review → 新 eval case**：把发现的 failure 固化为离线回归测试。

### Shadow、Canary 与 A/B 不等价

| 方法 | 首要问题 | 是否天然证明因果收益 |
|---|---|---|
| Offline eval | 已知任务、风险与回归是否通过？ | 否，缺少真实使用环境 |
| Shadow | 新系统能否旁路运行并产生可观测结果？ | 否，用户没有真正接受 treatment |
| Canary | 小流量下是否稳定、可回滚？ | 否，若不是随机对照，用户群和时间会混杂 |
| Randomized A/B | Treatment 是否导致 live outcome 改善？ | 在设计与数据质量成立时，是 |

一个 5% rollout 只有在分配随机、A/B 同期运行、曝光可追踪且比较方法预先定义时，才同时是 A/B experiment。

---

## 2. 从“试一个新 prompt”变成可证伪假设

一个可执行的 experiment brief 至少包含：

| 项目 | 示例 |
|---|---|
| Hypothesis | 给检索结果增加来源要求，会提高首次解决率 |
| Control A | 当前生产 prompt + retrieval + tools + harness 快照 |
| Treatment B | 只改变来源要求，其余配置保持一致 |
| Population | 满足条件的客服会话，排除内部测试与已知机器人流量 |
| Randomization unit | 用户或 conversation，而非每个 message |
| Primary metric | 规定时间内无需转人工的已确认解决率 |
| Guardrails | 引用准确性、越权率、投诉率、p95 latency、成功任务成本 |
| MDE / horizon | 值得采取行动的最小效果及预定样本/时长 |
| Stop / rollback | safety/security breach、错误率或 latency 超界即停 |
| Decision rule | ship、reject、继续收集或重设计的条件 |

### 一次实验尽量改变一个可解释因素

若同时更换 model、system prompt、retriever、tool schema 和 UI，即使 B 获胜也无法知道哪个变化产生收益，失败时也难以恢复。可以把整个 bundle 作为一次产品决策来测试，但必须诚实地把结论限定为“这个 bundle 相对 control 的整体效果”，不能归因到其中某一项。

### System under test 不是模型名称

LLM 应用的行为来自完整系统：

`model + prompts + context assembly + retrieval corpus/index + tools + routing + memory + retry/fallback + UI + policy`

A/B 期间应冻结或版本化这些依赖。若共享 corpus、routing policy 或 provider 配置中途改变，必须记录并评估是否破坏可比性。

---

## 3. Randomization unit：避免污染与伪独立

### Request-level randomization 何时危险

对多轮助手逐条 message 随机，会让同一 conversation 在 A/B 之间跳转：B 看到 A 留下的 context，用户体验也混合两个策略。这既是 contamination，也违反“每个观测相互独立”的简单分析假设。

选择 unit 的原则：

- **request**：一次性、无状态且没有 carryover 的独立任务；
- **conversation/session**：多轮行为只在本次会话内延续；
- **user/account**：偏好、memory、历史交互或学习效应会跨会话存在；
- **tenant/team cluster**：同一团队成员共享知识、workflow 或彼此影响，需避免 network spillover。

分配后使用 **sticky assignment**，并在分析时按同一随机化单位处理重复观测。大量 messages 不等于大量独立用户。

### 同期对照优于前后比较

“上周 A、这周 B”会把季节性、工作日结构、营销活动、文档更新和流量构成变化混入 treatment effect。A 与 B 同期运行、对同一 eligible population 随机分配，才能更好地隔离变更影响。

---

## 4. 指标：一个 Primary，加上不可被抵消的 Guardrails

实验前应根据 #18 的 metric hierarchy 声明：

1. **Primary metric**：直接代表假设、决定实验是否成功；
2. **Secondary metrics**：帮助理解机制和权衡，但不应事后挑一个显著结果宣布胜利；
3. **Guardrails**：任何显著改善都不能抵消 safety/security breach 或不可接受的质量、可靠性、cost、latency 退化；
4. **Diagnostic metrics**：tool call、fallback、retrieval hit、trace step 等，用于解释，不等同用户价值。

例如“平均答案长度增加”不是客服价值；“用户确认问题已解决且无需重复联系”更接近 outcome。Thumbs-up 也有自选择偏差和稀疏性，不能作为唯一 primary metric。

### Exposure 要记录真实发生，而不只是 assignment

用户被分到 B，但请求在进入 Claude 前失败或被 fallback 到 A，就没有真实接收 treatment。建议记录：

```text
experiment_id, randomization_unit_id, assigned_variant,
exposed_variant, config_snapshot_id, exposure_time,
request/trace_id, eligibility, fallback_reason,
primary_outcome, guardrail_events
```

- **Assignment log** 支持 intention-to-treat（按最初分配比较，保留随机化收益）；
- **Exposure/fallback log** 支持解释 non-compliance 与实施故障；
- 不应事后只删除“不顺利”的 B 请求，否则会产生 survivor bias。

同时检查 **Sample Ratio Mismatch (SRM)**：若计划 50/50，实际 eligible/assigned 或 exposed 数量出现无法由随机波动解释的偏差，先调查分流、日志、缓存、过滤与客户端兼容问题，不要直接相信效果估计。

---

## 5. 统计判断：显著不等于值得上线

CCAR-P 侧重架构判断，不要求把统计公式背成目的；但必须识别无效实验。

### 实验前

- 估计 baseline 和业务可接受的 **Minimum Detectable Effect (MDE)**；
- 根据随机化单位、方差、预期流量、power 与误报风险规划样本/时长；
- 预先声明 primary metric、关键 slices、停止规则和排除条件；
- 运行足够完整的业务周期，避免只覆盖某一天或短期 novelty effect。

### 实验中

- 监控数据完整性、SRM 与 hard guardrails；
- 不因每小时“偷看”到一次显著就停止。Repeated peeking 会增加 false positive；若确需动态停止，应使用预先设计的 sequential method；
- 不因看见结果后再换 metric、删 segment 或改变 horizon。

### 实验后

- 报告 effect size 与 confidence interval，而非只报 p-value；
- 比较统计不确定性与实际业务意义；
- 查看预先定义的关键 slices，防止总体平均掩盖某语言、tenant 或风险等级的伤害；
- 多指标、多变体和大量临时分群会增加偶然“胜出”，需要收紧决策规则或用后续实验确认。

“没有统计显著差异”不等于“两者等价”。置信区间很宽时，正确结论通常是 **inconclusive**：增加有效样本、降低噪声或重设计，而不是宣布 B 不劣。

---

## 6. LLM/Agent 实验的特殊难点

### 非确定性与长链行为

同一输入可能产生不同输出；一次用户 outcome 又可能由多次 tool calls、多轮对话和人工接管共同决定。需要固定 retry/fallback policy，记录完整 trace，并让 primary outcome 落到用户/任务层，而非只比较单个 response。

### Tool side effects

退款、发信、改权限等动作不可在 shadow 中无隔离重放。使用 sandbox、dry-run、synthetic account 或 effect suppression；真实高风险不可逆动作应有审批和硬边界，不能为了“实验纯度”牺牲安全。

### Adaptive routing 与 fallback 稀释 treatment

若 B 在困难任务上总回退 A，按实际 exposure 只看成功的 B 请求会高估效果；只看 assignment 又可能低估理想实现。主分析通常保留随机化分配，辅以曝光与 fallback 诊断，明确系统整体效果与组件能力不是同一结论。

### Cache、memory 与 corpus drift

Prompt caching、shared memory、RAG corpus/index 更新都可能造成跨 variant 污染或时间变化。缓存键和 memory namespace 应纳入 variant 隔离；共享更新应同步应用并记录时间，或暂停会破坏实验解释的变更。

### Human behavior 会适应系统

用户可能在几天后才学会利用新 UI 或对新 agent 建立/失去信任。短实验容易捕获 novelty，而错过长期 retention、重复联系或投诉。Duration 应匹配假设的影响窗口。

---

## 7. 安全的发布与回滚架构

```text
Offline gates
    ↓ pass
Dogfood / Shadow
    ↓ telemetry valid
Small canary
    ↓ stable + rollback proven
Randomized A/B
    ↓ primary wins + guardrails pass
Gradual rollout
    ↓ continuous monitoring
Full release or rollback
```

执行层应具备：

- server-side feature/experiment flag 与稳定哈希分桶；
- immutable `config_snapshot_id`，将 prompt、model、tools、retrieval 与 policies 绑定；
- 暴露事件、trace 与业务 outcome 可关联，但敏感内容最小化、脱敏并设 retention；
- kill switch、自动/人工 guardrail alert 与已演练 rollback；
- 对有副作用操作使用 idempotency key，避免重试或切换 variant 重复执行；
- 实验 owner、审批、开始/结束时间和 decision log。

不要把安全控制作为 treatment 的可选项：身份验证、授权、审计和不可突破的 policy 应在 A/B 两组均成立。

---

## 8. 从一次胜负到 iterative improvement

可靠闭环是：

1. 从产品目标、离线 failure 或 production trace 提出一个可证伪 hypothesis；
2. 先用离线 regression/safety suite 筛掉明显不安全或退化的 variant；
3. 预注册实验、随机化、验证 instrumentation 和 rollback；
4. 运行 A/B，不因中途结果随意改变规则；
5. 联合分析 primary、guardrails、uncertainty、关键 slices 与 traces；
6. 作出 `ship / reject / inconclusive / redesign` 决策并记录理由；
7. 把新发现的 failure 加入 versioned offline eval；
8. 下一轮只针对新的、可解释的 hypothesis 改进。

### 典型决策

| 结果 | 架构决策 |
|---|---|
| Primary 有实际意义地改善，所有 guardrails 通过 | 逐步 rollout，继续监控 |
| Primary 持平，但 cost/latency 明显改善且质量满足预定义 non-inferiority 条件 | 可考虑上线；不能事后发明“不劣”标准 |
| Primary 改善，但 prompt injection、数据泄露或越权率恶化 | 停止/拒绝；业务收益不能抵消 security hard gate |
| Point estimate 看似更好但区间很宽 | Inconclusive；增加信息或重设计 |
| 总体改善但关键语言/tenant 严重退化 | 不应仅按总体平均全量上线；修复、分层决策或保持 control |

---

## 9. 典型 failure modes

1. **前后比较代替随机对照**：时间趋势被误认为 treatment effect。
2. **按 message 随机多轮对话**：variant 交叉污染，用户体验不一致。
3. **一次改五个组件**：即使赢了也无法解释或安全回滚。
4. **只记录 assignment、不记 exposure/fallback**：无法知道用户实际收到什么。
5. **忽略 SRM**：分流或日志 bug 使结果失效。
6. **反复 peeking，显著即停**：提高 false positive。
7. **事后挑 metric/segment**：从噪声中挑“胜利”。
8. **只看平均值**：掩盖关键 slice 的退化。
9. **A/B 替代安全测试**：让已知危险变体直接接触用户。
10. **只分析完成会话**：失败、退出和 fallback 被排除，形成 survivor bias。
11. **把不显著当等价**：忽略统计能力不足。
12. **无回滚或不可逆副作用**：发现伤害后无法及时止损。

---

## 10. 考试判断清单

遇到 A/B testing scenario，按以下顺序判断：

1. 假设与用户/业务 outcome 是否清晰、可证伪？
2. 变体是否先通过离线 quality、safety、security gates？
3. Randomization unit 是否匹配 state、carryover 和 network effects？
4. A/B 是否同期、sticky，且只改变可解释因素？
5. Primary、guardrails、MDE、horizon、stopping/rollback 是否预先定义？
6. 是否记录 assignment、真实 exposure、fallback、config version 与 outcome？
7. 是否检查 SRM、peeking、multiple testing、污染和关键 slices？
8. 结论是否同时考虑 effect size、uncertainty 与 practical significance？
9. 是否把新 failure 回流到离线 regression suite，形成迭代闭环？

在选项中，优先选择能维持随机对照、保护用户、预先定义决策规则、记录实际曝光并把生产发现固化为 eval 的方案。

---

## 资料来源与核查日期

核查日期：**2026-09-26**。

1. Anthropic Engineering, [Demystifying evals for AI agents](https://www.anthropic.com/engineering/demystifying-evals-for-ai-agents)（2026-01-09）：评估层组合、A/B testing 的价值与限制、生产 feedback loop。
2. Anthropic Claude Platform Docs, [Define success criteria and build evaluations](https://platform.claude.com/docs/en/test-and-evaluate/develop-tests)：specific/measurable success criteria、multidimensional metrics 与 baseline/A-B comparison。
3. Microsoft Learn, [Experiments Best Practices and Recommendations](https://learn.microsoft.com/en-us/gaming/playfab/live-service-management/game-configuration/experiments/experimentation-keys)：作为通用 online experimentation 的补充一手资料，用于 SRM 与实验数据质量；不用于扩大 CCAR-P 范围。

### Guide 要求 vs 当前产品实现

- **Guide 要求**是能执行 A/B testing 并形成 iterative improvement；核心是实验设计、有效性、风险控制与决策逻辑。
- Feature flag、具体实验平台、Claude API 字段或某个统计库只是实现选择，不是 Guide 指定答案。
- Anthropic 的 A/B 描述支持“真实流量 + 实际 outcome + 控制混杂”的原则，但没有为所有业务规定统一流量比例、样本量、显著性阈值或实验时长。

### Needs verification

实施前必须重新核查当前 Claude model/API 名称、schema、feature availability、limits、pricing、数据处理条款，以及所选实验平台的统计方法。样本量、MDE、显著性/非劣阈值和 sequential design 必须结合真实 baseline、风险和实验平台验证，不能使用未经验证的通用默认值；这些可变信息不用于本课练习题的正确答案。
