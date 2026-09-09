# API/CLI 集成模式对比

> 学习序号 03 · 清单 #3 · Domain 3: Integration（19%）
> 核查日期：2026-09-09。考试依据：[Exam_Guide.md](../../../Exam_Guide.md) v1.0 第 6 节连接协议与集成机制选型目标；练习风格参考第 8 节样题。
> 前置：[01 MCP](../01_mcp-architecture-and-use-cases/knowledge.md)、[02 Agent-to-Agent](../02_agent-to-agent-communication-patterns/knowledge.md)。本课仅完成 API/CLI 对比，不展开 Claude Code 团队配置或完整鉴权课程。

## 1. 首先问：谁调用谁？

Claude 系统有两条经常被混淆的调用路径：

1. **应用调用模型**：你的 Java 后端通过 Claude API 请求模型响应，或者脚本启动 Claude Code CLI 执行工作。
2. **模型请求外部能力**：Claude 返回工具调用意图，应用再调用订单 API 或执行获准的命令，并把结果交回模型。

API（Application Programming Interface）是程序接口；CLI（Command-Line Interface）是命令行接口。SDK 则把 API 的请求、响应等封装成编程语言对象。它们不是互斥的网络协议：CLI 和 SDK 都可能在底层调用同一个 HTTP API。不能因为代码里有命令行，就认定它绕过了 API 的认证、计费或限流。

