# Progressive discovery 与 Monolithic context 策略

> 学习序号 11 · 清单 #11 · Domain 3: Integration（19%）。核查日期：2026-09-17。
> 考试依据：[Exam_Guide.md](../../../Exam_Guide.md) 第 6 节原文 “Evaluate progressive discovery vs. monolithic context strategy”。本课讨论的是架构决策：哪些信息/能力预先放入 Claude context，哪些只提供轻量线索并按需发现。Guide 使用 **progressive discovery**；Anthropic 的 Skills 与 context engineering 资料也常使用 **progressive disclosure**。两者在本课中共享“逐层按需暴露”的核心思想，但不把某个具体 API 功能等同于完整考点。

## 1. 核心矛盾：可用全集不等于当前工作集

Agent 可能理论上访问几百个工具、几十套操作手册和海量业务数据，但当前一次任务只需要其中很小一部分。架构必须决定如何构造每次 inference 的 **working context**。

### Monolithic context

在开始工作前，把预计需要的工具定义、文档、规则、示例或状态尽量一次性放入 context：

```text
request + all instructions + all tool schemas + all candidate data
  → Claude decides and acts
```

### Progressive discovery

先提供稳定规则和轻量目录/metadata；Claude 根据任务逐层发现并加载所需内容：

```text
request + invariant rules + catalog/metadata
  → discover relevant capability/source
  → load selected schema/instructions/data
  → act; fetch deeper detail only if needed
```

关键不是“把资料切成多个文件”，而是 **未被任务需要的详细内容不进入当前 context，同时仍可通过可靠索引被发现**。

