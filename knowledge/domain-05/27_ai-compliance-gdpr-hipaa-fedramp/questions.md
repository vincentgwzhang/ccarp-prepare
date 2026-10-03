# 原创练习题：GDPR、HIPAA 与 FedRAMP 在 LLM 架构中的应用

> 以下均为原创备考题，不是官方真题。题目只覆盖本知识点，并以 Exam Guide 的 scenario-based、architecture decision 与 trade-off 风格设计。

## Question 1 — GDPR 数据最小化

一家欧盟保险公司构建 Claude 理赔摘要助手。现有服务会把完整客户档案、全部历史保单和本次理赔材料一起发送给模型，因为团队认为更多上下文总能提高准确率。测试发现，大多数请求只需本次理赔材料和两项保单字段。架构师下一步的最佳行动是什么？

### Options

A. 保持完整上下文，只在 system prompt 中要求 Claude 不要输出无关个人数据  
B. 在应用边界按已定义任务提取必要字段，并用代表性评估验证摘要质量与漏取风险  
C. 取得一次用户 consent 后，继续发送所有可用数据且永久保存 prompt  
D. 对客户姓名做 hash 后即可发送全部档案，因为数据已经匿名化

### Correct Answer

**B**

### Explanation

**B 正确。** GDPR 的 purpose limitation、data minimization 和 privacy by design 要求架构从任务目的反推必要数据，并以测试验证最小上下文是否仍满足质量与安全要求。过滤应在可信应用边界确定性执行，而不是只依赖模型遵循提示。

**A 错误**：提示词不是数据最小化或访问控制；无关数据已经被处理和披露给下游。**C 错误**：consent 不是取消 purpose、retention 与其他义务的万能授权。**D 错误**：hash/pseudonymization 通常不自动使数据成为不可识别的 anonymous data，仍需结合可重识别性与完整处理判断。

---

## Question 2 — HIPAA applicability

一家独立 consumer wellness app 允许用户自愿记录睡眠和运动数据，并使用 Claude 生成生活方式建议。它不代表 health plan、clearinghouse 或进行相关电子交易的 provider 工作，也没有接收这些实体委托。产品经理声称“只要是健康数据就必须签 HIPAA BAA”。架构师应如何回应？

### Options

A. 正确；所有美国健康相关数据都是 HIPAA PHI  
B. 错误；先判断参与者是否为 covered entity/business associate 以及数据是否在该关系中构成 PHI，同时仍需评估其他适用隐私法律  
C. 错误；consumer app 永远不受任何隐私或安全要求约束  
D. 正确；只要使用 LLM，provider 自动成为 covered entity

### Correct Answer

**B**

### Explanation

**B 正确。** HIPAA 的适用性取决于受监管实体、业务关系和 PHI 定义，而非仅凭数据内容看起来与健康有关。题干未提供 covered entity 或 business associate 关系，因此最佳步骤是正式 applicability analysis；这不意味着其他联邦、州或消费者隐私规则不适用。

**A** 把所有健康数据都泛化为 PHI。**C** 从“不一定适用 HIPAA”错误推导为“没有任何义务”。**D** 混淆技术供应商与 covered entity 的法律角色。

---

## Question 3 — HIPAA cloud LLM（多选，选择 3 项）

一家医院计划把含 ePHI 的护理记录提交给云端 Claude 服务以生成交班摘要。上线前，哪三项行动最重要？

### Options

A. 确认医院与该精确服务/配置的责任关系，并在适用时签署覆盖该服务的 BAA  
B. 对完整端到端数据流执行 HIPAA Security Rule risk analysis，并落实 risk management  
C. 按交班任务限制 ePHI 字段，保护 prompt、RAG、日志、输出和人工复核队列  
D. 只要传输使用 TLS，就可以跳过 BAA 与内部访问控制  
E. 把“不得泄露 PHI”写进 system prompt，即可满足 administrative、physical 和 technical safeguards

### Correct Answer

**A、B、C**

### Explanation

**A、B、C 正确。** BAA/责任安排、组织自身 risk analysis/risk management，以及 minimum-necessary 导向的端到端保障必须组合使用。保护范围包含派生数据和 operational artifacts，而不只是 API 传输。

**D 错误**：encryption in transit 是一项控制，不能替代 BAA、授权、审计等要求；HHS 还明确说明 no-view/encrypted CSP 仍可能是 business associate。**E 错误**：模型提示不构成完整的行政、物理和技术保障。

---

## Question 4 — FedRAMP shared responsibility

美国联邦机构准备使用一个 FedRAMP Certified cloud AI offering。该机构另行连接自建 RAG、第三方 MCP server 和外部日志平台。项目经理认为 provider 的 certification 已覆盖整个解决方案，无需再做 agency authorization。最佳架构判断是什么？

### Options

A. 同意，因为 certification 自动扩展到所有与 offering 相连的系统  
B. 同意，只需在部署文档中写明“使用 FedRAMP provider”  
C. 验证使用的是认证边界内的精确 offering，并在 agency information system authorization 中评估 integrations、配置、数据和 agency-owned/shared controls  
D. 放弃所有 cloud service，因为 FedRAMP 禁止机构使用外部服务

