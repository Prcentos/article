# Demystifying evals for AI agents
- 日期：2026-11-24
- 链接：https://www.anthropic.com/research/demystifying-evals-for-ai-agents
- 分类：engineering
- 作者：Anthropic（Engineering）

## 讲了什么

这篇是 Anthropic 对 Agent 评测（evals）的方法论总结。核心主张：Agent 的自主性和灵活性让它难以评测，但正是这些能力让它有用，而解决"改了一处坏三处"的唯一办法是把预期行为编码成可重复运行的测试。

文章先定义了评测的最小构件：task（单个测试用例）、trial（一次尝试，因输出有随机性要跑多次）、grader（打分逻辑，分代码型、模型型、人工型三类）、transcript（完整轨迹记录）、outcome（环境最终状态，注意"订舱成功"要看数据库里有单，而不是 Agent 说了句"订好了"）。

然后按 Agent 类型给了评测配方：coding agent 用确定性测试（SWE-bench 式跑单测）+ LLM rubric 评代码质量；conversational agent 要第二个 LLM 扮演用户，同时检查任务完成和交互质量；research agent 用 groundedness（论断有出处）+ coverage（关键事实覆盖）+ 信源质量三层检查，并用精确匹配兜住有标准答案的部分；computer use agent 在真实/沙箱环境跑，检查后端状态而不只是页面。

非确定性部分给了两个关键指标：pass@k（k 次里至少对一次，衡量能力上限）和 pass^k（k 次全对，衡量可靠性），一个 75% 单试成功率的 Agent 跑 3 次全对只有约 42%。

落地路线是五步：20-50 个来自真实失败的任务就够起步；从手工测试清单和 bug tracker 转化任务；任务要写得"两个专家独立判分结果一致"，每个任务配一个参考解证明可解、且 grading 配置正确；任务集要正反平衡（既测该搜索时的搜索，也测不该搜索时的克制）；harness 环境每次 trial 从干净状态开始，避免共享状态污染结果。文中还点了两个典型翻车案例：Opus 4.5 在 CORE-Bench 上 42% 到 95%，纯粹是修掉了 grader 过刚和任务歧义；METR 发现有些任务让"遵守指令的模型"反而得分更低。

## 关键洞察

1. **outcome 和 transcript 的区分是评测的第一性原则**。"模型说了什么"和"环境发生了什么"是两回事，Agent 声称完成不等于完成。这背后是对"声明 vs 状态"的分离：把验证锚定在环境状态（数据库行、文件、服务响应）上，模型的话只算证据不算结论。这是分布式系统里"不要相信副本，读主库"的同一思想。
2. **pass@k 和 pass^k 回答的是两个不同问题：能力 vs 可靠性**。同一组 trial 数据，k 增大时一个趋向 100%、一个趋向 0。工具型 Agent（一次成功就行）看 pass@k，客服型 Agent（每次都行）看 pass^k。多数团队只报 pass@1，等于把"会不会"和"稳不稳"混在一个数字里。
3. **0% pass@100 通常是任务坏了，不是模型弱了**。这条经验极其实用：在 frontier 模型上如果一个任务跑 100 次全挂，先怀疑自己的 grader、任务描述、环境假设。对应到工程上就是"测试失败先查测试本身"，但 Agent 场景下这个直觉更难，因为失败看起来总在"模型行为"那层。
4. **评结果不评路径**。文章明确反对校验"工具调用序列必须按某顺序执行"，因为 Agent 经常走出评测设计者没预料到的合法路径。评 produced（产出了什么）不评 path（怎么走到那），本质上是给随机系统留自由度，只对确定性约束下手。这和"契约测试测行为不测实现"是同一个取舍。
5. **评测是产品与模型之间的接口**。有了 eval 集，新模型出来时你可以用几天而不是几周完成"验证 → 调 prompt → 升级"，还能免费获得延迟、token 成本、错误率的回归基线。评测的价值不在"发现 bug"，而在把"换模型"这个高不确定性动作变成低风险动作。

## 落到你手上

1. **给 HanaAgent/openhanako 建 20-50 个回归任务起步**。不用等几百个任务：把你踩过的坑（503 上下文过载、长会话截断、微信推送 24h 窗口、BMC 输出解析）各写成 1-2 个 task，配确定性 grader（输出文件存在、状态字段正确）+ 一个 LLM rubric 兜主观项。下次改模型或改 prompt，先跑一遍这个套件，再决定上不上。
2. **本地 Qwen3.5-27B 推理服务上验证 pass@k vs pass^k 差异**。挑 3-5 个固定任务（比如解析一段日志生成摘要、修一段带错代码），每个跑 5 次，分别算 pass@5 和 pass^5。27B 本地模型的单试成功率和"稳定可用"之间差距多大，这个实验一个晚上就能出数据，也直接回答"要不要再垫一层重试逻辑"这个工程决策。
3. **把"outcome 检查"用进自建服务的验收脚本**。chenliangorient.xyz 上的推理服务现在验收大概率是"curl 通了就是好了"。可以加一层状态检查：发一个有标准答案的请求（比如"17*23 等于多少"），断言响应里的数字正确、延迟在阈值内，每天跑一次作为回归信号。成本是几条 curl，换来的是"服务到底还正常吗"的客观答案。

## 原文摘录

> "The outcome is the final state in the environment at the end of the trial. A flight-booking agent might say 'Your flight has been booked' at the end of the transcript, but the outcome is whether a reservation exists in the environment's SQL database."

> "If your agent has a 75% per-trial success rate and you run 3 trials, the probability of passing all three is (0.75)³ ≈ 42%."

> "So as not to unnecessarily punish creativity, it is often better to grade what the agent produced, not the path it took."

> "With frontier models, a 0% pass rate across many trials (i.e. 0% pass@100) is most often a signal of a broken task, not an incapable agent."
