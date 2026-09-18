# Claude Certified Architect – Professional (CCAR-P) 备考知识点清单

> 依据官方 Exam Guide v1.0（2026年7月生效）第6节逐条拆解，共 43 个知识点，按领域权重排序。
> 标记说明：🟢 你已有底子，只需换语言对齐 | 🟡 有相关经验，需补 LLM/Claude 专属方法论 | 🔴 全新知识，需要从零学

> 阅读顺序：按知识点文件夹的 `01_`、`02_`、`03_`…前缀阅读。编号按实际学习顺序在所有 Domain 间连续递增；原清单条目编号不一定等于学习序号。每个知识点先读 knowledge.md，再做 questions.md。

---

## Domain 3 · Integration — 19%（权重最高，优先啃）

- [x] 🔴 **1. MCP（Model Context Protocol）架构与使用场景** — 学习序号 01 · 2026-09-07 · [学习](knowledge/domain-03/01_mcp-architecture-and-use-cases/knowledge.md) · [8 道练习](knowledge/domain-03/01_mcp-architecture-and-use-cases/questions.md)
- [x] 🟡 **2. Agent-to-Agent 通信模式** — 学习序号 02 · 2026-09-08 · [学习](knowledge/domain-03/02_agent-to-agent-communication-patterns/knowledge.md) · [8 道练习](knowledge/domain-03/02_agent-to-agent-communication-patterns/questions.md)
- [x] 🟡 **3. API/CLI 集成模式对比** — 学习序号 03 · 2026-09-09 · [学习](knowledge/domain-03/03_api-cli-integration-patterns/knowledge.md) · [6 道练习](knowledge/domain-03/03_api-cli-integration-patterns/questions.md)
- [x] 🟡 **4. 工具/Agent 能力配置的"capability bloat"评估与最小权限原则** — 学习序号 04 · 2026-09-10 · [学习](knowledge/domain-03/04_capability-bloat-and-least-privilege/knowledge.md) · [7 道练习](knowledge/domain-03/04_capability-bloat-and-least-privilege/questions.md)
- [x] 🟡 **5. Agentic 系统的鉴权/授权（Auth/Authz）架构设计** — 学习序号 05 · 2026-09-11 · [学习](knowledge/domain-03/05_agentic-authentication-and-authorization/knowledge.md) · [7 道练习](knowledge/domain-03/05_agentic-authentication-and-authorization/questions.md)
- [x] 🟡 **6. 准确率与延迟的权衡分析（accuracy-latency trade-off）** — 学习序号 06 · 2026-09-12 · [学习](knowledge/domain-03/06_accuracy-latency-tradeoffs/knowledge.md) · [6 道练习](knowledge/domain-03/06_accuracy-latency-tradeoffs/questions.md)
- [x] 🟡 **7. 大规模场景下的可观测性设计（日志、追踪、监控）** — 学习序号 07 · 2026-09-13 · [学习](knowledge/domain-03/07_observability-at-scale/knowledge.md) · [7 道练习](knowledge/domain-03/07_observability-at-scale/questions.md)
- [x] 🔴 **8. RAG 流水线设计：分块（chunking）策略** — 学习序号 08 · 2026-09-14 · [学习](knowledge/domain-03/08_rag-chunking-strategies/knowledge.md) · [7 道练习](knowledge/domain-03/08_rag-chunking-strategies/questions.md)
- [x] 🟡 **9. RAG 流水线设计：索引/向量库策略** — 学习序号 09 · 2026-09-15 · [学习](knowledge/domain-03/09_rag-index-and-vector-store-strategies/knowledge.md) · [7 道练习](knowledge/domain-03/09_rag-index-and-vector-store-strategies/questions.md)
- [x] 🔴 **10. 按数据形态/查询模式选择检索策略（混合检索、语义 vs 关键词）** — 学习序号 10 · 2026-09-16 · [学习](knowledge/domain-03/10_retrieval-strategies-by-data-and-query/knowledge.md) · [7 道练习](knowledge/domain-03/10_retrieval-strategies-by-data-and-query/questions.md)
- [x] 🔴 **11. Progressive discovery（渐进式发现）vs 单体上下文策略** — 学习序号 11 · 2026-09-17 · [学习](knowledge/domain-03/11_progressive-discovery-vs-monolithic-context/knowledge.md) · [7 道练习](knowledge/domain-03/11_progressive-discovery-vs-monolithic-context/questions.md)

---

## Domain 1 · Solution Design & Architecture — 17%

- [x] 🟢 **12. 业务问题 → Claude 解决方案的映射框架** — 学习序号 12 · 2026-09-18 · [学习](knowledge/domain-01/12_business-problem-to-claude-solution-mapping/knowledge.md) · [7 道练习](knowledge/domain-01/12_business-problem-to-claude-solution-mapping/questions.md)
- [ ] 🟢 **13. 端到端架构设计范式（输入→处理→输出→反馈闭环）**
- [ ] 🔴 **14. 架构模式选型：Workflow vs Agentic vs Augmented LLM（Anthropic 官方分类法）**
- [ ] 🟡 **15. 多 Agent 系统设计与编排策略**
- [ ] 🟢 **16. 复杂问题的拆解技术**
- [ ] 🟢 **17. 业务价值支柱对齐框架（效率/转型/生产力/成本/性能SLA）**

