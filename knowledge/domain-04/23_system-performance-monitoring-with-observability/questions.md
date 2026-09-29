# 23 · 基于日志和可观测性工具的系统性能监控：原创练习题

> 以下题目不是官方真题。题型和判断方式参考 Exam Guide 的 scenario-based / architecture decision 风格，只覆盖本知识点。

## Question 1（单选）

客服 agent 的平均 latency 本周几乎不变，但 VIP 用户大量报告超时。平台只保留全局平均值。最佳下一步是什么？

### Options

A. 只提高全局 timeout，不增加 instrumentation

B. 采集端到端 latency histogram，按受控的 route/tenant tier/tool path/release 分解 p95/p99，并检查高延迟 trace

C. 把每个 customer ID 设为 metric label

D. 只查看 Claude API 的平均响应时间

### Correct Answer

B

### Explanation

B 同时揭示 tail latency、受影响 slice 和 critical path，能够判断问题来自 queue、retrieval、Claude、tool 还是 retry。A 掩盖症状；C 会造成高 cardinality；D 忽略端到端链路和 VIP slice。

---

## Question 2（多选，选择三项）

团队在设计 telemetry schema。哪些映射最合理？

### Options

A. 用 metric histogram 观察端到端 latency 分布

B. 用 structured log/event 记录某次 policy decision、错误类别和版本

C. 用 distributed trace 还原 gateway → retrieval → Claude → tool 的因果与耗时

D. 将完整 prompt 作为 metric label，便于搜索

E. 只保留自然语言日志，不使用 task/trace ID

### Correct Answer

A、B、C

### Explanation

Metrics 适合聚合趋势和分位数，structured events 适合离散事实，trace 适合跨组件因果链。D 同时带来敏感数据和无界 cardinality；E 缺少稳定 schema 与关联键，难以规模化诊断。

---

## Question 3（单选）

新版本上线后，总体成功率只下降 0.3%，但西班牙语退款请求的 grounded-answer rate 下降 20%。哪项设计最能更早发现并定位该问题？

### Options

A. 只监控全局成功率月均值

B. 记录 prompt/model/index/release 版本，按 language、intent 和 route 等受控维度监控关键 quality slices

C. 将所有用户 ID 加入 dashboard legend

D. 只监控 CPU 和内存

### Correct Answer

B

### Explanation

B 能发现总体聚合掩盖的局部回归，并把它与具体变更关联。A 会稀释异常；C 造成高 cardinality 且未形成有意义 slice；D 只能反映部分基础设施健康，不能判断语义质量。

---

## Question 4（多选，选择四项）

生产系统没有即时 ground truth，团队担心回答质量随真实流量分布变化而漂移。哪些措施应组合使用？

### Options

A. 对 production samples 做脱敏抽样和 transcript review

B. 使用经 human-labeled set 校准的 grader，并持续检查 grader drift

C. 将用户反馈视为一个有偏但有价值的信号

D. 把生产 failure 加入版本化 regression eval，并在发布前重放

E. 只要 latency 与 HTTP 200 正常，就认定答案质量正常

### Correct Answer

A、B、C、D

### Explanation

线上监控覆盖真实行为，但信号嘈杂且缺少自动真值。A、B、C、D 以不同证据互补，并把发现转为预防性测试。E 混淆系统可用性与语义正确性。

---

## Question 5（单选）

平台偶尔出现短暂 429，SDK 重试后用户任务通常成功。当前告警对每个 429 都立即 page，导致值班疲劳。最佳调整是什么？

### Options

A. 完全忽略 429，不记录 retry

B. 结合持续窗口、rate-limit headroom、retry/queue、最终 task failure 和用户 SLO 设置分级告警

C. 禁用所有重试并把异常返回用户

D. 只增加模型输出 token 上限

### Correct Answer

B

### Explanation

B 保留 429 作为 leading signal，同时以持续性和用户影响决定 page、调查或容量 ticket。A 会丢失容量风险；C 未经评估地降低可靠性；D 与限流根因无直接关系。重试本身也必须可见，因为它会增加尾延迟与成本。

---

## Question 6（单选）

优化后每次 API request 的平均成本下降 25%，但任务失败和重试增多，人工升级率也上升。最应监控哪个主指标？

### Options

A. Cost per successful task，同时把 retries、tools、fallback 和人工处理纳入成本

B. 第一次 Claude call 的 input tokens

C. 单次请求的最低标价

D. 输出字符数

### Correct Answer

A

### Explanation

A 用成功业务结果作为分母，能揭示“单次便宜但端到端更贵”的假优化，并应与 quality/safety/latency guardrails 一起看。B、C、D 都是局部 proxy，不能衡量真实 cost-performance。

---

## Question 7（多选，选择四项）

文档索引刷新后，RAG agent 开始自信地给出错误答案。哪些 incident actions 最合理？

### Options

A. 按开始时间和受影响 query slices 确定范围

B. 检查代表性 trace 中的 retrieved chunks、index/corpus version 和 validator outcome

C. 若证据支持，rollback/disable 新索引或切回已知良好路径

D. 将该 failure 及修复加入 regression dataset，再渐进 rollout

E. 立即更换最大模型，不检查 retrieval

### Correct Answer

A、B、C、D

### Explanation

A–D 构成 detect → diagnose → mitigate → prevent 闭环，也与 Guide Sample 3 的首查 retrieval/indexing 相符。E 没有根据地改变模型，无法修复错误 context，还会混淆归因。

---

## Question 8（多选，选择四项）

医疗摘要系统需要保留足够证据定位罕见严重错误，同时保护敏感数据并控制 telemetry 成本。哪些设计最合理？

### Options

A. 默认记录 metadata、版本、结果类别和受控/不可逆标识，而非完整 prompt/response

B. 对敏感内容实施脱敏、访问控制、审计、retention 与 deletion policy

C. 提高错误、高延迟、新版本和严重安全事件 trace 的保留率，同时保留少量正常基线

D. 监控 telemetry pipeline 自身的 dropped records、export failure 和 queue saturation

E. 将患者 ID 与完整病历设为所有 metric 的 labels

### Correct Answer

A、B、C、D

### Explanation

A、B 贯彻 data minimization 和治理；C 用风险导向 sampling 保留诊断价值；D 防止监控盲区。E 会暴露高度敏感数据并产生无界 cardinality，既不安全也不可运营。
