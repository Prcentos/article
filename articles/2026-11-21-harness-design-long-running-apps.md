# Harness Design for Long-Running Application Development
- 日期：2026-11-21（收录）
- 原文链接：https://www.anthropic.com/research/harness-design-long-running-apps
- 分类：engineering（Labs 团队，作者 Prithvi Rajasekaran）

## 讲了什么

这是 Anthropic Labs 团队的一篇工程实践长文，核心问题只有两个：怎么让 Claude 做出有品味的界面，怎么让它无人值守地构建完整应用。作者的思路是从 GAN 里借结构：把系统拆成 **generator（生成者）+ evaluator（评审者）**，让"做事的"和"评判的"由两个独立 agent 担任，前端设计场景跑了 5~15 轮迭代循环，全栈场景扩展成 planner / generator / evaluator 三 agent 架构，单次自主构建长达 4 小时。

文中最扎实的部分是失败归因：长任务跑偏主要源于两件事。其一是"上下文焦虑"，Sonnet 4.5 在接近自认为的上下文上限时会提前收尾、质量塌方，压缩（compaction）保留连续性但治不了焦虑，彻底重置上下文 + 结构化交接物（context reset + handoff artifact）才是解法。其二是自我评估失效，agent 给自己打的分系统性地偏高，且对 LLM 生成的内容天然宽容；把评审拆出去、单独调教一个"多疑"的 evaluator，比让生成者自我批评可行得多。

结尾作者做了反向实验：随着 Opus 4.5/4.6 能力上移，逐个拆掉 harness 里不再起作用的组件（sprint 分解被整个砍掉），发现评估器的价值取决于任务是否超出模型独自可靠完成的边界，模型越强这条边界越往外推。4 小时、124 美元做出一个可玩的浏览器 DAW 是全文最抓人的结果。

## 关键洞察

1. **GAN 类比的底层是"生成与评判必须解耦"**。人类工程里开发和 QA 本来就是两个角色，LLM 单体自审时，同一段参数既负责"产出"又负责"挑刺"，而模型的对齐方向天然偏向讨喜、一致，自我批评几乎没有梯度可用。拆出独立 evaluator 后，"让它变严格"成了一个可独立调优的小问题，few-shot 打分示例就能校准它的口味并抑制分数漂移。这是用结构冗余换可训练性。
2. **"上下文焦虑"是模型行为而非资源问题**。它说明模型对自身状态有元认知，且这种元认知会反过来改变行为策略（提前收工）。reset 之所以有效，不是因为省 token，而是给了一个干净的"自我模型"；这提示长任务的可靠性瓶颈可能在模型的自我建模上，而不在上下文窗口大小本身。
3. **harness 的每个组件都是对模型能力短板的一次下注**。"every component encodes an assumption about what the model can't do"——组件会随着新模型发布而迅速贬值。于是 harness 工程变成一个持续再平衡的过程：新模型落地后先做减法（拆掉不再承重的那层），再做加法（利用新能力堆出之前做不到的事）。评估器值不值得跑，判断标准是任务相对"模型独力可靠边界"的位置，这是成本决策不是架构决策。
4. **评审标准的措辞本身就在塑造输出**。"museum quality" 这类短语让设计向特定视觉收敛；哪怕零轮反馈，仅凭标准的存在，第一轮产出就已显著好于无提示基线。标准即提示词，提示词即品味。

## 落到你手上

1. **给 HanaAgent 的长任务加"独立评审"环节，成本几乎为零**：用 workflow 跑多步任务时，最后一环加一个 `access: "read"` 的 agent 节点，prompt 就写"你是挑刺的 QA，列出没做完、有 stub、有 bug 的地方，不要客气"。陈亮现在的 subagent 用法是做完即交付，缺的正是这个解耦的评判者；本地 Qwen 27B 跑生成 + 同一个模型跑评审，质量上限不高但结构性收益（拦住 stub 和半成品）值得先验证一轮。
2. **验证"上下文重置 vs 压缩"**：拿 .114 上的 Qwen 27B 做一个小实验，同一个多步任务（比如搭个小服务）跑两次，一次连续跑，一次中途清空会话、用一份结构化交接文档（已完成/未完成/下一步/已知问题）开新会话续跑，对比质量。Qwen 有没有 Sonnet 4.5 那种"上下文焦虑"，直接看它的收尾行为就知道，这正好呼应他上周遇到的上下文过大导致服务满载的问题。
3. **新模型发布后做一次 harness 减法**：HanaAgent 里凡是"帮模型兜底"的脚手架（任务拆解、格式约束、重试逻辑），每换一次底层模型就重新审一遍哪些不再承重。这是原文给的最便宜的一条纪律，不需要任何新代码，只需要每次升级模型时花半小时读一遍 agent 的 trace。

## 原文摘录

> "Separating the agent doing the work from the agent judging it proves to be a strong lever to address this issue."

> "Every component in a harness encodes an assumption about what the model can't do, and those assumptions are worth stress testing, both because they may be incorrect, and because they can quickly go stale as models improve."

> "My conviction is that the space of interesting harness combinations doesn't shrink as models improve. Instead, it moves, and the interesting work for AI engineers is to keep finding the next novel combination."