---

## Domain 4 · Evaluation, Testing & Optimization — 16%

- [ ] 🔴 **18. LLM 评估指标体系（准确率/延迟/成本/安全/security）**
- [ ] 🔴 **19. 评估数据集设计与混合方法论测试框架**
- [ ] 🟡 **20. LLM 系统的 A/B 测试方法**
- [ ] 🔴 **21. 故障诊断：prompt failure vs 幻觉 vs 模型不匹配**
- [ ] 🟡 **22. Token 用量/延迟/成本优化技巧**
- [ ] 🟢 **23. 基于日志和可观测性工具的系统性能监控**

---

## Domain 5 · Governance, Safety & Risk Management — 14%

- [ ] 🔴 **24. Guardrails 与安全控制实现模式**
- [ ] 🔴 **25. LLM 系统的风险/局限性/失效模式分类学**
- [ ] 🟡 **26. Human-in-the-loop 验证策略设计**
- [ ] 🟡 **27. 面向 AI 系统的合规要求（GDPR/HIPAA/FedRAMP 在 LLM 场景下的应用）**
- [ ] 🔴 **28. 伦理 AI 考量（偏见、公平性、透明度）**

---

## Domain 6 · Stakeholder Communication & Lifecycle Management — 14%（你的强项区）

- [ ] 🟢 **29. 结构化发现与需求收集框架**
- [ ] 🟢 **30. 架构决策与权衡的沟通方法（ADR 等）**
- [ ] 🟢 **31. 干系人反馈闭环与 SLA 对齐管理**
- [ ] 🟢 **32. 架构文档标准与实施指导交付**
- [ ] 🟢 **33. AI 方案生命周期模型（发现→设计→交接→监控→迭代）**

---

## Domain 2 · Claude Models, Prompting & Context Engineering — 13%

- [ ] 🔴 **34. Claude 模型家族选型权衡（Opus/Sonnet/Haiku 的成本/延迟/能力对比）**
- [ ] 🟡 **35. System Prompt 设计与 Guardrail 模式**
- [ ] 🟡 **36. Prompt 工程技巧：zero-shot / few-shot / chain-of-thought**
- [ ] 🟡 **37. 上下文窗口管理与 Token 优化（长上下文策略、"lost in the middle"问题）**
- [ ] 🔴 **38. Prompt Caching 机制**
- [ ] 🟡 **39. 模块化 Prompt 设计**
- [ ] 🔴 **40. Claude Skills 功能**

---

## Domain 7 · Developer Productivity & Operational Enablement — 7%（权重最低，放最后）

- [ ] 🔴 **41. Claude Code 团队环境配置**
- [ ] 🟡 **42. AI 辅助开发者工作流模式**
- [ ] 🟢 **43. Claude 系统的调试与运维问题排查**

---

## 统计概览

| 标记 | 数量 | 含义 |
|---|---|---|
| 🟢 强项，快速对齐 | 11 | 你已有等价经验，主要是换 Claude 语境的术语 |
| 🟡 有基础，需补方法论 | 17 | 工程直觉在，需要补 LLM/Claude 专属做法 |
| 🔴 全新知识 | 15 | 需要从零系统学习 |

**建议击破顺序**：结合 Exam_Guide.md 的明确要求、Domain 权重、前置依赖与实际完成情况，每次选择一个知识点。优先考虑 Domain 3 的 MCP、RAG 等薄弱项，必要时先补前置基础。Domain 6 占 14%，已有经验可帮助加快学习，但不能跳过指南目标；颜色只代表学习基础，不代表考试重要性。

## 执行记录

