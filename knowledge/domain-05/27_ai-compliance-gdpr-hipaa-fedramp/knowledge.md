# 面向 AI 系统的合规要求：GDPR、HIPAA 与 FedRAMP

> **Exam Guide 对齐**：Domain 5 · Governance, Safety & Risk Management（14%）中的 **“Ensure compliance with regulations (e.g., GDPR, HIPAA, FedRAMP)”**。
>
> **学习目标**：面对 Claude/LLM 方案，能够先判断制度是否适用，再把法律、合同或授权要求落实为架构控制、责任边界和可审计证据。本文用于架构考试准备，不构成法律意见；实际项目应由 privacy、security、legal/compliance 与业务 owner 共同确认。

## 1. 考试真正考什么

这三套制度不是三个可互换的“安全认证”：

| 制度 | 首要问题 | 核心对象 | 典型责任/证据 |
|---|---|---|---|
| **GDPR** | 为什么、以什么依据处理个人数据，个人如何行使权利？ | EU/EEA 数据主体的 personal data 及相关 processing | controller/processor 责任、lawful basis、透明度、数据主体权利、DPIA、处理记录、跨境传输保障 |
| **HIPAA** | 参与者是否是 covered entity / business associate，数据是否为 PHI/ePHI？ | 由受监管实体或其 business associate 创建、接收、维护或传输的 PHI/ePHI | Privacy/Security/Breach Notification Rules、risk analysis、BAA、minimum necessary、保障措施与事件记录 |
| **FedRAMP** | 美国联邦机构是否在其信息系统中使用某个在授权边界内的 cloud service offering？ | 联邦云服务使用场景及其系统/授权边界 | 可复用的 cloud assessment evidence、agency authorization、SSP、责任分配、配置与持续监控 |

考试场景通常不会要求背诵全部条文，而会测试：

1. 是否先做 **applicability 与 scope 判断**，而不是看到医疗、欧盟用户或政府客户就直接下结论；
2. 是否理解 **供应商能力/认证 ≠ 客户整个系统合规**；
3. 是否能把义务落到 LLM 数据流的每个组件；
4. 是否保留足以证明设计和运行有效的 **evidence**；
5. 是否把模型、工具、RAG、日志和人工流程的变化纳入持续治理。

## 2. 一个通用的合规工程方法

### 2.1 从义务到证据的五层链条

不要从“买哪个合规产品”开始，而应建立下面的 traceability：

| 层 | 关键问题 | 例子 |
|---|---|---|
| Requirement | 哪项法律、合同或授权要求适用？ | GDPR data minimization；HIPAA risk analysis；FedRAMP agency-owned control |
| Risk | 在本数据流中会怎样违反？ | prompt 带入无关 PHI；跨境日志未纳入 transfer assessment |
| Control | 哪个 preventive/detective/corrective control 降低风险？ | 字段级过滤、RBAC、retention job、人工审批、incident playbook |
| Owner | 谁实施、运行、复核？ | app team、privacy、CSP、agency、security operations |
| Evidence | 如何证明控制存在且有效？ | data-flow inventory、配置导出、访问审计、测试记录、BAA/合同、SSP/POA&M |

“有 policy”而无运行控制，或“有控制”而无责任人与证据，均不是完整答案。

### 2.2 七步工作流

1. **界定 use case 与参与者**：用户、客户、data subject、controller/processor、covered entity/business associate、CSP、federal agency 分别是谁。
2. **盘点数据流**：输入、system prompt、RAG 文档、embedding/vector index、tool call/result、模型输出、human review、cache、trace、backup、support export 全部画入 data-flow diagram。
3. **分类数据与目的**：personal data、special-category data、PHI/ePHI、federal information、credentials；记录 purpose、lawful/contractual authority、来源与接收方。
4. **确定边界与合同**：DPA/processor terms、BAA、subprocessor chain、跨境机制、FedRAMP 精确 offering/region/interface/authorization boundary。
5. **控制设计**：最少数据、最小权限、隔离、加密、retention/deletion、审计、输出验证、人工控制、事件响应与业务连续性。
6. **验证并形成证据**：测试 deletion、权限穿透、RAG ACL、日志脱敏、incident drill、恢复、配置漂移与数据主体请求流程。
7. **持续治理**：模型、prompt、tool、数据源、subprocessor、region 或 retention 改变时重新评估；把生产事件回流到 risk register 与 regression tests。

