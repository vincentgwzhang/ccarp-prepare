# 21 · Prompt Failure、Hallucination 与 Model Mismatch 故障诊断

## Guide 对齐与学习边界

- **Domain**：4 · Evaluation, Testing & Optimization（16%）
- **Exam Guide 明确要求**：`Diagnose system issues (prompt failure, hallucinations, model mismatch)`。
- **本课目标**：从 trace 与可控实验出发，区分三类故障并选择最小、可验证的修复，而不是看到错误答案就立刻改 prompt 或换更大模型。
- **前置知识**：#18 的指标、#19 的 evaluation framework、#20 的对照实验为诊断提供复现集和比较方法。
- **边界**：本课讨论 diagnosis；不提前展开 #22 的完整 token/latency/cost 优化，也不把 Domain 5 的所有风险与安全控制并入本课。

核心原则：

> **错误输出是 symptom，不是 root cause。** 先验证系统实际给了模型什么，再用一次只改变一个因素的对照实验定位责任层。

---

## 1. 三个术语的工作定义

### 1.1 Prompt failure

模型原则上有能力完成任务，输入事实也足够，但 instruction contract 没有把期望行为表达清楚，或多个指令彼此冲突，导致系统性偏离。

常见信号：

- 成功标准、角色、输出格式、禁止事项或优先级不清；
- system、developer/application、user、few-shot example 之间矛盾；
- prompt 太抽象，假设模型知道未提供的业务规则；
- 把大量 edge cases 写成脆弱的 if/else，产生意外覆盖；
- tool description、参数、边界或返回值语义含糊；
- 同一类输入稳定地产生相似错误，澄清指令或加入少量 canonical examples 后显著改善。

Prompt failure 不只存在于 system prompt。Tool schema/description、retrieval instruction、output contract 与 agent handoff 都属于广义 prompting/context contract。

### 1.2 Hallucination

模型生成了没有证据支持、与已提供 context 冲突，或把不确定内容当事实表达的 claim。Anthropic 当前文档将其描述为 factual incorrectness 或与给定 context 不一致。

例子：

- 文档没有提到退款期限，回答却自信声称“30 天”；
- 引用不存在的段落、URL、case number 或 tool result；
- 正确证据已在 context 中，却捏造另一数字；
- tool 返回 `not_found`，最终回答却宣称记录存在。

注意：**hallucination 是输出层现象，未必是最终根因标签。** 如果 retriever 把错误版本的政策放进 context，而模型忠实复述，首要根因是 retrieval/data freshness；如果 prompt 要求“必须给一个答案”，它又会放大 hallucination。修复应对准真正责任层。

### 1.3 Model mismatch

选择的模型与任务所需能力、模态、上下文处理、复杂推理、tool use 可靠性或质量门槛不匹配。即使 prompt 清晰、context 正确、工具契约合理，目标模型仍在代表性 eval 上持续达不到要求，而更合适的模型在**相同条件**下显著改善。

Model mismatch 不是“这个模型偶尔错了”，也不是“更大一定更好”。需要用真实 prompts/data 和同一 benchmark 比较 accuracy、quality、edge cases、latency 与 cost，才能证明是否应换模型、路由或升级。

---

## 2. 不要遗漏第四类：System / Context / Tool Failure

Guide 要求区分三类，但考试 scenario 常先给出系统层线索。以下问题不能被草率归为三者之一：

- RAG index 没刷新、embedding/corpus 版本不一致、ACL filter 错误；
- context 被截断、拼接顺序错误、模板变量为空；
- tool schema 或参数映射错误、tool 返回 stale/error，但应用吞掉异常；
- conversation state、memory、cache 或 tenant 数据串线；
- orchestration 跳过必要步骤、重试导致重复副作用；
- API error、timeout、stop condition 或 parser 把完整答案截断。

Exam Guide Sample 3 的关键逻辑正是：文档刷新后突然出现 confident-but-wrong，而 model 与 latency 未变，第一检查点应是 retrieval/indexing，而不是猜测模型权重或 temperature。

因此更完整的责任链是：

```text
Input / data
  → retrieval & context assembly
  → prompt / tool contracts
  → model capability & inference
  → tools / orchestration
  → output parsing / presentation
```

任何一层都可能制造最终“回答错误”的表象。

---

## 3. 先建立可复现证据包

排障前保存失败 trial 的实际输入输出，而不是只看用户截图：

