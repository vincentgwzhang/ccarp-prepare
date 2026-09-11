# Agentic 系统的 Authentication / Authorization 架构

> 学习序号 05 · 清单 #5 · Domain 3: Integration（19%）
> 核查日期：2026-09-11。考试依据：[Exam_Guide.md](../../../Exam_Guide.md) v1.0 第 6 节 “Analyze authentication and authorization requirements to identify security gaps”；题目风格参考第 8 节 Sample 1 的最小权限与错项逻辑。
> 前置：[04 Capability bloat 与最小权限](../04_capability-bloat-and-least-privilege/knowledge.md)。上一课决定“agent 应获得哪些能力”；本课决定“每一次能力调用以谁的身份、对哪些资源、在什么条件下获准”。

## 1. 考试首先要分清四件事

| 概念 | 回答的问题 | 例子 |
|---|---|---|
| Authentication（AuthN） | 你是谁？凭证是否真实有效？ | 验证用户 session、JWT 或 workload identity |
| Authorization（AuthZ） | 你能对这个具体资源做这个动作吗？ | 用户能否退款其所属租户的订单 |
| Consent | 资源所有者是否同意某客户端代表自己访问？ | 用户同意 MCP client 获取 `calendar:read` |
| Capability exposure | Agent 是否应当看到或调用这个工具？ | 普通客服 agent 不暴露删除账户工具 |

成功 AuthN 不代表通过 AuthZ；用户登录成功，也可能没有读取另一个租户订单的权限。Consent 也不是永久、无限范围的授权：用户同意“读日历”，不等于允许删事件，更不等于允许访问另一个服务。

考试场景往往把这几层混在一起。最常见的错误答案是：

- 用 Claude API key 已验证，便认为最终用户已获业务授权；
- 工具 schema 合法，便认为目标资源可以操作；
- 模型说“用户已同意”，便跳过真实 consent / approval；
- 有日志或更强模型，便认为能弥补缺失的执行层权限检查。

## 2. Agentic 系统至少有三条身份链

Claude 系统不是“一个 token 从入口传到底”这么简单。

```text
最终用户 ──用户凭证──> 你的应用 / Agent Orchestrator
                           │
                           ├──应用凭证──> Claude API
                           │                 （应用是谁、在哪个 workspace）
                           │
                           └──受控调用──> Tool Gateway / MCP Server
                                             │
                                             └──下游凭证──> 业务资源服务
                                                               （谁能对哪个对象做什么）
```

这三条链解决不同问题：

1. **User → application**：确认最终用户、租户、会话强度和原始请求。
2. **Application → Claude API**：确认调用 Anthropic 的 workload；它通常不表达用户对你公司订单、CRM 或数据库的权限。
3. **Tool layer → business service**：以 delegated user identity 或受限 service identity 调用下游，并在资源服务再次执行 AuthZ。

核心原则：**Claude 参与规划，但不是身份提供者、权限数据库或最终 Policy Enforcement Point。** `tool_use` 中的 `user_id`、`tenant_id`、角色或“已获批准”都属于模型生成数据，不能作为可信身份来源。

## 3. Claude API 凭证的正确边界

