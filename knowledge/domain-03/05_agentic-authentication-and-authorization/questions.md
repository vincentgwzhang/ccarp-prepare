# Agentic Authentication / Authorization：练习

> 学习序号 05 · 清单 #5 · Domain 3（19%） · 2026-09-11
> 7 道原创场景题，非官方真题。题型和 rationale 参考 [Exam_Guide.md](../../../Exam_Guide.md) 第 8 节；核查来源见 [knowledge.md](knowledge.md)。

## 1. Claude API key 的边界

### Question

一个多租户客服应用使用有效 Claude API 凭证。Claude 生成 `get_invoice(invoice_id)` 工具调用后，handler 直接按该 ID 查询数据库，没有验证当前用户或租户。哪项是最关键的修复？（选择 1 项）

### Options

- A. 因为 Claude API 已认证应用，所以无需其他检查。
- B. 从已验证的服务端 security context 取得用户和租户，并在领域服务检查该 invoice 的访问权。
- C. 要求 Claude 在工具参数中同时输出 `user_id`，然后信任它。
- D. 增加工具调用日志，但保持查询逻辑不变。

### Correct Answer

B。

### Explanation

B 把 Claude 平台调用身份与业务用户授权分开，并补上 object/tenant-level AuthZ。A 错把应用对 Claude 的 AuthN 当成用户对发票的 AuthZ。C 的身份字段由模型生成，可被错误推理或注入影响，不是可信来源。D 只能帮助事后调查，不能阻止越权读取。

## 2. 后台索引任务的身份

### Question

夜间任务从批准的内部知识库重建共享搜索索引，没有在线最终用户。团队应优先采用哪种身份方案？（选择 1 项）

### Options

- A. 使用某位开发者的个人长期密钥，因为他创建了任务。
- B. 使用专用于该 workload 的受限 service identity，只允许读取批准的数据源，并记录触发任务与执行身份。
- C. 随机选择最近登录用户的 token，以便保留用户语义。
- D. 将管理员 token 放在 system prompt 中，方便 Claude 在需要时使用。

### Correct Answer

B。

### Explanation

B 与无人值守系统任务的真实主体一致，便于权限收敛、生命周期管理和审计。A 让任务依赖个人且权限/生命周期可能错误。C 伪造 delegation 语义。D 会把高权凭证暴露给模型上下文，并扩大泄漏面。

## 3. MCP token passthrough

### Question

一个 MCP server 接收 audience 为该 MCP server 的用户 access token，然后把同一 token 原样发送给下游 CRM API。CRM 恰好也接受它。最佳整改是什么？（选择 1 项）

### Options

- A. 保持转发，只要两个服务都使用 HTTPS 就安全。
- B. MCP server 验证 inbound token 是签发给自己的；访问 CRM 时通过合法 delegation/token exchange 或独立受限身份获取 CRM audience 的 token。
- C. 把 token 加入查询字符串，便于两个服务共同读取。
- D. 让 Claude 判断 token 是否“看起来可信”。

### Correct Answer

B。

### Explanation

B 保留两条独立信任关系和 audience 边界，避免 token passthrough 与 confused deputy。A 的传输加密不能修复错误 audience 和跨服务复用。C 进一步增加 token 泄漏风险。D 不能替代 cryptographic validation 和资源服务策略。

## 4. Scope 足够但对象不属于用户

### Question

用户的 token 有 `orders:read` scope。Agent 请求读取同一租户内另一个用户的私有订单。API 只检查 scope，因此返回了订单。暴露的主要安全缺口是什么？（选择 1 项）

### Options

- A. 模型 context window 太大。
- B. 缺少 resource/object-level authorization；scope 只说明动作类别，不能证明该 subject 可访问该订单。
- C. Tool description 不够长。
- D. API 返回速度太快，导致策略来不及运行。

### Correct Answer

B。

### Explanation

B 精确指出 AuthZ 缺口：`orders:read` 是粗粒度 action permission，还需检查订单与 subject 的 ownership/relationship。A、C 与对象权限无关。D 把实现缺失误解成性能问题；授权必须位于执行路径中，而不是依赖延迟。

## 5. 审批后参数改变

### Question

财务用户批准“向供应商 X 支付 500 欧元”。Agent 在后续回合把工具参数改为供应商 Y、5,000 欧元，系统因本会话已有批准而直接执行。哪两项修复最合适？（选择 2 项）

### Options

- A. 将批准绑定到具体 action、resource、关键参数摘要和有效期限。
- B. 在执行副作用前重新验证当前参数、权限与审批条件；不匹配时重新请求批准。
- C. 提高模型能力，批准即可继续覆盖所有参数变化。
- D. 只在付款完成后发送审计邮件。
- E. 把 temperature 设为 0，便不需要重新授权。

### Correct Answer

A、B。

### Explanation

A 防止把窄批准扩大成会话级万能许可，B 确保 authorization decision 与实际执行内容一致。C、E 不能把模型行为变成安全边界。D 有助于发现问题，却已经晚于副作用发生。

## 6. Resource server 的 token 验证

### Question

一个 tool gateway 收到结构合法且签名可验证的 JWT。哪三项仍是决定是否允许调用的关键检查？（选择 3 项）

### Options

- A. Token 的 issuer、有效期以及撤销/失效状态符合策略。
- B. Token 的 audience 是当前 gateway/resource server。
- C. Scope/subject/tenant 允许当前 action 和目标 resource。
- D. Token 字符串长度超过 500 个字符。
- E. Claude 在自然语言中称该用户为管理员。

### Correct Answer

A、B、C。

### Explanation

A 检查凭证是否仍有效且来自可信签发方，B 阻止为其他服务签发的 token 被横向复用，C 完成 action 与对象级授权。D 不是安全语义。E 是模型生成内容，不能替代已验证 claims 和服务端 policy。

## 7. 401、403 与恢复策略

### Question

Agent 调用受保护工具：第一次因 access token 过期收到 401；刷新后身份有效，但因缺少 `calendar:write` scope 收到 403。最佳处理是什么？（选择 1 项）

### Options

- A. 对两种响应都无限重试，权限最终会自动出现。
- B. 401 走合法的重新认证/刷新路径；403 停止普通重试，按策略请求最小必要 scope 的 step-up consent，或拒绝操作。
- C. 将 403 改写成 200，让 Claude 自己判断是否继续。
- D. 使用另一个用户的宽权限 token 完成请求。

### Correct Answer

B。

### Explanation

B 区分身份凭证无效与已认证但权限不足，并保持最小 scope。A 会放大负载，且不会修复权限缺失。C 隐藏授权失败并破坏控制流。D 构成身份混用和越权，审计语义也错误。
