# 伦理 AI：偏见、公平性与透明度

> **Exam Guide 对齐**：Domain 5 · Governance, Safety & Risk Management（14%）中的 **“Address ethical AI considerations (bias, fairness, transparency)”**。
>
> **学习目标**：能够把伦理原则转化为可设计、可测试、可解释、可申诉、可持续监控的 Claude/LLM 系统要求；能识别“整体准确率高”“删除敏感属性”“加免责声明”等看似合理但不足的答案。

## 1. 伦理 AI 不是宣言，而是 socio-technical architecture

伦理问题不只存在于 foundation model。一个最终系统的行为由以下因素共同决定：

- model 及其训练/对齐形成的行为；
- system prompt、few-shot examples、policy 与 guardrails；
- RAG corpus、chunking、ranking、metadata 和访问控制；
- tools、外部 API、数据库与业务规则；
- UI 如何展示、排序、隐藏和引导；
- 人工 reviewer 的信息、权限、激励和 workload；
- 真实用户群体、语言、地区、无障碍需求与社会环境；
- feedback loop：哪些反馈被收集、谁更容易反馈、反馈如何影响下一版本。

因此，“模型本身通过 bias benchmark”不能证明部署后的招聘、信贷、医疗或客服系统公平。考试中应评价 **end-to-end system in context**。

## 2. 四个概念必须分清

| 概念 | 核心问题 | 典型误区 |
|---|---|---|
| **Bias** | 是否存在系统性的倾向、误差或不对称？来源可能是数据、模型、检索、流程或人 | 所有 bias 都必然有害；或只要模型不读取 protected attribute 就没有 bias |
| **Harmful bias** | 某种偏差是否给个人或群体带来不合理伤害？ | 只测平均 accuracy，不看伤害分布和严重度 |
| **Fairness** | 在此 use case 中，什么结果或过程才算公正？谁承担 trade-off？ | 认为存在一个适用于所有场景的“公平分数” |
| **Transparency** | 向谁披露什么，使其能正确使用、监督、质疑或追责？ | 把透明度等同于公开 raw chain-of-thought 或贴一条“AI 生成”标签 |

还要区分：

- **Explainability/interpretability**：系统或输出为什么这样产生，相关因素和限制是什么；
- **Accountability**：谁对设计、上线、审批、纠正和损害负责；
- **Contestability/recourse**：受影响者如何提出异议、补充信息、请求人工复核或获得纠正。

透明度若没有 owner、review 和 recourse，往往只是在“公开地不可负责”。

## 3. 从伤害开始，而不是从指标开始

### 3.1 两类常见伤害

**Allocative harm** 影响机会、资源或服务分配，例如：

- 某些群体更容易被招聘筛选器淘汰；
- 同等风险的申请人获得不同的信贷建议；
- 某些语言用户更难通过身份核验或获得客服升级。

**Representational harm** 影响群体如何被描述或对待，例如：

- 把职业、能力、犯罪或家庭角色与特定群体刻板关联；
- 使用贬损、物化或排斥性语言；
- 对某些观点一贯提供更浅、更负面或更高拒答率的响应。

此外还应检查：

- **quality-of-service harm**：不同语言、口音、设备或残障用户的成功率差异；
- **procedural harm**：缺乏通知、理由、复核和申诉，即使总体结果看似接近；
- **automation harm**：人把模型建议当作客观事实，导致偏差规模化；
- **feedback-loop harm**：过去的不公平结果被重新作为“真实标签”，强化下一轮系统。

### 3.2 先写 ethical impact statement

对每个 consequential use case，至少回答：

1. 决策对象和受影响群体是谁？谁可能没有声音？
2. 系统是提供信息、推荐、排序，还是直接执行决定？
3. false positive 与 false negative 分别伤害谁、严重度如何？
4. 哪些 protected、vulnerable、language、region 或 accessibility slices 可能表现不同？
5. 什么是该业务语境下的公平目标？谁批准这个规范性选择？
6. 人能否理解系统角色、发现错误、提出异议并获得有效补救？
7. 哪些用途应禁止，哪些必须降级为 decision support 或要求人工决定？

