# 32 · 架构文档标准与实施指导交付：原创练习题

> 以下题目不是官方真题。题型和判断方式参考 Exam Guide 的 scenario-based / architecture decision 风格，只覆盖本知识点。

## Question 1（单选）

架构师为一个 Claude 客服 Agent 交付了一张包含 Web、Backend、Claude 和 Vector DB 的高层图。开发团队仍然不知道如何实现 tool authorization、失败处理或验收。最有效的下一步是什么？

### Options

A. 在同一张图上加入更多厂商 logo

B. 补充关键 runtime/failure scenarios、接口与 tool contracts、可测 acceptance gates、owner，以及 rollout/rollback guidance

C. 将高层图导出为更高分辨率的 PDF

D. 让每位开发者自行选择授权和错误处理方式

### Correct Answer

B

### Explanation

B 把概念图转成可实施的行为、契约、验证和责任。A、C 改善不了语义缺口；D 会导致安全控制与错误语义不一致。架构文档应帮助团队理解系统，也要给出实施与验收路径。

---

## Question 2（多选，选择三项）

在传统微服务文档之外，Claude Agent 架构最需要额外记录哪三类 artifact contract？

### Options

A. Prompt/template 的版本、变量、输出契约及关联 eval

B. Context/RAG 的数据来源、freshness、ACL、index compatibility 与异常处理

C. Tool 的 schema、authorization、side effect、idempotency、HITL 与错误语义

D. 每位开发者偏好的 IDE 主题

E. 团队所有聊天消息的完整副本

### Correct Answer

A、B、C

### Explanation

Claude 系统行为取决于 prompt、context 和 tools，三者必须像代码/API 一样有 owner、version、contract 和 verification。D 与架构行为无关；E 不仅噪声巨大，还可能泄露敏感信息，不能替代结构化决策与契约。

---

## Question 3（单选）

安全评审需要理解：外部文档中的 indirect prompt injection 如何进入 context、诱导 tool call，又在哪一步被 authorization policy 和人工审批阻断。哪种文档最合适？

### Options

A. 只列出使用的编程语言版本

B. 一张只显示系统名称的 context diagram

C. 标出 trust boundaries、消息与状态变化的 runtime/sequence view，并链接 tool authorization 与 HITL contract

D. 只提供月度成本报表

### Correct Answer

C

### Explanation

C 能显示攻击输入、context、model decision、tool request 和确定性控制的时序及边界。Context diagram 可提供范围，但单独使用不足以解释运行时攻击路径。A、D 不回答安全评审的问题。

---

## Question 4（单选）

一个文档把“当前生产架构”和“未来目标架构”混在同一张图中，没有状态标记。运维团队因此误以为新的 fallback 已经部署。最佳修正是什么？

### Options

A. 保留原图，在会议中口头解释即可

B. 明确分离或标注 current、target 与 migration views，并为每个状态记录版本、实施 gate 和 owner

C. 删除所有 target-state 信息

D. 将所有箭头改成不同颜色，但不提供 legend

### Correct Answer

B

### Explanation

B 消除状态歧义，并把目标状态连接到实施责任与验证。口头知识不可持续；删除目标状态会失去规划价值；无 legend 的颜色仍然含糊，也无法证明功能已经部署。

---

## Question 5（多选，选择三项）

架构师把一个高风险 Claude 决策支持系统拆成 implementation work packages。哪三项最能使这些 packages 真正可执行？

### Options

A. 明确 affected building blocks、interfaces、data/prompt/tool contracts 和 dependencies

B. 为每个增量定义 quality/safety acceptance criteria、observability evidence 与 owner

C. 定义 staged rollout、rollback 和失败时的 fallback

D. 只给出最终目标架构图，不规定实施顺序

E. 以“使用最佳实践”作为唯一完成标准

### Correct Answer

A、B、C

### Explanation

A、B、C 分别覆盖要改什么、怎样证明完成、怎样安全上线。D 留下大量隐含决策；E 不可测、不可审计。实施指导必须把 architecture intent 转成具体契约、门槛和恢复机制。

---

## Question 6（单选）

合规要求规定“不得跨租户返回检索内容”。团队已有 server-side tenant filter，但文档没有对应测试或责任人。哪项改进最直接暴露并关闭该缺口？

### Options

A. 在文档中增加更多 LLM 定义

B. 建立 requirement → architecture element → control → verification/evidence → owner 的 traceability mapping

C. 只在 system prompt 中加入“不要泄露数据”

D. 等待生产事故后再补测试

### Correct Answer

B

### Explanation

B 会显示控制存在但验证和 ownership 缺失，并推动 negative ACL test 与证据归属。C 不能替代确定性的 tenant authorization；A 与缺口无关；D 将可预防风险推迟到事故之后。

---

## Question 7（单选）

为了让 handoff 文档“自包含”，工程师建议把生产 API key、完整敏感 prompt 和最近的用户 traces 一并提交到 architecture repository。最佳回应是什么？

### Options

A. 接受，因为完整文档优先于 secret management

B. 只要 repository 是 private 就可以无条件接受

C. 文档记录 secret/config 的引用、owner、分类和获取流程；凭证保留在受控 secret store，trace/prompt 按最小化与访问控制处理

D. 把 key 做 Base64 编码后提交

### Correct Answer

C

### Explanation

C 保留实施所需的可发现性，但不复制敏感值。Private repository 也不等于最小权限的 secret store；Base64 不是加密。架构文档本身也必须符合数据分类和访问控制。

---

## Question 8（单选）

团队频繁修改 tool schema、prompt 和 deployment configuration，但架构文档每季度才人工更新，已多次误导 on-call。最好的长期改进是什么？

### Options

A. 在文档首页加入“可能过期”的免责声明

B. 将相关文档、diagram source、schema 和 ADR 纳入 version control 与 PR change policy，并用自动检查和 owner review 管理 drift

C. 停止维护所有架构文档，只读源码

D. 每季度生成更长的 PDF

### Correct Answer

B

### Explanation

B 把文档更新纳入发生变更的同一流程，并对可自动核对的 schema、链接和 artifact references 执行检查。A 只承认问题；C 会让非开发角色和跨系统行为失去共同模型；D 增加体积但没有改变漂移机制。
