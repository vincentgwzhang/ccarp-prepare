# 21 · Prompt Failure、Hallucination 与 Model Mismatch：原创练习题

> 以下题目不是官方真题。题型和判断方式参考 Exam Guide 的 scenario-based / architecture decision 风格，只覆盖本知识点。

## Question 1（单选）

一个员工福利助手在知识源迁移后，开始向新员工推荐已经停用的保险计划；model identifier、prompt 和 latency 均未变化。最合理的第一检查点是什么？

### Options

A. 直接换成能力最强的模型

B. 检查新数据源同步、index、corpus version、metadata filter 与实际 retrieved chunks 是否返回了现行计划

C. 降低 temperature，并认定问题已经解决

D. 在 prompt 中重复五次“不要 hallucinate”

### Correct Answer

B

### Explanation

故障紧随 knowledge-source migration 出现，而 model/prompt 未变，最强因果线索指向数据同步与 retrieval/indexing。应先查看实际 chunks 和版本。A、C、D 都在未验证输入 evidence 前调整生成层，可能掩盖而不能修复 stale context。

---

## Question 2（单选）

分类服务收到充分且正确的输入，但 prompt 只写“按适当方式分类”，没有类别定义、边界或输出 contract。多个模型都稳定产生不同标签；加入明确 label rubric 和 canonical examples 后原模型通过。最可能的根因是什么？

### Options

A. Model mismatch

B. Retrieval index corruption

C. Prompt failure

D. API authentication failure

### Correct Answer

C

### Explanation

任务 contract 原本含糊，保持 model/data 不变、仅澄清 prompt 后稳定修复，是 prompt failure 的强证据。A 缺乏“清晰 prompt 下原模型仍失败而候选模型通过”的对照；B、D 与场景不符。

---

## Question 3（多选，选择三项）

为诊断一个 agent 偶发地声称“退款成功”但数据库没有退款记录，最关键的三类证据是什么？

### Options

A. 完整 rendered messages、model/config 与 request/trace ID

B. 只保存 UI 中最后一句“退款成功”

C. Tool definition、实际 tool calls/arguments、results/errors 及最终数据库 outcome

D. Retry/fallback、stop reason、parser 与 raw output

E. 只询问开发者记忆中的 prompt 内容

### Correct Answer

A、C、D

### Explanation

A、C、D 可以区分 prompt、model、tool execution、错误吞没、截断和后处理问题，并验证真实 outcome。B 只有 symptom；E 可能与实际 rendered request 不同，无法形成可重放证据。

---

## Question 4（单选）

一个摘要助手在 source 中没有任何金额时，自信生成了“交易价值 500 万欧元”。最佳初始缓解方案是什么？

### Options

A. 要求答案必须包含一个具体金额，以保持格式完整

B. 只多采样十次，选择最常出现的金额

C. 允许明确表示信息不足，要求关键 claim 引用 source quote；找不到支持证据时撤回该 claim

D. 隐藏 source，让模型更自由地使用预训练知识

### Correct Answer

C

### Explanation

这是 unsupported factual claim。C 让不确定性成为合法结果，并将 claim 与可核验 evidence 绑定。A 会强迫猜测；B 的一致性不等于真实性；D 移除 grounding，风险更高。

---

## Question 5（单选）

团队在一个复杂、多文档推理 suite 上进行了如下控制实验：retrieval 和 graders 已验证；oracle context 已提供；三种清晰 prompt 均未使当前模型达标；候选模型在相同 prompt、context 和 tools 下稳定通过，同时仍满足 latency/cost guardrails。最合理的诊断是什么？

### Options

A. 一定是 index 没刷新

B. 有充分证据支持 model mismatch

C. 只能继续无限扩写 prompt

D. 因为任何模型都会偶尔失败，所以不能作架构决策

### Correct Answer

B

### Explanation

场景控制了 data、context、prompt、tools 与 grader；候选模型在相同条件下稳定改善且满足非功能门槛，这构成 model mismatch 的合理证据。A 与已验证 retrieval 冲突；C 会增加脆弱性；D 忽略了代表性 eval 的决策价值。

---

## Question 6（多选，选择三项）

一个 tool-using agent 经常把 `customer_id` 填入 `order_id`。为区分 prompt/tool-contract failure 与 model mismatch，最有价值的三项动作是什么？

### Options

A. 检查 tool description、parameter names、examples 与两个 ID 的语义边界

B. 不看 trace，直接把所有错误标为 hallucination

C. 保持 model 不变，改为不易混淆的参数名和 validation，再在 failure set 上重跑

D. 同时更换 model、prompt、tool schema 和 database client

E. 在相同 tool contract 和 cases 下进行受控 model comparison

### Correct Answer

A、C、E

### Explanation

A 定位接口歧义，C 是 tool-contract-only 实验，E 是 model-only 对照，组合后可以判断归因。B 没有证据；D 同时改变多个变量，即使恢复也无法知道根因。

---

## Question 7（单选）

模型的 raw response 是包含全部四个字段的合法 JSON，但应用 parser 丢掉了 `risk_level`，UI 因此显示“不完整”。应如何分类和处理？

### Options

A. Hallucination；要求模型多解释

B. Model mismatch；换更大模型

C. Post-processing/integration failure；修 parser，并为 schema mapping 增加 deterministic test

D. Prompt failure；把字段名重复十遍

### Correct Answer

C

### Explanation

模型原始输出正确，损坏发生在 parser 层，因此是 deterministic integration failure。A、B、D 都把责任错误归给生成层。正确修复应落在最早出错的组件并留下回归测试。

---

## Question 8（多选，选择三项）

团队确认正确 policy chunk 已进入 context，但助手仍偶发捏造例外条款。哪些做法最适合作为下一轮诊断与改进？

### Options

A. 将输出拆成 atomic claims，检查每条是否被 chunk 支持、反驳或无法验证

B. 只保留一次回答正确的 replay，宣布问题消失

C. 保持 context/model 不变，测试允许“不知道”、仅使用 supplied documents、quote/citation verification 的 prompt variant

D. 在 versioned failure set 上进行多 trials，并在 prompt-only 试验后再做受控 model swap

E. 将 temperature 变化、模型升级、prompt 重写和 retriever 更新一次性上线

### Correct Answer

A、C、D

### Explanation

A 建立 groundedness 证据，C 隔离 prompt/grounding 策略，D 处理非确定性并为 model mismatch 提供后续对照。B 是 cherry-picking；E 同时改变多个变量，无法诊断，也难以回归验证。