截至核查日，Anthropic 官方 Authentication 文档区分 API key、Workload Identity Federation 等认证方式，并建议共享或自动化 workload 使用自己的身份，而不是共享个人密钥；静态密钥应存放在 secret manager，泄漏时轮换或停用。[Claude API Authentication](https://platform.claude.com/docs/en/manage-claude/authentication)

API 文档还说明请求凭证与 workspace 选择相关；云平台入口可使用各自 IAM。它们用于控制应用对 Claude 平台的访问、用量归属和管理范围，不会自动完成你的业务对象授权。[Claude API overview](https://platform.claude.com/docs/en/api/overview)

因此，一个多租户 Java 服务可以用服务身份调用 Claude，但仍必须从已验证的 server-side security context 取得 `principalId` 与 `tenantId`，并在执行工具时检查订单所属租户。不要把 Claude API key 暴露给浏览器、移动客户端、prompt、tool result 或日志。

这些认证方式和字段是**当前产品实现**，可能变化；Guide 要求掌握的是信任边界和缺口分析，并未要求死记 header 或某类凭证名称。

## 4. 下游调用：Delegation 还是 Service Identity？

### Delegated / on-behalf-of identity

适合“agent 代表当前用户访问其个人资源”的场景，例如读取用户日历或创建其本人有权创建的工单。下游可以保留用户级归属和审计语义。

设计要点：

- token 的 audience 只面向目标资源服务；
- scope 只覆盖当前操作需要的权限；
- token 生命周期短，绑定正确 client / tenant；
- token 留在受信的 credential broker 或 tool handler，不进入模型上下文；
- 下游仍检查对象归属、条件和业务规则。

### Service identity

适合无人值守 batch、公共知识索引或明确属于系统职责的后台操作。它不应伪装成最终用户；审计需同时记录 workload identity 和触发来源。

共享 service account 的主要风险是权限容易变成所有用户权限的并集。若必须使用，应按服务/环境/任务隔离身份、收窄资源和操作，并让服务端根据可信调用上下文实施 tenant/object 约束。

### 两者的选择

| 约束 | 较合适的身份模型 |
|---|---|
| 必须保留用户同意、个人数据范围和用户级审计 | Delegated user identity |
| 无在线用户、由队列触发的系统任务 | Scoped service/workload identity |
| 高影响操作要求近期重新验证用户 | Delegation + step-up authentication / approval |
| 下游不支持 delegation | 受限 service identity + 服务端明确保存原始 actor 与强制资源策略 |

“实现简单”不能成为把管理员凭证共享给所有 agent 的理由。

## 5. Authorization 必须在执行时重新判断

Agent 可能运行多个回合，期间角色、订单状态、审批或 token 都可能变化。授权检查不能只在会话开始或 plan 生成时做一次。

一次高质量决策至少包含：

```text
Decision = subject + action + resource + tenant + context + policy version
```

- **subject**：来自已验证 security context 的 user/workload，不来自 prompt；
- **action**：如 `invoice.read` 或 `refund.create`，避免含义不清的 `admin`；
- **resource**：具体订单、账户、仓库或文档；
- **tenant**：多租户边界应是服务端约束，不信任模型传入；
- **context**：金额、环境、设备/会话强度、时间、审批状态；
- **policy version**：便于审计当时依据了什么规则。

高影响操作在**实际副作用发生前**重新检查。批准的应是具体 action/resource/参数摘要，而不是笼统批准“本会话以后的一切操作”。若执行参数在审批后改变，应重新授权。

### Java / Spring 风格伪代码

以下是架构示意，不依赖某个 Anthropic SDK 版本：

```java
Authentication auth = SecurityContextHolder.getContext().getAuthentication();
TrustedActor actor = actorResolver.fromVerifiedAuthentication(auth);

// orderId 可来自模型，但 actor/tenant 绝不能由模型声明。
Order order = orderRepository.findWithinTenant(request.orderId(), actor.tenantId());

Decision d = policyEngine.authorize(
    actor, "order.refund", order.id(),
    Map.of("amount", request.amount(), "approvalId", approval.id())
);
if (!d.allowed()) throw new AccessDeniedException(d.reasonCode());

ScopedCredential downstream = tokenBroker.forAudienceAndScope(
    "payments-api", Set.of("refund:create"), actor
);
payments.refund(downstream, order.id(), request.amount(), idempotencyKey);
```

关键不在类名，而在数据来源：资源参数可以由 Claude 建议；可信 actor、tenant、审批和下游 credential 必须由执行环境提供并强制验证。

## 6. Policy Enforcement 放在哪里？

采用 defense in depth，但每层职责要清楚。

| 层 | 适合做什么 | 不能只靠它做什么 |
|---|---|---|
| Agent/tool configuration | 不暴露不需要的工具；按角色路由 | 不能证明某个订单属于用户 |
| Orchestrator / tool gateway | 验证调用上下文、策略、审批、预算；发放窄凭证 | 不应成为唯一对象授权层 |
| MCP server / adapter | 验证自己的 access token、audience、scope；映射安全操作 | 不应盲传 token 给下游 |
| Domain service | 最终 object/tenant/business-rule authorization | 不应信任模型陈述的角色和归属 |
| Audit/monitoring | 记录允许/拒绝、actor、resource、policy、结果 | 不能阻止首次越权 |

RBAC 适合粗粒度职责，例如 `support_agent`；ABAC / relationship-based checks 适合“当前用户属于该租户且是订单所有者，金额低于阈值”。生产设计经常组合两者，而不是让一个宽泛角色决定所有对象。

## 7. Token 验证：不能只检查“有 token”

Resource server 应根据采用的凭证机制验证至少以下语义：

- 签名或等价真实性，以及可信 issuer；
- token 是否过期、是否尚未生效，必要时处理撤销；
- **audience 是否就是当前服务**；
- scope / role 是否覆盖请求 action；
- subject、client/workload 与 tenant 是否符合策略；
- token 的强度是否足以执行敏感动作；
- 实际 resource 是否属于允许范围。

不要只 decode JWT 后相信 claims，也不要接受“由别的服务签发、但格式看起来正确”的 token。短生命周期减少泄漏窗口，却不能替代 audience、scope 和对象级验证。

## 8. MCP 场景的关键安全判断

最新版 MCP 2026-07-28 Authorization 规范将受保护的 HTTP MCP server 视为 OAuth resource server。它要求 access token 绑定目标 resource/audience、由 MCP server 验证，并明确禁止接受或转送为其他资源签发的 token；无效/过期凭证对应 401，权限或 scope 不足对应 403。[MCP Authorization 2026-07-28](https://modelcontextprotocol.io/specification/2026-07-28/basic/authorization)

### 禁止 token passthrough

错误架构：

```text
Client token（audience = MCP Server）
        └──原样转发──> CRM API
```

正确思路是分开两条信任关系：MCP server 验证发给自己的 inbound token；若它还要访问 CRM，则以自己作为 OAuth client，通过合法 delegation/token exchange 或独立受限凭证取得 **CRM audience** 的 token。

MCP Security Best Practices 将 token passthrough 列为 anti-pattern：它会破坏 audience、审计与服务边界，并可能造成 confused deputy。该文档还强调按客户端保存 consent、精确验证 redirect URI、保护 state，以及把 scope 降到必要范围。[MCP Security Best Practices 2026-07-28](https://modelcontextprotocol.io/docs/2026-07-28/tutorials/security/security_best_practices)

MCP 协议细节是 Claude 集成的实际例证。Guide 只要求分析 AuthN/AuthZ 安全缺口，未声明必须背诵 OAuth discovery 字段或全部 MCP normative rules。

## 9. 常见 security gaps 与修复

| Gap | 为什么危险 | 首选修复方向 |
|---|---|---|
| Prompt 里的 `user_id` 被当成身份 | 用户或注入内容可伪造 | 从验证过的 server-side context 取 actor |
| 一个跨租户管理员 token 服务所有请求 | 任一缺陷影响所有资源 | 每服务/租户/任务收窄身份与对象检查 |
| 只检查 scope，不检查 resource ownership | 产生 BOLA/IDOR 类横向越权 | Domain service 做 object-level AuthZ |
| 一个 token 被多个服务接受 | token 泄漏可横向重放 | 每个 resource 验证 audience，分离 token |
| MCP server 原样转发 inbound token | 边界、审计与 consent 被绕过 | 验证 inbound；为 downstream 取得独立 token |
| 授权只在 agent 开始时检查 | 长任务期间权限/状态可能变化 | 每次敏感 tool execution 重新判定 |
| 审批后参数可变 | 批准 A，最终执行 B | 将批准绑定 action/resource/参数摘要和期限 |
| 长期凭证进入 prompt、日志或 tool result | 模型上下文和遥测扩大泄漏面 | 令牌留在 credential broker；日志脱敏 |
| 403 被无限重试 | 权限不足通常不会自行恢复 | 停止重试，触发合法 step-up 或拒绝 |
| Session ID 被当作认证 | 被猜中/窃取即可冒充 | 每个请求仍验证真实凭证；session 只关联状态 |
| Cache / memory 未带 tenant key | 跨租户数据污染或泄漏 | tenant-aware key、ACL 与检索过滤 |

## 10. Trade-offs：安全不是无限增加弹窗

### 细粒度策略 vs 复杂度

更细的 scope 和对象规则能减小风险，但会增加策略管理、测试和排障成本。解决办法是从高价值资源和高影响 action 开始建模，使用一致的 policy vocabulary，并测试 allow 与 deny 两类路径，而不是退回万能管理员权限。

### Short-lived token vs 延迟与可用性

短期 token 减少泄漏窗口，但 token exchange、刷新或 IdP 故障会增加延迟和依赖。可在受控 broker 中缓存到安全期限、提前刷新并设置有界失败策略；不能因为性能担忧就把长期 token 放进 agent 上下文。

### Central policy gateway vs service-local enforcement

中央 gateway 便于统一审计和粗粒度策略；领域服务最了解对象归属与业务状态。较稳妥的组合是 gateway 负责共性检查和窄凭证，领域服务保留最终对象级判定。

### Fail closed vs 业务连续性

对退款、删除、敏感数据等高影响操作，授权组件不可用时通常应 fail closed。对低风险读取，可设计经过风险评审的降级路径；不能让模型自行决定绕过认证。

## 11. 考试答题框架

看到 Auth/AuthZ 场景时，依次问：

1. **Subjects**：最终用户、agent workload、MCP client、tool server 分别是谁？
2. **Credentials**：每一跳使用什么凭证？是否共享、长期或暴露给模型？
3. **Audience**：token 是签发给哪个服务的？是否被错误复用/转发？
4. **Action + resource**：不仅有没有 scope，还要看能否操作这个具体对象和租户。
5. **Decision time**：是在执行副作用前检查，还是只在登录/规划时检查？
6. **Delegation/consent**：系统是在代表用户，还是以自身 workload 身份行动？授权是否与真实语义一致？
7. **Failure behavior**：过期/无效凭证、权限不足和授权服务故障分别怎样处理？
8. **Audit**：能否重建谁、代表谁、对什么资源、为何被允许以及结果？日志是否泄密？

优先选择能修复信任边界根因的答案。增加 prompt、模型尺寸、日志或无限重试，通常不能修复身份冒充、audience 错误或对象授权缺失。

## 12. 范围与核查结论

list.md #5 与 Guide 的 AuthN/AuthZ 缺口分析目标一致，本次无需修正清单粒度。教材使用 Claude API 和 MCP 作为官方实现例证，没有把 OAuth 全协议或特定 IAM 产品扩展为新的考试目标。

**Needs verification**：无阻碍本课核心结论与练习答案的未核实事项。Claude API 支持的认证方式、workspace 行为，以及 MCP Authorization 的版本和具体 normative 字段属于会变化的实现细节；实际部署前须按目标平台和协议版本重新核查。

下一步：完成 [questions.md](questions.md)，重点检查自己是否把“应用已能调用 Claude”误判成“最终用户已能调用业务工具”。
