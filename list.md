# Claude Certified Architect – Professional (CCAR-P) 备考知识点清单

> 依据官方 Exam Guide v1.0（2026年7月生效）第6节逐条拆解，共 43 个知识点，按领域权重排序。
> 标记说明：🟢 你已有底子，只需换语言对齐 | 🟡 有相关经验，需补 LLM/Claude 专属方法论 | 🔴 全新知识，需要从零学

---

## Domain 3 · Integration — 19%（权重最高，优先啃）

- [ ] 🔴 **1. MCP（Model Context Protocol）架构与使用场景**
- [ ] 🟡 **2. Agent-to-Agent 通信模式**
- [ ] 🟡 **3. API/CLI 集成模式对比**
- [ ] 🟡 **4. 工具/Agent 能力配置的"capability bloat"评估与最小权限原则**
- [ ] 🟡 **5. Agentic 系统的鉴权/授权（Auth/Authz）架构设计**
- [ ] 🟡 **6. 准确率与延迟的权衡分析（accuracy-latency trade-off）**
- [ ] 🟡 **7. 大规模场景下的可观测性设计（日志、追踪、监控）**
- [ ] 🔴 **8. RAG 流水线设计：分块（chunking）策略**
- [ ] 🟡 **9. RAG 流水线设计：索引/向量库策略**
- [ ] 🔴 **10. 按数据形态/查询模式选择检索策略（混合检索、语义 vs 关键词）**
- [ ] 🔴 **11. Progressive disclosure（渐进式披露）vs 单体上下文策略**

---

## Domain 1 · Solution Design & Architecture — 17%

- [ ] 🟢 **12. 业务问题 → Claude 解决方案的映射框架**
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
| 🟢 强项，快速对齐 | 13 | 你已有等价经验，主要是换 Claude 语境的术语 |
| 🟡 有基础，需补方法论 | 15 | 工程直觉在，需要补 LLM/Claude 专属做法 |
| 🔴 全新知识 | 15 | 需要从零系统学习 |

**建议击破顺序**：优先扫掉 Domain 3（Integration，19% 权重 + 最多🔴项）里的 MCP、RAG chunking、检索策略这几个纯新知识点；同步补 Domain 2 的 Claude 模型选型和 Prompt Caching；Domain 6 基本可以直接跳过或快速过一遍，把省下来的时间全砸在 🔴 项上。

## 学习资源建议
- 官方 Anthropic 文档（Claude API、MCP、Skills、Prompt Engineering 指南）是核心一手资料
- 动手做一个端到端 Claude 项目（RAG + 评估 + 可观测性），把清单里的知识点在实操中串起来记忆
- 每学完一个 🔴 项，回到 Exam Guide 第8节的样题风格，自己写一道类似难度的题目检验理解