## 4. Bias 从哪里进入 LLM 系统

### 4.1 数据与标签

- 历史数据可能反映过去的歧视，而非理想决策标准；
- 某些群体样本不足，导致估计不稳定；
- human labels 可能把主观偏好包装成 ground truth；
- “是否被录用”“是否获批”等历史 outcome 可能受旧流程影响；
- 公开网络语料可能包含刻板印象和不均衡代表性。

### 4.2 Retrieval 与知识源

- corpus 中某些地区、语言或群体缺少资料；
- ranking 对主流语言和高频表达更有利；
- query rewrite 改变方言或少数群体术语的原意；
- stale/low-quality sources 对特定群体产生不成比例的错误；
- ACL 或数据缺口让不同用户获得不同证据基础。

### 4.3 Prompt、policy 与 safety layer

- few-shot examples 只展示单一文化或家庭结构；
- prompt 把 protected attribute 暗示为风险因素；
- safety classifier 对某些方言、身份词或敏感主题过度拒答；
- 为了“中立”而机械呈现虚假对等，反而掩盖事实证据；
- system prompt 对某类观点提供不同深度、语气或引用标准。

### 4.4 Tools 与业务流程

- downstream API 本身带有偏差；
- tool availability 因地区或语言不同；
- recruiter、医生、客服等 human reviewer 对模型建议产生 automation bias；
- UI 默认排序、颜色和 confidence wording 放大某些建议；
- 申诉渠道复杂、只提供某种语言或需要高数字能力。

### 4.5 生产反馈

只收集“点赞/点踩”的系统会忽略沉默用户、退出用户和无法完成流程者。若高资源群体更容易反馈，后续优化可能持续偏向他们。因此要把 abandonment、escalation、appeal、override 和 complaint 与主动反馈一起分析。

## 5. Fairness 是规范选择，不是单一数学答案

### 5.1 常见公平目标

| 目标 | 直观含义 | 适合思考的问题 |
|---|---|---|
| **Demographic parity** | 各群体获得正向结果的比例接近 | 机会/曝光分配是否失衡？但不考虑真实 base rate 或资格差异 |
| **Equal opportunity** | 实际正类中，各群体 true positive rate 接近 | 合格者是否同等容易得到机会？ |
| **Equalized odds** | 各群体 TPR 与 FPR 都接近 | 漏过与误伤两类错误是否都公平分布？ |
| **Predictive parity** | 获得某预测结果的人中，正确率跨群体接近 | “被标为高风险”在各群体中是否同样可信？ |
| **Calibration** | 相同 score 在各群体中代表近似相同 outcome likelihood | 风险分数是否可用相同含义解释？ |
| **Individual fairness** | 与任务相关特征相似的人得到相似处理 | “相似”如何定义，是否引入代理变量？ |
| **Procedural fairness** | 规则、信息、复核和申诉过程一致且可及 | 决策过程是否尊重当事人并可纠错？ |

这些目标可能彼此冲突，尤其当 base rates 不同时。架构师不应自行选择“最科学”的指标，而应：

1. 从具体伤害和业务/法律要求确定优先目标；
2. 记录选择、放弃的替代指标及 trade-off；
3. 让 domain、legal/ethics、产品和 affected stakeholders 参与；
4. 同时设置 utility、safety 与 fairness guardrails；
5. 定期重审，而非一次签字永久有效。

### 5.2 不要简单删除 protected attributes

“Fairness through unawareness”通常不足：邮编、学校、语言、姓名、职业历史等变量可能成为 proxy；系统还可能从文本推断身份。完全不采集群体信息也会让 disparity 无法测量。

更合理的做法是：

- 在明确目的、授权、隐私保护和访问隔离下，保存审计所需的敏感属性；
- 不让其进入不应使用它的业务决策路径；
- 在受控 evaluation/monitoring 环境中用于 slice analysis；
- 设置严格 retention、least privilege 和审计。

