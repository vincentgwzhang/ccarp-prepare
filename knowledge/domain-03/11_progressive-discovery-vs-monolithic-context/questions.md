# 练习题：Progressive discovery 与 Monolithic context

> 学习序号 11 · Domain 3: Integration。以下均为原创练习题，并非官方真题。题型和 rationale 参考 `Exam_Guide.md` Sample Questions，重点是依据场景约束选择架构，而不是背当前 API 参数。

## Question 1（单选）

### Question

企业 Agent 可访问 240 个工具，来自 GitHub、工单、监控、HR 和财务系统。一次典型请求只需要 2–4 个工具。将所有 schema 预载后，input context 很大且经常选中名称相似的错误工具。最佳改进是什么？

### Options

- **A.** 增加更多相似工具，让 Claude 有更多选择。
- **B.** 保留核心策略和少量高频工具，其他工具通过受授权的 catalog 按需发现并加载完整 schema。
- **C.** 删除所有工具描述，只保留函数名以节省 token。
- **D.** 提高生成 temperature，使工具选择更多样。

### Correct Answer

**B**

### Explanation

B 直接处理大工具集的 context bloat 与选择歧义，同时保留常用低延迟路径。A 会扩大问题；C 破坏 discoverability 和参数理解；D 增加随机性，不能修复上下文设计。

---

## Question 2（单选）

### Question

某风控服务只有 4 个短小工具，每次请求都会按固定顺序调用全部工具，端到端 SLA 非常严格。团队考虑强制增加 tool search step。最佳判断是什么？

### Options

- **A.** 必须增加，因为 progressive discovery 在所有场景都优于预加载。
- **B.** 继续预加载 4 个工具更合理；discovery 的额外延迟和失败路径缺少收益。
- **C.** 移除所有工具，在 system prompt 中模拟调用结果。
- **D.** 把安全规则也延迟到搜索命中后再加载。

### Correct Answer

**B**

### Explanation

题干是小、稳定且每次全部使用的 toolset，monolithic context 简单、快速并减少 discovery miss。A 将模式绝对化；C 无法获得真实执行结果；D 可能让 Agent 在看到安全规则前行动。

---

## Question 3（多选，选择 **3** 项）

### Question

团队把企业操作手册改造成三层 progressive discovery：metadata → 主说明 → 专项 reference。哪些设计最重要？

### Options

- **A.** Metadata 明确写出能力、触发条件和用户常用术语。
- **B.** 核心授权与高风险 approval 规则始终可见，不依赖按需检索。
- **C.** 主说明提供清晰导航，只在任务需要时加载专项 reference。
- **D.** 使用“处理各种事情”作为所有模块的统一 description。
- **E.** 允许搜索结果绕过用户权限，以免漏掉资料。

### Correct Answer

**A、B、C**

### Explanation

A 保证可发现性，B 保持不可延迟的安全 invariant，C 实现真正的分层加载。D 会让模块难以区分；E 将 discovery 错误地变成权限提升，违反 least privilege。

---

## Question 4（单选）

### Question

开发者说：“我们的 monolithic prompt 已启用 prompt caching，所以无关的 70,000-token 工具和文档不会再影响 Claude。”最佳回应是什么？

### Options

- **A.** 正确；缓存后这些内容既不计入 context，也不可被 Claude看到。
- **B.** 不完整；caching 可优化重复前缀的部分成本/延迟，但无关内容仍在模型可见 context 中竞争注意力。
- **C.** 正确；caching 会自动删除过期规则和冲突版本。
- **D.** 不正确，唯一原因是缓存会改变工具权限。

### Correct Answer

**B**

### Explanation

Prompt caching 与 context curation 是不同层面。缓存不能证明大而杂的 context 具有高信号，也不会自动解决选择歧义、冲突或注意力稀释。A、C 都赋予 caching 不存在的语义；D 把问题错误归因于授权。

---

## Question 5（单选）

### Question

代码 Agent 先读取 repository tree，然后搜索 symbol，只加载目标类、调用者和相关配置。另一个实现把整个 repository 文本一次性拼入 prompt。大型且持续变化的代码库中，前者最主要的架构优势是什么？

### Options

- **A.** 它保证所有代码永远都被模型看到。
- **B.** 它利用结构化线索逐步缩小范围，让当前 context 保持高相关性并可继续 drill down。
- **C.** 它不再需要任何权限、版本或来源记录。
- **D.** 它保证零额外延迟且永远不会走错探索路径。

### Correct Answer

**B**

### Explanation

B 描述 just-in-time progressive discovery：树、路径和 symbol 是导航线索，详细文件按需进入 working context。A 与机制相反；C 错误取消治理；D 忽略搜索步骤、dead-end 和 discovery miss 的真实 trade-off。

---

## Question 6（单选）

### Question

上线 progressive tool discovery 后，input token 降低 70%，但长尾财务工具的任务成功率显著下降。Trace 显示工具存在且用户有权使用，但 description 仅为“process data”，搜索从未命中。第一项最合理的修复是什么？

### Options

- **A.** 优化工具 name/description，加入明确能力、触发条件与领域术语，并用长尾 query set 回归 discovery recall。
- **B.** 提高 Claude 的输出 token 上限。
- **C.** 隐藏所有 discovery trace，避免团队看到失败。
- **D.** 只统计 token 节省，忽略任务成功率。

### Correct Answer

**A**

### Explanation

失败位于 catalog discoverability，而不是工具执行。A 修复检索入口并用真实 query 验证。B 不影响工具搜索；C 移除诊断能力；D 把局部效率指标置于业务结果之上。

---

## Question 7（单选）

### Question

一个 Agent 在查找政策例外时反复打开目录和摘要，进行了 18 次搜索仍未找到证据，延迟持续增长。哪项设计最能控制该 failure mode？

### Options

- **A.** 允许无限探索，因为更多步骤必然提高正确率。
- **B.** 设置搜索步数/token/时间 budget 和停止条件；失败时扩大或切换检索策略、请求澄清或明确 abstain，并记录 trace。
- **C.** 搜索失败后自动把所有企业数据无过滤地塞入 context。
- **D.** 删除核心政策约束以腾出空间。

### Correct Answer

**B**

### Explanation

B 为 dead-end exploration 提供可观测、受控的 fallback。A 会造成无界成本和延迟；C 重新制造 monolithic bloat，还可能越权；D 牺牲正确性和安全，不能修复 discoverability。
