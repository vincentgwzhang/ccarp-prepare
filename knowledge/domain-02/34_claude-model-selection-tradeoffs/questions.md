# 34 · Claude 模型选型权衡：原创练习题

> 以下题目不是官方真题。题型和判断方式参考 Exam Guide 的 scenario-based / architecture decision 风格，只覆盖本知识点。

## Question 1（单选）

某电商每天需要对数百万条短消息执行意图分类。结果可由规则抽样校验，延迟和成本都很敏感。团队尚未做过本地评估。最佳起点是什么？

### Options

A. 无条件使用最高 capability tier，因为它在所有任务上总成本最低

B. 从快速、经济型候选开始，在代表性数据和 edge cases 上验证质量/安全门槛，只有发现 capability gap 才升级

C. 只比较厂商公开 benchmark，跳过真实消息测试

D. 随机选择一个 model，并在生产事故后再调整

### Correct Answer

B

### Explanation

B 是适合高吞吐、可校验任务的 efficiency-first 方法，但仍要求真实 eval 和 hard gates。A 把更高 capability 错当作普遍最低总成本；C 不能代表本地语言、输入分布和错误代价；D 没有发布证据和风险控制。

---

## Question 2（单选）

一个高自治 research Agent 需要跨多个工具运行数小时。balanced-tier model 在最高合理 effort 下仍频繁遗漏依赖，且这些失败已由任务级 eval 复现。下一步最合理的是？

### Options

A. 在不改变任何测试的情况下直接宣布 Agent 不可实现

B. 试验更高 capability 候选，并在相同任务、工具、风险门槛下比较完成率、总时长和 cost per successful task

C. 只增加 context window 中的无关文档

D. 删除失败案例，提高总体通过率

### Correct Answer

B

### Explanation

现有证据表明当前 model/config ceiling 可能不足，B 用 capability-first escalation 并保持可比评估。A 过早停止；C 增加噪声但未解决推理缺口；D 破坏 eval，掩盖真实风险。

---

## Question 3（多选，选择四项）

架构师正在为医疗行政 Assistant 编写 model selection contract。以下哪四项应成为主要决策依据？

### Options

A. 目标任务、真实输入分布、edge cases 和高风险 slices 上的质量/安全结果

B. End-to-end latency、throughput、retry/escalation 和 human-review burden

C. Cost per successful task，包括失败和下游处理成本

D. Required features、目标平台/region、data/compliance 和 model lifecycle

E. Model 名称听起来是否更高级

F. 仅比较单次 input token 的 list price

### Correct Answer

A、B、C、D

### Explanation

A–D 覆盖 task fitness、运行表现、真实经济性和部署硬约束。E 不是证据；F 忽略 output、turns、tools、cache、失败、重试和人工成本，无法代表 end-to-end outcome。

---

## Question 4（单选）

候选 X 的 token 单价是候选 Y 的两倍，但 X 通常一次完成，Y 经常需要重试并转人工。应该使用什么指标做主要经济比较？

### Options

A. 只比较 input token 单价

B. 只比较单次 API call 的 median latency

C. 比较 cost per successful task，计入 turns、retries、tools、human escalation 和 failure impact

D. 选择名称更新的 model，不再计算成本

### Correct Answer

C

### Explanation

C 衡量业务真正购买的“成功结果”。较高 token price 可能因更少 turns/retries 而更便宜，也可能没有收益，必须实测。A、B 都只覆盖局部；D 以版本新旧替代 workload evidence。

---

## Question 5（单选）

团队收到生产 model 即将 retired 的通知。应用包含在线服务、夜间 batch 和隐藏 fallback route。最佳迁移方案是什么？

### Options

A. 等到 retirement 当天统一修改 model string

B. Inventory 所有调用路径，使用现有 prompts/data/tools 对 replacement 做 regression 与 prompt audit，再 shadow/canary、验证 rollback，并在截止日前完成 cutover

C. 只修改在线服务，因为 batch 和 fallback 不会调用 model

D. 相信 replacement 一定完全兼容，因此无需测试

### Correct Answer

B

### Explanation

B 覆盖依赖发现、行为/契约验证、渐进发布和恢复。A 留不出修复时间；C 会留下退休 model 的隐性调用；D 忽略 prompt behavior、API、tool use、latency 和成本差异。

---

## Question 6（多选，选择三项）

团队计划让低成本 model 处理大多数请求，复杂请求升级到高 capability model。哪三项最关键？

### Options

A. 评估 router 自身在 normal、hard 和 high-risk slices 上的错误

B. 定义可观测的 escalation/fallback 条件，并保留 route、model/config version 和 outcome telemetry

C. 与单模型 baseline 比较端到端质量、成本、延迟和运维复杂度

D. 允许用户通过 prompt 自报“低风险”，并直接绕过安全控制

E. 只分别测试两个执行模型，不测试完整 routing pipeline

### Correct Answer

A、B、C

### Explanation

多模型系统的整体质量受 router 和 fallback 控制。A、B、C 分别验证路由误差、运行控制和是否真正优于简单 baseline。D 可被操纵；E 会漏掉 misrouting、双重调用、延迟和错误传播。

---

## Question 7（单选）

工程师认为所有不带日期的 Claude model ID 都会自动升级到最新 weights，因此生产无需记录 exact ID。根据当前官方 versioning 规则，最佳回应是什么？

### Options

A. 正确；所有 dateless ID 都是 evergreen alias

B. 错误；当前 4.6 generation 及之后的 dateless canonical ID 是 pinned version，而较早型号的某些短名称才可能是 alias；生产仍应记录 exact ID 和完整配置

C. 是否 pinning 不重要，因为 model 行为永远不会变化

D. 只记录 family 名称，例如“Sonnet”，就足以重现结果

### Correct Answer

B

### Explanation

B 符合当前官方文档，也避免从字符串外观错误推断。即使 weights 被 pin，serving infrastructure、prompt、tools 和 data 仍需版本化。A 混淆 canonical IDs 与旧 aliases；C 忽略基础设施和系统变化；D 不够精确，无法回滚或审计。

---

## Question 8（单选）

一个候选 model 支持远大于应用典型输入的 context window。架构师建议把全部企业文档放入每次请求，从而取消 retrieval 和权限过滤。最佳判断是什么？

### Options

A. 正确；只要内容放进 context，模型一定准确使用且不会泄露

B. 不正确；context capacity 只是硬上限，仍需按 relevance、ACL、freshness、成本、延迟和 injection risk 设计并评估 context pipeline

C. 正确；更长 context 自动消除 hallucination

D. 只需换最高 capability model，就不再需要数据治理

### Correct Answer

B

### Explanation

大 window 不等于有效、授权且可靠的 context。B 保留 retrieval、权限和数据治理，并要求在真实 context distribution 上评估利用能力。A、C、D 都把模型能力误当成确定性的数据与安全控制。