隐私与公平可能存在张力，必须以治理决策解决，而非假装其中一项不存在。

## 6. 如何评价 Claude/LLM 系统的偏见与公平性

### 6.1 评价矩阵

不要只跑一个公开 benchmark。至少按以下维度组合：

| 维度 | 例子 |
|---|---|
| Task | 回答、摘要、分类、排序、推荐、tool action |
| Harm | allocative、representational、quality-of-service、procedural |
| Slice | 性别、年龄、族群、语言、地区、残障，以及 intersectional combinations |
| Metric | task success、error type、refusal、response depth、tone、citation quality、escalation、appeal overturn |
| Context | clear/ambiguous、单轮/多轮、RAG/tool/no-tool、正常/adversarial |
| Evaluator | deterministic rule、domain expert、affected-user research、calibrated human/LLM grader |

### 6.2 配对与 counterfactual tests

对同一场景只改变一个群体属性或身份线索，例如：

- 相同简历只改姓名/代词；
- 相同客服问题改语言或方言；
- 相同政治请求从对立观点提出；
- 相同 medical symptom 改年龄/性别等相关维度。

比较：答案内容、深度、证据、语气、拒答、建议、tool selection 与升级路径。Anthropic 当前公开的 political even-handedness evaluation 也采用成对的对立观点请求，并分别观察回答深度/质量、是否承认其他观点及拒答率；这说明“偏见”不能压缩为一个单值。

但 counterfactual test 也有边界：若变化的属性在任务中确实相关，要求答案完全相同可能反而错误。测试必须由 domain expert 确认哪些差异是合理的。

### 6.3 Slice 与 intersectional analysis

总体 95% accuracy 可能掩盖某小群体 70% 的表现。至少报告：

- 每个 slice 的样本量与置信区间；
- TPR/FPR、拒答、成功率、严重错误率；
- worst-group performance，而不只 macro/micro average；
- 两个以上属性交叉的结果；
- 是否因样本过少而无法可靠结论。

样本不足时正确答案是扩大数据、定性研究或限制部署，不是把不确定性写成“没有发现差异”。

### 6.4 Mixed-method evaluation

伦理风险尤其需要：

- quantitative metrics 发现规模和趋势；
- qualitative review 理解语气、刻板印象、尊严和情境；
- domain experts 判断真实伤害；
- affected users/communities 验证设计者是否漏掉关键影响；
- adversarial/red-team tests 寻找边界案例；
- production complaints、appeals、overrides 与 abandonment 验证实验室结论。

Anthropic 对 AI evaluations 的公开总结也强调：即使是 BBQ 等 bias benchmark，其实现、score 定义和解释都需要大量判断；单一 benchmark 不是“公平证书”。

## 7. 缓解策略：优先修系统，不只修措辞

按照控制力度从强到弱考虑：

1. **重新定义任务或禁止用途**：若伤害高且无法可靠评价，不让模型做最终决定；
2. **限制 autonomy**：让 Claude 生成 evidence-backed recommendation，而非直接分配机会或执行不可逆操作；
3. **改善数据与 retrieval**：补足代表性、修正 labels、更新知识源、平衡语言/地区资料、验证 ranking；
4. **确定性业务约束**：禁止使用不相关属性，统一资格规则，验证输入/输出与 tool parameters；
5. **prompt/policy 改进**：提供多样化 examples、要求基于证据、避免未经支持的群体推断；
6. **human review 与 recourse**：给 reviewer 充分上下文和推翻权，为受影响者提供可访问的申诉；
7. **分层发布**：shadow、pilot、limited cohort、rollback trigger；
8. **持续监控**：group metrics、complaint、override、appeal overturn、drift 和变更影响。

模型升级可能改善某 benchmark，却让另一群体的拒答、语言质量或 tool success 回归。因此每次更换 model、prompt、classifier、RAG corpus 或 tool，都要跑相同的 fairness regression suite。

