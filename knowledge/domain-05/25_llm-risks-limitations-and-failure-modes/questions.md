# 25 · LLM 系统的风险、局限性与失效模式：原创练习题

> 以下题目不是官方真题。题型和判断方式参考 Exam Guide 的 scenario-based / architecture decision 风格，只覆盖本知识点。

## Question 1（单选）

企业政策文档昨晚更新，但客服助手今天仍回答旧政策。哪项是最合理的第一步？

### Options

A. 立即将其归类为模型 hallucination 并更换最大模型

B. 检查 ingest/index version、cache、实际 retrieved chunks 和最终 prompt，再判断是 retrieval、context 还是 generation failure

C. 提高 temperature，让模型探索新答案

D. 只在 system prompt 中写“务必使用最新信息”

### Correct Answer

B

### Explanation

错误答案是 symptom，不是根因。文档刷新后仍答旧内容优先指向 ingestion、index、cache 或 retrieval 链路，必须检查端到端证据包。A 过早归因且成本高；C 会增加变化，不能修复知识供应链；D 不能让缺失的新文档进入 context。

---

## Question 2（单选）

团队写下：“LLM 可能 hallucinate，所以可能给出不存在的退款条款，导致客服错误退款并产生财务损失。”哪项映射最准确？

### Options

A. Hallucination 是 impact；错误退款是 limitation；财务损失是 trigger

B. Hallucination/无依据生成是局限或原因类别；不存在的条款是 failure mode；财务损失是 impact

C. 三者都是同一个 failure mode，无需区分

D. 财务损失是 model limitation

### Correct Answer

B

### Explanation

B 保留了 cause/limitation → observable failure → impact 的因果链，便于选择检测与控制。A 和 D 混淆概念；C 会让 risk register 无法定位根因、影响与责任人。

---

## Question 3（多选，选择四项）

为一个能读邮件并调用内部工具的 agent 建立风险分类时，哪些失效应纳入系统级分析？

### Options

A. 邮件中的 indirect prompt injection 改变工具选择

B. Tool timeout 后重试导致非幂等写操作重复

C. 长任务 compaction 丢失关键约束

D. 平均准确率稳定，但某语言 slice 的严重错误显著增加

E. 只记录模型输出措辞不够优雅，忽略所有下游动作

### Correct Answer

A、B、C、D

### Explanation

A–D 分别覆盖对抗性内容、分布式副作用、context/state 与 slice drift，说明风险属于完整系统而非只属于模型。E 把注意力限制在表面文本，遗漏权限、状态和业务影响。

---

## Question 4（单选）

退款工具的 JSON 始终符合 schema，但 agent 偶尔对错误客户的订单退款。最准确的风险归因是什么？

### Options

A. Structured Outputs 失败，因为任何业务错误都属于 schema failure

B. 参数结构可能完全正确，但 identity/ownership、object-level AuthZ 或 semantic validation 失效

C. 这是纯粹的语言偏见问题

D. 只要把 temperature 设为 0，授权风险就消失

### Correct Answer

B

### Explanation

Schema-valid 只说明结构满足契约，不证明对象属于当前用户、动作获授权或业务状态正确。A 混淆结构与语义；C 与场景不符；D 不能替代确定性 authorization boundary。

---

## Question 5（多选，选择三项）

一个 agent 的平均任务成功率很高，但会执行不可逆的高影响操作。风险排序时，除发生频率外还应优先考虑哪些因素？

### Options

A. 影响严重度与 blast radius

B. 可检测性、发现时延与可逆性

C. 现有控制强度以及控制后的 residual risk

D. UI 使用的品牌颜色

E. 只看全局平均成功率

### Correct Answer

A、B、C

### Explanation

低频但高严重度、难发现且不可逆的事件仍可能最高优先。现有控制不代表风险归零，必须评估 residual risk。D 不影响本场景的风险；E 会掩盖严重长尾和关键 slices。

---

## Question 6（单选）

可信员工让 research agent 总结一个外部网页。网页隐藏指令，诱导 agent 上传内部文件。应如何分类？

### Options

A. 可信员工发起的 direct jailbreak

B. Indirect prompt injection；威胁来自 agent 处理的不可信第三方内容，并被过宽能力放大

C. 单纯的过时知识

D. 仅是输出格式不一致

### Correct Answer

B

### Explanation

用户可以是可信的，第三方内容仍可能由攻击者控制。隐藏指令属于 indirect injection，文件上传能力扩大了后果。A 错认攻击主体；C、D 都没有解释目标改变和越权副作用。

---

## Question 7（多选，选择三项）

哪些条目最适合作为可执行 risk register 的组成部分？

### Options

A. 具体 scenario/trigger、failure mode、cause 和 impact

B. Existing controls、detection signal、owner 与 residual-risk decision

C. 严重度、可检测性、可逆性和受影响范围

D. 只写“AI 有风险”，不指定组件和责任人

E. 因为已有 guardrail，把 residual risk 固定写成零

### Correct Answer

A、B、C

### Explanation

A–C 让风险可定位、可排序、可验证并可追责。D 不能驱动架构决策；E 忽略控制失效、误报漏报与未知风险，也混淆 inherent risk 和 residual risk。

---

## Question 8（单选）

团队的离线 benchmark、生产监控和用户投诉给出相互矛盾的信号。最佳判断是什么？

### Options

A. 只相信离线 benchmark，因为它可复现

B. 只相信生产投诉，因为它来自真实用户

C. 分析各信号的覆盖与盲区，结合 automated eval、生产 slices、trace/transcript 和人工复核定位 failure mode

D. 取三个指标的简单平均数并关闭调查

### Correct Answer

C

### Explanation

离线 eval 可能不代表真实分布，生产监控可能缺少 ground truth，投诉又稀疏且自选择。C 利用互补证据重建具体失败路径。A、B 都把单一信号绝对化；D 会抹去指标含义、严重度和分群差异。