## 3. LLM 数据流中容易漏掉的处理点

传统 Web 系统只盯业务数据库远远不够。LLM 方案至少要检查：

- 原始 user prompt 和上传文件；
- prompt template、few-shot 示例和会话 history；
- RAG 原文、chunk、metadata、embedding 和 vector index；
- tool request/result，以及 MCP 或其他 integration 的中间状态；
- 模型生成内容、citation、moderation/guardrail 结果；
- traces、debug logs、evaluation datasets、human-review queues；
- prompt cache、application cache、backup、dead-letter queue；
- vendor support、telemetry、abuse monitoring 与 subprocessors；
- fine-tuning 或其他二次使用（若有）。

同一事实可能在多个派生存储中存在。删除主数据库记录而保留可检索 chunk、含原文的 trace 或 review screenshot，通常不能视为完成端到端删除。

## 4. GDPR：以目的、权利和 accountability 驱动架构

### 4.1 先判断 applicability 和角色

GDPR 保护的是 **personal data**，不仅是姓名。可识别个人的 prompt、客户 ID、位置、在线标识、工单内容、语音转写和与个人相关的推断都可能进入范围。医疗等 special categories 还需要额外条件；不要把所有去掉姓名的数据自动当作 anonymous data。

架构设计前要明确：

- 谁决定处理目的与方式（通常是 **controller**）；
- 谁代表 controller 处理数据（**processor**），以及后续 subprocessors；
- 每个 processing purpose 的 lawful basis；涉及 special-category data 时还要确认相应额外条件；
- 数据来自本人还是第三方、向谁披露、保存多久、是否跨境。

“用户勾选 consent”不是万能答案。consent 只是可能的 lawful basis 之一，且必须满足相应有效性要求；有些处理会依据合同、法律义务或 legitimate interests 等其他基础。具体选择必须由合规/法律团队结合场景判断。

### 4.2 关键义务如何映射到 LLM 架构

| GDPR 关注点 | LLM 架构含义 |
|---|---|
| Purpose limitation / data minimization | 只给 Claude 完成任务必需的字段；先在 application boundary 做确定性过滤，不靠提示词请求模型“忽略敏感数据” |
| Accuracy | 区分 source truth 与模型输出；允许更正源数据和派生索引；高影响决策不得把生成文本当作未经验证的事实 |
| Storage limitation | 为会话、trace、RAG、cache、review artifact 与 backup 分别定义 retention/deletion lifecycle |
| Integrity/confidentiality | RBAC/ABAC、tenant isolation、RAG ACL、secret management、encryption、tool allowlist、审计与事件响应 |
| Transparency | 对处理目的、数据类别、接收方/processor、保存期、自动化使用和权利提供可理解说明 |
| Data-subject rights | 建立 access、rectification、erasure、restriction、portability、objection 的身份核验、定位、执行和留痕流程 |
| Privacy by design/default | 在默认配置中限制收集、可见性、保存和复用；上线前 threat/privacy review，而不是事后补控制 |
| Processor governance | 合同、指令边界、subprocessor、删除/返还、协助权利请求和事件响应；验证实际服务配置与合同一致 |
| DPIA | 对可能造成高风险的处理，在上线前系统评估必要性、比例性、风险及缓解；不是一份一次性模板 |
| International transfers | 标明处理地点和接收方；依据适用机制与风险评估处理第三国传输，不把“region 选择”当作全部答案 |

### 4.3 Automated decision-making 的考试陷阱

GDPR Article 22 针对的是 **solely automated processing** 且产生法律效果或类似重大影响的决定，并非所有自动化或所有 Claude 输出。考试中应先判断这两个条件，再考虑适用例外、告知、可争议性和 safeguards。

