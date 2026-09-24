# 18 · LLM 评估指标体系

## Guide 对齐与学习边界

- **Domain**：4 · Evaluation, Testing & Optimization（16%）
- **Exam Guide 明确要求**：`Define evaluation metrics (accuracy, latency, cost, safety, security)`。
- **本课目标**：把“模型效果不错”拆成可计算、可解释、能驱动 go/no-go 与架构决策的多维指标体系。
- **边界**：本课聚焦指标的含义、选择、聚合、阈值与 trade-off。下一知识点才深入 evaluation dataset、数据切分、测试框架和 mixed methodologies。

Anthropic 的官方评估指南强调：先定义具体、可衡量、可实现且与应用相关的 success criteria，再设计 evaluation。大多数真实用例都需要多维评估，而不是一个总分。

---

## 1. 从业务风险推导指标，而不是从指标库反推

一套可用的 metric contract 至少回答：

| 字段 | 要回答的问题 |
|---|---|
| Objective | 要证明什么业务结果或控制什么风险？ |
| Metric | 用什么量测？分子、分母和单位是什么？ |
| Population | 在哪些任务、用户、语言、风险层级上测？ |
| Threshold | pass/fail 门槛是什么？是目标还是硬约束？ |
| Aggregation | 看平均值、分位数、最坏分群，还是事件率？ |
| Measurement | code、human、LLM grader 或生产 telemetry？ |
| Owner | 谁解释失败并决定是否发布？ |

例如“准确率至少 90%”仍然不完整：是 exact match、claim groundedness 还是 task success？在哪类请求上？关键高风险样本是否允许被总体平均值掩盖？

常见结构是：

1. **Primary outcome**：系统最重要的任务结果；
2. **Quality dimensions**：正确性、相关性、完整性、一致性；
3. **Operational metrics**：latency、availability、error rate；
4. **Economic metrics**：cost per successful outcome；
5. **Hard guardrails**：safety、security、privacy、compliance。

不要简单把这些维度平均成一个总分。安全或授权失败往往是 release blocker，即使平均质量很高也不能被抵消。

---

## 2. Accuracy：先定义“正确”是什么

LLM 输出开放、非确定，`accuracy` 不是单一万能指标。应按任务形态选择。

### 2.1 分类与抽取

若有明确 label，可使用：

- **Accuracy**：`正确预测数 / 总样本数`。类别均衡、错误代价接近时直观。
- **Precision**：`TP / (TP + FP)`。关注“系统判为正的结果有多少是真的”。误报代价高时优先。
- **Recall**：`TP / (TP + FN)`。关注“真实正例有多少被找到”。漏报代价高时优先。
- **F1**：precision 与 recall 的调和平均，适合需要平衡二者的场景。

例：安全筛查中，漏掉危险请求和误拦正常用户的代价不同，不能只看 accuracy。类别极不平衡时，一个“永远预测正常”的系统也可能得到很高 accuracy。

### 2.2 开放式生成

摘要、问答、建议和分析往往存在多个合理答案。常见维度包括：

- **Factual correctness**：事实是否正确；
- **Groundedness / faithfulness**：关键 claim 是否由给定 context/source 支持；
- **Relevance**：是否直接回应任务；
- **Completeness / coverage**：是否覆盖必要要点；
- **Instruction adherence**：是否满足格式、范围和约束；
- **Consistency**：相似输入或重复运行是否保持语义一致。

Exact match 适合 label、固定 schema 或确定性答案，却不适合用来否定措辞不同但语义等价的好答案。ROUGE、BLEU、embedding similarity 等 proxy 可规模化，但不能自动等同于事实正确或业务成功。

### 2.3 RAG 与 agentic system

应分层定位质量：

- **Retrieval**：Recall@K、Precision@K、MRR、nDCG 等，检查相关证据是否被检索并排在合适位置；
- **Generation**：grounded claim rate、citation correctness/completeness、answer correctness；
- **Agent**：task completion、正确 tool selection、argument validity、unauthorized action rate、loop/timeout rate；
- **End-to-end**：用户目标是否完成，而不只是某个模型 turn 得分高。

Exam Guide 的 Sample 3 正是分层诊断思维：文档刷新后出现 confident-but-wrong，而 model/latency 未变，应先检查 retrieval/indexing，不要把所有 failure 都归因于模型。

### 2.4 Grader 选择

- **Code-based grader**：exact/string/schema/range 检查；快、稳定、可扩展，但缺少语义细腻度。
- **Human grader**：可处理高风险和模糊判断，但慢、贵且存在 inter-rater disagreement。
- **LLM-as-a-judge**：能按 rubric 评价复杂输出，扩展性较好，但自身会有偏差、位置/措辞敏感性和不稳定性，必须先用人工标注校准。

