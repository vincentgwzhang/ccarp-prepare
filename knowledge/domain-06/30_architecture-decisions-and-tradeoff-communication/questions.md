# 30 · 架构决策与权衡的沟通方法：原创练习题

> 以下题目不是官方真题。题型和判断方式参考 Exam Guide 的 scenario-based / architecture decision 风格，只覆盖本知识点。

## Question 1（单选）

团队决定为内部知识助手采用 RAG。现有“ADR”只有一句：`Use vector database because semantic search is better.` 最重要的改进是什么？

### Options

A. 添加更多产品宣传截图

B. 记录业务 context、权威数据与 ACL 约束、候选检索方案、代表性 eval、latency/cost/security trade-offs、决定后果和复审条件

C. 删除所有被拒绝的方案，以免团队再次讨论

D. 将文档改名为会议纪要即可

### Correct Answer

B

### Explanation

B 让决定可复核、可执行并能在 context 改变时重新判断。原句没有定义“better”的 workload、指标或代价。A 不是证据；C 会丢失 rationale 并导致重复争论；D 不解决内容缺陷，ADR 与会议纪要目的也不同。

---

## Question 2（多选，选择三项）

架构师比较固定 workflow 与开放 agent。哪些内容最能形成诚实的 trade-off communication？

### Options

A. 任务是否可预先编码，以及需要多大动态决策能力

B. 两种方案在代表性任务上的 quality、p95、cost per successful task 与 failure evidence

C. autonomy、工具权限、stop condition、人工审批和 residual risk

D. 只描述 agent 的创新性，不记录额外成本与风险

E. 只采用架构师个人最喜欢的框架，不讨论需求

### Correct Answer

A、B、C

### Explanation

A 说明为什么需要或不需要 agent，B 提供 workload-specific evidence，C 暴露行动边界和风险代价。D 是 benefit-only pitch；E 把个人偏好冒充 decision driver，均无法支持可追溯决定。

---

## Question 3（单选）

一个 weighted decision matrix 显示方案 X 总分最高，但 X 在 red-team 中发生跨租户数据泄露；团队将 security 与 cost、latency、UX 一起求平均。最佳处理是什么？

### Options

A. 接受 X，因为总分最高

B. 提高 UX 权重，让结果更稳定

C. 将跨租户泄露设为 hard constraint / release blocker，先淘汰 X；不能让其他维度抵消该失败

D. 删除 red-team 结果，因为它不是普通流量

### Correct Answer

C

### Explanation

不可接受的 security failure 是约束，不是可用低成本或高 UX 补偿的普通评分项。C 先应用 hard guardrail，再对剩余可行方案进行 trade-off 比较。A、B 都在进行风险“平均洗白”；D 会破坏决策证据。

---

## Question 4（单选）

两个月后，新的 eval 证明原 ADR 的核心假设不成立。团队需要更换模型路由策略。应如何维护 decision history？

### Options

A. 直接编辑旧 Accepted ADR，使它看起来一直支持新策略

B. 删除旧 ADR，避免团队误读

C. 创建新 ADR，记录新证据与决定，将旧 ADR 标记为 Superseded 并双向链接

D. 只在聊天频道宣布，不更新 decision log

### Correct Answer

C

### Explanation

C 保留了当时 context、理由和时间线，同时清楚标出当前有效决定。Accepted ADR 应视为 append-only history。A、B 破坏审计和学习；D 会使决定不可发现，也无法被代码、incident 或后续设计引用。

---

## Question 5（多选，选择三项）

一个面向高管的 Claude 平台 decision brief 应保留哪些信息，即使技术细节被压缩？

### Options

A. 推荐方案与要解决的业务结果

B. 关键 trade-offs、重大风险、核心假设和置信度

C. 需要谁作决定、预算/时间影响及何时复审

D. 每个 SDK 方法签名和全部日志字段

E. 只报告收益，不提负面 consequence，以免产生阻力

### Correct Answer

A、B、C

### Explanation

Executive communication 应压缩实现细节，但不能改变事实基础或隐藏风险。A、B、C 支持知情决策和明确的 accountability。D 更适合工程附录；E 会使 sponsor 无法理解代价与接受 residual risk。

---

## Question 6（单选）

团队要选择 Claude model。有人建议根据厂商通用 benchmark 排名直接采用最强型号。最佳架构沟通方式是什么？

### Options

A. 接受建议，因为更强模型在任何 workload 都必然是最佳架构

B. 用实际 prompts、data、edge cases 和目标负载比较 quality、latency、cost 与安全门槛，并记录 selection/routing policy，而非永久绑定当前型号

C. 只比较每百万 token 的标价

D. 只做一次成功 demo，然后将结果写成事实

### Correct Answer

B

### Explanation

B 将模型选择与具体 requirements 和可复核 evidence 绑定，并为产品演进保留替换空间。A 泛化通用排名；C 忽略 task success、延迟和外围成本；D 的样本不足以支持生产决策。

---

## Question 7（多选，选择三项）

以下哪些情况通常值得创建 ADR？

### Options

A. 决定 agent 是否拥有不可逆写工具及其审批边界

B. 选择会影响 data residency、vendor lock-in 和迁移成本的平台

C. 确定跨团队共享 API contract 与 failure semantics

D. 给局部变量改名

E. 调整一条可随时撤销且无跨组件影响的 debug log 文案

### Correct Answer

A、B、C

### Explanation

A、B、C 都影响结构、关键 quality attributes、接口或难以撤销的风险边界，值得进入 decision log。D、E 是低成本局部实现细节；若所有小改动都写 ADR，重要决定会被噪声淹没。

---

## Question 8（单选）

评审会上，安全团队反对推荐方案，产品团队支持，最终 decider 在阅读证据后接受方案并要求增加控制。最合适的记录方式是什么？

### Options

A. 写“全体一致通过”，避免暴露分歧

B. 记录 decision、decider、security concern、增加的 mitigation、accepted residual risk、owner 和复审触发器

C. 不记录安全意见，因为它没有改变最终结论

D. 推迟所有决定，直到每个人都把该方案视为个人首选

### Correct Answer

B

### Explanation

有效架构治理不要求所有人偏好相同方案，但要求必要意见被听取、风险被知情接受并有 owner。B 保留了 dissent 对控制和复审的影响。A、C 会伪造 decision history；D 把 unanimity 错当作决策质量，可能无限阻塞。
