# Capability bloat 评估与最小权限

> 学习序号 04 · 清单 #4 · Domain 3: Integration（19%）
> 核查日期：2026-09-10。考试依据：[Exam_Guide.md](../../../Exam_Guide.md) v1.0 第 6 节 “Evaluate tool/agent configuration for capability bloat”；第 8 节 Sample 1 直接以 least privilege 考查移除不需要的工具。
> 前置：[01 MCP](../01_mcp-architecture-and-use-cases/knowledge.md)、[02 Agent-to-Agent](../02_agent-to-agent-communication-patterns/knowledge.md)、[03 API/CLI](../03_api-cli-integration-patterns/knowledge.md)。本课聚焦能力面收敛；完整身份、租户隔离和授权架构留给清单 #5。

## 1. 先抓住考试核心

**Capability bloat（能力膨胀）**是指某个 agent 为完成其职责，被配置了超过必要范围的工具、操作、资源访问或自主权。例如只负责生成回复草稿的客服 agent，同时拥有退款、账户删除和任意 SQL 工具。

它不要求多余能力已经被调用。只要能力对 agent **可达**，攻击面和误操作面就已经扩大：错误推理、含恶意指令的外部内容、配置错误或凭证泄露，都可能把“理论上的能力”变成真实副作用。

**Least privilege（最小权限）**要求每个主体只在完成当前职责所需的范围和时间内拥有必要能力。Guide Sample 1 的判题标准非常明确：若角色根本不需要退款和删除能力，最佳方案是把工具从配置中移除；日志和确认只是补偿控制，不能消除不必要的能力。

一句话记忆：

> 先缩小可做什么，再约束怎样做，最后监测做了什么。

## 2. “能力”不只是工具名称

评估配置时要逐层盘点，而不是只数 `tools` 数组有几个元素。

| 层次 | 要问的问题 | 膨胀示例 |
|---|---|---|
| Tool surface | 这个角色是否需要看到或调用该工具？ | 草稿 agent 能调用 `delete_account` |
| Operation | 是否只暴露所需动作？ | 只需读订单，却暴露通用 `execute_sql` |
| Resource scope | 能访问哪些租户、仓库、目录、记录？ | 工单 agent 可读取所有客户 |
| Credential scope | 执行身份具有什么后端权限？ | 只读工具持有数据库管理员凭证 |
| Argument scope | 参数是否允许扩大目标或影响范围？ | 文件工具接受任意绝对路径 |
| Autonomy | 哪些动作可自动执行，哪些需审批？ | 高额退款无需确定性校验或人工批准 |
| Lifetime | 权限是否比任务或会话活得更久？ | 临时分析任务使用长期共享密钥 |