### Correct Answer

**C**

### Explanation

**C 正确。** FedRAMP evidence 可复用，但 agency authorizing official 仍为具体系统使用接受风险。RAG、MCP 和外部日志可能位于不同边界并引入机构负责的 controls，必须进入 system boundary、SSP、配置和持续监控分析。

**A、B** 都犯了 certification inheritance fallacy。**D** 与 FedRAMP 通过标准化证据支持联邦机构安全采用云服务的目的相反。

---

## Question 5 — GDPR erasure 的端到端执行

一个支持 GDPR erasure request 的客服助手，只从主 CRM 删除客户记录。之后该客户的旧工单仍可通过 vector search 检索，debug trace 中也保留了完整 prompt。最合适的修复是什么？

### Options

A. 无需修复；embedding 和 trace 从不属于个人数据  
B. 只在 UI 隐藏搜索结果，后端副本可以永久保留  
C. 建立数据 lineage 和删除编排，按适用的保留/例外规则处理 chunk、index、cache、trace、review/evaluation copy 与 backup，并保留执行证据  
D. 要求 Claude 在输出中不要提到该客户即可

### Correct Answer

**C**

### Explanation

**C 正确。** 权利处理必须覆盖实际 personal-data footprint，并考虑合法保存义务或其他适用例外；架构需要能定位派生副本、执行删除/限制并证明结果。embedding 是否构成个人数据不能脱离可链接性和处理环境一概断言。

**A** 是无依据的绝对结论。**B** 只改变可见性而没有处理后端数据。**D** 把数据治理错误降级为提示词行为。

---

## Question 6 — 合规 evidence plane（多选，选择 3 项）

企业要证明其高风险 LLM workflow 的合规控制不只存在于设计文档中。哪三项最能形成有效的运行证据闭环？

### Options

A. 建立 requirement → risk → control → owner → evidence 的可追踪映射  
B. 对权限隔离、删除、事件响应和恢复流程做周期性测试，并保存结果与 remediation  
C. 监控 model、tool、data source、subprocessor 与配置变更，必要时触发重新评估  
D. 保存一张供应商首页的“enterprise-ready”截图，替代合同和配置审查  
E. 让 Claude 自己声明每次响应“符合所有法规”

### Correct Answer

**A、B、C**

### Explanation

**A、B、C 正确。** 三者分别提供责任可追踪性、控制有效性验证与变更后的持续保证，构成 evidence plane。证据应与实际数据流、配置和责任绑定。

**D** 的 marketing claim 不能证明精确服务、合同或系统配置。**E** 是模型生成的断言，不是独立、可审计的 compliance evidence。

---

## Question 7 — GDPR automated decision

一家银行让 Claude 自动建议是否冻结客户账户。系统把建议交给审核员，但要求每 20 秒处理一单，界面不显示依据，也不允许覆盖建议。团队称“有人点击批准，所以 Article 22 不相关”。最佳判断是什么？

### Options

A. 正确；任何人工点击都必然构成 meaningful human involvement  
B. 应先判断决定是否 solely automated 且产生法律或类似重大影响；当前人工环节缺乏信息、时间和推翻权，不能仅凭点击断言风险已解决  
C. Article 22 禁止一切使用 AI 的银行流程，因此必须立即移除 Claude  
D. 只要模型准确率高于人工，透明度、争议和人工控制都不再需要

### Correct Answer

**B**

### Explanation

**B 正确。** 正确分析先看 Article 22 的适用条件，再看例外与 safeguards。题目中的 reviewer 不能独立评估或改变结果，接近 rubber stamp；架构应提供依据、合理时间、覆盖权限、记录和 contest/escalation path，并由合规团队确认具体义务。

**A** 把形式点击等同于实质参与。**C** 过度扩大 Article 22 的范围。**D** 混淆模型性能与程序性、透明度及权利保障。

---

## Question 8 — 精确产品边界与变更治理

一个团队已完成某云服务版本的 DPA、BAA 和 FedRAMP package 审查。为提升功能，他们准备切换到同厂商的新 region、新 model endpoint，并启用外部 web-search tool。最好的发布决策是什么？

### Options

A. 同厂商产品自动继承所有合同、授权边界和控制，因此直接发布  
B. 只回归测试回答质量；合规状态与 endpoint、region、tool 无关  
C. 将变更视为 data-flow/boundary change，重新核查合同覆盖、数据位置、subprocessor、FedRAMP scope、tool 权限、retention 和责任控制，再按风险完成审批  
D. 只让 Claude 在每个 web request 前请求用户口头同意

### Correct Answer

**C**

### Explanation

**C 正确。** 合规判断绑定精确服务、数据流、region、interface 和责任边界。外部 tool 还可能新增接收方、egress、日志和攻击面；应触发 change impact analysis 与相应审批/验证。

**A、B** 都把 vendor identity 错当成边界不变。**D** 既不能替代合同/授权/安全分析，也未解决用户是否有权同意、数据最小化和下游处理等问题。