| 证据 | 为什么需要 |
|---|---|
| Request/trace ID、时间、tenant/user segment | 关联跨服务日志并识别分群问题 |
| Model identifier 与 inference configuration | 确认实际调用的版本/路由，而非配置期望值 |
| 完整 system/user messages 与模板变量 | 发现缺值、冲突、escaping、拼接和注入问题 |
| Context 顺序、长度与是否被裁剪/压缩 | 判断证据是否真正进入模型可见输入 |
| Retrieved chunks、score、source/version/ACL | 区分 retrieval miss、staleness 与生成错误 |
| Tool definitions、calls、arguments、results/errors | 判断模型选错工具、契约含糊或执行端失败 |
| Stop reason、parser/schema validation、retry/fallback | 识别截断、解析或路由故障 |
| Final raw output 与 UI 展示结果 | 区分模型输出和后处理损坏 |
| Expected answer / supporting evidence | 让“错在哪里”可判定 |

同时固定 corpus、tools、prompt、model、grader 与 application version。若无法重放原条件，诊断结论只能标为 tentative。

LLM 输出非确定，单次 replay 通过不等于问题不存在。对代表性 failures 运行多次 trials，报告发生率与 slice，而不是 cherry-pick 一次结果。

---

## 4. 分层排障流程

### Step 1：把 symptom 写成可检验 assertion

不要写“Claude 很笨”，而要写：

- factual：回答声称 X，但 source of truth 为 Y；
- groundedness：claim C 没有任何 supplied source 支持；
- instruction following：要求四字段 JSON，但缺少 `risk_level`；
- tool use：应调用 `lookup_policy`，却直接回答；
- capability：在复杂多文档比较 suite 上正确率低于已定义门槛。

### Step 2：先查 deterministic 层

按数据流从外向内确认：

1. 用户请求是否解析正确；
2. 模板变量是否存在、顺序是否正确；
3. retriever 是否返回相关、授权且新鲜的 evidence；
4. 完整 evidence 是否进入实际 request；
5. tool call/result 是否成功且未被吞错；
6. raw model output 是否被 parser/UI 改坏。

只要这些层存在明确故障，就先修它，不能用“改 prompt”掩盖坏数据或坏工具。

### Step 3：构造最小可判别实验

在同一 failure set 上一次只改变一个因素：

- **Oracle-context test**：手工提供已验证、最小且充分的正确 evidence；
- **Prompt test**：保持 model/context 不变，只澄清 instruction、优先级、格式或 canonical examples；
- **Model-swap test**：保持 prompt/context/tools 不变，换候选模型；
- **Tool-contract test**：不换模型，消除工具重叠、重命名参数、加入明确边界；
- **Output-enforcement test**：格式问题使用 schema validation/Structured Outputs，而非把事实错误误叫格式问题。

### Step 4：看整体与关键 slices

诊断必须在代表性 dataset 上复现。一个 hand-picked example 修好，只能证明该例改变，不能证明根因成立。查看 intent、语言、context length、risk tier、tool path 与 corpus version 等 slices。

### Step 5：选择最小修复并防回归

修复后把 failure 变成 regression case，重跑相关 capability、safety/security、latency 与 cost guardrails。局部 prompt patch 可能修一题、破坏另一题；换模型也可能提高质量但突破成本/时延 SLA。

---

## 5. 诊断矩阵：实验结果如何解释

| 观察 | 更可能的根因 | 下一步 |
|---|---|---|
| Oracle context 使原模型恢复；原 retrieval 缺失/陈旧 | Retrieval/context failure | 修 index、query、filter、freshness；增加 retrieval eval |
| 同 model/context 下，明确 prompt 或 canonical example 稳定修复 | Prompt failure | 简化并版本化 prompt；测试冲突与边界 |
| 正确 evidence 已存在，但输出仍增加无来源 claim | Hallucination / grounding failure | 允许不确定、限制外部知识、要求 quotes/citations/claim verification |
| 多种高质量 prompt 均失败；相同输入下合适候选模型稳定达标 | Model mismatch | 在质量、cost、latency guardrails 下换模或路由 |
| Tool result 本身错误或 error 被吞 | Tool/integration failure | 修执行端、错误传播、schema 与 contract test |
| Raw output 正确，parser/UI 丢字段 | Post-processing failure | 修 parser/schema/UI；不是 prompt 或模型问题 |
| 所有模型只在长 context 尾部证据上退化；压缩高信号 context 后恢复 | Context design issue | 重排、检索、压缩；不能只归因于模型大小 |
| 失败仅发生在截断/stop limit | Request/configuration issue | 处理 continuation、预算与 stop condition；实施前核查当前 API 行为 |

