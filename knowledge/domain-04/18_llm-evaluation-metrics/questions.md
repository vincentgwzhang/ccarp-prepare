# 18 · LLM 评估指标体系：原创练习题

> 以下题目不是官方真题。题型和判断方式参考 Exam Guide 的 scenario-based / architecture decision 风格，只覆盖本知识点。

## Question 1（单选）

一个内容审核分类器处理的请求中，99.5% 为正常内容，0.5% 为需要拦截的严重违规内容。模型把所有请求都判为正常，总体 accuracy 为 99.5%。架构师下一步最应该怎么做？

### Options

A. 接受该模型，因为 accuracy 已接近 100%

B. 只增加测试集规模，仍只报告 accuracy

C. 分别报告违规类的 precision、recall 和 F1，并按严重性设定 recall / unsafe-compliance 门槛

D. 改用平均输出长度作为质量指标

### Correct Answer

C

### Explanation

类别极度不平衡时，总体 accuracy 会被正常类支配。C 能直接暴露严重违规内容全部漏检的问题，并把高风险 false negative 纳入门槛。A 会接受一个对核心任务完全失效的系统；B 不能修复 metric mismatch；D 与分类正确性无关。

---

## Question 2（单选）

团队评估一个基于内部政策文档的开放式问答系统。多个答案可以有不同措辞，但每个事实 claim 都必须得到文档支持。最合适的主要质量指标是什么？

### Options

A. 与单一 reference answer 的逐字符 exact match

B. 输出 token 数越多越好

C. Claim-level groundedness / citation support，加上答案相关性与必要要点覆盖率

D. 只测模型请求是否返回 HTTP 200

### Correct Answer

C

### Explanation

该任务允许多种措辞，核心风险是无依据 claim。C 直接对应 groundedness、relevance 和 completeness。A 会错误惩罚语义等价答案；B 是活动量而非质量；D 只说明调用成功，不能证明内容正确。

---

## Question 3（多选，选择三项）

一个实时 Claude agent 的用户抱怨“偶尔卡住很久”。团队目前只报告平均模型调用时延。哪三项改进最有诊断价值？

### Options

A. 报告端到端 p95/p99 latency

B. 分解 queue、retrieval、model、tool、validation 和 retry 的耗时

C. 分别观察 TTFT、总完成时间、timeout/retry rate，并按 tool path 分群

D. 继续只看平均值，因为分位数会增加仪表盘复杂度

E. 将 streaming 开启状态直接当作总时延达标

### Correct Answer

A、B、C

### Explanation

A 暴露 tail latency，B 定位慢在哪个阶段，C 区分感知响应、完整完成和异常路径。D 会继续掩盖少量但严重的慢请求；E 混淆 streaming 带来的 perceived latency 改善与 total completion latency。

---

## Question 4（单选）

方案 X 每次请求成本较低，但任务失败和重试较多；方案 Y 单次调用更贵，却一次完成率高且人工复核较少。哪项指标最适合比较真实经济性？

### Options

A. 单次 Claude API 请求标价

B. 每月请求总数

C. 包含模型、tools、retries、基础设施和人工复核的 cost per accepted successful task

D. 平均输出 token 数，不考虑结果是否成功

### Correct Answer

C

### Explanation

C 使用成功业务结果作为分母，并纳入端到端成本，能够识别“调用便宜但失败昂贵”的方案。A 和 D 忽略成功率及外围成本；B 是规模指标，不能比较单位经济性。

---

## Question 5（多选，选择两项）

团队正在评估一个可读取邮件并调用财务工具的 agent。哪些指标最直接衡量 Security，而不是一般 Safety？

### Options

A. 间接 prompt injection 导致未授权 tool call 的成功率

B. 跨用户敏感数据泄露率

C. 回复语气的礼貌程度

D. 正常用户对拒绝措辞的满意度

E. 摘要的 ROUGE 分数

### Correct Answer

A、B

### Explanation

A 测试对抗性内容是否突破授权边界，B 测试 confidentiality / tenant isolation，都是 security outcomes。C、D 可能属于体验或 safety/refusal quality；E 是摘要相似度 proxy，不衡量攻击抵抗或权限边界。

---

## Question 6（单选）

一个医疗助手在普通问题上质量优秀，但 red-team 中有一次输出了真实患者信息。团队用一个 composite score 将 accuracy、latency、cost、privacy 平均后，系统仍然高于发布门槛。最佳决策是什么？

### Options

A. 发布，因为高 accuracy 足以抵消一次隐私失败

B. 将严重隐私/数据泄露设为 hard guardrail 或 release blocker，调查 failure path 后重新评估

C. 增加 latency 权重，让 composite score 更稳定

D. 删除该失败样本以免影响平均分

### Correct Answer

B

### Explanation

严重 security/privacy failure 不能被其他高分平均掉。B 使用 hard guardrail 保持不可妥协的风险边界，并要求定位与回归验证。A、C 都允许无关维度抵消严重事件；D 破坏评估完整性。

---

## Question 7（多选，选择三项）

团队计划使用 LLM-as-a-judge 对客服回复的专业性和问题解决质量评分。哪些做法最能提高 grader 的可信度？

### Options

A. 为每个维度提供清晰 rubric 和结构化评分定义

B. 用专家标注样本校准 judge，并持续检查一致性和失败切片

C. 隐去候选模型身份和无关元数据，减少偏差

D. 认为 judge 也是 LLM，所以其评分天然客观且无需验证

E. 用一个模糊问题“这个答案好吗？”替代分维度 rubric

### Correct Answer

A、B、C

### Explanation

A 降低标准歧义，B 验证 judge 与目标判断是否一致，C 减少品牌/位置等无关偏差。D 把 grader 当作 ground truth；E 会造成评分不可解释、不稳定，也难以定位改进方向。

---

## Question 8（单选）

一个全球客服助手总体 task success 为 94%，超过 90% 门槛。但西班牙语高价值账户只有 68%，英语普通账户占绝大多数。最合理的评估结论是什么？

### Options

A. 总体分数达标，因此所有用户群都可以发布

B. 只要扩大英语样本，总体分数会更精确

C. 必须按语言、账户风险和任务类型分群，并为关键 slice 设独立门槛；总体平均不能掩盖该失败

D. 删除西班牙语样本，因为它们不是多数流量

### Correct Answer

C

### Explanation

C 识别了 aggregation masking：高频英语样本掩盖关键少数群体的明显退化。若该 slice 具有高业务价值或风险，应有独立 threshold。A 忽略分群失败；B 会进一步稀释问题；D 制造不具代表性的评估。
