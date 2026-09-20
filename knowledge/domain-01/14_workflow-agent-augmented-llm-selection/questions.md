# 练习题：Workflow、Agent 与 Augmented LLM 选型

> 以下为原创练习题，并非官方真题。题型与判断方式参考 Exam Guide 的 Sample Questions；每题只围绕当天知识点。

## Question 1（单选）

企业内部问答系统接收员工问题，按其权限检索最新政策文档，然后生成带出处的回答。每次请求路径相同，不需要执行外部动作。最合适的模式是什么？

### Options

A. 使用 retrieval 增强的 Augmented LLM，并在应用层执行 ACL 和出处验证  
B. 使用 autonomous Agent，让模型自行发现可访问的部门  
C. 使用多 Agent 系统，由多个 Agent 投票决定员工权限  
D. 使用长时 Agent session，不设置停止条件

### Correct Answer

**A**

### Explanation

A 满足事实 grounding 与权限要求，同时保持最小复杂度。固定的检索—生成路径不需要 Agent 动态规划。B、C 都把 authorization 交给模型且过度设计；D 既不必要，也引入运行失控风险。

---

## Question 2（单选）

供应商准入流程固定为：字段抽取、必填校验、风险分类、按风险进入两个调查分支，最后由合规人员审批。哪个方案最合理？

### Options

A. Workflow，由代码控制步骤和分支，Claude 处理抽取/分类节点，审批由确定性 gate 控制  
B. Agent，由 Claude 自由增加或跳过合规步骤  
C. 单次 prompt，让 Claude 完成全部步骤并直接批准  
D. 只使用向量数据库，不需要 orchestration

### Correct Answer

**A**

### Explanation

题干的步骤、分支和审批均可预定义，Workflow 提供可预测性、step-level validation 和审计。B 给予与需求不相称的自由；C 将多个高风险职责塞入单次调用并绕过审批；D 只解决检索问题，不能表达业务流程。

---

## Question 3（单选）

团队要修复大型代码库中的疑难缺陷。开始时不知道涉及哪些文件，系统需要搜索代码、修改、运行测试，并依据测试失败继续调整。哪种模式最适合？

### Options

A. 固定单次摘要调用  
B. 受约束 Agent，配备代码/测试工具、sandbox、预算、停止条件和 human review  
C. 只建立固定的“修改一个指定文件”Workflow，无论测试结果如何都结束  
D. 让模型生成代码后直接部署生产，以减少 latency

### Correct Answer

**B**

### Explanation

题目同时具备未知路径、动态工具选择和可验证 environment feedback，符合 Agent 的适用条件。A 无法执行探索闭环；C 的固定路径不能适应未知影响范围；D 缺少测试、隔离和审查，风险不可接受。

---

## Question 4（多选，选择三项）

架构师正在判断是否应将现有 Workflow 升级成 Agent。哪三项最能支持升级？

### Options

A. 实际任务所需步骤无法可靠预先枚举  
B. 工具或环境能返回可验证结果，Agent 可据此修正计划  
C. 评估显示 Agent 在受控预算和权限内显著改善业务 outcome  
D. “Agent”这个名称更容易获得管理层关注  
E. 当前流程只有一次稳定的分类调用

### Correct Answer

**A、B、C**

### Explanation

A 说明固定路径不足，B 提供可靠 feedback loop，C 用数据证明额外复杂度有净收益。D 是 branding 而非 architecture reasoning；E 恰恰说明简单 Augmented LLM 或 Workflow 可能已足够。

---

## Question 5（单选）

一个应用用 Claude 对工单分类，然后代码根据分类结果进入预先定义的退款、技术支持或一般咨询分支。团队成员称它为 Agent，因为“下一步由 Claude 选择”。最准确的判断是什么？

### Options

A. 是 Agent，因为任何模型决策都会使系统成为 Agent  
B. 是 Workflow：模型执行 routing，但允许类别和后续主路径由代码预定义  
C. 是多 Agent，因为有三个分支  
D. 既不是 Workflow 也不是 Augmented LLM，因为它使用了分类

### Correct Answer

**B**

### Explanation

Claude 决定 route label，不代表它控制整个任务轨迹；系统仍在预定义控制图中运行，因此是 Workflow。A 混淆局部语义决策与全局 control ownership；C 把分支误作 Agent；D 忽略分类正是 routing workflow 的常见节点。

---

## Question 6（单选）

某客服 Agent 可以动态查询订单、检索政策并准备退款建议，但真正的退款必须通过资格服务和人工审批。该设计应如何分类与评价？

### Options

A. 它不能算 Agent，因为存在人工审批  
B. 它是 Hybrid：Agent 负责开放式调查，确定性 Workflow 负责高影响动作的 policy gate 和审批  
C. 它是纯 Augmented LLM，因为所有使用工具的系统都属于 Augmented LLM  
D. 它设计错误，因为 Agent 必须拥有完全自治权

### Correct Answer

**B**

### Explanation

B 用控制边界而不是营销标签分类：开放探索由 Agent 完成，资金副作用留给可审计 Workflow。HITL 不会自动取消 agentic behavior，因此 A 错；C 只描述了可能使用的基础 building block，遗漏控制方式；D 错把 autonomy 理解成无限权限。

---

## Question 7（多选，选择三项）

一个团队决定部署长时运行 Agent。哪些三项是其 harness 最关键的控制？

### Options

A. 工具 allowlist、执行时 authorization 与 sandbox/credential isolation  
B. Iteration、time、token/cost budget 和明确停止条件  
C. Durable session、checkpoint/interrupt、tracing 与人工升级  
D. 允许 Agent 读取所有生产凭证，以便减少工具错误  
E. 失败时无限重复同一工具调用

### Correct Answer

**A、B、C**

### Explanation

A 限制权限和 blast radius，B 防止 runaway loop，C 支持恢复、可观测和人工控制。D 违反 least privilege 并暴露 credential；E 会扩大成本、延迟和副作用，不能替代分类重试与停止策略。

---

## Question 8（单选）

一个文档摘要服务目前使用一次 Claude 调用即可达到质量和 SLA。架构师建议改成五步 Agent loop，理由是“Agent 更先进”。最佳回应是什么？

### Options

A. 接受，因为 Agent 在所有任务上都比单次调用准确  
B. 接受，因为更多模型调用必然降低 cost  
C. 保持当前 Augmented LLM，除非评估证明更复杂方案带来足以抵偿 latency、cost 和风险的改进  
D. 改成多 Agent，以进一步体现架构成熟度

### Correct Answer

**C**

### Explanation

C 符合 Anthropic“从最简单可行方案开始、只有在可测量改善时增加复杂度”的原则。A 没有普遍成立的保证；B 通常与更多调用的资源消耗相反；D 进一步扩大无依据的复杂度。
