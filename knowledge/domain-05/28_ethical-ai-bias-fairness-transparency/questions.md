# 原创练习题：伦理 AI（偏见、公平性与透明度）

> 以下均为原创备考题，不是官方真题。题目只覆盖本知识点，并以 Exam Guide 的 scenario-based、architecture decision 与 trade-off 风格设计。

## Question 1 — 平均准确率掩盖群体伤害

一家企业使用 Claude 对求职材料生成 shortlist recommendation。离线测试总体准确率为 94%，但对一种非主流语言撰写的简历只有 71%，且这组申请者更常被错误淘汰。团队希望以总体准确率达标为由上线。最佳架构决策是什么？

### Options

A. 上线，因为总体准确率是唯一客观指标  
B. 增加一条“AI 可能出错”的免责声明后上线  
C. 暂停或限制该用途，补充该语言数据和评估，分析 false negative harm、worst-group performance，并设置分群发布门槛  
D. 从输入中删除语言字段即可消除差异

### Correct Answer

**C**

### Explanation

**C 正确。** 招聘属于 consequential use case，平均值不能掩盖特定群体的严重 false negative。应评估完整系统，明确公平目标、改进数据/流程并以 group/worst-group guardrail 决定是否部署。

**A** 忽略伤害分布。**B** 的披露不能修复差别结果。**D** 无法消除文本中的语言信号、proxy 或模型能力差异，也会让审计更困难。

---

## Question 2 — Fairness metric 选择

一个贷款辅助系统的主要伦理风险是：在实际具备还款能力的申请人中，某群体被错误拒绝的比例明显更高。哪项最直接对应这一问题？

### Options

A. 只比较各群体申请人数  
B. 比较各群体实际正类中的 true positive rate，并结合 false negative harm 评估 equal opportunity  
C. 要求所有群体获得完全相同的平均文本长度  
D. 只确认总体 calibration，不再检查任何错误率

### Correct Answer

**B**

### Explanation

**B 正确。** 题目关心“真正合格的人是否同等容易获得正向结果”，与 equal opportunity/TPR disparity 直接对应。最终选择仍要结合 ground truth 质量、法律要求和其他 guardrails。

**A** 不测结果或错误。**C** 与分配伤害无关。**D** 的 calibration 不能单独回答合格者被漏过是否公平，且不同 fairness goals 可能冲突。

---

## Question 3 — Counterfactual evaluation（多选，选择 3 项）

团队要测试 Claude 客服助手是否对不同身份群体提供一致服务。哪三项做法最合理？

### Options

A. 构造成对 prompts，只改变一个身份线索，比较答案质量、语气、拒答和升级路径  
B. 对语言、地区、年龄及相关 intersectional slices 报告样本量、置信区间和 worst-group 结果  
C. 让 domain experts 判断某些属性变化是否合理地应影响答案  
D. 只运行一个公开 bias benchmark，并把通过结果视为生产系统永久公平  
E. 删除所有投诉记录，避免评价者受到负面案例影响

### Correct Answer

**A、B、C**

### Explanation

**A、B、C 正确。** Paired tests 可隔离不相关的差别待遇；slice/intersectional analysis 防止平均数掩盖问题；domain review 避免错误地要求对任务相关差异也输出完全相同结果。

**D** 忽略企业自己的 prompt、RAG、tools 和用户环境，也忽略 drift。**E** 删除了重要 production signal，破坏问责和改进闭环。

---

## Question 4 — Protected attribute 的处理

招聘团队提出：为了证明系统公平，删除性别、年龄等 protected attributes，并且不在任何评价环境中保存这些信息。架构师应选择什么方案？

### Options

A. 接受；模型看不到字段就不可能产生群体差异  
B. 在明确授权和隐私控制下，将审计属性隔离在受控 evaluation 环境用于 disparity analysis，同时禁止不相关属性进入业务决策路径并检查 proxies  
C. 把所有敏感属性直接加入 production prompt，让 Claude自行决定何时使用  
D. 只检查模型输出中是否出现身份词，不评估结果差异

### Correct Answer

**B**

### Explanation

**B 正确。** Fairness through unawareness 通常不足，因为文本和其他特征可能成为 proxy；完全不保留审计属性还会使群体差异无法测量。合理方案需要 purpose limitation、隔离、least privilege、retention 和审计。

**A** 忽略 proxy 与模型推断。**C** 扩大了敏感数据使用且缺乏确定性约束。**D** 只能发现显式措辞，不能发现 allocative 或 quality-of-service harm。

