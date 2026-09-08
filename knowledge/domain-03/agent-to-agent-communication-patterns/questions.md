# Agent-to-Agent 通信模式：场景练习

> 清单 #2 · Domain 3: Integration（19%） · 2026-09-08
> 8 道原创练习，非官方真题。结构参考 [Exam_Guide.md](../../../Exam_Guide.md) 第 8 节：根据场景约束选最直接方案，并解释关键干扰项。每题标明选择数量。
> 先学习 [knowledge.md](knowledge.md)。题目考通信与编排决策，不要求背诵某个会变化的 SDK 字段。

## 1. Tool 还是 Agent？

### Question

客服助手只需根据订单号读取一个稳定订单 API，并返回状态。查询没有多步规划，API 已有清晰 schema 和权限校验。哪种方案最合适？（选择 1 项）

### Options

- A. 把订单查询包装成自主 agent，并允许它自行创建更多 agent。
- B. 直接将订单查询作为 tool/API 调用，保留确定性的契约和权限校验。
- C. 建立三个订单 agent 互相投票，以避免 API 返回错误。
- D. 让模型从训练记忆推测订单状态，减少集成复杂度。

### Correct Answer

B。

### Explanation

B 与问题的确定性、单步骤性质匹配。A 和 C 引入没有价值依据的规划、成本和协调面。D 无法取得实时私有订单数据。Agent-to-agent 适合委派需要自主执行的任务，并非所有外部调用的默认形式。

## 2. 可并行的高价值研究

### Question

公司要评估一次高价值收购，必须分别调查财务、法律和安全风险；三个维度可独立研究，最后需要统一、可追责的建议。哪种模式最合适？（选择 1 项）

### Options

- A. Lead agent 将三个有明确边界的任务并行委派给专业 worker，再验证并综合结果。
- B. 三个 agent 不设角色、预算或终止条件，自由相互发送任务直到意见一致。
- C. 让一个随机 worker 直接发布最终建议，省去综合环节。
- D. 将所有资料发送给三个 agent，但不规定来源、输出格式或完成标准。

### Correct Answer

A。

### Explanation

A 利用子任务的独立性降低延迟，同时由 lead 保持最终责任。B 容易循环、失控且责任不明。C 缺少跨维度综合。D 虽然使用多个 agent，却没有有效委派契约，容易重复、遗漏且难以验证。

## 3. 高共享状态任务

### Question

一个代码重构任务要求每一步都基于同一组不断变化的文件，模块之间高度耦合；团队预算有限，单 agent 已能完成任务。哪项决策最合理？（选择 1 项）

### Options

- A. 仍拆成十个并行 agent，因为 agent 数越多结果必然越好。
- B. 优先保留单 agent 或受控 workflow；只有找到真正独立的工作包时再引入 worker。
- C. 让每个 agent 各自维护不同版本的代码，最后按多数票合并。
- D. 把所有 agent 改成 peer-to-peer，以消除状态一致性问题。

### Correct Answer

B。

### Explanation

题目明确给出高度共享状态、依赖关系和有限预算，这些都削弱并行多 agent 的收益。A 的“必然”不成立。C 会扩大版本冲突。D 增加协调自由度，并不会自动解决一致性，反而更难治理。

## 4. 好的委派契约

### Question

Lead agent 委派“研究供应链问题”，三个 worker 重复查同一地区，同时遗漏法规风险。哪两项改进最直接？（选择 2 项）

### Options

- A. 为每个 worker 明确互斥的范围、目标与 non-goals。
- B. 规定统一输出 schema、证据要求和完成标准。
- C. 仅把每个 worker 的 temperature 调低，继续使用原始模糊指令。
- D. 允许 worker 无上限创建更多 agent，以增加覆盖率。
- E. 删除所有任务 ID，让 worker 更自由地返回结果。

### Correct Answer

A、B。

### Explanation

A 直接解决重复与遗漏，B 让汇总者检查覆盖面和证据。C 不会补齐任务边界。D 可能把重复和成本问题进一步放大。E 削弱追踪能力，与问题无关。