## 8. Transparency：对不同受众提供可行动信息

### 8.1 四层透明度

| 层级 | 受众 | 应提供的信息 |
|---|---|---|
| **System-level** | 治理者、采购、安全/合规 | intended use、禁止用途、系统边界、数据流、模型/版本、已知限制、评估范围、owner、change history |
| **Operator-level** | 一线员工与 reviewer | AI 的职责边界、证据来源、uncertainty、何时升级、如何覆盖、如何记录 |
| **User/subject-level** | 使用者及受决定影响者 | 正在与 AI 交互或 AI 参与决定、用途、关键数据来源、重要限制、是否有人负责、如何纠错/申诉 |
| **Decision-level** | 具体个案 reviewer/subject | 影响结果的可理解因素、使用的 evidence/provenance、适用规则、缺失信息和 recourse path |

Transparency 的目标是让受众采取正确行动，不是倾倒最大量信息。一个 80 页 model card 对普通申请人可能不够可用；一句“AI assisted”对审计员又远远不够。

### 8.2 Transparency 不等于 raw chain-of-thought

不应把模型内部 chain-of-thought 当作权威解释：它可能不忠实、不稳定，也可能泄露敏感信息或安全机制。应提供：

- 使用了哪些 sources、facts、rules 和 tools；
- 哪些关键信息缺失或不确定；
- 输出是 recommendation 还是 final decision；
- 哪个人或组织负责；
- 怎样更正输入、请求复核或申诉。

可验证的 provenance、decision record 和可操作理由，通常比“展示模型思考过程”更可靠。

### 8.3 Model/system cards 的正确用法

Anthropic 的 system cards 记录模型 capabilities、safety evaluations 和 responsible deployment decisions；Transparency Hub 进一步公开 evaluation methodology、governance、societal impact 等信息。架构师应把它们用作：

- model selection 和 known-limitations 的输入；
- 自己场景评估的起点；
- 采购/风险审查和变更管理证据之一。

它们不能替代企业对自己的 prompt、RAG、tools、users 和业务结果的 end-to-end evaluation。

## 9. 一个可落地的治理闭环

可以借用 NIST AI RMF 的四个动作组织工作：

### Govern

- 定义伦理原则、决策权、risk appetite 和禁止用途；
- 指定 product、model risk、domain、legal/ethics、data 与 operations owners；
- 建立 exception、appeal、incident 和 change-management 流程。

### Map

- 描述 context of use、affected stakeholders 和潜在伤害；
- 画完整 data/model/RAG/tool/human flow；
- 明确公平目标、透明度受众与已知知识空白。

### Measure

- 建立 paired、slice、intersectional、qualitative 和 production tests；
- 记录指标为何适合、哪些风险无法可靠测量；
- 检查 reviewer 行为、explanation usefulness 和 recourse effectiveness。

### Manage

- 按 severity、likelihood、exposure 和 reversibility 排序；
- mitigate、限制部署、接受 residual risk 或停止项目；
- 用 release gate、owner、deadline、rollback trigger 跟踪行动；
- 持续监控真实影响，重新执行 Govern/Map/Measure。

## 10. Trade-offs 与 failure modes

### 10.1 必须显式处理的 trade-offs

- **Fairness vs raw utility**：提高某群体 recall 可能降低整体 accuracy；要以伤害严重度和业务目标决策；
- **Fairness vs privacy**：审计 disparity 需要 group labels，但敏感属性又必须最少化和保护；
- **Transparency vs security/privacy**：披露应足够可行动，但不能泄露个人数据、攻击路径或 secrets；
- **Safety vs access**：过度拒答可能对某些方言、身份词或敏感主题群体造成不均等服务；
- **Consistency vs individual need**：同样处理不一定公平，无障碍 accommodation 可能需要不同交互。

### 10.2 典型 failure modes