如果业务依靠“人工复核”降低风险，人工必须获得足够上下文、权限和时间进行实质判断，并能推翻建议。只让审核员机械点击“批准”，不能自动证明已经排除 solely automated decision 的风险，也不能替代其他 GDPR 义务。

### 4.4 LLM 场景的典型 failure modes

- 只删除用户 profile，没有删除 RAG index、trace 或 evaluation copy；
- 用 production prompt/log 训练或评估另一用途，却没有重新检查 purpose、lawful basis 和通知；
- vector store 只按相似度搜索，没有 tenant/data-subject ACL；
- 把 pseudonymized data 当作彻底匿名，从而取消全部保护；
- 只签 DPA，却不核查当前实际 product、subprocessor、retention 与 transfer path；
- 以模糊“AI involved”提示代替具体、可理解的透明度信息。

## 5. HIPAA：先判断实体与 PHI，再落实风险管理

### 5.1 “医疗数据”不自动等于 HIPAA PHI

HIPAA 的判断依赖 **实体、关系和数据**。HHS 将 covered entity 限定为特定 health plan、health care clearinghouse，以及进行某些电子交易的 health care provider；代表其执行涉及 PHI 的活动或提供相关服务者可能是 business associate。由此：普通 consumer wellness app 的健康数据可能受其他法律约束，但不能仅因“是健康数据”就断言它属于 HIPAA PHI。

反过来，如果 cloud/LLM provider 为 covered entity 或 business associate 创建、接收、维护或传输 ePHI，它即使看不到解密后的数据，也可能仍是 business associate。加密不能替代角色判断和必要的 BAA。

### 5.2 三类要求在架构中的落点

| HIPAA 维度 | LLM 场景要求 |
|---|---|
| Privacy Rule | 限制 uses/disclosures；执行适用的 **minimum necessary**；处理个人权利和授权流程。不要把完整病历默认塞入 prompt |
| Security Rule | 以 risk analysis/risk management 为基础，实施 administrative、physical、technical safeguards，保护 ePHI 的 confidentiality、integrity、availability |
| Breach Notification Rule | 定义 detection、triage、取证、风险评估、通知责任与时间流程；BAA 应规定 security incident/breach reporting |

具体到 Claude/RAG 系统：

- 在调用前按任务字段最小化 ePHI，并隔离 tenant/patient；
- 对应用、RAG、tools 和 review queue 实施唯一身份、least privilege、访问撤销和审计；
- 保护传输与存储，管理 key、secret 和 support access；
- 对来源与输出实施 integrity control，避免生成建议覆盖 authoritative clinical record；
- 记录并监控 access、tool action、policy decision 和异常，但日志本身也必须按 ePHI 保护；
- 设计 availability、backup、disaster recovery 与 emergency-mode operation；
- 在传输 ePHI 前，确认**精确服务/配置**受 BAA 覆盖，并管理 BA/subcontractor 链；
- 对 prompt injection、错误患者上下文、跨租户 retrieval、过度披露和人工 reviewer access 纳入 risk analysis。

### 5.3 HIPAA 常见错误

- “供应商愿意签 BAA，所以系统已合规”：BAA 是必要责任安排之一，不替代客户自身 risk analysis、配置和运行控制；
- “数据加密，供应商就不是 BA”：HHS cloud guidance 明确说明，仅提供 no-view 服务也可能是 BA；
- “全部病历有助于提高准确率”：这会与 minimum necessary、访问控制和风险降低目标冲突；
- “只记录 prompt 便于审计”：未经保护的完整 prompt 日志会新增 ePHI 副本和泄露面；
- “模型输出不是原始病历，因此无风险”：输出可能含 PHI、错误信息或进入正式 record，仍须治理。

## 6. FedRAMP：授权边界、证据复用与 shared responsibility

### 6.1 它不是通用的“政府合规标签”