Anthropic 将 context 描述为有限资源，并提出目标应是用最小的高信号 token 集合支持期望行为；随着 context 增长，注意力可能被稀释。[Effective context engineering for AI agents](https://www.anthropic.com/engineering/effective-context-engineering-for-ai-agents)

## 2. Monolithic context 不是错误做法

它在以下场景可能是最佳选择：

- 工具数量很少、schema 简短，几乎每个请求都要使用；
- 资料很小、稳定、彼此高度关联，任务需要全局对照；
- discovery 多一次往返会违反严格 latency SLA；
- 漏发现的代价高，而完整内容放入 context 已通过质量、成本和安全验证；
- 需要一个简单、确定、容易调试的 baseline。

优势：

- 没有 search/navigation step，首个决策更快；
- Claude 一开始即可看到全部小型信息集，减少 discovery miss；
- orchestration 简单，trace 和失败路径较少；
- 稳定前缀可与 prompt caching 配合，但 caching 只降低重复处理的部分成本，不消除注意力竞争。

主要风险：

- **Context bloat**：工具 schema、文档和历史挤占任务证据与输出空间；
- **Attention dilution / context pollution**：无关或相似内容增加选错工具、混淆版本和忽略关键约束的概率；
- **成本与延迟**：每次请求都携带很少会用到的内容；
- **维护与冲突**：重复规则、过期文档和相似工具同时出现；
- **暴露面扩大**：不需要的数据若进入模型、日志或下游组件，会增加治理负担。

“context window 装得下”只证明没有超过硬限制，不证明这是最佳上下文。长 context 的容量与有效注意力是两个不同问题。

## 3. Progressive discovery 的四个典型层次

### 3.1 能力发现：从目录到完整 tool schema

启动时只暴露能力类别、工具 metadata 或 search tool；任务需要某能力时，检索匹配工具，再加载完整 schema。Anthropic 当前 Tool Search Tool 文档将此描述为按需发现并加载工具，而不是预先把全部定义放入 context。[Tool Search Tool](https://platform.claude.com/docs/en/agents-and-tools/tool-use/tool-search-tool)

```text
core tools + searchable catalog
  → search "create issue in repository"
  → load github issue tools only
  → select and invoke one tool
```

这是 progressive discovery 的一个产品实现，不是唯一实现。系统也可在客户端做 deterministic routing、semantic/BM25 tool search，或按身份/任务类别生成允许的工具子集。

### 3.2 指令发现：Skill metadata → 主说明 → 专项资源

Anthropic Agent Skills 展示了明确的渐进层级：启动时加载名称/描述；Skill 相关时再读取 `SKILL.md`；引用资料和脚本按需读取或执行。官方文档强调只有相关内容占用 context。[Agent Skills overview](https://platform.claude.com/docs/en/agents-and-tools/agent-skills/overview)

这说明 metadata 不是装饰：description 必须准确表达“做什么”和“何时使用”，否则详细 Skill 即使正确也不会被触发。主文件应像目录与操作入口，细节放在明确命名的 reference files 中。[Skill authoring best practices](https://platform.claude.com/docs/en/agents-and-tools/agent-skills/best-practices)

### 3.3 数据发现：identifier / hierarchy → targeted content

Agent 先看到文件路径、标题、时间、大小、schema 或检索摘要，再用 `grep`、查询、RAG/API 等加载相关数据。Anthropic 的 context engineering 文章称这种方式为 just-in-time context，并指出路径、命名与时间等 metadata 可帮助下一步探索。

例如分析大型代码库时：

```text
repo tree → module README → symbol search → relevant class → referenced config
```

而不是把 repository 全部文件一次性塞给 Claude。每次发现产生新线索，下一步范围进一步缩小。

### 3.4 结果发现：执行与过滤后只引入高价值输出

大规模日志、表格或 API 结果可在外部执行环境中筛选、聚合，再将小型结果集放入 context。目标不是隐藏证据，而是避免几万行中间数据占据工作记忆。原始结果仍应有可追溯引用，并允许在需要时 drill down。

## 4. 哪些内容不能延迟发现

Progressive discovery 不等于“启动时什么都不给”。以下内容通常必须始终可见：

- system-level 目标、角色和输出契约；
- 不可违反的安全、隐私、合规和授权原则；
- 高风险操作的 approval / human-in-the-loop 规则；
- 当前用户身份与可信 authorization boundary；
- 结束条件、关键业务约束和任务成功标准；
- 如何发现内容、哪些来源可信的导航规则。

如果把安全政策本身设为“需要时才检索”，Claude 可能在发现它之前已经采取动作。正确模式是：**核心 invariant 常驻，详细操作手册按需加载**。同样，progressive discovery 不能扩大权限；catalog/search 返回的工具与资料必须先受 AuthN/AuthZ 和 least privilege 约束。

## 5. Hybrid 通常比两个极端更实用

生产系统常采用：

```text
Always loaded
  - core policy / identity / success criteria
  - discovery instructions
  - 3–5 high-frequency compact tools

On demand
  - long-tail tools and MCP servers
  - domain Skills and detailed references
  - source documents, logs, schemas and large results
```

高频、低风险、schema 很小的工具预加载，可避免每次搜索；长尾工具、庞大文档按需发现。Anthropic 的当前 Tool Search 文档也建议保留少量常用工具为 non-deferred；但具体数量、阈值和 API 字段是当前产品建议，不是 Guide 的永久考试常数。

还可按任务类别预组装 bundle：客服请求预载 ticket read/draft 工具；财务请求预载只读报表工具；其他能力仍可搜索。这比单一巨大工具集更可预测，又保留长尾覆盖。

## 6. 选择策略的决策矩阵

| 约束 | 更偏向 Monolithic | 更偏向 Progressive discovery |
|---|---|---|
| 能力/资料规模 | 小且稳定 | 大、持续增长、多域 |
| 每次使用比例 | 大部分都会用 | 每次只用少量长尾项 |
| 首步 latency | 极敏感 | 可接受一次或多次发现 |
| 内容变化 | 少 | 频繁更新、运行时状态多 |
| 任务结构 | 一步、明确 | 多步、需要探索 |
| Discovery miss 风险 | 极高且全集很小 | 可用 fallback/多路检索缓解 |
| Token / attention 压力 | 低 | 高 |
| 工具相似度 | 少且边界清晰 | 多，需要检索、namespace 与描述治理 |

决策不能只看 token 数。还要测 tool-selection accuracy、task success、发现失败、额外 tool calls、p95/p99、成本、安全事件与维护复杂度。

## 7. 如何设计可发现的信息空间

### 7.1 Catalog 质量

每个条目应有清晰、互斥度尽可能高的 name/description、用途、输入、数据域、版本与权限标签。加入用户真实用语和领域同义词，但不要用含糊描述如“处理数据”。一致 namespace（如 `github_`、`billing_`）可帮助检索和审计。

### 7.2 层级与导航

目录先展示 scope，再链接细节；长 reference 加目录；文件名、路径、标题与时间应传递意义。避免 A → B → C → D 的过深链条，也避免一个 overview 引用几十个无解释文件。

### 7.3 Budget 与停止规则

限定 discovery steps、候选数量、累计 token 与时间；命中高置信度结果后停止继续搜索。若多次搜索无结果，执行 fallback：扩大 query、切换索引、请求澄清、使用受控默认路径，或明确报告无法完成，而不是无限探索。

### 7.4 Cache 与复用

同一会话中已经发现且仍有效的工具/资料可复用，避免重复搜索。但必须考虑版本、权限和 freshness；用户身份或 scope 改变后不能复用旧的授权结果。Prompt caching、retrieval cache 与“内容是否进入 context”是不同层面，不能混为一谈。

### 7.5 Observability

记录：用户 intent、catalog/version、search query、候选、最终加载项、拒绝/授权结果、探索深度、token、latency、调用结果与 fallback。没有这些数据，就无法判断失败来自 catalog 不可发现、routing、工具 schema、执行，还是 Claude reasoning。

## 8. 重要概念比较

- **Progressive discovery vs RAG**：RAG 是获得相关外部证据的一组方法；它可用于 discovery，但若一次返回大量候选并全部预载，仍可能是 monolithic behavior。
- **Progressive discovery vs prompt caching**：caching 可减少重复前缀的成本/延迟；被缓存的无关内容仍会占据模型可见 context 和注意力。
- **Progressive discovery vs compaction**：discovery 决定新内容何时进入；compaction 处理已积累的历史如何压缩。
- **Progressive discovery vs capability bloat**：按需加载 schema 不等于移除权限。用户永远不应获得的工具必须从授权 catalog 中删除，而不是仅 defer。
- **Progressive disclosure vs lazy loading**：lazy loading 描述加载时机；有效 discovery 还需要准确 metadata、检索、决策、授权、fallback 与评价闭环。
- **Long context vs monolithic context**：支持长 context 只是容量能力；是否应该一次放入全部内容仍是架构选择。

## 9. Failure modes

- **Vague metadata**：正确工具存在但描述不可检索，造成 false negative。
- **Similar tool ambiguity**：多个近似工具同时命中，仍然选错或参数错误。
- **Discovery tax**：小型固定任务也强制多层搜索，延迟超过收益。
- **Over-defer**：连核心工具、导航说明或安全 invariant 都未预载。
- **Permission laundering**：搜索目录返回调用者不应知道或使用的能力。
- **Dead-end exploration**：缺少 budget/fallback，Agent 反复搜索相同范围。
- **Stale reference**：catalog 指向已删除、旧版或权限已变化的资料。
- **Partial-view error**：Agent 过早停止，没有加载回答所需的例外或关联文件。
- **Monolithic fallback explosion**：一旦搜索失败就把全集塞入 context，重新制造 bloat。
- **只测 token 节省**：上下文变小，但任务成功率、latency 或安全变差。

## 10. Evaluation 方法

用真实任务集比较至少三个 variant：全量预载 baseline、progressive discovery、hybrid hot-set。按简单/复杂、常用/长尾、单域/跨域、高风险任务分片，测量：

- end-to-end task success 与 grounded correctness；
- tool/Skill/source selection precision、recall，尤其 discovery miss；
- 首个正确动作时间、总 p95/p99 latency；
- input token、context peak、tool/search steps 和总成本；
- 错误参数、重复搜索、dead-end 与 fallback rate；
- unauthorized discovery/invocation 必须为零；
- catalog 更新后回归与版本兼容性。

不能因为 context token 减少就宣布成功。若长尾工具始终找不到，或额外搜索使关键工作流超时，架构需要改进 metadata、保留 hot set、增加 deterministic routing，甚至对该小场景退回 monolithic。

## 11. 考试决策速记

1. 把 context 当有限的 attention budget，不以“装得下”为充分理由。
2. 小、稳定、常用全集可 monolithic；大、增长快、每次只用少量内容更适合 progressive discovery。
3. 核心 policy/identity/approval/成功标准常驻，细节和长尾能力按需加载。
4. 用 metadata/catalog → instructions/schema → data/resources 的分层结构。
5. 预载 compact hot set + defer long tail 往往是稳健折中。
6. Discovery 不改变授权边界，也不能代替 least privilege。
7. 共同评价质量、发现率、token、latency、成本、安全与维护性。

## 12. 范围、来源与待核查

`list.md` 的 #11 与 Guide 原文一一对应，术语已使用 **progressive discovery**；本课解释其与 Anthropic 官方资料中 **progressive disclosure / just-in-time context** 的关系，无需修正清单粒度。至此 Domain 3 的 11 个拆分知识点已完成，但本次没有生成其他 Domain 的内容。

资料核查于 2026-09-17：[Anthropic Effective Context Engineering](https://www.anthropic.com/engineering/effective-context-engineering-for-ai-agents)、[Tool Search Tool](https://platform.claude.com/docs/en/agents-and-tools/tool-use/tool-search-tool)、[Agent Skills Overview](https://platform.claude.com/docs/en/agents-and-tools/agent-skills/overview)、[Skill Authoring Best Practices](https://platform.claude.com/docs/en/agents-and-tools/agent-skills/best-practices)及 [MCP Tools specification 2026-07-28](https://modelcontextprotocol.io/specification/2026-07-28/server/tools)。

**Needs verification**：无影响核心架构原则或练习答案的未核实事实。实施前须核查所用 Claude model/API surface 是否支持当前 Tool Search/Skills 功能，以及 `defer_loading`、搜索方式、限制、计费、数据保留和第三方平台差异；本文不把当前 model list、固定 token 数、效果百分比或 schema version 当成考试答案。

下一步：[questions.md](questions.md)。