这些是基于证据的“更可能”，不是凭单个信号下绝对结论。多个根因可以串联：retrieval 缺失 + prompt 禁止说“不知道”会共同制造 hallucination。

---

## 6. Prompt failure 的识别与修复

Anthropic 的 context engineering 建议在两个极端间取平衡：一端是把复杂业务逻辑硬编码成脆弱 prompt，另一端是过于空泛、假定 shared context。有效 prompt 应清晰、直接，在正确抽象层表达预期行为，并使用少量多样、canonical examples。

### 高价值检查项

1. **任务与完成条件**：做什么、何时算完成；
2. **输入语义**：每个字段代表什么，缺失时如何处理；
3. **约束与优先级**：事实来源、权限、必须/禁止行为冲突时谁优先；
4. **不确定性策略**：何时澄清、拒绝、升级或说不知道；
5. **输出 contract**：人类可读格式或 machine schema；
6. **Examples**：覆盖典型正例、负例与边界，不堆砌全部例外；
7. **Tool contract**：什么时候用、什么时候不用、参数/返回/error 语义。

### 不要用 prompt 修所有问题

- 需要 guaranteed JSON schema 时，应考虑支持该能力的结构化输出/严格 validation，而不是无限追加“必须输出 JSON”；
- source 本身错误时，再好的 prompt 也不能创造真相；
- 任务确实超出选定模型能力时，继续堆 prompt 会增加 token、维护成本与冲突。

---

## 7. Hallucination 的识别与缓解

### 先确定 claim 与 evidence 的关系

将输出拆成 atomic claims，对每条标记：

- **Supported**：由提供的 source/tool result 明确支持；
- **Contradicted**：与 source of truth 冲突；
- **Unsupported**：context 没有证据，模型却断言；
- **Unverifiable**：当前资料不足，不能强行判真伪。

若 claim 是根据错误 chunk 得出的，生成阶段可能是 grounded 的，根因仍是 retrieval。若正确 evidence 已在 context 中却输出相反内容，才更聚焦模型的 grounding/reasoning 行为。

### Anthropic 官方建议的缓解方向

- 明确允许回答 “I don't know” 或 information insufficient；
- 长文档任务先提取 direct quotes，再基于 quotes 分析；
- 要求 claims 带 citation，并在无支持 quote 时撤回；
- 限制只能使用 supplied documents，而非任意外部知识；
- 对关键事实做独立验证和人工/规则 hard check。

多次采样发现不一致可以作为风险信号，但“一致地说错”仍是可能的，所以 consensus 不能替代 source verification。高风险结论必须由权威 source/tool state 验证。

---

## 8. Model mismatch 的证明与决策

### 什么时候才有资格怀疑 model mismatch

至少先满足：

- task contract 清晰，grader 与 expected outcome 可信；
- 正确、充分、授权的 context 可见；
- tool/interface 与 orchestration 正常；
- failure 在代表性 suite 和多 trials 中持续；
- prompt 改进已达到合理程度，而不是明显缺字段或冲突；
- 候选模型在完全相同条件下显著改善关键指标。

### 匹配维度

- task complexity 与推理深度；
- 多轮/长任务一致性；
- 需要的 modalities；
- tool use 与 agentic recovery；
- 目标语言、领域与 edge cases；
- latency、throughput 与 cost SLA；
- safety/security/quality 门槛。

Anthropic 当前 model-selection 文档建议基于特定 use case 创建 benchmark，用实际 prompts/data 比较 accuracy、response quality 与 edge cases，并同时权衡 performance 和 cost。这意味着“换最大模型”不是默认答案。

可选修复包括：使用更合适模型、按难度 routing、advisor/executor、简化任务、decomposition，或把确定性步骤移到 code/tool。选择必须由 evaluation 证明。

---

## 9. 示例：RAG 客服助手为何答错政策

症状：用户问新版退款政策，助手自信回答旧的 30 天，但新版为 14 天。

### 错误做法

直接在 system prompt 加一句“不要 hallucinate”，或未经验证切换更昂贵模型。

