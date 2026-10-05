# 29 · 结构化发现与需求收集框架：原创练习题

> 以下题目不是官方真题。题型和判断方式参考 Exam Guide 的 scenario-based / architecture decision 风格，只覆盖本知识点。

## Question 1（单选）

业务负责人要求团队“本季度上线一个能回答所有员工问题的 Claude agent”。架构师尚未看到真实问答记录、权威数据源或成功标准。最佳第一步是什么？

### Options

A. 立即选择能力最强的模型并开始 prompt engineering

B. 访谈 sponsor 即可，因为 sponsor 最了解所有员工的日常例外

C. 收集代表性真实案例和当前 baseline，识别用户、受影响者、数据 owner 与风险 owner，再定义任务边界和验收标准

D. 先赋予 agent 所有内部系统的读写权限，后续再缩小范围

### Correct Answer

C

### Explanation

C 从 problem、evidence、stakeholders 和 measurable success 开始，能判断是否需要 AI、workflow 还是 agent。A 过早绑定方案；B 会漏掉操作细节、数据与受影响群体；D 在需求未知时扩大权限和 blast radius，违反最小权限原则。

---

## Question 2（多选，选择三项）

团队正在发现一个保险理赔辅助系统的需求。以下哪三项最应该在 task contract 中明确？

### Options

A. 输入来源、权威性、freshness、访问权限与冲突处理

B. 输出内容/schema、证据要求以及不确定时的 abstain / escalation 行为

C. 可调用工具、可执行动作、审批点、幂等与回滚要求

D. 只记录“使用 Claude”，无需说明业务 outcome

E. 将当前某一模型版本写成永久业务目标，即使模型以后下线也不可改变

### Correct Answer

A、B、C

### Explanation

A 定义可信上下文，B 定义可验收输出和失败语义，C 定义 agentic action 的安全边界。D 把实现手段当作需求并遗漏价值；E 将易变产品细节错误固化为业务目标，阻碍后续架构替换。

---

## Question 3（单选）

临床文档助手需要标记可能遗漏的严重药物过敏。漏报可能直接伤害患者，误报主要增加人工复核时间。哪项 requirement 最合理？

### Options

A. 只要求总体 accuracy 高于一个平均门槛，不区分错误类型

B. 优先定义高风险过敏 slice 的 recall / 漏报门槛，并要求低置信或证据冲突时转人工；同时监控误报负担

C. 让模型始终给出确定答案，以减少升级数量

D. 用响应字数代替临床正确性指标

### Correct Answer

B

### Explanation

该场景的 false negative 代价远高于 false positive。B 将 error-cost asymmetry 转化为风险切片、指标和 HITL 行为，同时没有忽略误报造成的运营成本。A 会让多数低风险样本掩盖严重漏报；C 禁止合理 abstention；D 与风险无关。

---

## Question 4（单选）

发现阶段显示：采购审批步骤固定，金额与类别规则明确，任何付款写入都必须经过指定经理确认并留有审计记录。哪种初始架构方向最符合发现结果？

### Options

A. 让完全自主 agent 动态决定审批路径并直接付款

B. 使用确定性 workflow 编排审批和写入，Claude 仅负责适合的抽取、解释或起草步骤

C. 采用多 agent 系统，因为组件越多越容易合规

D. 跳过 architecture assessment，只优化 system prompt

### Correct Answer

B

### Explanation

固定规则、高风险写操作和严格审计需要可预测的代码路径与明确 approval gate。Claude 可增强非结构化环节，但不应替代服务端授权和业务校验。A 扩大不必要的自主性；C 把复杂度当优势；D 无法满足审批与审计要求。

---

## Question 5（多选，选择三项）

一个 RAG 助手将同时接入旧版 Wiki、已批准政策库和员工私有文档。哪些发现问题最关键？

### Options

A. 哪个来源在冲突时具有权威，政策如何版本化和失效

B. 用户身份如何映射到 document / chunk ACL，撤销权限后索引如何同步

C. 每个来源的 coverage、freshness、owner 和删除/保留规则

D. 哪种向量数据库 logo 最符合公司品牌

E. 是否能让模型自己猜测最新政策，避免治理数据源

### Correct Answer

A、B、C

### Explanation

A、B、C 共同定义 data/context contract，分别覆盖权威性、授权以及生命周期。D 与 requirement 无关；E 把数据治理缺口交给概率性生成解决，会造成 confident-but-wrong 和越权泄露。

---

## Question 6（单选）

产品经理写下验收标准：“客服助手应该准确、快速并让用户满意。”架构师最好的改进是什么？

### Options

A. 保留原句，因为自然语言越宽泛越容易通过验收

B. 分别定义任务 population、正确性/groundedness rubric、p95 起止点与负载、用户 outcome、hard safety/security gates、baseline 和 owner

C. 只记录平均模型响应时间

D. 以 demo 中的三个成功例子作为全部验收依据

### Correct Answer

B

### Explanation

B 将形容词变成可测 metric contract，并防止平均值或满意度掩盖严重失败。A 不可验证；C 只覆盖单一组件指标；D 样本过少且缺少边界、失败和代表性场景。

---

## Question 7（单选）

一家银行只访谈了购买该系统的部门主管和直接操作界面的员工。系统给出的建议将影响未直接使用系统的贷款申请人。下一步最佳做法是什么？

### Options

A. 不需行动，因为申请人不是系统用户

B. 将申请人及相关子群作为 indirect / affected stakeholders，调查潜在影响、申诉需求和差异化风险，并纳入需求与评价

C. 只让工程团队推测申请人的关注点

D. 等系统上线并产生投诉后再识别受影响群体

### Correct Answer

B

### Explanation

Stakeholder 不等于 UI user。B 把受系统 operation 影响的人纳入需求、风险和 recourse 设计，也能发现 aggregate metric 掩盖的群体差异。A 和 D 延迟必要治理；C 缺少参与式证据，builder assumptions 不能替代 affected stakeholder input。

---

## Question 8（多选，选择三项）

一个客服 agent 被允许提交退款。为使需求足以支持安全设计，哪三项最重要？

### Options

A. 明确退款资格与金额由服务端以权威交易状态重新校验

B. 定义人工批准触发条件，并把批准绑定到 exact amount、account 和 action payload

C. 要求 idempotency、审计记录以及失败后的 compensation / recovery 行为

D. 允许模型在 tool error 后无限重试，直到出现成功消息

E. 把“agent 表示退款成功”定义为业务 outcome，无需检查支付系统状态

### Correct Answer

A、B、C

### Explanation

A 防止模型绕过确定性业务与授权规则；B 防止批准内容与实际执行漂移；C 控制重复副作用并支持审计和恢复。D 可能导致 loop 和重复退款；E 混淆 transcript 与真实 outcome，必须以外部系统状态为准。
