# An update on recent Claude Code quality reports（"变笨"不是玄学：一次产品级退化的三连故障复盘）

- **日期**：2026-04-23（收录于 2026-11-19）
- **原文链接**：https://www.anthropic.com/research/april-23-postmortem
- **分类**：engineering / 产品复盘（推理档位、prompt 缓存、system prompt 管理）

## 【讲了什么】

用户大规模反馈"Claude 变笨了"，Anthropic 排查出三个互相独立的变更叠加导致：(1) 3 月 4 日把默认 reasoning effort 从 high 降到 medium 以降低延迟，4 月 7 日回滚；(2) 3 月 26 日上线"闲置超 1 小时的会话清除旧思考"的缓存优化，实现有 bug，变成整个会话每一轮都在丢历史思考，4 月 10 日修复；(3) 4 月 16 日在 system prompt 里加了一条"工具调用间文本 ≤25 词、最终回复 ≤100 词"的简洁指令，扩大评测后发现 Opus 4.6/4.7 都掉 3%，4 月 20 日回滚。

三个变更影响不同流量、发生在不同时间，叠加后看起来像"全面的、不一致的退化"。最难的排查过程：内部 dogfooding、代码评审、单测、e2e 全都没拦住 bug #2，两个无关实验（内部消息队列实验 + 思考展示方式的正交改动）掩盖了症状，拖了一周多才定位。最终用 Opus 4.7 对肇事 PR 做回溯 code review 找到了 bug（4.6 找不到）。事后补偿：全体订阅者重置用量。

## 【关键洞察】

1. **reasoning effort 是 test-time compute 曲线上的选点问题，而"选哪一点当默认"是产品问题不是技术问题。** 曲线本身（思考越多→越聪明、越慢、越贵）是模型层的客观事实，但默认值定在哪，本质是在"多数任务的边际收益"和"重度用户的体验悬崖"之间站队。他们的教训：internal evals 显示 medium 在多数任务上"稍笨但快很多"，数据上成立，却忽略了高难度任务用户（恰恰是最付费、最敏感的人群）掉出悬崖。默认值应该对"你最不能失去的用户"优化，而不是对任务分布的平均值优化。

2. **缓存、上下文管理、思考记录三者的交互会产生"隐性成本放大器"。** bug #2 的杀伤力不在"丢思考"本身，而在链式反应：每轮丢思考 → 每轮都 cache miss → 用量烧得比预期快 → 用户觉得"又慢又贵又健忘"。一个效率优化同时制造了三个用户可感知的退化，且彼此无法单独归因。自建推理服务同理：KV cache 淘汰策略、context 截断策略、thinking 保留策略，任何一个改动都会通过缓存命中率放大成延迟和成本问题，改之前要推演这条链。

3. **单行 system prompt 可以通过数周内部测试然后仍然砸掉 3% 的智能。** 简洁性指令对"容易任务"几乎无害（所以内部测试全过），只在"难任务"上暴露（思考被截短、中间结论丢失、工具调用间的信息传递断裂）。这暴露了 eval 覆盖的结构性盲区：如果评测集里难任务占比不够，prompt 回归是测不出来的。他们的修复动作值得抄：system prompt 每个字都进版本管理，逐行 ablation，每次改动跑全模型矩阵的评测，"可能影响智能"的变更强制 soak 期 + 灰度。

4. **"看起来像全面退化"本身就是诊断信号：多半是多变更叠加，而不是模型坏了。** 三个 bug 各自只影响一个流量切片，单看任何一个都"不算严重"。当你（或你的用户）观察到退化呈现"时好时坏、因人而异、没有规律"的形态，第一反应不应该是"模型不行了"或"某个大变更搞的"，而是去查最近一周所有变更（代码、配置、prompt、参数），做二分到单一变量。他们自己最初也误判成"正常反馈波动"，拖了数周。

## 【落到你手上】

1. **把 .114 上 Qwen3.5-27B 的上下文/输出限制当"曲线选点"来测，而不是拍脑袋。** 你刚把总长定到 262144、单次输出 65536。建议跑一个最小任务集（3~5 个你真实会用的任务：长文总结、代码修 bug、多轮问答），分别在"当前配置"和"减半配置"下各跑一遍，记录首字延迟、总时长、结果是否可用。如果减半后任务完成率没掉，你的默认值就该降一半（省钱省电），把大配置留给显式 opt-in 的复杂任务。这是他们这次教训的最小可执行版。

2. **给 HanaAgent 或本地推理服务的 system prompt 建一个"一次只改一处"的纪律 + 固定回归题集。** 准备 3~5 个固定问题（含至少 1 个难任务），存成一个脚本。每次改 prompt 或参数后跑一遍，对比输出。你现在 503/中断问题排了两天，如果期间改过多个配置，用这个脚本二分能省时间。成本约等于零。

3. **排查"服务变慢/变笨"时先拉变更清单再怀疑硬件。** 如果哪天 .114 的响应质量突然不稳定，先 git log 最近一周（代理配置、模型参数、system prompt、docker compose）再动别的。"时好时坏"= 多因素叠加的指纹，单变量假设通常错。

## 【原文摘录】

- "In our internal evals and testing, medium effort achieved slightly lower intelligence with significantly less latency for the majority of tasks... This was the wrong tradeoff."
- "Instead of clearing thinking history once, it cleared it on every turn for the rest of the session."
- "One of these evaluations showed a 3% drop for both Opus 4.6 and 4.7. We immediately reverted the prompt as part of the April 20 release."
