# 练习题：复杂问题的拆解技术

> 以下为原创练习题，并非官方真题。题型与判断方式参考 Exam Guide 的 Sample Questions；每题只围绕当天知识点。

## Question 1（单选）

保险公司要生成理赔决定草稿。流程稳定：抽取材料、核验保单、计算规则、生成解释、合规校验，且每一步都依赖前一步的结构化结果。最合适的拆解是什么？

### Options

A. 固定 sequential workflow，为每步定义 schema 和 gate，只有验证通过才进入下一步

B. 将五步同时并行，并让各步骤自行猜测其他步骤的输出

C. 把所有职责放进一次无结构 prompt，并要求 Claude 自称已核验

D. 创建多个 Agents 对最终决定投票，不读取保单规则

### Correct Answer

**A**

### Explanation

题目中的步骤固定且存在明确数据依赖，适合 prompt chaining / sequential decomposition。结构化 contract 和中间 gate 能阻断错误传播。B 违反依赖关系；C 难以定位或验证失败；D 用投票替代 authoritative policy 和计算规则。

---

## Question 2（单选）

架构团队要审查一个方案的 security、performance、cost 和 compliance。四个方面都基于同一版本的设计文档，彼此可独立分析，但最后必须处理 trade-off。最佳方案是什么？

### Options

A. 按 concern 进行 parallel sectioning，各自按统一 rubric 输出，再由 synthesis step 检查冲突与全局约束

B. 让四个分析步骤修改同一可变报告，最后写入者覆盖前面内容

C. 只做 cost 分析，因为它最容易量化

D. 将报告随机切成四段，每个步骤只读其中一段

### Correct Answer

**A**

### Explanation

四个关注点能够独立处理，因此可以并行；但它们的建议可能相互冲突，需要显式 recomposition。B 会产生竞态和丢失更新；C 缺少覆盖；D 按文本长度切分与问题结构无关，可能让每个 reviewer 缺失必要上下文。

---

## Question 3（多选，选择四项）

一个 subtask 将被交给独立 Claude worker。为了使其可验证、可重试且能被下游消费，contract 最应包含哪四项？

### Options

A. 单一 objective，以及明确 scope 和 exclusions

B. 带版本的 inputs、必要 constraints 与 allowed authority

C. 结构化 output artifact 和必须保留的 provenance

D. Acceptance test、completion criteria 和 failure semantics

E. “尽力而为”作为唯一成功标准

F. 允许 worker 随意改变总体目标以便更快结束

### Correct Answer

**A、B、C、D**

### Explanation

A–D 共同构成可执行工作包：边界清楚、输入可重现、输出可组合、成功与失败可判断。E 不可测量；F 会产生 goal drift，并使局部结果无法保证总体 outcome。

---

## Question 4（单选）

一个 coding Agent 接到“迁移整个大型应用”的目标后，试图一次修改所有模块，经常在 context 耗尽时留下半完成代码，下一轮也不知道哪些功能已通过。最合适的改进是什么？

### Options

A. 建立带依赖和验收步骤的 feature backlog；每轮只交付一个 bounded increment，保存进度并在 clean state 下做端到端测试

B. 增大 prompt 中的鼓励语气，但仍要求一次完成全部迁移

C. 每轮清除所有历史、进度文件和测试结果

D. 看到部分文件已修改后就将整个迁移标记完成

### Correct Answer

**A**

### Explanation

A 将长任务拆为可验证增量，并通过持久状态与测试解决跨 context 连贯性和过早完成。B 没有改变任务结构；C 破坏可恢复性；D 把局部活动误认为全局验收通过。

---

## Question 5（单选）

事故诊断系统开始时不知道根因涉及数据库、网络、部署还是第三方依赖，所需调查步骤取决于实时 telemetry。应选择哪种拆解方式？

### Options

A. 受约束的 dynamic decomposition：根据证据更新 plan，同时限制 tools、预算、停止条件和升级路径

B. 预先固定只检查数据库，忽略所有不匹配的 evidence

C. 无限制地创建调查步骤，直到 token 耗尽

D. 跳过环境证据，仅依据最常见根因生成结论

### Correct Answer

**A**

### Explanation

题干无法预先枚举路径，适合运行时规划，但动态计划仍需 guardrails 和 ground-truth feedback。B 的静态路径过窄；C 缺少控制；D 会把先验概率当成事实，无法可靠诊断当前事故。

---

## Question 6（多选，选择三项）

架构师考虑把一个 work unit 继续拆小。哪些现象最能说明当前拆分仍然过粗？

### Options

A. 同一步同时负责检索、事实判断、写作和不可逆执行

B. 步骤失败时只能从头重跑，无法知道是哪类错误

C. 无法为该步骤定义独立 completion test

D. 该步骤只有一个目标、一个结构化输出且能局部重试

E. 下游能直接消费其 artifact，不需重新解释

### Correct Answer

**A、B、C**

### Explanation

A 表示职责混杂，B 表示缺少 failure isolation，C 表示边界不可验证，都是 under-decomposition 信号。D、E 恰好说明 work unit 的粒度较健康，没有理由仅为增加步骤数量而继续拆分。

---

## Question 7（单选）

团队把一个报告拆成 30 个极小步骤，每一步只改写上一段的一两句话。结果调用次数、延迟和信息丢失都增加，几乎每步仍需要完整原文。首要修正是什么？

### Options

A. 合并没有独立 objective、artifact 或 acceptance test 的步骤，减少无价值 handoff

B. 再增加 30 个步骤，使每次输出更短

C. 删除最终 coverage 和 consistency 检查

D. 让每一步都重新摘要完整历史后再工作

### Correct Answer

**A**

### Explanation

这是 over-decomposition：步骤没有独立价值，却增加 serialization、context duplication 和 lossy handoff。A 恢复合适粒度。B、D 进一步放大问题；C 会让信息丢失和矛盾更难被发现。

---

## Question 8（单选）

系统先生成付款建议，再调用支付 API。业务要求任何实际付款都必须经过确定性 policy check 和财务人员确认。拆解中最关键的设计是什么？

### Options

A. 将 read-only analysis/proposal 与 controlled execution 分离，在执行前设置 policy 和 human approval gate，并使用幂等标识

B. 把建议与付款放进同一个 Agent step，并靠 prompt 要求它谨慎

C. 让多个 Agents 投票后自动付款，因为多数票等同授权

D. 失败时无条件重复整个流程，包括支付调用

### Correct Answer

**A**

### Explanation

A 是 risk-first decomposition：把不确定推理与不可逆副作用隔离，并由系统强制授权；幂等标识降低响应丢失后重复付款的风险。B 只靠 prompt 且扩大 blast radius；C 混淆共识与权限；D 可能重复执行副作用。