“工具描述得更清楚”可以改善模型选对工具的概率，却不会收窄后端权限。Anthropic 的工具定义文档说明，工具定义会进入模型上下文，`description` 用来描述何时及如何使用工具，`input_schema` 约束预期参数；这些是接口与行为引导，不等于业务授权。[Define tools](https://platform.claude.com/docs/en/agents-and-tools/tool-use/define-tools)

## 3. 评估方法：从职责到最小能力集合

### 步骤 A：写清角色和允许结果

先定义 agent 的单一职责、数据边界和允许副作用。例如：

- 角色：一级客服回复助手；
- 允许：读取当前租户的工单、检索批准的知识库、生成草稿；
- 禁止：发送消息、退款、修改客户记录、跨租户读取；
- 升级：需要退款时转交专用退款流程。

如果连允许结果都写不清，就无法证明某项能力“必要”。

### 步骤 B：建立 capability matrix

把每项能力映射到业务任务和风险：

| 能力 | 必须完成的任务 | 影响 | 决策 |
|---|---|---|---|
| `read_ticket` | 理解当前工单 | 泄露客户数据 | 保留；限制为当前租户/工单 |
| `search_kb` | 查批准的答复依据 | 可能摄入不可信内容 | 保留；限制数据源与返回量 |
| `draft_reply` | 产生草稿 | 错误内容 | 保留；明确仅草稿 |
| `send_reply` | 一级客服不负责发送 | 对外通信 | 移除 |
| `issue_refund` | 不属于该角色 | 财务损失 | 移除并路由至独立流程 |

证据不足的工具默认不应进入生产配置。以“未来可能有用”为理由保留高影响能力，正是 capability bloat 的常见来源。

### 步骤 C：先消除，再收窄，再加控制

按以下顺序决策：

1. **Remove**：职责从不需要的工具，完全不提供给该 agent。
2. **Split/route**：少数场景才需要的高风险能力，交给独立角色或确定性 workflow；使用不同工具集和凭证。
3. **Scope**：必须保留的工具，收窄操作、资源、参数、凭证和时间范围。
4. **Gate**：对不可逆或高影响操作加入确定性策略校验、额度限制、step-up authentication 或 human approval。
5. **Contain**：用 sandbox、文件边界、网络 egress allowlist 和隔离运行环境限制 blast radius。
6. **Observe**：记录调用主体、工具、授权决策、目标资源和结果，并告警异常行为。

Anthropic 对 agent 的建议是按具体 use case 定制能力，保持接口清楚，并只在能证明效果时增加复杂性；自主 agent 还会带来错误累积风险，适合在沙箱和 guardrails 下充分测试。[Building Effective AI Agents](https://www.anthropic.com/engineering/building-effective-agents)

## 4. 三道边界：模型、编排器、后端

可靠架构不能把安全决定只交给提示词。

```text
用户/外部内容
      │
      ▼
Claude：选择已暴露的工具并生成参数（不拥有最终授权）
      │ tool_use
      ▼
编排器：验证 schema、身份、策略、审批状态、预算和幂等性
      │ authorized request
      ▼
业务服务：以 scoped credential 再做资源级授权并执行
```

Claude API 的 client tool 由应用实际执行；模型返回 `tool_use` 是执行请求，不是授权或成功证明。[Tool use overview](https://platform.claude.com/docs/en/agents-and-tools/tool-use/overview)

因此：

- System prompt 中写“不要退款”只影响模型意图，不是强制边界。
- JSON Schema 或 strict tool use 可减少参数形状错误，不会证明用户有权操作目标资源。
- 编排器的策略检查不能替代业务服务的 object-level authorization。
- 后端凭证如果权限过大，即使工具表面只叫 `read_order`，仍可能留下很大的 blast radius。

## 5. 什么时候移除，什么时候保留并加审批？

| 情况 | 首选决策 | 原因 |
|---|---|---|
| 该角色从不需要该能力 | 移除工具 | 消除攻击面，符合 Guide Sample 1 |
| 只有另一角色需要 | 分离 agent/workflow 和凭证 | 避免所有会话共享高权限 |
| 当前角色偶尔需要且影响可逆、边界清楚 | 保留最小操作并按风险 gate | 避免为简单任务全面提权 |
| 高影响、不可逆或法规要求人工决定 | 确定性校验 + human approval；必要时由专用服务执行 | 模型自信不是审批依据 |
| 不可信输入可能影响工具选择 | 减少可达工具、隔离上下文和执行环境 | 降低 prompt injection 可利用的能力 |

Human approval 不是万能替代品。若每次普通操作都弹窗，会形成 **approval fatigue**，用户可能机械批准。优先让低风险、边界明确的读取自动化；只对真正敏感的动作升级审批，并在批准界面展示对象、金额和影响。

## 6. 防御控制的强弱与用途

| 控制 | 主要作用 | 不能替代什么 |
|---|---|---|
| 移除不必要工具 | 消除不需要的能力面 | 必需工具内部的授权 |
| Scoped credentials / resource checks | 强制限制真实资源访问 | 正确的工具选择与业务验证 |
| Human confirmation | 对敏感决定增加监督 | 全面工具删减；也会受疲劳影响 |
| Sandbox / egress allowlist | 限制文件、进程、网络影响范围 | 业务层对象授权 |
| Schema validation / strict tool use | 保证参数结构符合约定 | 参数语义、用户权限和事实正确性 |
| Logging / alerting | 追溯和侦测异常 | 阻止首次危险操作 |
| 更强模型或更低 temperature | 可能改变决策质量/一致性 | 强制权限边界 |

Claude Code 当前文档提供细粒度 allow/ask/deny 权限；bare tool deny 可让模型上下文中完全没有该工具，而 scoped rule 是在调用时阻止匹配操作。文档也明确：权限由运行时执行，而不是由模型提示执行。[Configure permissions](https://code.claude.com/docs/en/permissions)

其安全指南还将只读起步、工作目录边界、sandbox、网络控制、MCP 信任验证和审计列为不同层次的防护，并提醒处理不可信内容时审查命令与隔离执行。[Claude Code security](https://code.claude.com/docs/en/security)

这些 Claude Code 机制是当前产品实现示例。考试要求是能评估能力膨胀并作架构判断，不要求死记某个版本的规则语法。

## 7. 常见 failure modes

### “全部工具都给一个万能 agent，靠 prompt 管住”

Prompt injection、上下文误解或错误规划都可能绕过行为约束。应按角色分割能力，并让执行层做强制授权。

### 用通用工具代替窄工具

`execute_sql(sql)`、任意 `bash`、任意 HTTP fetch 很灵活，但难以证明允许操作的上界。对固定业务动作，优先提供语义明确的窄接口，如 `get_order(order_id)`；在后端固定只读查询并校验租户。

### 只缩 schema，不缩 credential

表面 schema 只接受订单号，但 handler 仍使用跨租户管理员凭证，程序缺陷就可能突破预期边界。工具契约和执行身份必须一起收窄。

### 把日志当预防控制

日志可以回答“发生了什么”，但通常不能阻止第一次误删。高影响操作需要消除能力、强制授权和执行前 gate；审计是补充。

### 因误调用率低就忽略高影响工具

风险不仅看发生概率，也看影响和可利用性。一个很少被误选、但能删除生产数据的工具，仍应从不需要它的角色中移除。

### 共享一个高权限 agent 给所有租户

即使每个请求有用户 ID，若资源级检查遗漏，就可能横向越权。能力清单、租户上下文、scoped credential 和后端授权要共同形成边界。

## 8. 考试答题决策树

面对场景题，依次判断：

1. 题目明确说角色**不需要**某能力吗？若是，优先移除，而不是记录、提示或确认。
2. 能力是否偶尔需要，但只属于特定角色/流程？优先分离和路由。
3. 保留能力是否被收窄到必要动作、资源、参数、凭证和时长？
4. 高影响动作是否有模型之外的确定性校验或审批？
5. 日志、sandbox 和监控是否作为纵深防御，而非替代最小权限？
6. 某选项是否用“大模型、更详细 prompt、低 temperature”回答授权问题？通常是错位干扰项。

要特别区分：

- **Capability bloat**：agent 被提供了不需要或过宽的能力。
- **Tool-selection quality**：在被提供的工具中，模型能否选对工具。
- **Authorization gap**：调用方是否有权对特定资源执行动作。

三者相关，但解决办法不同。移除多余工具处理第一项；清晰描述和评估帮助第二项；执行层策略与资源级授权处理第三项。

## 9. 范围与核查结论

本课与 list.md #4 粒度一致，无需修改 Guide 要求或新增考试重点。外部资料只用于解释 Guide 已明确要求的能力面、工具接口和防护实现。

**Needs verification**：无阻碍本课概念或练习答案的未核实核心事实。Claude Code 的具体 permission mode、规则语法、工具类型及 API 可选字段属于快速变化的产品实现；实际部署前应重新核查官方文档，不作为本课死记内容。

下一步：完成 [questions.md](questions.md)。先独立作答，再用解释检查自己是否把补偿控制误当成了能力消除。
