# 22 · Token、Latency 与 Cost–Performance Trade-off：原创练习题

> 以下题目不是官方真题。题型和判断方式参考 Exam Guide 的 scenario-based / architecture decision 风格，只覆盖本知识点。

## Question 1（单选）

一个合规审查服务的每次请求都包含相同的大型 policy、tool definitions 和 examples，只有末尾的合同条款不同。质量已达标，但 input cost 和 TTFT 偏高。最佳第一步是什么？

### Options

A. 删除 policy 的后半部分，不做回归测试

B. 将稳定内容组成一致前缀、动态合同放在后面，启用并监控 prompt caching

C. 每次随机重排 examples，避免模型过拟合

D. 无条件切换到最便宜模型

### Correct Answer

B

### Explanation

B 保留必要信息，同时让重复大前缀被重用，直接针对重复处理成本和 TTFT。A 可能破坏合规质量；C 会降低 cache 命中且改变行为；D 未验证 task capability，可能增加 failure 和 retry。

---

## Question 2（单选）

团队为长报告启用 streaming 后，用户更快看到第一段文字，但完整报告的生成时间、token 数与账单基本不变。最准确的解释是什么？

### Options

A. Streaming 主要改善 perceived responsiveness/TTFT，不保证降低 total completion time 或 cost

B. Streaming 自动压缩了整个 prompt

C. Streaming 等价于 asynchronous batch

D. 只要看到第一 token，业务任务就一定完成

### Correct Answer

A

### Explanation

Streaming 让客户端增量接收输出，用户更早获得反馈；它通常不减少模型必须生成的 tokens，也不自动降低完整生成时间或费用。B、C 混淆不同机制；D 忽略完整结果、validation 和 tool outcome。

---

## Question 3（单选）

公司每天夜间需要对 50 万条历史 feedback 做一次分类，结果第二天上午可用即可，单条交互 latency 不重要。哪种方案最符合 cost–latency trade-off？

### Options

A. 每条记录由用户界面发起同步 streaming 请求

B. 使用适合非实时 workload 的 asynchronous batch，设计 custom ID、失败重试和结果对账

C. 把全部 feedback 拼进一个无法验证的超大 prompt

D. 为每条记录启动一个长期 autonomous agent

### Correct Answer

B

### Explanation

该任务吞吐大、deadline 宽，适合异步 batch 的单位成本优势；同时需要识别每个结果并处理 errored/expired items。A 为不存在的实时需求付费；C 会遇到容量、隔离和可追踪问题；D 增加不必要的 orchestration、token 和风险。

---

## Question 4（多选，选择三项）

一个 agent 需要查询天气、读取库存和获取客户偏好，三项互不依赖，随后综合推荐商品。哪些优化最合理？

### Options

A. 并行执行三项只读查询，并定义 partial-failure policy

B. 记录各 tool latency、整个 critical path、token 与最终 task success

C. 对 tool result 去除无关字段，只把综合所需信息放入 context

D. 无界重试所有失败调用，直到成功

E. 在查询结果返回前让模型编造缺失数据

### Correct Answer

A、B、C

### Explanation

A 缩短独立 calls 的 wall-clock critical path；B 能确认真实瓶颈；C 减少低信号 context。D 会放大 tail latency/cost 和副作用风险；E 破坏 groundedness。若 calls 有数据依赖，则不能照搬并行策略。

---

## Question 5（单选）

方案 X 每次请求成本最低，但只有 60% 任务成功，失败后通常重试两次并有 20% 转人工；方案 Y 单次调用更贵，但 95% 一次完成。架构决策最应比较什么？

### Options

A. 只比较模型标价

B. 只比较第一次请求的 input tokens

C. Cost per successful task，并包含 retries、tools、infrastructure 和人工升级，同时检查质量/SLA

D. 选择输出文字更短的方案，不考虑 outcome

### Correct Answer

C

### Explanation

C 使用业务 outcome 作为分母，能暴露便宜但失败频繁的路径所隐藏的总成本。A、B、D 都是局部指标，无法判断端到端 cost-performance。

---

## Question 6（单选）

开发者将 `max_tokens` 设得很低以降低成本，结果 JSON 经常在字段中间被截断并触发昂贵重试。最佳改进是什么？

### Options

A. 继续降低上限，让失败更快发生

B. 定义精简但完整的 output schema，设置经过测量的合理上限，检查 completion/stop 与 schema validation

C. 删除所有 validation，直接消费半截 JSON

D. 用 streaming 代替完整性检查

### Correct Answer

B

### Explanation

`max_tokens` 是 hard cap，过低会制造截断和重试。B 同时控制预期输出和完整性，并应在真实分布上确定预算。A、C、D 都没有解决合法完整输出的问题。

---

## Question 7（多选，选择三项）

一个平台同时处理简单字段提取和复杂多文档风险分析。团队想使用 model/config routing。哪些措施最重要？

### Options

A. 用真实 prompts/data 分别评估各 route 的 success、edge cases、latency、cost 与 safety

B. 定义 complexity/confidence signal、escalation 和 fallback，并监控 route mix

C. 以满足质量门槛的最低成本路径处理每类任务，而非所有任务统一使用同一配置

D. 总是将所有请求送到最小模型，不允许升级

E. 只根据输入字符数路由，不验证错误分流

### Correct Answer

A、B、C

### Explanation

A 建立证据，B 管理 routing 自身的错误，C 实现 capability–speed–cost 匹配。D 会牺牲复杂任务；E 可能是一个 feature，但不能单独代表语义难度，且缺少 eval。

---

## Question 8（多选，选择四项）

一个客服 agent 的平均 latency 尚可，但 p99 和月度费用持续升高。哪些步骤最符合系统化优化？

### Options

A. 按 intent/context size/tool path/retry/route 分解 p95/p99、tokens 和 cost per success

B. 查找慢 tools、serial dependencies、retry storms、低 cache hit 与冗余 context

C. 一次只改变一个主要 lever，并重跑 quality/safety eval 与 tail-latency/cost 指标

D. Canary/A-B 验证后逐步 rollout，并保留 rollback 条件

E. 同时删一半 context、换模型、修改 retriever 和关闭 validation，然后只看平均 latency

### Correct Answer

A、B、C、D

### Explanation

A、B 定位真正的 tail/cost 驱动项，C 保持可归因并防止质量退化，D 验证生产收益和可恢复性。E 同时改变多个因素、移除安全网且只看均值，无法判断收益来源或关键 slice 的伤害。