截至核查日，Anthropic 官方还提供 `ant` CLI 访问 Claude API；它体现的是“命令行形式的 API 客户端”。不要将所有 Claude 相关 CLI 都当成 Claude Code 的代理执行环境。[官方 ant CLI 文档](https://platform.claude.com/docs/en/cli-sdks-libraries/cli/quickstart)

## 2. 按责任比较，而不按缩写选答案

下表是本课的工程选型推理，不是官方强制选型规则。

| 方式 | 调用方负责什么 | 更适合什么 | 主要代价 |
|---|---|---|---|
| 直接 HTTP API | 请求构造、响应解析、状态和错误处理 | 已有 HTTP 基础设施，需精细控制网络行为 | 需维护序列化与协议适配 |
| 语言 SDK | 业务流程、权限、资源预算；核实 SDK 默认行为 | Java 等常驻后端中的模型集成 | SDK 版本、默认重试等也需管理 |
| 普通 CLI/API CLI | 启动进程、输入输出、退出状态、凭证和运行环境 | 运维脚本、CI 或只有成熟 CLI 的现有工具 | 进程开销、版本漂移、解析和交互问题 |
| Claude Code CLI | 委托已有 agent runtime；管理工作目录和允许能力 | 仓库分析、开发任务、脚本驱动的代理工作 | agent 可能多步执行；需约束文件、命令与会话 |

API 不天然更准确，CLI 不天然更慢。实际延迟包含模型生成、工具执行、网络往返、进程启动和重试。应测量符合真实负载的延迟与成功率，不能凭接口名字推导 SLA。

## 3. Claude API 的调用边界

Messages API 提供直接模型交互，适合自行控制流程。多轮上下文由调用方组织并随请求提供，不能假设远端自动记住上一次 HTTP 请求的对话。[Messages API 文档](https://platform.claude.com/docs/en/build-with-claude/working-with-messages)

对自定义 **client tool**，典型过程如下：

1. 应用提交用户问题和可用工具定义。
2. Claude 返回 assistant 消息中的 `tool_use`，包含调用 ID、工具名和参数。
3. 应用校验参数和权限，执行真实业务 API/受控命令。
4. 应用保留前述 assistant 消息，再以 user 消息中的 `tool_result` 回传结果，用 `tool_use_id` 关联调用。
5. Claude 根据结果继续回答或提出下一次调用。

失败结果可设置 `is_error: true`。这里是 Claude API 字段，不要与 MCP 的 `isError` 混用。Server tools 的执行责任不同，不能把所有工具都按 client tool 处理。[官方工具调用处理](https://platform.claude.com/docs/en/agents-and-tools/tool-use/handle-tool-calls)

模型输出工具调用，只表示“请求执行”；应用才拥有执行权。例如 `get_order(order_id)` 必须以已认证用户检查该订单的可见性。JSON 参数合法不代表业务请求获准。

## 4. CLI 接入怎样成为可靠的程序接口？

CLI 并不必然要求人工操作。Claude Code 官方支持 `claude -p` 非交互运行，并提供 `--output-format json` 等输出方式；`--json-schema` 可用于请求符合指定结构的输出。[官方非交互运行文档](https://code.claude.com/docs/en/headless)

教学示例，未在本次任务执行，不涉及真实模型调用：

```bash
claude -p '根据提供的文本列出待确认的问题' --output-format json
```

这条命令只展示入口，**不是完整的生产沙箱配置**。集成程序还应明确：工作目录、可用工具、非交互认证、配置来源、超时、退出码、结果中的失败信息，以及期望字段。机器解析应针对结构化输出，不能截取终端里“看起来像答案”的最后一行。有效 JSON 也不等于内容真实或业务任务成功。

### Java 包装现有命令的思路

假设企业已有只读订单查询 CLI，名称为 `orderctl`。下面是自定义教学伪代码，`orderctl` 不是 Anthropic 产品：

```java
// executable、subcommand 和工作目录由应用固定；orderId 先作业务校验。
var pb = new ProcessBuilder("/opt/company/bin/orderctl", "get",
                            "--id", validatedOrderId, "--format", "json");
pb.directory(approvedWorkingDirectory);
// 启动后：并发、限量读取 stdout/stderr；设置 deadline；超时终止进程。
// 最后：检查退出状态、解析 JSON、验证业务结果，回传对应 tool_result。
```

将参数逐个传递可以避免把用户文本解释成 shell 语法；仍需防止危险选项、任意路径和工具本身的高权限行为。不要把模型返回的整段字符串交给 `sh -c`。stdout/stderr 如果没有被及时消费，子进程还可能阻塞；输出大小和进程数量也要限制。

## 5. 如何作出架构决定？

| 约束 | 优先考虑 | 判断依据 |
|---|---|---|
| 多租户在线服务，只需分类/提取，要求明确请求边界 | API/SDK | 便于集成既有身份、并发和请求生命周期管理 |
| CI 对仓库作多步分析，已有获准的 Claude Code 运行环境 | 非交互 Claude Code CLI | 复用现有代理工作能力，核实输出和权限契约 |
| 内部系统没有 API，但有稳定、支持 JSON 的 CLI | 受控 CLI adapter | 可满足需求，需承担进程和环境管理 |
| 多个 AI 应用需要复用同一组业务工具 | 可在 API/CLI 外包装 MCP | 复用发现/调用接口；底层业务仍通过原入口执行 |
| 对方需要自主规划、长任务状态和补充输入 | Agent-to-agent 机制 | 接入对象具有独立任务语义；HTTP 只是承载方式之一 |

这些倾向受题目约束影响。例如在线服务也可以通过受控 job worker 使用 CLI，但应解释收益如何抵偿进程隔离和运维复杂度。CI 也可以直接调用 API；“CI”本身不构成必须采用 CLI 的理由。

## 6. Failure modes：定位到真正出错的层

| 现象 | 先查什么 | 对应处理 |
|---|---|---|
| CLI 在本机成功，在 CI 卡住 | 是否等待登录/权限输入，工作目录和配置是否相同 | 预配置非交互环境；不通过全面放权掩盖配置缺口 |
| CLI 升级后解析失败 | 输出格式与 schema 是否改变 | 固定兼容版本，优先机器输出，验证字段 |
| 模型声称工具成功，实际没执行 | tool 调度与结果关联是否正确 | 以真实执行结果为准，不把调用意图当成功 |
| 收到 API 429 | 限流与重试策略 | 有界退避、遵守服务反馈；避免多个层次叠加重试 |
| HTTP 401/403 | 身份、权限或资源配置 | 修正凭证/授权；连续盲重试一般不会恢复权限 |
| 写操作超时后重复扣款 | 下游已执行但响应丢失 | 查询业务状态、使用业务幂等键；不要盲重放完整 agent 流程 |

Claude API 的错误类型与请求 ID 可用于定位服务侧问题；流式响应开始后也可能出现错误，所以收到初始 HTTP 成功响应不能证明整次生成已经成功。[官方错误文档](https://platform.claude.com/docs/en/api/errors)

重试预算应端到端考虑：若 SDK 重试、应用重试和队列重试各自都启动新模型工作，成本与副作用可能被放大。相同输入也不保证得到完全相同的模型输出；可重放日志用于排障，不能当作业务幂等保证。

## 7. 考试判断与核查边界

先识别调用方向和接入对象，再找硬约束：运行环境、是否需要多步代理能力、机器输出、权限和失败契约。最后在能满足约束的方案中比较维护成本与延迟；优先排除“换大模型即可解决认证/解析问题”等不对应根因的干扰项。

Guide 未指定必须背诵某个 CLI 名称或参数，本课命令仅为当前官方产品示例。清单 #3 与 Guide 本次无新增冲突。资料只使用上文五个官方页面，核查于 2026-09-09；未引用社区文章确定考试重点。

**Needs verification**：无阻碍本课概念与答案的未核实核心事实。具体项目的 CLI/SDK 版本、认证方式、权限配置、SLA 与重试默认值须部署时验证；本次未安装或执行 Claude CLI、未进行真实 API 测试，也未断言模型价格和限额。

下一步：完成 [questions.md](questions.md)，重点解释为何错误选项没有解决题目约束。
