# 20 · LLM 系统 A/B 测试与迭代改进：原创练习题

> 以下题目不是官方真题。题型和判断方式参考 Exam Guide 的 scenario-based / architecture decision 风格，只覆盖本知识点。

## Question 1（单选）

一个客服助手准备测试新 system prompt。对话通常有 6–10 轮，且后续消息依赖前文。哪种实验设计最合理？

### Options

A. 每条用户消息都重新随机到 A 或 B，以快速增加样本量

B. 以 conversation 或 user 为随机化单位，保持 sticky assignment；A/B 同期运行且仅改变目标 prompt

C. 本周全部使用 A，下周全部使用 B，直接比较平均解决率

D. 让客服人员自行选择他们认为更好的 variant

### Correct Answer

B

### Explanation

B 避免同一多轮上下文在变体间交叉污染，并用同期随机对照降低用户构成与时间趋势的混杂。A 把相关 messages 误当独立样本且产生 carryover；C 混入季节性、流量和知识库变化；D 产生 selection bias。

---

## Question 2（单选）

团队想直接把一个尚未经过测试的新 tool-using agent 投放 50% 生产流量，用 A/B 观察它是否比旧系统更安全。最佳做法是什么？

### Options

A. 立即 A/B，因为随机化能自动消除安全风险

B. 先通过离线 regression、safety/security 与权限测试，再 dogfood/shadow、canary，确认 telemetry 和 rollback 后才进行受控 A/B

C. 只要 treatment 的平均回答更长，就可以上线

D. 取消 control，把所有用户都切到新 agent，以便更快发现问题

### Correct Answer

B

### Explanation

A/B 用于比较 live outcome，不应替代上线前安全验证。B 先把已知风险挡在离线和小流量阶段，再用随机实验验证实际收益。A、D 会让未经验证的高风险行为直接影响大量用户；C 的回答长度既不是安全指标，也不是业务 outcome。

---

## Question 3（多选，选择四项）

团队要预注册一个“增加 citation 要求是否提高企业搜索助手可信度”的实验。以下哪四项最应在看到结果前确定？

### Options

A. 可证伪 hypothesis 与 control/treatment 配置

B. 一个 primary metric 和不可被抵消的 safety/security、latency、cost guardrails

C. Randomization unit、eligible population、预计时长/样本或 MDE

D. Stopping、rollback、排除条件和最终 decision rule

E. 等结果出现后再从所有 metrics 中选择提升最大者作为 primary

F. 只保留 treatment 成功完成的会话

### Correct Answer

A、B、C、D

### Explanation

A–D 让实验假设、分配、统计能力和风险决策可审计，避免结果驱动的规则变化。E 是事后挑指标，会增加 false discovery；F 造成 survivor bias，并破坏随机对照所代表的整体系统效果。

---

## Question 4（单选）

一个多轮助手按 request 随机分流。用户第一轮收到 A，第二轮收到 B；B 使用了 A 产生的 summary。分析却把每条 message 当独立观测。主要问题是什么？

### Options

A. Variant contamination、carryover，以及对独立样本数的高估

B. 唯一问题是 token cost 较高

C. 这会让随机化比 user-level assignment 更严格，因此没有问题

D. 只要最终答案流畅，就不影响因果结论

### Correct Answer

A

### Explanation

同一会话跨 A/B 会造成 treatment 污染；多条 message 也不是彼此独立的用户样本。应按 conversation/user 等能覆盖状态延续的单位 sticky 分配并分析。B、C、D 都忽略了实验有效性。

---

## Question 5（单选）

实验团队每小时查看一次结果，一旦 `p < 0.05` 就停止并宣布 B 获胜，但没有预先设计 sequential test。最大风险是什么？

### Options

A. Repeated peeking 提高 false-positive 概率，停止规则使名义显著性失真

B. 观察越频繁，随机化一定越强

C. 唯一风险是 dashboard 成本增加

D. 只要最终样本超过 100，任何停止规则都有效

### Correct Answer

A

### Explanation

反复检验并在偶然显著时停止会增加误报。应预先规定固定 horizon，或使用经过设计的 sequential method。B、C、D 都没有解决多次查看造成的统计偏差；不存在适用于所有指标和基线的固定样本魔法数字。

---

## Question 6（单选）

Treatment 将任务完成率从 control 的 70% 提高到 74%，但越权读取内部文档的比例也超过预先定义的 security hard gate。应如何决定？

### Options

A. 立即全量上线，因为 primary metric 提高了 4 个百分点

B. 只对没有投诉的用户上线

C. 停止或拒绝 treatment，修复 security failure 并加入 regression suite 后再评估

D. 删除 security metric，因为它不是 primary metric

### Correct Answer

C

### Explanation

Security hard gate 不是可由平均业务收益抵消的 secondary metric。C 保护用户，并把生产发现固化为后续离线测试。A、B、D 都是在指标改善的名义下接受已知越权风险。

---

## Question 7（单选）

B 的任务成功率 point estimate 比 A 高 1%，但 confidence interval 很宽，同时包含有实际意义的改善和退化。最佳结论是什么？

### Options

A. B 已证明更优，因为 point estimate 为正

B. A 与 B 已证明完全等价，因为结果不显著

C. 结果 inconclusive；检查数据质量与 power，继续收集预定样本或重设计实验

D. 从 dozens of segments 中挑一个 B 显著的群体，并据此宣布整体胜利

### Correct Answer

C

### Explanation

宽区间说明信息不足，既不能证明 B 胜出，也不能证明等价。C 尊重 uncertainty，并回到 MDE、样本与噪声设计。A 忽略区间，B 把“不显著”误解为“相同”，D 是事后多重比较。

---

## Question 8（多选，选择三项）

团队把新 RAG pipeline 部署给 5% 流量，称为“A/B test”。哪些额外条件最能使它成为可信的在线对照实验？

### Options

A. Eligible users 被随机、sticky 地分到同期 control 与 treatment

B. 记录 assignment、实际 exposure/fallback、版本化配置、trace 与业务 outcome，并检查 SRM

C. 预先定义 primary metric、guardrails、horizon 和 rollback/decision rule

D. 只监控 treatment 的 server uptime，不保留 control

E. 让 B 同时更换模型、prompt、index、UI 和权限，并把收益全部归因于模型

### Correct Answer

A、B、C

### Explanation

A 提供可比的随机对照，B 使实际 treatment 与数据质量可审计，C 防止事后改规则并保护用户。D 只是 canary/稳定性监控；E 即使能比较整个 bundle，也不能把效果单独归因给模型，而且增加定位和回滚难度。
