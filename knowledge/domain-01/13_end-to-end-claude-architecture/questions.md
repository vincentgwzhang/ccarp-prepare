# 练习题：端到端 Claude 架构

> 以下为原创练习题，并非官方真题。题型与判断方式参考 Exam Guide 的 Sample Questions；每题只围绕本知识点。

## Question 1（单选）

团队为内部知识助手画出的生产架构只有 `Web UI → Claude API → Web UI`。员工问题必须依据按部门授权的内部资料回答。哪个改进最先补上关键的端到端缺口？

### Options

A. 把 Claude API 调用复制到三个服务中，提高代码复用率  
B. 在调用前验证身份并按 ACL 检索资料，在输出前验证证据与权限，同时记录可关联的结果和反馈  
C. 增大输出长度，让 Claude 在一次响应中包含更多内容  
D. 允许模型自行决定能访问哪些部门的文档

### Correct Answer

**B**

### Explanation

B 同时补齐 input trust boundary、受权 context、output validation 和 feedback/observability。A 只是复制调用且可能扩大不一致；C 不解决数据权限和真相来源；D 把 authorization 交给模型，违反系统边界和 least privilege。

---

## Question 2（单选）

一个客服应用收到 Claude 的成功 HTTP 响应后，直接把 `content` 展示给用户。线上偶尔出现句子中断，另一些请求实际上是在要求调用订单查询工具。最合理的修复是什么？

### Options

A. 只要 HTTP 是 200，就应把内容视为完整结果  
B. 检查响应的内容类型和完成原因，并分别处理自然结束、tool request、截断、拒绝与 fallback  
C. 对每个响应无条件调用一次订单工具  
D. 将所有异常响应原样写回 prompt，直到返回更长文本

### Correct Answer

**B**

### Explanation

B 正确区分 transport success 与 task completion；不同 stop/completion 状态需要不同控制流。A 会交付截断或未完成结果；C 可能越权且与任务无关；D 缺少停止条件、权限与错误分类，可能形成 runaway loop。

---

## Question 3（单选）

退款服务调用 Claude 生成建议，随后调用支付工具。Claude API 第一次请求超时，应用重试后用户被退款两次。哪个设计最直接防止这类后果？

### Options

A. 对 Claude 和支付工具都进行无限次快速重试  
B. 仅在 prompt 中要求 Claude“绝不要重复退款”  
C. 为业务操作建立 durable task state 和 idempotency key，执行前重新验证订单状态，并只对可安全重试的失败做 bounded retry  
D. 使用更长的 system prompt 描述支付流程

### Correct Answer

**C**

### Explanation

C 在确定性事务边界解决重复副作用，并把模型/API retry 与业务 action retry 分开。A 会放大故障；B、D 都不能给支付 transaction 提供 exactly-once-like protection，模型指令也不能代替数据库状态和幂等控制。

---

## Question 4（多选，选择三项）

法务合同审查任务可能运行数分钟，并在高风险条款处等待律师批准。以下哪三项最适合异步端到端设计？

### Options

A. Durable queue 与可恢复的 task state machine  
B. Task ID、结果存储以及查询或通知机制  
C. Idempotency、timeout/cancel 与人工审批状态  
D. 将单个 HTTP 连接永久保持，且不持久化中间状态  
E. 在进程内变量中保存所有任务，部署时直接丢弃

### Correct Answer

**A、B、C**

### Explanation

A 提供排队与断点恢复，B 让调用者可靠获得结果，C 管理重复执行、生命周期和 human checkpoint。D 把可靠性依赖于长连接；E 无法承受重启或扩缩容，二者都不适合长任务与审批等待。

---

## Question 5（单选）

产品团队希望把每个用户的 thumbs-up/thumbs-down 立即自动写入 production system prompt，使助手“实时学习”。最佳架构决策是什么？

### Options

A. 直接写入；用户反馈天然等于事实正确性  
B. 只采用 thumbs-up，忽略 thumbs-down  
C. 将反馈作为有噪声的信号隔离收集，经脱敏、标注、eval/regression、review 和受控 rollout 后再修改生产版本  
D. 把所有反馈追加到每次请求，且永不删除

### Correct Answer

**C**

### Explanation

C 构成安全的 improvement loop，并防止反馈投毒、偏差和不可回滚变化。A 把满意度误当成正确性；B 引入单向选择偏差；D 会造成隐私、context 膨胀和恶意内容进入指令面的风险。

---

## Question 6（多选，选择三项）

一个合同抽取系统的 Claude 输出已通过 JSON schema validation。哪些检查仍然是必要的？

### Options

A. 关键字段是否由合同原文证据支持  
B. 金额、日期和交叉字段是否满足业务一致性规则  
C. 高风险发现是否需要律师确认后才写入 matter record  
D. JSON 能解析，因此不再需要任何检查  
E. 只检查输出是否足够长

### Correct Answer

**A、B、C**

### Explanation

Schema validation 只证明 syntactic validity。A 检查 semantic grounding，B 检查 business consistency，C 建立与风险匹配的 authority/HITL gate。D 混淆结构正确与事实/业务正确；E 与真实性和授权无直接关系。

---

## Question 7（单选）

一个 coding assistant 在每次修改后运行测试。如果测试失败，它读取失败信息并修复；达到最大迭代次数后转给工程师。测试结果属于哪一种反馈，设计中的最大迭代次数主要解决什么问题？

### Options

A. 属于跨请求 improvement feedback；最大次数用于扩大训练数据  
B. 属于 runtime control feedback；最大次数限制 runaway loop、成本与复合错误  
C. 属于用户满意度反馈；最大次数保证代码一定正确  
D. 不属于反馈，因为测试工具不是 LLM

### Correct Answer

**B**

### Explanation

测试为当前任务提供 environment ground truth，帮助 orchestrator 决定继续还是停止，因此是 runtime control loop。迭代上限控制资源和错误传播，但不能保证成功。A 混淆了运行时闭环与跨版本改进闭环；C 错把停止条件当正确性证明；D 忽略了工具/环境反馈正是端到端 agentic execution 的关键组成。