- 2026-09-18：完成 #12 业务问题 → Claude 解决方案映射框架（学习序号 12，两个文件、7 道练习）。直接对齐 Guide Domain 1 的 “Translate business problems into Claude-based AI solutions”；核查 Anthropic Building Effective AI Agents 与 success criteria/evaluation 官方文档。无新增清单冲突；本课限定在业务目标、任务契约、数据/真相源、风险/autonomy、价值与验收的映射，没有提前替代后续端到端架构或 workflow/agent 模式选型知识点。
- 2026-09-17：完成 #11 Progressive discovery vs monolithic context（学习序号 11，两个文件、7 道练习）。直接对齐 Guide Domain 3 的独立目标；核查 Anthropic context engineering、Tool Search、Agent Skills 与 MCP Tools 官方资料。无新增清单冲突；说明 Guide 的 progressive discovery 与官方资料 progressive disclosure/just-in-time context 的关系，并明确核心安全规则不可延迟加载。Domain 3 的清单知识点至此全部完成，本次未生成其他 Domain 内容。
- 2026-09-16：完成 #10 按数据形态/查询模式选择检索策略（学习序号 10，两个文件、7 道练习）。直接对齐 Guide Domain 3 的独立 retrieval strategy 目标；核查 Anthropic Contextual Retrieval/Cookbook 与 OpenSearch hybrid/fusion/optimization 官方文档。无新增清单冲突；明确 exact identifier、semantic intent、structured query 与 hybrid 的适用边界，不把 Anthropic 实验数字或某一搜索产品配置当成考试必背结论。
- 2026-09-15：完成 #9 RAG 索引/向量库策略（学习序号 09，两个文件、7 道练习）。对齐 Guide Domain 3 的 RAG indexing 目标与 Sample 3 的文档刷新故障诊断；核查 Anthropic Embeddings、Contextual Retrieval 及 pgvector 官方 exact/ANN/filter 文档。无新增清单冲突；厂商实现仅作权衡例证，不将当前 embedding 型号、dimension 或 index 默认参数列为考试必背内容。
- 2026-09-14：完成 #8 RAG chunking 策略（学习序号 08，两个文件、7 道练习）。直接对齐 Guide Domain 3 的 RAG pipeline chunking/indexing 目标，并限定本课只深入分块；核查 Anthropic Contextual Retrieval 与 Claude Search Results/Citations 官方资料。无新增清单冲突；明确 retrieval chunk 与 Claude citation block 的层次差异，不把 Anthropic 2024 实验参数或效果数字当成通用默认值或考试要求。
- 2026-09-13：完成 #7 大规模 Claude 系统可观测性设计（学习序号 07，两个文件、7 道练习）。直接对齐 Guide Domain 3 的 observability challenges 与 monitoring strategies at scale 目标；核查 Anthropic API request ID、errors/rate limits、Usage & Cost API、Claude Code OTel 及 OpenTelemetry traces/sampling 官方资料。无新增清单冲突；明确与 Domain 4 持续性能监控的交集和侧重点，不把 Claude Code 的 telemetry 实现泛化为所有 Claude API 应用的默认能力。
- 2026-09-12：完成 #6 准确率与延迟架构权衡（学习序号 06，两个文件、6 道练习）。对齐 Guide Domain 3 的 accuracy-latency trade-offs 与配置论证目标；核查 Anthropic 的 agent 架构、并行工具、streaming 和 prompt caching 文档。无新增清单冲突；教材不把具体模型性能或价格当成 Guide 要求。
- 2026-09-11：完成 #5 Agentic 系统 Authentication / Authorization 架构（学习序号 05，两个文件、7 道练习）。对齐 Guide Domain 3 的 AuthN/AuthZ requirements 与 security gaps 分析目标；核查 Anthropic Claude API 认证/调用边界及最新版 MCP 2026-07-28 Authorization、Security Best Practices。无新增清单冲突；具体 Claude 凭证类型、workspace 字段和 OAuth/MCP 规范细节仅作当前实现例证，不扩展为 Guide 必背要求。
- 2026-09-10：完成 #4 capability bloat 评估与最小权限（学习序号 04，两个文件、7 道练习）。直接对齐 Guide Domain 3 的能力配置目标及 Sample 1 rationale；核查 Anthropic 工具定义/执行边界、agent 设计原则和 Claude Code 权限/安全文档。无新增清单冲突；Claude Code 规则语法只作当前实现例证，不扩展为 Guide 必背要求。
- 2026-09-09：完成 #3 API/CLI 集成模式对比（学习序号 03，两个文件、6 道练习）。对齐 Guide Domain 3 的集成机制选型目标；核查 Claude Messages API、工具结果处理、API 错误、Claude Code 非交互模式及 ant CLI 官方文档。无新增清单冲突；具体 CLI 名称与参数作为实现示例，不扩展为指南明确要求的必背内容。
- 2026-09-08：完成 #2 Agent-to-Agent 通信模式（两个文件、8 道练习）。依据 Guide 第 6 节 Domain 3 的集成机制选型目标，使用 Anthropic 的 orchestrator–worker 实践与 A2A 官方规范解释模式和边界。Guide 使用的是泛称 agent-to-agent，未点名 A2A Protocol，因此不将具体协议字段提升为必考范围。本次未发现新的清单冲突，未完成其他知识点。
- 2026-09-07：完成 #1 MCP 架构与使用场景（两个文件、8 道练习）。依据 Guide 第 6 节 Domain 3 的集成机制选型目标和第 8 节样题设计。将 #11 英文术语对齐 Guide 的 progressive discovery；移除“Domain 6 可以跳过”的建议；按实际条目修正颜色统计为 11/17/15。本次未完成其他知识点。最新 MCP 协议行为与旧版区别在教材中标注，不将版本变化擅自扩为考试重点。

## 学习资源建议

- 官方 Anthropic 文档（Claude API、MCP、Skills、Prompt Engineering 指南）是核心一手资料
- 动手做一个端到端 Claude 项目（RAG + 评估 + 可观测性），把清单里的知识点在实操中串起来记忆
- 每学完一个 🔴 项，回到 Exam Guide 第8节的样题风格，自己写一道类似难度的题目检验理解
