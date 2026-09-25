# 19 · Evaluation Dataset 与 Mixed-method Framework：原创练习题

> 以下题目不是官方真题。题型和判断方式参考 Exam Guide 的 scenario-based / architecture decision 风格，只覆盖本知识点。

## Question 1（单选）

团队要为一个已上线的客服 agent 建立第一版 evaluation dataset。当前资源有限。哪种起步方式最有价值？

### Options

A. 等收集到数万条完美标注后才开始评估

B. 从产品验收要求、人工必测场景、support tickets 和真实失败中整理一组无歧义 cases，并逐步扩充

C. 只让另一个 LLM 随机生成大量 prompts，不检查来源和覆盖

D. 直接采用公开通用 benchmark，完全不加入业务场景

### Correct Answer

B

### Explanation

B 能快速形成与真实任务和已知风险相关的高信息量 suite，并建立“生产失败 → regression case”的闭环。A 不必要地延迟反馈；C 容易产生 synthetic monoculture；D 无法覆盖企业 workflow、tools、权限与业务标准。

---

## Question 2（多选，选择三项）

一个内部搜索助手只有“应该搜索”的测试，优化后几乎所有问题都会调用搜索工具。为修复 dataset 设计，最应该加入哪三类 case？

### Options

A. 应搜索的时效性或内部知识问题

B. 不应搜索、可直接根据已提供上下文回答的问题

C. 信息不足时应澄清或拒绝猜测的问题

D. 删除所有原有正例，只保留不搜索的 case

E. 把每个原 case 复制十遍，不改变分布

### Correct Answer

A、B、C

### Explanation

A/B 构成行为的正反覆盖，防止 under-triggering 与 over-triggering；C 覆盖第三种重要边界：既不盲搜也不编造。D 会把 one-sided problem 反向重现；E 增加数量但不增加覆盖。

---

## Question 3（单选）

团队每次修改 prompt 后都会查看 held-out test set 的失败样本并针对它们调参，直到分数提高。最大的评估风险是什么？

### Options

A. Test set 已逐渐变成 development set，分数可能高估泛化能力

B. Held-out set 使用次数越多，统计上一定越可靠

C. 只要不改变 label，就不会发生任何泄漏

D. 唯一风险是 API cost 增加

### Correct Answer

A

### Explanation

当开发决策反复利用 test failures 时，团队实际上对该集合过拟合。应保留真正受控的 holdout 或补充新样本。B、C 忽略信息泄漏；D 只看到运行成本，没看到 validity 损失。

---

## Question 4（多选，选择三项）

团队评估一个可执行退款的客服 agent。哪些 grader 组合最合理？

### Options

A. 用数据库 state check 验证退款是否存在、金额是否正确

B. 用 deterministic checks 验证高风险退款前是否完成身份验证和授权

C. 用带清晰 rubric 的 LLM grader 评估解释是否清楚、语气是否合适，并用人工样本校准

D. 只检查 agent 最后的文本是否包含“退款成功”

E. 强制所有正确方案逐字采用同一段回复

### Correct Answer

A、B、C

### Explanation

A 验证真实 outcome，B 检查必要的安全过程约束，C 处理开放式交互质量并进行可靠性校准。D 只相信 self-report，可能与真实状态不符；E 会拒绝措辞不同但正确的回答，过度 brittle。

---

## Question 5（单选）

一个 coding agent task 要求修复漏洞，但没有说明文件路径；grader 却只接受在某个隐藏路径生成脚本。Agent 以另一种正确结构修复并通过完整功能测试，却被判失败。最佳处理方式是什么？

### Options

A. 继续保留 grader，因为隐藏要求能增加难度

B. 修订 task/grader，使所有被检查的必要条件在任务契约中明确，并优先按 outcome 判定

C. 把该失败直接记为模型能力不足

D. 要求 agent 猜测 evaluator 的目录习惯

### Correct Answer

B

### Explanation

这是 task ambiguity 和 brittle grading，不是能力失败。B 恢复评估公平性和 construct validity；A、C、D 都会奖励猜 grader 而非解决真实任务。

---

## Question 6（多选，选择两项）

一个 customer-facing agent 单次 trial 的成功率为 80%。产品要求用户每次使用都应稳定成功。哪两项做法最合适？

### Options

A. 对关键 tasks 运行多次 trials，并报告每次成功率与一致性指标（如 pass^k）

B. 自动重试十次，只保留最好的一次且不报告重试

C. 固定并记录 retry policy、环境与配置，把失败和额外成本都计入

D. 只报告 pass@10，因为至少一次成功就等同可靠

### Correct Answer

A、C

### Explanation

A 直接测“每次都可靠”的要求；C 确保测试可解释，并保留真实 failure/cost。B cherry-pick 结果，D 的 pass@k 更适合“多次中一次成功即可”的任务，会夸大面向用户 agent 的可靠性。

---

## Question 7（单选）

团队有一套离线自动 eval，所有分数持续很好，但生产用户仍报告未覆盖的复杂失败。最合理的改进是什么？

### Options

A. 宣布用户反馈无效，因为自动 eval 更客观

B. 只增加现有 cases 的重复次数

C. 结合生产监控、用户反馈和 transcript review 识别分布缺口，将脱敏后的真实失败加入 regression/capability suites，并保留人工校准

D. 删除生产监控以避免与离线结果冲突

### Correct Answer

C

### Explanation

没有单一评估层能覆盖所有 failure。C 使用 mixed methodologies 发现 offline–production gap，并把新知识固化为可重复测试。A、D 忽略真实 outcome；B 只减少已有估计的方差，不增加任务覆盖。

---

## Question 8（多选，选择三项）

团队为 RAG agent 构建并行 evaluation harness。哪些措施最能提高结果的可复现性和有效性？

### Options

A. 每个 trial 从隔离的干净环境开始

B. 版本化 prompt、model/config、tool schema、corpus/index snapshot 与 graders

C. 保存 transcript 和最终环境 outcome，以区分 agent、grader 与基础设施 failure

D. 让所有 trials 共用可写数据库和缓存，不做清理

E. 只记录最终平均分，不保存 task 或 slice 结果

### Correct Answer

A、B、C

### Explanation

A 防止共享状态污染，B 使结果可重放和跨版本比较，C 支持公平性审查与根因定位。D 会引入相关 failure 或不公平提示；E 掩盖关键 slices，也无法审计评估是否测到了预期行为。