优先使用“足够可靠且最简单”的 grader。LLM grader 应有清晰 rubric、结构化输出、盲化候选身份，并监控与专家判断的一致性；不能把 judge 分数当作绝对真相。

---

## 3. Latency：测端到端分布，而不是一个平均数

LLM 系统的用户时延可能包含 queue、retrieval、Claude request、tool execution、validation、retry 和 response rendering。

重要指标：

- **TTFT / time to first token**：用户多快看到首个流式输出；
- **Time to first useful result**：首个可采取行动的结果；
- **Total completion latency**：任务完整结束时间；
- **p50 / p95 / p99**：典型体验与 tail latency；
- **Timeout、retry 和 queue wait rate**；
- **Throughput / concurrency**：单位时间可处理的工作量。

平均时延可能被少量极慢请求掩盖；p95 表示 95% 请求不超过该值，但仍要按关键 segment、输入长度、tool path 与负载区分。Streaming 通常改善 perceived latency，不保证总完成时间下降。

Latency 指标必须写明起止点与 workload。例如“p95 < 3s”需要说明是 API round trip、TTFT 还是端到端完成，以及在何种并发和输入分布下测得。

---

## 4. Cost：以成功结果为分母

模型费用只是系统成本的一部分。可记录：

- input/output token、cache write/read 等用量类别；
- model、embedding、reranking、tool/API、storage 与 compute 成本；
- retry、fallback、重复 agent step；
- human review、升级与返工；
- evaluation、observability 与日常运营。

有用的派生指标：

```text
cost_per_request = total_cost / requests

cost_per_successful_task = total_cost / accepted_successful_tasks

tokens_per_successful_task = total_tokens / accepted_successful_tasks
```

第二个通常更能支持架构决策。低价方案若成功率差、tool loop 多或人工复核重，可能具有更高的 `cost per successful task`。

必须按 model、feature、tenant、任务类型和成功/失败路径分解成本，避免总体平均掩盖昂贵长尾。当前价格、计费类别和 Usage & Cost API 字段会变化；考试应掌握测量逻辑，而不是背价格。

---

## 5. Safety：系统是否避免造成有害结果

Safety 关注输出或行为对人、组织和社会造成的伤害，例如 harmful content、错误高风险建议、偏见、不当披露、危险工具动作。

常见指标：

- **Harmful output rate / policy violation rate**；
- **Unsafe compliance rate**：面对有害请求仍照做的比例；
- **Appropriate refusal rate**：应拒绝时正确拒绝；
- **Over-refusal rate**：正常请求被错误拒绝；
- **Severity-weighted harm**：高严重性错误不能被大量轻微成功稀释；
- **Escalation / human-review correctness**。

Safety evaluator 本身也有 false positive/negative，因此需要明确 threat/risk taxonomy、严重程度和人工 adjudication。高风险指标常用硬门槛或“零容忍事件 + 统计上限”，而非与质量分数做简单平均。

Anthropic 官方指南说明，即使伦理和安全等较模糊概念，也应通过明确事件定义、试验规模和阈值变得可量测；同时，高风险信息仍需人工验证，因为 mitigation 无法完全消除 hallucination。

---

## 6. Security：系统能否抵抗对抗者并守住权限边界

Safety 与 Security 有交集但不等价：

- **Safety**：结果是否造成伤害，即使没有攻击者；
- **Security**：面对恶意输入、被污染内容或越权尝试时，系统是否保持 confidentiality、integrity 与 authorization boundary。

Security 指标应基于 threat model，例如：

- jailbreak / direct prompt-injection attack success rate；
- indirect prompt-injection attack success rate；
- secret/system-prompt/data exfiltration rate；
- unauthorized tool call / privileged action rate；
- cross-tenant data leakage rate；
- malicious tool result 被执行或采信的比例；
- detection、block、containment 与 recovery rate。

分母必须清楚：`成功攻击数 / 有效攻击尝试数`。只报告“拦截 99%”而不说明攻击类别、攻击强度、是否包含 adaptive attacker，没有解释力。

Security evaluation 还应验证纵深防御：输入筛查失败时，least privilege、sandbox、approval gate 和 output validation 能否限制 blast radius。官方文档区分 direct injection 与来自网页、邮件、文件、tool result 的 indirect injection，并建议持续监控与 red-team；因此只测普通用户输入不够。

---

## 7. 聚合、分群与统计陷阱

### Macro / micro / weighted

- **Micro average**：汇总所有样本后计算，容易被高频类别支配；
- **Macro average**：各类别等权，能暴露少数类表现；
- **Weighted average**：按业务流量或风险权重聚合。

