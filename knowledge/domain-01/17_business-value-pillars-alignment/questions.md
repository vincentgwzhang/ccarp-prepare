# 17 · 业务价值支柱对齐：原创练习题

> 以下题目不是官方真题。题型和判断方式参考 Exam Guide 的 scenario-based / architecture decision 风格，只覆盖本知识点。

## Question 1（单选）

一家客服团队计划用 Claude 起草回复。业务负责人要求“提高效率”，同时强调不能牺牲客户问题解决质量。哪个成功标准最合适？

### Options

A. 每天生成的回复数量提高 50%

B. 平均 token 输出量降低 20%

C. p95 工单解决周期降低 30%，同时首次解决率不下降且重开率不升高

D. 所有工单都允许 Claude 自动关闭

### Correct Answer

C

### Explanation

C 将主要价值支柱（Efficiency）连接到端到端、分位数口径的业务结果，并用首次解决率和重开率防止“更快但更差”。A 是 activity metric，生成更多不代表问题被解决；B 可能影响成本，但不能证明流程效率或客户结果；D 把 autonomy 当成目标，且忽略风险和任务差异。

---

## Question 2（单选）

一家法律服务团队部署了 Claude 文档助手。上线后模型每天生成的文档翻倍，但律师花在核对与返工上的时间也显著增加。哪个指标最能判断 Productivity 是否真正改善？

### Options

A. 每天的模型调用次数

B. 每位律师每周完成并通过质量审查的案件数，以及每案复核/返工时间

C. 模型每分钟生成的 token 数

D. 提示词模板的数量

### Correct Answer

B

### Explanation

Productivity 衡量单位人员或团队产生的有效价值。B 同时观察 accepted outcomes 与为此付出的复核/返工工作，能识别“输出增加但净生产力下降”。A、C、D 只测活动、技术吞吐或资产数量，不能证明业务结果。

---

## Question 3（多选，选择四项）

架构团队比较两个 Claude 方案的单位经济性。为了估算 `cost per successful resolution`，最应该纳入哪四项？

### Options

A. 模型、retrieval 和 tool 调用成本

B. 支撑该方案的基础设施与运营成本

C. Human review、重试与返工成本

D. 成功且被接受的 resolution 数量

E. 公司去年已支付、与两个候选方案均无关的办公室装修费用

F. 只记录每次 API 请求的标价，不记录成功率

### Correct Answer

A、B、C、D

### Explanation

A、B、C 构成方案相关的主要 TCO，D 是把总成本归一化为业务结果所需的分母。E 是与决策无关的 sunk/共同成本；F 忽略失败、重试、人工工作和结果质量，会让“单次调用便宜但整体失败多”的方案看起来虚假地更优。

---

## Question 4（多选，选择三项）

一个面向企业客户的实时助手将签订性能 SLA。哪些设计最能形成可验证且有业务意义的 SLA？

### Options

A. 定义代表性峰值负载下的 p95 端到端时延

B. 定义 availability / error rate，并明确测量窗口

C. 为关键回答定义不可牺牲的最低质量或安全门槛

D. 只使用实验室环境中的平均模型响应时间

E. 把启用 streaming 等同于完整任务一定更快

### Correct Answer

A、B、C

### Explanation

A 捕捉用户真正经历的端到端 tail latency，B 使可靠性承诺可度量，C 防止团队用错误或不安全的快速回答“达成”性能目标。D 忽略 production load、retrieval、tools、validation 与长尾；E 混淆 perceived responsiveness 与完整完成时间，streaming 并不保证后者缩短。

---

## Question 5（单选）

一家保险公司每天夜间处理数十万份无须立即返回的文档摘要，要求在次日工作开始前完成，并希望改善 throughput 与 cost。最合理的初始架构方向是什么？

### Options

A. 为每份文档建立同步交互会话，并保持连接直到完成

B. 使用异步队列/批处理，配合幂等、状态追踪、失败重试和结果回收；用代表性负载验证截止时间

C. 使用多 Agent 自由协商，因为 agent 数量越多吞吐一定越高

D. 取消所有输出验证以减少延迟

### Correct Answer

B

### Explanation

场景没有即时交互 SLA，核心是截止时间内的高吞吐与单位经济性，因此异步 batch/queue 是更匹配的模式；幂等、追踪和恢复保证运营可靠性。A 无谓地占用同步资源；C 把复杂性当成吞吐保证；D 以质量为代价，不构成可接受的成本或性能优化。具体批处理价格、限制和完成时间必须按当前官方文档与目标环境验证。

---

## Question 6（单选）

一家银行希望用 Claude 推出过去无法规模化提供的多语言小企业顾问服务。这项计划的主要价值支柱是 Transformation。哪组衡量方式最合适？

### Options

A. 只测单次响应 token 数

B. 只测原有客服流程的平均处理时间

C. 测新覆盖的客户/语言范围、符合条件用户的采用率、成功完成的咨询目标和风险调整后的业务价值

D. 只要 prototype 能回答一个演示问题就宣布成功

### Correct Answer

C

### Explanation

C 衡量的是新能力是否真正被目标用户采用并创造结果，同时保留风险视角，符合 Transformation。B 更偏已有流程的 Efficiency；A 是技术活动指标；D 没有代表性、adoption 或业务结果，无法支持投资决策。

---

## Question 7（单选）

团队发现方案 X 的 p95 时延最低，但依据性准确率低于业务规定的最低门槛；方案 Y 稍慢但满足准确率、隐私和 SLA。产品经理要求“优先性能”。架构师应如何决策？

### Options

A. 选择 X，因为最低延迟总是最高业务价值

B. 在满足准确率、隐私等硬门槛的候选方案中优化性能，因此选择 Y，并继续寻找不破坏门槛的降延迟方法

C. 把准确率与延迟简单相加，分数较高者胜出

D. 取消准确率门槛，因为它不是性能指标

### Correct Answer

B

### Explanation

价值对齐通常是 constrained optimization：先满足不可牺牲的质量、安全和隐私门槛，再优化主要价值支柱。B 保留业务约束并允许持续改进。A、D 会产生快速但不可接受的结果；C 将量纲和严重性不同的指标压成不透明分数，可能掩盖硬性失败。

---

## Question 8（多选，选择三项）

一个团队准备为内部采购助手开展 pilot，并在 8 周后做 go/no-go 决策。哪些做法最能支持可信的价值判断？

### Options

A. 上线前记录当前流程的周期、质量、成本和分群表现

B. 预先定义目标、guardrails、数据来源与指标 owner

C. 在代表性但风险受控的流量上测量 cost per accepted outcome、adoption、质量和端到端 SLA

D. 只挑最简单的十个成功案例制作演示

E. 因为项目已经投入开发成本，所以无论结果如何都扩大部署

### Correct Answer

A、B、C

### Explanation

A 提供可比较 baseline，B 让 go/no-go 条件可审计，C 同时验证经济性、采用、质量和运营表现，并减少只看单一指标的误判。D 存在严重 selection bias；E 是 sunk-cost reasoning，而不是基于价值证据的决策。
