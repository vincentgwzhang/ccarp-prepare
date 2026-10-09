# 33 · AI 方案生命周期模型：原创练习题

> 以下题目不是官方真题。题型和判断方式参考 Exam Guide 的 scenario-based / architecture decision 风格，只覆盖本知识点。

## Question 1（单选）

某保险公司要求团队“尽快选一个最强的 Claude model”来自动审批理赔，但尚未定义哪些理赔可自动处理、错误后果、人工角色或成功标准。架构师最合适的第一步是什么？

### Options

A. 先对所有当前 Claude models 做 latency benchmark，再选择最快的

B. 先完成 structured discovery：明确 workflow、intended use / anti-scope、baseline、风险、autonomy、success/guardrail metrics 与 stop criteria

C. 直接构建 fully autonomous prototype，让业务在生产中决定需求

D. 在 system prompt 中要求 Claude “始终做出正确合规的审批”

### Correct Answer

B

### Explanation

B 先确定问题、价值、边界和可验证结果，才能判断是否应使用 LLM、需要何种 autonomy 以及怎样选 model。A 在任务和约束未知时优化了错误的问题；C 把 discovery 与风险验证推给生产用户；D 的文字要求不能替代 deterministic controls、评估和人工责任。

---

## Question 2（单选）

一个 Claude 财务分析 prototype 在 12 个精心挑选的演示案例上表现很好。Sponsor 要求本周向所有客户开放。哪项回应最符合生命周期治理？

### Options

A. 演示已经证明方案有效，可以直接全量上线

B. 只要在界面加入免责声明，就不需要额外测试

C. 先建立可重现的 release candidate，完成代表性/边界/对抗 eval、security/privacy 与 operational readiness，并采用有 rollback trigger 的 staged rollout

D. 永久停留在 prototype，因为 LLM 系统无法上线

### Correct Answer

C

### Explanation

C 承认 prototype 只降低特定不确定性，不能证明 production distribution、风险和运维能力。A 把 curated demo 当成生产证据；B 不能控制数据泄露、错误决策或系统故障；D 又忽略了通过 validation、guardrails 和渐进发布管理风险的可能性。

---

## Question 3（多选，选择三项）

Claude 客服 Agent 即将从项目团队交给 24×7 operations。以下哪三项最能证明 handoff 已真正完成？

### Options

A. Receiving owner 已接受 RACI，并获得 dashboard、runbook、访问权限、training 与 escalation path

B. 交付物绑定 application、model/prompt、tool、RAG/index、policy 与 eval 的版本及 known residual risks

C. Canary、pause/rollback triggers 和 incident drill 已验证，operations 完成 sign-off

D. 架构师发送了 repository 链接，因此默认由架构师永久承担 on-call

E. 项目团队删除所有 failure records，以免影响接手团队信心

### Correct Answer

A、B、C

### Explanation

A、B、C 分别证明责任和能力被接受、交接对象可识别且风险透明、以及生产控制可执行。D 只是传递链接，没有明确接受责任，并制造隐性单点；E 破坏经验、证据和风险透明度。

---

## Question 4（单选）

一个已稳定运行数月的 RAG Assistant 在夜间 index refresh 后开始引用错误租户的内容。团队建议立即改 system prompt，要求“只使用正确租户数据”。最佳生命周期响应是什么？

### Options

A. 接受建议，因为 prompt 修改是最快的安全控制

B. 先 containment 并回滚/隔离问题 index；用 trace 和 version bundle 定位 refresh/ACL failure，将案例加入 regression tests，修复 deterministic filtering 后再 staged rollout

C. 等待更多用户投诉，以确认问题是否真实

D. 只调整 temperature，不调查 index 或 authorization

### Correct Answer

B

### Explanation

这是严重的跨租户 security/privacy incident。B 先限制伤害，再用版本与 trace 归因，把真实 failure 固化为回归测试，并修复确定性授权层。A 的 prompt 不能可靠执行 tenant authorization；C 延迟 containment；D 与根因无关。

---

## Question 5（单选）

团队只改了 tool schema 中一个看似很小的字段名称，认为无需重新评估 Claude Agent。架构师应该如何决定？

### Options

A. 所有变更都必须重新执行组织内每一种测试，无论影响如何

B. 只要代码能编译，tool schema 变化永远不需要 LLM evaluation

C. 根据影响和风险选择复验范围，至少检查 tool selection、schema/semantic validation、authorization、side effects、错误处理以及相关 eval slices

D. 让生产用户自行发现兼容性问题

### Correct Answer

C

### Explanation

C 是 risk-based revalidation：字段变化可能改变模型生成参数、tool handler compatibility 和授权语义，必须验证相关路径；但不要求所有低影响变化机械执行完全相同的流程。A 是无差别 checklist；B 忽略 model-tool contract；D 把可预防风险推给生产。

---

## Question 6（多选，选择三项）

团队准备定义 Claude 医疗行政 Assistant 的 production monitoring。哪三类信号组合最符合完整生命周期？

### Options

A. Availability、error/timeout、end-to-end latency、tool dependency 与 queue health

B. Task success、abstention、groundedness、safety/security failure，并按任务、语言和风险 cohort 分片

C. Cost per successful task、human override/escalation、workflow outcome、投诉或 harm signal

D. 只看总体平均 token 数，因为平均值可以代表全部风险

E. 只记录模型输出，不关联 prompt/tool/index versions 或 trace

### Correct Answer

A、B、C

### Explanation

A、B、C 同时覆盖技术运行、行为与风险、成本和真实业务/人工结果。D 的单一总体平均值会掩盖高风险 slices，也不能衡量成功；E 失去版本和因果上下文，使问题难以复现和回归验证。

---

## Question 7（单选）

一位项目经理把 NIST AI RMF 的 Govern、Map、Measure、Manage 解释为四个一次性、严格线性的阶段：完成 Manage 后不再需要 Map 或 Measure。哪项修正最准确？

### Options

A. 该解释正确，AI 风险只需在发布前评估一次

B. 这些 functions 相互作用、可按情境迭代应用；Governance 跨阶段，生产变化和反馈可触发重新 Map、Measure 与 Manage

C. 只要使用 Claude，就不需要组织治理

D. AI RMF 只适用于基础模型训练，任何应用系统都不适用

### Correct Answer

B

### Explanation

B 符合 lifecycle risk management：阶段有助于组织工作，但不是做完即丢弃的 checklist。新用户、数据、tool、model、威胁或 policy 都可能使旧 assumptions 失效。A 忽视 production drift；C、D 都错误缩小了系统与组织责任。

---

## Question 8（单选）

公司决定退役一个不再创造价值、但仍可调用多个写操作 tools 的旧 Agent。以下哪项是最佳方案？

### Options

A. 停止宣传即可，保留所有 credentials 和 routes 以备未来使用

B. 关闭 dashboard，以减少运营成本，但继续接受请求

C. 执行 decommission plan：停止流量、通知用户和下游、撤销 credentials/permissions、移除 integrations、按 retention policy 处理数据并保存必要审计证据

D. 删除 repository 的 README，其他资源无需处理

### Correct Answer

C

### Explanation

C 关闭访问、权限、数据和组织责任的完整链条，并保留必要证据。A、B 会留下没有 owner 和监控的高权限 orphaned system；D 只删除文档，不会停止运行组件、凭证、数据或下游依赖。

