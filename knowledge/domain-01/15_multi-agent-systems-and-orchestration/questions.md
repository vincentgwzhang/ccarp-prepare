# 练习题：多 Agent 系统设计与编排策略

> 以下为原创练习题，并非官方真题。题型与判断方式参考 Exam Guide 的 Sample Questions；每题只围绕当天知识点。

## Question 1（单选）

一家咨询公司需要在两小时内完成某新市场的初步研究。任务包括法规、竞争者、客户需求和供应链四个互不依赖的方向，最终报告必须保留来源并处理相互矛盾的结论。哪种架构最合理？

### Options

A. 一个 orchestrator 向四个范围清晰的 research workers 并行委派，再按来源质量验证、解决冲突并综合报告
B. 四个 workers 共享一个可变 prompt，并让最后结束的 worker 覆盖其他人的结果
C. 将同一完整任务交给四个 workers，以简单多数票生成事实结论
D. 固定选择第一个 worker 的答案，以避免 synthesis latency

### Correct Answer

**A**

### Explanation

题目具有明确、可独立并行的广度型工作包，适合 orchestrator–workers。A 同时提供 scope partition、provenance 和冲突处理。B 会产生并发状态竞争；C 重复工作且多数票不能修复相关错误；D 牺牲覆盖率并浪费其他 worker 的成果。

---

## Question 2（单选）

银行的贷款通知流程固定为：读取审批结果、选择合规模板、填充字段、规则校验、人工批准、发送。步骤和分支稳定，且每次都必须依序完成。首选架构是什么？

### Options

A. 固定 workflow，在必要节点调用 Claude，并以确定性规则和人工 gate 控制发送
B. 多 Agent peer-to-peer 网络，让 Agents 自行决定是否跳过审批
C. Orchestrator 每次动态创建多个 workers 来投票选择模板
D. 让单个 autonomous Agent 获得发送权限且不设停止条件

### Correct Answer

**A**

### Explanation

题干是可枚举、强依赖、受监管的顺序流程，固定 workflow 是最小充分方案。多 Agent 不会带来有效并行，却增加协调和权限风险。B、D 还允许模型绕过高影响 gate；C 对稳定模板选择属于过度设计。

---

## Question 3（多选，选择四项）

架构师准备将“审查三个独立服务的变更”委派给三个 specialist Agents。为了让结果可汇聚，委派契约中最应包含哪四项？

### Options

A. 每个 worker 的 objective、service scope 和明确排除项
B. 输出 schema、证据/引用要求和 completion criteria
C. 可用工具、权限边界、时间与调用预算
D. 输入版本或 commit，以及 blocked/partial 的返回方式
E. 允许每个 worker 自行扩大到所有服务并无限创建下级 Agent
F. 要求 workers 只返回“完成”，不提供 artifacts 或依据

### Correct Answer

**A、B、C、D**

### Explanation

A–D 共同定义了类似 API contract 的可执行边界：做什么、基于什么、能做什么、如何返回以及何时停止。E 会造成范围重叠、能力膨胀和预算失控；F 使 orchestrator 无法验证或综合结果。

---

## Question 4（单选）

一个 lead Agent 将同一安全审查请求同时发给六个 workers。监控显示它们反复检查相同文件，token 成本大幅上升，仍遗漏了未被任何 worker 分配的模块。最佳改进是什么？

### Options

A. 先建立 coverage map，按组件或威胁面划分互斥 scope，并在汇聚阶段检查覆盖缺口
B. 再增加六个 workers，以提高命中遗漏模块的概率
C. 删除 task IDs，使 workers 无法知道任务是否重复
D. 让每个 worker 获得全仓库写权限，以便主动修复任何问题

### Correct Answer

**A**

### Explanation

问题根因是 delegation 没有 partition 与 coverage control。A 同时减少重叠并暴露漏项。B 放大成本而不解决结构问题；C 破坏去重和可观测性；D 与审查目标无关，还扩大副作用和安全风险。

---

## Question 5（单选）

三个 research Agents 对同一法规生效日期给出不同答案。协调者接下来最合适的动作是什么？

### Options

A. 比较各答案的来源权威性和发布日期，对冲突点定向复查；仍无法解决时显式报告不确定性
B. 选择两个 Agents 相同的答案，因为多数票必然可靠
C. 选择生成文本最长的答案
D. 将三个答案拼接在一起，不说明冲突

### Correct Answer

**A**

### Explanation

事实冲突应靠 provenance、source authority、freshness 和 targeted verification 解决。Agents 可能共享同一错误来源，因此多数票不是事实保证。C 没有质量依据；D 把协调责任转嫁给用户且可能产生自相矛盾的输出。

---

## Question 6（多选，选择三项）

一个多 Agent 财务分析系统中，研究 workers 只需读取报表；最终执行 Agent 可以提交付款，但必须得到财务人员批准。哪三项设计最符合安全原则？

### Options

A. 只给研究 workers 只读、最小范围的数据和工具权限
B. 把付款动作集中到受控执行角色，并在工具层强制 human approval
C. 将 worker 输出视为不可信输入，验证其 schema、来源和潜在注入内容后再汇聚
D. 在 coordinator 的 prompt 中写“不要滥用权限”，然后给所有 Agents 相同的付款凭证
E. 通过 Agent 间消息明文传递共享凭证，以减少配置工作

### Correct Answer

**A、B、C**

### Explanation

A 限制 blast radius；B 将不可逆副作用置于确定性控制和人工批准之后；C 防止错误或恶意 worker 结果污染协调者。D 仅靠 prompt 不能替代授权控制且违反 least privilege；E 会扩散凭证并破坏审计边界。

---

## Question 7（多选，选择三项）

团队把长时间的多 Agent 调查从同步调用改为 queue-based asynchronous orchestration。新增设计中最重要的是哪三项？

### Options

A. task correlation、幂等 key，以及 duplicate/stale result 处理
B. deadline 和 cancellation 传播、partial completion 与恢复策略
C. 带版本的状态与 durable artifact 引用
D. 假设每个消息只会送达一次，因此删除去重逻辑
E. 让所有 workers 无条件覆盖同一共享状态，以保证“最新”

### Correct Answer

**A、B、C**

### Explanation

异步架构把等待转化为分布式状态、投递和生命周期问题。A–C 分别处理消息重复/关联、任务控制/恢复和状态可重建性。D 是危险的交付假设；E 会导致 lost update、竞态和不可复现结果。

---

## Question 8（单选）

四个 workers 中三个已完成并写入带 provenance 的 artifacts，第四个在 deadline 前超时。用户允许明确标注缺口的部分结果。最佳处理是什么？

### Options

A. 保留已完成 artifacts；根据剩余预算对超时任务做有界、幂等的重试或重新分配，仍未完成则综合部分结果并标明缺口
B. 丢弃全部结果并无限重启整个多 Agent 系统
C. 假装第四个 worker 已成功，以保持报告完整
D. 对所有已完成 workers 执行同一外部写操作作为重试

### Correct Answer

**A**

### Explanation

A 符合 failure containment、预算控制和题目允许的 graceful degradation：成功工作不应因单个 worker 失败而丢失。B 无界且浪费成本；C 伪造状态；D 把读/分析失败错误地转化为可能重复的副作用。
