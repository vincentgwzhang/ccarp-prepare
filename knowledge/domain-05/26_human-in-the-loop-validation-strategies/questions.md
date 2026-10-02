# 26 · Human-in-the-loop 验证策略：原创练习题

> 以下题目不是官方真题。题型和判断方式参考 Exam Guide 的 scenario-based / architecture decision 风格，只覆盖本知识点。

## Question 1（单选）

一个 Agent 可以替客户接受服务条款并发起付款。哪种 HITL 设计最合适？

### Options

A. 付款完成后随机抽查 1%

B. 执行前向有权用户展示条款、金额、收款方和确切动作，并取得绑定这些参数的明确批准

C. 让 Agent 自己判断用户是否“应该同意”

D. 只记录日志，不中断执行

### Correct Answer

B

### Explanation

接受条款和付款都产生现实后果并需要 affirmative consent，应在执行前批准，且批准要绑定具体 payload。A 和 D 都只能事后发现，无法撤回首次影响；C 不能替用户作出同意。

---

## Question 2（单选）

Reviewer 批准向供应商 A 支付 5,000 欧元。批准后 Agent 将收款方改为 B、金额改为 8,000 欧元。系统应如何处理？

### Options

A. 继续执行，因为“付款”这个动作已经获批

B. 只要模型 confidence 高就执行

C. 使原批准失效，重新校验并对新的收款方和金额发起审批

D. 执行后再通知 reviewer

### Correct Answer

C

### Explanation

批准必须绑定 action、resource、arguments、版本和期限。关键参数变化后继续执行会形成“approve A, execute B”及 TOCTOU 风险。A、B、D 都没有阻止未授权的新动作。

---

## Question 3（多选，选择四项）

高风险 tool call 的 review package 应包含哪些内容？

### Options

A. 用户原始 intent、身份/授权上下文

B. 确切 action、target、arguments 和预期副作用

C. 原始证据、provenance、矛盾/缺失信息与触发 gate 的原因

D. 可逆性、替代方案、policy/risk reason 和升级入口

E. 只显示 Agent 生成的“建议批准”一句话

### Correct Answer

A、B、C、D

### Explanation

A–D 让 reviewer 能独立验证事实、权限和影响，并在证据不足时升级。E 容易造成 automation bias，且无法核对模型摘要是否遗漏或歪曲事实。

---

## Question 4（单选）

某低风险、可逆的分类任务每日有一百万条。团队要求人工逐条审批，导致队列积压并开始绕过流程。最佳改进是什么？

### Options

A. 继续逐条审批并把 review SLO 取消

B. 对确定性规则和低风险正常路径自动处理，异常/灰区转人工，并进行风险分层抽样审计

C. 删除所有人工复核和监控

D. 审批超时后全部自动通过

### Correct Answer

B

### Explanation

B 将有限人工注意力投入真正需要判断的案例，同时用抽样发现漂移和漏报。A 形成不可运营的控制；C 丢失异常处理；D 把容量问题变成安全绕过。

---

## Question 5（多选，选择三项）

哪些措施最能降低 automation bias 和 approval fatigue？

### Options

A. 显示原始证据、关键差异、未知项和反证，而不是只显示模型结论

B. 按风险触发审批，避免所有动作都弹窗

C. 测量 reviewer disagreement、overturn、错误捕获与工作负载，并做质量校准

D. 默认选中“批准”，并以点击速度作为唯一绩效指标

E. 隐藏模型的不确定性，使界面更简洁

### Correct Answer

A、B、C

### Explanation

A 支持独立判断，B 保护稀缺注意力，C 让人类控制本身可评价。D 会鼓励橡皮图章式审批；E 剥夺关键风险信息。

---

## Question 6（单选）

高额退款审批在 15 分钟内无人响应。哪种默认策略最合理？

### Options

A. 自动批准，以满足延迟 SLA

B. 让 Agent 自己假装 reviewer 已批准

C. Fail closed，转更高级队列或取消；若业务允许则降级为只生成草稿

D. 先退款，之后发送审计邮件

### Correct Answer

C

### Explanation

高影响写操作的审批超时不应变成 allow。C 保持安全边界，并给出转人工或无副作用降级路径。A、B、D 都在未授权时产生现实后果。

---

## Question 7（多选，选择三项）

团队要评价 HITL gate 是否有效，应重点监控什么？

### Options

A. 应升级案例的 recall、升级 precision 和严重 false negatives

B. Reviewer agreement、overturn/edit/escalation 及下游 incident/near miss

C. Queue age、time-to-decision、SLO miss、工作负载和 stale approvals

D. 只统计审批按钮点击总数

E. 只统计 Agent 的 token 数

### Correct Answer

A、B、C

### Explanation

A 衡量 routing/gate，B 衡量判断质量和实际结果，C 衡量控制是否可运营。D 无法证明审批正确；E 可用于成本分析，但不足以评价 HITL 有效性。

---

## Question 8（单选）

团队计划把所有 reviewer 的批准/拒绝直接当作无误标签，实时训练分类器。最佳判断是什么？

### Options

A. 合理，因为人工决定天然就是 ground truth

B. 不合理；先评估一致性、偏见与错误，进行抽样 adjudication、隐私处理和版本控制后再用于校准或训练

C. 只要 reviewer 是经理就无需验证

D. 只保留批准案例，拒绝案例没有价值

### Correct Answer

B

### Explanation

人工 reviewer 也会受疲劳、信息不足、偏见和 policy 理解差异影响。B 把人工反馈当作需要质量治理的信号。A、C 将人类判断绝对化；D 会造成选择偏差并丢失重要边界样本。