## 5. 长任务通信

### Question

合规 agent 的分析通常需要 30 分钟，期间可能要求人工补充文件。调用方不能一直阻塞连接，还要支持恢复和取消。最佳交互设计是什么？（选择 1 项）

### Options

- A. 只返回一次无 task ID 的同步响应，超过 60 秒就从头重试。
- B. 使用可追踪的异步 task，持久化状态，提供状态查询/通知、补充输入与取消语义。
- C. 每秒创建一个新 agent，直到其中一个返回完整结果。
- D. HTTP 接受请求后立即把任务标记 completed，稍后再尝试生成结果。

### Correct Answer

B。

### Explanation

B 满足长时间、人工输入、非阻塞、恢复和取消全部硬约束。A 会造成昂贵重复且无法确认原任务状态。C 缺少关联和成本控制。D 混淆“请求已接受”与“任务已完成”。

## 6. 超时与副作用

### Question

采购 agent 委派付款 agent 执行转账。网络在返回前超时，调用方不知道转账是否已经发生。最安全的下一步是什么？（选择 1 项）

### Options

- A. 立即以新 task ID 重试同一转账，直到收到成功响应。
- B. 先用原 task/correlation ID 查询状态；重试时复用业务幂等键，并只按明确定义的可重试条件操作。
- C. 让更大的模型判断银行是否可能已经转账。
- D. 把超时记为成功，以避免重复付款。

### Correct Answer

B。

### Explanation

B 面对的是经典的“不确定执行结果”，通过状态查询和业务幂等控制副作用。A 可能重复付款。C 没有事实依据。D 也可能把失败误报成功。Transport failure 不等于业务动作未发生。

## 7. MCP 与 agent-to-agent

### Question

企业已有一个能自主规划、多步调用内部 ERP 的采购 agent。另一个跨部门 agent 需要把完整的供应商评估目标委派给它，同时采购 agent 内部仍要查询 ERP。哪项职责划分最清晰？（选择 1 项）

### Options

- A. Agent-to-agent 机制用于委派评估任务；采购 agent 内部可用 MCP/tool 接入 ERP。
- B. MCP 必须承担 agent 的任务生命周期、跨部门协商和最终责任，因此不需要 agent-to-agent 机制。
- C. Agent-to-agent 机制直接替代 ERP 的确定性访问控制。
- D. 两种机制不能在同一系统中使用，因为它们互相竞争。

### Correct Answer

A。

### Explanation

A 区分 agent 协作与 agent-to-tool/data 接入，并允许组合。B 扩大了 MCP 的职责。C 错误移除业务授权边界。D 与两者互补的设计相反。Guide 的核心是根据连接对象和需求选机制，而不是记住某个缩写就排斥其他方式。

## 8. 跨组织 Agent 的信任边界

### Question

一个已认证的外部风险 agent 返回结构化 artifact，其中含来源链接和建议。Lead agent 将据此决定是否自动冻结供应商账户。哪两项控制最重要？（选择 2 项）

### Options

- A. 验证 artifact schema、来源和结论是否满足预定成功标准。
- B. 因对方已认证，直接授予它冻结所有供应商的权限。
- C. 对冻结操作实施本地授权策略，并在风险等级要求时加入 human approval。
- D. 将 lead agent 的完整会话和长期凭证返回给外部 agent，方便其自证。
- E. 只增加日志；即使发现越权，也不阻止动作。

### Correct Answer

A、C。

### Explanation

A 把远端结果当作需要验证的输入，C 保留敏感动作的本地决策与权限边界。B 混淆 authentication 与 authorization。D 违反最小数据和最小凭证原则。E 只有侦测价值，不能替代预防性控制。

## 复盘

错题不要只记答案字母。分别标记错误属于：调用对象判断、拓扑选择、委派契约、同步/异步、失败与幂等、MCP/A2A 边界，还是安全授权。能用场景约束排除最诱人的错误项，才接近 Exam Guide 样题要求的判断方式。
