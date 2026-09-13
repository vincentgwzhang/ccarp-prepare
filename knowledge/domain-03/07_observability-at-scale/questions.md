# 大规模 Claude 系统的可观测性设计：练习

> 学习序号 07 · 清单 #7 · Domain 3（19%） · 2026-09-13；7 道原创场景题，非官方真题。题型和 rationale 风格参考 [Exam_Guide.md](../../../Exam_Guide.md) 第 8 节；资料见 [knowledge.md](knowledge.md)。

## 1. HTTP 成功但答案错误

### Question

一个 RAG 客服系统的 Claude API 成功率保持 99.9%，但文档更新后客户开始投诉答案引用过期政策。当前只记录 HTTP 状态和模型调用总时长。哪项改造最直接提高定位能力？（选择 1 项）

### Options

- A. 只提高日志级别，保存每台服务器的更多框架启动信息。
- B. 将一次业务任务的 trace 关联到检索结果/索引版本、Claude request ID、prompt/config 版本和最终质量反馈。
- C. 把 API 200 响应全部视为正确答案，并调低投诉告警阈值。
- D. 删除文档版本字段，避免 dashboard 出现更多维度。

### Correct Answer

B。

### Explanation

B 能判断错误证据来自哪次索引、经过何种配置生成，并把技术调用与语义结果关联。A 增加无关噪声，未覆盖检索和质量。C 混淆技术成功与任务正确。D 恰好删除了定位文档刷新回归所需的低基数版本信息。

## 2. Metric cardinality 爆炸

### Question

平台团队把 `session_id`、完整用户问题和 `request_id` 都设置为每个 latency metric 的 labels。请求增长后，监控存储和查询成本急升。最佳调整是什么？（选择 1 项）

### Options

- A. 保留这些 labels，因为 metric 必须能唯一标识每个请求。
- B. Metrics 只保留 route、region、稳定配置版本和 error class 等有限维度；高基数标识与安全处理后的详情放入可关联的 logs/traces。
- C. 停止采集所有 metrics，只保存完整 prompt。
- D. 对 `request_id` 按字母排序即可降低 cardinality。

### Correct Answer

B。

### Explanation

B 在可聚合告警与个案诊断之间正确分工，可控制 time-series cardinality。A 使几乎每个请求生成独立序列。C 既失去 SLO 趋势，也放大隐私风险。D 改变排序不改变唯一值数量。

## 3. 高流量 trace 采样

### Question

一个多 Agent 平台无法经济地保存 100% traces，但必须调查罕见工具错误和 p99 延迟，同时需要正常链路作对照。哪项策略最合适？（选择 1 项）

### Options

- A. 只用 0.1% 随机 head sampling，不区分结果。
- B. 全量保留所有 prompt 正文，删除 traces。
- C. 保留少量正常基线，并用 tail/risk-aware sampling 优先保留含错误、高延迟、安全事件或新版本的 traces，同时监控采样器容量。
- D. 只保留每日平均延迟，因为它可以还原每条调用链。

### Correct Answer

C。

### Explanation

C 让有限预算覆盖关键异常并保留比较基线，且认识到 tail sampler 本身有状态与容量成本。A 可能丢掉稀有严重事件。B 暴露敏感内容且无法恢复因果/耗时链。D 的聚合平均数不能还原 p99 个案。

## 4. 异步 Agent 链路

### Question

入口服务将研究任务写入队列，worker 数分钟后调用 Claude 和多个工具。当前团队只能按相近时间戳猜测入口请求与 worker trace 的关系。哪两项改造最合理？（选择 2 项）

### Options

- A. 在队列消息中安全传播 task/trace correlation context。
- B. 用 span link 或等价的因果关联表达 producer 与后续 consumer trace。
- C. 强制让入口 HTTP 连接保持数分钟，以便所有 span 必须处于一个同步调用栈。
- D. 删除 task ID，避免 worker 知道任务来源。
- E. 只增加 CPU dashboard，不记录消息处理结果。

### Correct Answer

A、B。

### Explanation

A 提供稳定的跨异步边界关联键，B 能表达延后执行的因果关系。C 将异步任务错误地改造成脆弱的长连接，且不是建立可观测性的必要条件。D、E 都让跨边界定位更困难。

## 5. Prompt/response 的隐私

### Question

医疗问答团队为方便调试，计划把所有完整 prompt、RAG chunk、response 和工具参数永久发送到第三方 telemetry 后端。最佳架构建议是什么？（选择 1 项）

### Options

- A. 默认记录，因为 observability 的价值总是高于隐私风险。
- B. 默认记录 metadata、版本、长度、结果类别和受控关联标识；只有在明确用途、访问控制、脱敏、保留期和合规依据下，才对内容做受限采样。
- C. 只把患者姓名替换成 user ID，其他医疗内容永久保留即可。
- D. 将后端改名为 audit system 后，就不需要数据治理。

### Correct Answer

B。

### Explanation

B 遵循数据最小化，并保留大多数性能和因果诊断能力。A 无视敏感信息与凭证泄露风险。C 的其余内容仍可能识别患者或包含机密。D 改名称不改变访问、留存和合法性要求；调试 trace 与合规审计也不应混为一谈。

## 6. 隐藏的重试

### Question

系统 dashboard 显示任务最终成功率稳定，但 p95 和 token 成本突然上升。SDK 会对部分 transient failure 自动重试，而应用目前只记录最终响应。哪两项最能验证“隐藏重试”假设？（选择 2 项）

### Options

- A. 为每次 attempt 记录开始/结束、错误类别、耗时及可用的 provider request ID，并关联到最终 task outcome。
- B. 按时间观察 429/5xx/timeout、retry count 与端到端 p95、每任务调用/token 的共同变化。
- C. 删除错误事件，仅保留最终成功，以避免重复计数。
- D. 只查看平均输出字符数。
- E. 禁用鉴权，让重试更快完成。

### Correct Answer

A、B。

### Explanation

A 展示实际 attempt 链，B 检验它与延迟和成本变化是否相关。C 正是隐藏问题的现有做法。D 无法区分额外 API 尝试。E 引入严重安全缺口，且没有验证根因。

## 7. Usage API 的边界

### Question

组织已用 Anthropic Usage & Cost API 制作每日 workspace 成本报表。某租户报告一次工具调用执行了两次，团队提议只靠该报表排查完整调用链。最佳判断是什么？（选择 1 项）

### Options

- A. 足够；聚合成本报表必然包含每个工具的业务幂等键与调用因果。
- B. 不足；保留它做组织用量/成本趋势，同时需要应用侧按 task/trace 关联 Claude attempts、tool_use、实际工具执行、重试和幂等结果。
- C. 足够；只要当天总 token 没变化，就能证明没有重复副作用。
- D. 删除应用日志，因为两个数据源不能共存。

### Correct Answer

B。

### Explanation

B 正确区分聚合管理报表与逐任务分布式诊断。A 赋予 Usage & Cost API 未被题设支持的业务调用信息。C 中 token 汇总不能证明外部工具只执行一次。D 放弃了定位业务副作用所需的细粒度证据。
