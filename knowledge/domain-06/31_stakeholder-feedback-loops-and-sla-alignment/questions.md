# 31 · 干系人反馈闭环与 SLA 对齐管理：原创练习题

> 以下题目不是官方真题。题型和判断方式参考 Exam Guide 的 scenario-based / architecture decision 风格，只覆盖本知识点。

## Question 1（单选）

上线后，用户通过 thumbs-down 报告客服助手答案变差。团队只保存评分和日期，无法知道当时使用的 prompt、检索内容、模型或工具。最优先的架构改进是什么？

### Options

A. 每次 thumbs-down 都立即更换模型

B. 将反馈事件关联到 end-to-end trace、版本、retrieval/tool path、业务 outcome 和允许保存的交互上下文

C. 删除 free-text feedback，只保留总体平均分

D. 认为用户感受主观，因此不处理

### Correct Answer

B

### Explanation

B 使反馈可复现、可切片并能定位责任层。没有 trace 和版本，团队无法区分 model、prompt、retrieval、tool、UI 或 expectation gap。A 会对稀疏信号过度反应；C 进一步丢失诊断信息；D 会忽略 offline eval 未覆盖的真实失败。

---

## Question 2（多选，选择三项）

团队确认了一批真实生产失败。哪三项最能真正关闭反馈闭环？

### Options

A. 去敏后将代表性 failure 固化为 regression eval，并记录 expected outcome

B. 修复后运行邻近场景和完整 suite，检查质量、安全、成本与延迟回归

C. staged rollout 后监控原受影响 slice，并向相关 stakeholders 更新结果

D. 创建 ticket 后立即标记 feedback resolved，不等待验证或发布

E. 把原始用户内容未经审核直接加入 prompt 或训练数据

### Correct Answer

A、B、C

### Explanation

A 将一次失败变成未来可重复防线；B 防止局部修复引入新回归；C 验证真实环境效果并完成沟通。D 混淆“记录工作”与“解决结果”；E 可能引入隐私、数据质量和 prompt-injection 风险。

---

## Question 3（单选）

某文档写着：“客服 AI 的 SLA 是 p95 latency 小于 4 秒。”但未达标时没有任何约定后果。最准确的判断是什么？

### Options

A. 这是完整 SLA，因为出现了数字

B. 这更像 SLO；若要成为 SLA，还需明确适用范围、测量方法以及未达标的协议后果

C. 这是 SLI，因为任何 target 都称为 SLI

D. 这是 error budget，因为 4 秒等于允许失败量

### Correct Answer

B

### Explanation

SLI 是测量，SLO 是该测量的目标，SLA 则包含与用户或客户约定的后果。该声明有 target，却没有 consequence，也缺少完整 measurement contract。C 混淆 indicator 与 objective；D 混淆 latency threshold 与 error budget。

---

## Question 4（多选，选择三项）

以下哪三项是一条可执行 SLO 不可缺少的要素？

### Options

A. 明确 eligible event 与 good event，即 numerator / denominator

B. target、measurement window 与数据来源

C. owner，以及 miss / burn 时采取的行动规则

D. “尽可能快”这样的开放描述

E. 将所有可采集 metric 都放进同一 SLO

### Correct Answer

A、B、C

### Explanation

A 定义测量语义，B 定义目标和观察边界，C 使目标能够驱动行动。D 不可量测；E 会导致指标过多且难以判断服务健康，SLO 应聚焦少量用户真正关心的 service levels。

---

## Question 5（单选）

一个跨租户数据泄露事件只影响一名用户，没有耗尽月度 availability error budget。产品经理建议等月末统一处理。最佳决策是什么？

### Options

A. 接受建议，因为 error budget 尚未耗尽

B. 将严重 security/privacy 事件作为独立 hard incident trigger，立即 containment、调查与通知；不能由 availability budget 抵消

C. 把该事件计为一次普通 latency miss

D. 删除该样本，以免少数异常影响总体指标

### Correct Answer

B

### Explanation

Error budget 用于管理可接受的 service-level miss，不是购买严重安全违规的额度。B 根据 severity 和不可接受后果立即响应。A、C 把不同风险错误聚合；D 隐藏关键 failure 并破坏治理证据。

---

## Question 6（单选）

用户反馈“系统变笨”，但总体质量分和平均 latency 无明显变化。架构师下一步最应该做什么？

### Options

A. 用总体平均值否定反馈

B. 立即全局切换到最昂贵的模型

C. 将反馈关联版本和 trace，按任务、语言、model/prompt/index/tool path 与时间切片，并用 SME review 和 targeted eval 验证

D. 只提高满意度问卷发送频率

### Correct Answer

C

### Explanation

不同变更可能只影响某些 traffic slices 或尾部行为，总体平均会掩盖退化。C 同时尊重用户信号并要求可复核证据。A 可能漏掉真实分群问题；B 在未定位根因时扩大成本和变更风险；D 增加信号数量但不提高诊断能力。

---

## Question 7（多选，选择三项）

为对齐一个高风险 Claude 决策支持系统的 stakeholder expectations，哪些内容必须明确？

### Options

A. intended use、anti-scope、known limitations 与 uncertainty behavior

B. Claude 是建议者还是最终决策者，以及人工 review / escalation 责任

C. 数据 source / freshness / ACL、服务目标、failure mode 与 incident communication

D. 承诺所有回答 100% 正确，以增强信心

E. 隐藏 fallback 和降级行为，避免用户担忧

### Correct Answer

A、B、C

### Explanation

A、B、C 共同定义能力、权责、数据和运营边界，使用户知道何时信任、何时升级以及故障时会发生什么。D 对开放式生成不现实且不可可靠验证；E 会导致错误预期，并削弱 incident 与 human-control 设计。

---

## Question 8（单选）

某服务过去 28 天已经耗尽可靠性 error budget，但团队计划发布一个会改变 prompt、retrieval 和 tool routing 的大型功能。最佳治理方式是什么？

### Options

A. 照常发布，因为新功能与历史故障可能无关

B. 按预先同意的 error-budget policy 暂停或强化非必要变更，优先恢复可靠性；仅按明确例外处理紧急安全修复

C. 提高 SLO 数字，使现有表现重新达标

D. 只向用户发送道歉，不调整发布策略

### Correct Answer

B

### Explanation

Error-budget policy 的目的，是在可靠性变差时为团队提供共同、数据驱动的行动规则。B 避免继续叠加风险，同时保留预先定义的紧急例外。A 忽略 change risk；C 是修改目标掩盖问题；D 没有改变导致重复 miss 的工程行为。