- 只报告 aggregate accuracy，不报告 group/worst-group results；
- 只跑一次 benchmark，未测试自己的 RAG、prompt 和 tools；
- 用 LLM-as-judge 评价偏见，却未校准 grader 自身偏差；
- 删除 protected attribute 后宣称公平，却保留大量 proxy；
- 只调整措辞，不改变有偏的分配规则或 source data；
- “有人审核”但 reviewer 没有证据、时间或推翻权；
- 有 AI disclosure，却没有理由、owner、纠错和 appeal；
- 以“样本太少未显著”为“证明公平”；
- 只监控投诉数量，忽略不敢投诉或已退出的用户；
- model/prompt/RAG 变更后沿用旧 fairness sign-off。

## 11. 架构题答题顺序

1. **先界定影响**：是内容建议，还是会影响机会、资源、权利和安全的 consequential decision？
2. **识别 affected groups 与 harm**：谁可能被误伤、漏过、刻板描述或排除？
3. **选择 fairness definition**：根据实际错误成本和规范要求选择，而非默认 demographic parity；
4. **评价完整系统**：paired + slice + intersectional + qualitative + production signals；
5. **设计控制**：数据/RAG、确定性规则、scope、HITL、recourse、release gate；
6. **透明且可追责**：向不同受众提供可行动信息、provenance、owner 与 appeal；
7. **持续治理**：版本化指标、阈值、例外和 residual risk，监控 drift 并复评。

## 12. Guide 要求与当前实现细节

### Guide 要求掌握

- 能识别 Claude/LLM 系统中 bias 的多层来源和不同伤害；
- 理解 fairness 是 context-dependent 的规范选择，并能为指标与 trade-off 提供理由；
- 能设计分群、配对、定性和生产反馈结合的评价；
- 能用 audience-specific transparency、accountability 和 recourse 支持负责任部署。

### 当前实现细节（会变化，不应死记）

- 某个 Claude model/system card 的具体 benchmark、分数或发布日期；
- 当前模型在政治 even-handedness、BBQ、拒答或特定语言上的表现；
- Anthropic Transparency Hub、system-card schema、公开 evaluation suite 的当前字段；
- 任何第三方 fairness tool、classifier 或 LLM grader 的当前能力。

### Needs verification

实际实施前必须核查当前 model/system card、API/service behavior、applicable law 和组织伦理政策；fairness definition、protected slices、threshold、confidence interval、review/appeal SLA 必须由具体 use case、数据和 affected stakeholders 决定。本文不把当前某个 Claude 模型的分数当作考试答案。

## 13. 一页复习

- **Bias** 是系统性倾向；**harmful bias** 关注伤害；**fairness** 决定如何分配错误、机会与程序保障。
- 公平是 end-to-end、context-dependent 的 socio-technical property，不是模型单项属性。
- 从 harm 和 affected groups 开始，再选择 metric；不同 fairness metrics 可能冲突。
- paired/counterfactual tests 看差别待遇，slice/intersectional tests 防止平均数掩盖弱势群体。
- transparency 要 audience-specific、可行动、带 owner/recourse；不等于 raw chain-of-thought。
- model/system cards 是输入，不是部署系统的公平证明。
- 最佳架构闭环：Govern → Map → Measure → Manage → production feedback → 重新评估。

## 资料来源（核查日期：2026-10-04）

- [Anthropic — Challenges in evaluating AI systems](https://www.anthropic.com/news/evaluating-ai-systems)
- [Anthropic — Model system cards](https://www.anthropic.com/system-cards)
- [Anthropic — Transparency Hub](https://www.anthropic.com/transparency)
- [Anthropic — Introducing Anthropic's Transparency Hub](https://www.anthropic.com/news/introducing-anthropic-transparency-hub)
- [NIST AI RMF — AI Risks and Trustworthiness](https://airc.nist.gov/airmf-resources/airmf/3-sec-characteristics/)
- [NIST AI RMF Playbook — Measure](https://airc.nist.gov/airmf-resources/playbook/measure/)
- [NIST AI RMF Playbook — Manage](https://airc.nist.gov/airmf-resources/playbook/manage/)