FedRAMP 关注联邦机构使用 cloud service offering（CSO）时的标准化安全评估、认证/授权证据与持续监控。关键区分是：

- FedRAMP 对 **特定 cloud service offering 和边界**形成可复用的 certification evidence；
- agency authorizing official 仍要为该 agency information system 的**具体使用**接受风险；
- 机构必须考虑所处理信息、所选配置、集成和自身负责的 controls；
- 使用商业版、不同 region、额外 API、RAG/vector database、MCP server 或日志服务，不能默认都在已认证边界内。

因此，最佳答案通常不是“选择 FedRAMP-certified vendor 即完成”，而是验证 exact offering，并在 agency system authorization 中记录继承控制、客户控制、配置与 residual risk。

### 6.2 LLM 架构审查清单

1. **Scope**：当前 model endpoint、API、region、管理面、日志/telemetry、support path 是否在 certification/authorization package 描述的边界内？
2. **System boundary**：RAG ingestion、vector store、MCP/tools、identity provider、human-review system 和 SIEM 哪些属于 agency system，哪些是 external service？
3. **Data categorization**：任务数据的安全类别与影响是否适合该 offering/class，并满足 agency-specific requirements？
4. **Control inheritance**：哪些 control 由 CSP 完成、哪些由 agency 完成、哪些 shared？安全组、IAM、logging、key、data handling 和 incident response 不能留空。
5. **Secure configuration**：关闭未用能力，约束 tools/egress，配置 identity、logging、monitoring、records、privacy 和 incident response。
6. **Evidence**：在 agency SSP/authorization 中描述实际实现；跟踪 agency-owned findings/POA&M，不复制 provider package 冒充自身系统描述。
7. **Continuous monitoring**：消费 ongoing certification/vulnerability data，监控配置漂移和接口变化；重大架构/边界变化触发 change review。

### 6.3 生成式 AI 的特殊变化风险

模型升级、tool 权限扩展、新数据源和新的 agent loop 可能不改变底层云名称，却改变系统行为、数据流和攻击面。FedRAMP evidence reuse 不能替代本系统的：

- change impact analysis；
- regression/security testing；
- data-flow 与 boundary 更新；
- customer-responsibility control 验证；
- incident/continuous-monitoring integration。

## 7. 三套制度如何组合，而不是相互替代

某方案可能同时受多套要求约束。例如美国联邦医疗机构采购云端 clinical assistant：

- HIPAA 回答 PHI、BAA、minimum necessary 和 ePHI safeguards；
- FedRAMP 回答 federal cloud offering、授权边界、control inheritance 与 agency risk acceptance；
- 若处理落入 GDPR 范围的个人数据，GDPR 又增加 lawful basis、权利、transparency、DPIA 和 transfer 等问题。

一份 FedRAMP package 不自动解决 HIPAA Privacy Rule；一份 BAA 不自动解决 FedRAMP agency authorization；GDPR consent 也不替代 Security Rule 或 FedRAMP controls。

可以将系统分为三个平面：

| 平面 | 设计职责 | 典型产物 |
|---|---|---|
| **Data plane** | 数据最小化、隔离、RAG ACL、tool execution、输出/人工复核 | data flow、schema、policy enforcement、test results |
| **Control plane** | identity、keys、配置、retention、region、vendor/subprocessor、change approval | config baseline、access review、DPA/BAA、responsibility matrix |
| **Evidence plane** | 可证明 requirement→control→owner→evidence 的运行状态 | ROPA/DPIA、risk analysis、audit logs、SSP/POA&M、incident/drill records |

## 8. 供应商尽调：问“哪个服务、哪项责任、什么证据”

不要只问“你们是否合规”。至少核查：

- exact product、API、deployment/region 和合同实体；
- provider 在该用例中的角色及数据处理指令；
- prompts、outputs、files、logs、feedback 的保存、删除与二次使用；
- encryption、identity、support access、audit、incident notification；
- subprocessors、跨境传输与数据位置；
- DPA/BAA 是否覆盖当前服务和配置；
- FedRAMP package/status/boundary 是否覆盖实际 interface，以及 customer responsibilities；
- 数据导出、权利请求、termination 后返还/删除与证据；
- model/service 更新如何通知，客户如何验证变更。