### 证据化诊断

1. 查 trace：retriever 返回的确实是旧版 policy chunk；
2. 用新版 14 天条款作为 oracle context 重放，原模型回答正确；
3. 结论：首要根因是 corpus/index freshness，不是 model mismatch；
4. 修复 ingest/version/filter，加入 `policy_version` 和 freshness assertion；
5. 增加 regression：每次政策更新后验证 retrieve → answer → citation；
6. 再加防御：如果证据版本不明，回答 information insufficient 并升级，而不是猜测。

若 retriever 返回正确 14 天条款，模型却持续回答 30 天，则继续做 prompt test 和 model-swap test，才能区分 instruction/grounding failure 与 model mismatch。

---

## 10. 典型 Failure Modes

1. **所有错误都叫 hallucination**：掩盖 stale index、bad tool result 或 parser bug。
2. **先换更大模型**：没有同条件 benchmark，成本上升但根因仍在。
3. **只改 prompt、不看实际 rendered request**：模板变量可能根本没注入。
4. **一次改 prompt、model、retriever**：即使恢复也无法归因。
5. **用单次 replay 下结论**：忽略非确定性和 slice-specific failure。
6. **只看最终文本，不看 trace/outcome**：agent 可能声称成功但 tool state 未改变。
7. **把格式错误等同事实错误**：应分别测试 schema compliance 与 groundedness。
8. **用 model consensus 当真相**：多个输出可一致地错误。
9. **对每个失败追加 prompt rule**：prompt 变成冲突、脆弱的规则堆。
10. **修复后不建 regression case**：同类问题随下一次变更重现。

---

## 11. 考试判断清单

遇到诊断题时，依次问：

1. 什么在何时改变？model、prompt、document、index、tool、config 还是 traffic？
2. 错误是 unsupported claim、指令偏离、能力不足，还是 deterministic pipeline 故障？
3. Source of truth 是什么？模型实际看到了它吗？
4. Trace 是否证明 retrieval/tool/parser 正常？
5. Oracle context、prompt-only 与 model-only 对照各说明什么？
6. 结论是否在代表性 suite、多 trials 和关键 slices 上成立？
7. 修复是否对准最早出错的责任层，并留下 regression test？

优先选择“先查最可能且与最近变化直接相关的系统层，再做最小隔离实验”的选项。没有证据时，不要把偶发错误直接定性为 model mismatch。

---

## 资料来源与核查日期

核查日期：**2026-09-27**。

1. Anthropic Claude Platform Docs, [Reduce hallucinations](https://platform.claude.com/docs/en/test-and-evaluate/strengthen-guardrails/reduce-hallucinations)：hallucination 定义、允许不确定、quotes/citations、claim verification 与外部知识限制。
2. Anthropic Engineering, [Effective context engineering for AI agents](https://www.anthropic.com/engineering/effective-context-engineering-for-ai-agents)：prompt 抽象层、context signal、canonical examples 与 tool ambiguity。
3. Anthropic Claude Platform Docs, [Choosing the right model](https://platform.claude.com/docs/en/about-claude/models/choosing-a-model)：用 use-case-specific benchmark 和真实 prompts/data 比较模型，并权衡质量、edge cases、cost 与 performance。
4. Anthropic Claude Platform Docs, [Increase output consistency](https://platform.claude.com/docs/en/test-and-evaluate/strengthen-guardrails/increase-consistency)：区分一般 prompt 格式约束与严格 schema enforcement。
5. Anthropic Engineering, [Building Effective AI Agents](https://www.anthropic.com/engineering/building-effective-agents)：tool definitions 同样需要 prompt engineering，并应通过 examples 观察错误、迭代接口。

### Guide 要求 vs 当前产品实现

- **Guide 要求**是诊断 prompt failure、hallucination 与 model mismatch，并不要求背诵当前 Claude 产品型号、API 字段或某一模型排行榜。
- 官方文档中的具体功能与 model 名称只说明当前实现；本课的考试结论依赖 trace、对照实验与责任层，而不依赖特定型号。

### Needs verification

实施前需重新核查当前 model identifiers/capabilities、context 与 output limits、Structured Outputs 支持范围、API response/stop fields、tool behavior、pricing、availability 与数据条款。任何 model mismatch 决策都必须在目标环境用实际 prompts、data、tools 与 SLA 重新 benchmark；这些可变事实不用于确定练习题答案。
