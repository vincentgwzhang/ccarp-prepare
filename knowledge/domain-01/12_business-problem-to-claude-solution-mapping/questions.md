# 练习题：业务问题 → Claude 解决方案映射

> 以下为原创练习题，并非官方真题。题型与判断方式参考 Exam Guide 的 Sample Questions；每题只围绕本知识点。

## Question 1（单选）

一家保险公司的负责人提出：“竞争对手都在用生成式 AI，我们也要部署 Claude。”架构师需要确定第一个试点。以下哪一步最合理？

### Options

A. 先选择能力最强的 Claude model，再让各团队提交适合该模型的需求  
B. 先把现有所有理赔文档写入一个 prompt，观察 Claude 能做什么  
C. 选择一个有明确用户、baseline、输入输出、错误后果和可量化业务目标的流程，再判断 Claude 适合参与哪一步  
D. 先建立多 Agent 平台，以免未来试点受到架构限制

### Correct Answer

**C**

### Explanation

C 从业务问题和可验收任务出发，能先验证价值、适用性和风险边界。A 与 B 都是 solution-first，模型能力或演示效果不能替代业务映射；D 在需求尚未明确时引入平台和多 Agent 复杂度，违反从最小可行方案开始的原则。

---

## Question 2（单选）

一家电商希望让 Claude 根据订单金额、国家税率和促销规则计算最终退款，并直接更新账本。税率和规则已经在确定性服务中完整实现。最佳设计是什么？

### Options

A. 让 Claude 在 prompt 中完成全部计算，并要求它返回高 confidence  
B. 让 Claude 理解用户诉求并解释结果，由确定性服务计算、校验并提交退款 transaction  
C. 用多个 Claude agent 相互投票决定退款金额  
D. 将温度设为最低后，让 Claude 直接更新账本

### Correct Answer

**B**

### Explanation

B 把模糊语言理解交给 Claude，把精确计算、业务规则和事务提交留给权威的确定性服务。A 的自报 confidence 不是正确性证明；C 增加成本与复杂度，仍不能提供确定性；D 即使降低采样随机性，也不会把概率模型变成账本规则引擎或授权主体。

---

## Question 3（单选）

客服部门希望降低退款工单的平均处理时间。退款会产生真实资金影响，当前规则复杂且会频繁更新。哪个第一阶段方案最合适？

### Options

A. 给 Claude 所有客服工具，让它自行检索、判断并退款  
B. 让 Claude 只使用自身知识回答，不接入订单或政策系统  
C. 让 Claude 检索获授权的当前政策并生成结构化建议；确定性服务判断资格，坐席审批退款  
D. 暂不定义评估指标，先以是否能成功调用退款工具判断试点成功

### Correct Answer

**C**

### Explanation

C 同时利用 Claude 的语言理解能力和权威系统的事实/规则，并按资金风险设置 human approval。A 权限过大且政策判断不可控；B 会把模型记忆误当成 source of truth；D 只证明技术连通性，没有衡量正确性、越权、处理时间或业务价值。

---

## Question 4（多选，选择三项）

团队要评估“用 Claude 自动整理供应商合同”是否值得进入原型阶段。以下哪三项信息对 problem-to-solution mapping 最关键？

### Options

A. 当前人工流程的处理时间、错误率和主要瓶颈  
B. 合同输入类型、期望输出 schema、权威数据源与样例分布  
C. 错误后果、审批责任和哪些字段必须由确定性规则验证  
D. 市场上最流行的 Agent framework 排名  
E. Claude 聊天界面的配色偏好

### Correct Answer

**A、B、C**

### Explanation

A 建立业务 baseline，B 定义可执行和可评估的任务契约，C 决定风险控制与 Claude/确定性代码的职责边界。D 可能在实施阶段影响工程选择，但在问题尚未映射前不是关键证据；E 与本架构判断无关。

---

## Question 5（多选，选择三项）

一个内部知识助手的负责人把成功标准写成“回答听起来专业”。为了形成可验收方案，架构师最应该补充哪三类指标？

### Options

A. 基于代表性问题集的答案正确性、groundedness 和 citation 质量  
B. p95 latency、单位成功任务成本和 fallback rate  
C. 未授权内容泄露、危险建议与越权访问的发生情况  
D. prompt 中形容词的平均数量  
E. 模型每次回答的总字数越多越好

### Correct Answer

**A、B、C**

### Explanation

A 覆盖任务质量，B 覆盖可运营性，C 覆盖安全与权限，三者构成与用途相关的多维 success criteria。D 与目标缺乏因果关系；E 把长度误作质量，长答案可能增加 latency、cost 和认知负担。

---

## Question 6（单选）

一项高吞吐量任务要从不同格式的发票中提取字段供下游 ERP 使用。少数字段缺失或矛盾时必须由财务人员处理。哪个 solution envelope 最合适？

### Options

A. Claude 输出符合 schema 的候选字段；程序校验类型、总额和必填项；异常进入人工队列  
B. Claude 输出自由文本摘要，下游用字符串切割写入 ERP  
C. Claude 自主修改 ERP schema，以适应每一张发票  
D. 只要抽样的十张发票都成功，就直接无人工 fallback 上线

### Correct Answer

**A**

### Explanation

A 将 Claude 用于非结构化理解，同时用 schema、确定性校验和人工异常处理保护下游系统。B 的输出契约脆弱；C 给予与任务不相称的高权限；D 的样本缺乏代表性和 edge cases，且没有处理题干明确存在的异常。

---

## Question 7（单选）

试点中，Claude 的摘要在专家评分上达到目标，但每份摘要的端到端成本高于人工流程，p95 latency 也违反用户 SLA，且没有减少后续处理时间。架构师应如何判断？

### Options

A. 模型质量达标，所以方案已经成功  
B. 只要换成 Agent 架构，成本和 latency 必然下降  
C. 方案尚未满足多维 success criteria，应优化、缩小适用范围或停止，而不能仅凭模型质量上线  
D. 忽略业务指标，因为 CCAR-P 只关注技术准确率

### Correct Answer

**C**

### Explanation

C 正确连接了任务质量、SLA、cost 和业务结果。A 把单一 model metric 当成业务成功；B 没有依据，增加 agentic complexity 往往还会增加调用、latency 和 cost；D 与 Exam Guide 的业务价值、成本和 performance SLA 对齐要求相反。