选择取决于风险。如果罕见类别代表高价值或高伤害事件，只看 micro average 会产生误导。

### 平均分不能代替 failure distribution

至少同时看：

- overall 与关键 slices（语言、客户层、任务难度、输入长度）；
- average 与 p95/p99 / worst-case；
- pass rate 与 failure taxonomy；
- point estimate 与样本量、置信区间或不确定性；
- aggregate trend 与具体严重 incident。

LLM 输出有随机性时，应考虑重复运行和方差。两个方案只差 0.2 个百分点，但测试集很小，未必构成可信改进。

### Threshold、target 与 guardrail

- **Threshold**：最低发布门槛；
- **Target**：希望优化到的目标；
- **Guardrail**：优化其他指标时不能退化的维度。

例如：正确率 ≥ 90% 是门槛；安全严重事件为 0 是 hard guardrail；满足这些条件后，在 cost 与 p95 latency 间选择 Pareto 更优方案。

---

## 8. Offline 与 online 指标的关系

- **Offline eval**：可重复、便于回归和定位；主要回答“在定义好的任务分布上能否满足标准”。
- **Online metrics**：反映真实流量、用户行为、operational load 与 adoption；主要回答“上线后是否真正有效”。

二者不可互相替代。高 offline accuracy 不保证用户采纳；高 thumbs-up 也可能掩盖低频严重安全事件。上线前通过离线门槛，上线后用 telemetry、抽样 review、用户结果和 incident metrics 持续监控。

下一知识点会详细处理如何构造代表性数据集和 mixed-method test framework。

---

## 9. 典型 failure modes

1. **One-metric optimization**：只追 accuracy，忽略安全、成本或 SLA。
2. **Metric–task mismatch**：用 exact match 评价存在多个合理答案的生成任务。
3. **Accuracy paradox**：类别失衡时总体 accuracy 很高，关键少数类几乎全错。
4. **Proxy becomes target**：优化 similarity/ROUGE，却没有改善事实正确性或业务结果。
5. **Average hides tail**：平均 latency 达标，p99 和高价值用户体验失败。
6. **Cost per request only**：忽略失败、重试、agent steps 与人工返工。
7. **Safety = Security**：只做 harmful-content 测试，却没有 injection、exfiltration 和 authorization 测试。
8. **Judge not calibrated**：LLM grader 未与专家标注比对就大规模使用。
9. **No segmentation**：总体分数掩盖语言、租户、风险层级或长输入退化。
10. **Composite score masks blockers**：严重 security failure 被其他高分平均掉。

---

## 10. 考试决策模板

遇到评估指标题，依次判断：

1. 任务的“正确”是 label、事实依据、语义质量、tool action 还是 end-to-end outcome？
2. 哪类错误代价更高：false positive 还是 false negative？
3. 哪些是 primary metric，哪些是 hard guardrail？
4. 是否需要 precision/recall/F1、分群或 severity weighting，而非总体 accuracy？
5. latency 是否端到端、使用 p95/p99 并明确负载？
6. cost 分母是否为 successful/accepted outcome？
7. safety 与 security 是否分别覆盖，并有清楚 threat model？
8. grader 是否适合任务，是否经过可靠性校准？
9. 是否用一个平均或 composite score 掩盖关键失败？

---

## Guide 要求与当前实现细节

Guide 要求掌握 accuracy、latency、cost、safety、security 的指标设计与权衡。当前 Claude 文档中的模型名称、价格、API 字段、grader 示例和产品能力只是实现参考，不是稳定的考试事实。

### Needs verification

实施前重新核查：当前模型与 API 能力、计费/usage 字段、价格、structured output 与 evaluation tooling 支持、rate limits、安全分类器行为及平台差异。所有 threshold、baseline 和 grader reliability 必须在目标业务数据与风险模型上验证，不能直接复制文档示例数值。

---

## 核查来源（2026-09-24）

- Anthropic，[Define success criteria and build evaluations](https://platform.claude.com/docs/en/test-and-evaluate/develop-tests)
- Anthropic，[Reducing latency](https://platform.claude.com/docs/en/test-and-evaluate/strengthen-guardrails/reduce-latency)
- Anthropic，[Reduce hallucinations](https://platform.claude.com/docs/en/test-and-evaluate/strengthen-guardrails/reduce-hallucinations)
- Anthropic，[Mitigate jailbreaks and prompt injections](https://platform.claude.com/docs/en/test-and-evaluate/strengthen-guardrails/mitigate-jailbreaks)
- Anthropic，[Usage and Cost API](https://platform.claude.com/docs/en/manage-claude/usage-cost-api)