---

## Question 5 — 可行动的透明度

一家保险公司用 Claude 生成高风险理赔 recommendation。它已在网页底部写明“本服务使用 AI”，但客户无法知道哪些材料被使用、谁做最终决定或如何纠错。最重要的改进是什么？

### Options

A. 公开模型的完整 raw chain-of-thought 作为唯一解释  
B. 提供与受众匹配的说明，包括 AI 的角色、关键 evidence/provenance、重要限制、责任主体和更正/人工复核/申诉路径  
C. 增加更多营销性描述，说明模型非常先进  
D. 隐藏 AI 的参与，以免客户不信任结果

### Correct Answer

**B**

### Explanation

**B 正确。** Meaningful transparency 的目标是让客户理解系统角色并能采取行动。对于 consequential decision，还必须与 accountability 和 recourse 结合。

**A** 的 chain-of-thought 可能不忠实、不稳定或泄露敏感/安全信息；可验证 sources、rules 和 decision record 更可靠。**C** 不是风险说明。**D** 进一步削弱知情、纠错与信任。

---

## Question 6 — Safety 与公平的 trade-off

一个 safety classifier 能有效阻止有害请求，但生产数据显示，包含特定群体身份词的无害咨询被拒绝的比例显著更高。最佳下一步是什么？

### Options

A. 保持不变，因为 safety 指标永远优先且无需衡量 over-refusal  
B. 完全移除 classifier，以保证所有群体相同通过率  
C. 分群评估 harmful-request detection 与 benign over-refusal，补充代表性 edge cases，校准控制并同时设置 safety 与 access guardrails  
D. 要求受影响用户改用主流身份词

### Correct Answer

**C**

### Explanation

**C 正确。** 这是 safety 与 fair access 的真实 trade-off，不能只优化一端。应测量每个群体的 true harm detection 和 benign refusal，改善数据/阈值/流程，并保持对真正有害请求的控制。

**A** 把不均等服务当作无需管理的副作用。**B** 可能制造严重安全风险。**D** 把系统缺陷转嫁给用户并加剧排斥。

---

## Question 7 — Model card 的边界

某 Claude model 的公开 system card 显示其在多个 bias evaluations 上表现良好。企业准备将它接入自有 RAG、招聘规则和自动排序工具。项目经理认为无需再做伦理评估。最佳判断是什么？

### Options

A. 正确；model-level evaluation 自动覆盖任何 downstream application  
B. 错误；system card 是 model selection 和 known-limitations 的输入，企业仍须评价自己的数据、prompt、RAG、tools、human workflow 与真实结果  
C. 正确；只要供应商公开评估，客户就不再对部署结果负责  
D. 错误；公开 system card 没有任何架构价值，应完全忽略

### Correct Answer

**B**

### Explanation

**B 正确。** System card 提供重要的 capabilities、safety evaluation 和 deployment context，但 downstream system 会引入新的数据、规则、工具和影响。Ethical AI 是 end-to-end property，责任不能外包给基础模型文档。

**A、C** 都犯了 model-to-system inheritance fallacy。**D** 又走向另一个极端：可靠的 model documentation 是风险评估的重要输入，只是不是充分条件。

---

## Question 8 — 持续伦理治理（多选，选择 3 项）

一个多语言公共服务助手通过了上线前 fairness review。之后团队更换模型、更新 RAG corpus，并新增自动 tool action。哪三项最能维持负责任部署？

### Options

A. 对版本化的 paired/slice/intersectional regression suite 重新评估，并记录指标与阈值变化  
B. 监控 refusal、task success、complaint、abandonment、human override 和 appeal overturn 等 production signals  
C. 对新增 autonomy、数据源和受影响群体重新执行 impact/boundary review，并保留 rollback trigger  
D. 沿用首次签署的 fairness approval，因为同一产品名称代表风险不变  
E. 只监控总体 token cost，因为伦理风险无法测量

### Correct Answer

**A、B、C**

### Explanation

**A、B、C 正确。** 模型、知识源和 action capability 的变化会改变行为、证据基础与影响范围，因此必须重新 Map/Measure/Manage，并把真实用户结果接回治理闭环。

**D** 把 sign-off 当作永久属性。**E** 错误地放弃可测量的 group、process 和 recourse signals；无法可靠量化的风险也应被记录并通过定性研究、限制部署或人工控制管理。