即使 vendor 提供强控制，客户仍负责自己的 prompt、RAG corpus、tool permissions、identity、业务审批和下游使用。

## 9. 架构题的答题模板

面对法规场景，按下面顺序排除干扰项：

1. **适用吗？** 识别主体、数据、地域/机构与处理活动；
2. **边界在哪？** 画出完整数据流、服务版本、region、integrations 和派生副本；
3. **谁负责？** 明确 controller/processor、CE/BA、CSP/agency 与 shared controls；
4. **先做什么？** 先 risk/privacy analysis、合同与最小化，再传真实敏感数据；
5. **如何 enforce？** 优先 deterministic access、filter、retention、approval 和 isolation，不把 policy 只写进 prompt；
6. **如何证明？** 选择可复核的配置、测试、日志、合同和持续监控证据；
7. **变更怎么办？** 将 model、tool、data source、vendor 和 boundary 变更纳入重新评估。

## 10. Guide 要求与当前实现细节

### Guide 要求掌握

- 能将 GDPR、HIPAA、FedRAMP 的适用范围和责任映射到 Claude/LLM 架构；
- 能识别 shared-responsibility、数据治理、访问控制、审计、事件响应和持续合规中的缺口；
- 能在场景题中拒绝“买了认证产品即整体合规”“签了合同即完成控制”等过度简化答案。

### 当前实现细节（会变化，不应死记）

- Claude/云平台的 DPA、BAA 可用性、数据处理条款、retention、region、subprocessor 与具体 feature 覆盖；
- 某一 Claude service、cloud marketplace offering 或接口的 FedRAMP certification/authorization 状态和边界；
- FedRAMP 2026 rules、designation/class/package 字段与 agency-specific workflow；
- GDPR/HIPAA 的最新监管解释、成员国/州法叠加要求和当前拟议规则状态。

### Needs verification

实际实施前必须让 legal/privacy/security 团队核查适用法律，并用当前官方合同、trust/compliance material 和 FedRAMP package 验证**精确服务、版本、region、接口、数据流与责任边界**。本文没有断言任何当前 Claude 产品已满足 HIPAA、GDPR 或 FedRAMP，也不以未验证的产品状态决定练习题答案。

## 11. 一页复习

- **GDPR**：personal data + purpose/lawful basis + rights + privacy by design + processor/transfer + accountability。
- **HIPAA**：先判断 CE/BA 与 PHI/ePHI；BAA + risk analysis + minimum necessary + admin/physical/technical safeguards + breach process。
- **FedRAMP**：精确 CSO/boundary + evidence reuse + agency system authorization + control inheritance/customer controls + continuous monitoring。
- 共同原则：完整数据流、least privilege、最少数据、retention/deletion、审计/事件响应、vendor governance、change management、证据闭环。
- 最常见陷阱：把 vendor certification、BAA、DPA、encryption 或“human review”中的任一项，当成整体系统合规的充分条件。

## 资料来源（核查日期：2026-10-03）

- [Regulation (EU) 2016/679 (GDPR) — EUR-Lex 官方文本](https://eur-lex.europa.eu/eli/reg/2016/679/oj/eng)
- [European Commission — Data protection](https://commission.europa.eu/law/law-topic/data-protection_en)
- [HHS — Guidance on HIPAA & Cloud Computing](https://www.hhs.gov/hipaa/for-professionals/special-topics/health-information-technology/cloud-computing/index.html)
- [HHS — The HIPAA Security Rule](https://www.hhs.gov/hipaa/for-professionals/security/index.html)
- [FedRAMP — Using a FedRAMP Certified Cloud Service](https://www.fedramp.gov/2026/agencies/use/)
- [FedRAMP — Continuous Monitoring](https://www.fedramp.gov/2026/authority/m-24-15/monitoring/)

