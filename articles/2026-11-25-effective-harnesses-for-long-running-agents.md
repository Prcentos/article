# Effective harnesses for long-running agents

- 日期：2026-11-25（收录日）
- 原文链接：https://www.anthropic.com/research/effective-harnesses-for-long-running-agents
- 分类：engineering（engineering 栏目，作者 Justin Young，Code RL & Claude Code 团队）

## 讲了什么

长时程 agent 的核心矛盾：任务要跨小时甚至跨天，但每个 context window 都是失忆的。把 agent 当成轮班工程师——每个新人上班时不记得上一班发生了什么。作者指出光靠 compaction（上下文压缩）不够，直接让 Opus 4.5 在循环里跑 "build a clone of claude.ai" 会稳定失败，失败有两种形态：一是想一口气做完，上下文烧在实现中途，留下半成品无文档；二是后期 agent 看到已有进展，直接宣布完工。

解法是一个两角色 harness：initializer agent 只在第一次运行，负责搭好环境——init.sh 启动脚本、claude-progress.txt 进展日志、git 初始提交、一份 200+ 条的 feature list JSON（全部初始标为 failing）；coding agent 每次新会话只做三件事：读进展文件和 git log 搞清楚现状、跑一遍端到端冒烟测试确认没被留坑、挑一个最高优先级的 failing feature 做完再收工，收尾时 git commit + 更新 progress 文件，要求代码保持"可以直接 merge 到 main 的干净状态"。测试环节则强制 agent 用 Puppeteer 浏览器自动化像真实用户一样操作界面验证，而不是只看代码和单元测试。

## 关键洞察

1. **状态必须住在模型外面。** 这是最底层的一条。compaction 是有损压缩，信息丢没丢下一个 agent 无从知道；而 git 历史、progress 文件、feature list 是无损的状态外化。这和分布式系统的原则同构：system of record 必须独立于任何单个进程，进程可以随便挂，状态不能。agent 的"进程"就是 context window，它的"磁盘"就是这些文件。
2. **两个失败模式其实是同一个信息缺口的两面。** one-shotting 是单个 session 承担了超出窗口的规划；过早宣告完工是后期 session 缺少全局目标信息，用局部进展推断了整体。feature list 同时解决两者，因为它是一个"不可改结构、只可更新状态"的外部目标函数：agent 不能删测试（结构受保护），只能把 passes 从 false 翻成 true。
3. **用 JSON 而不是 Markdown 存 feature list 是个容易被忽视的对齐细节。** 结构化格式有严格的语法边界，模型乱改会立刻产生格式错误，而且训练语料里 JSON 的"改值不改结构"模式远比 Markdown 的"自由重写"清晰。格式选择本质上是在利用模型的训练分布做约束。
4. **"干净状态"就是 CI 思想在 agent 域的移植。** 每个 session 的交接成本被摊薄：小提交、可回滚、随时可恢复工作版本，等价于持续集成里"增量小、测试快、随时可发布"。反过来，让 agent 一次做大块，等于把 CI 的所有好处全部放弃，最后只能靠下一个 session 考古。
5. **验证的 oracle 要贴近用户。** 单元测试过了但端到端不工作，本质是 agent 的验证标准比用户的使用标准低。给它浏览器自动化让它以用户视角操作，等于把验收标准物理性地抬高了。局限也诚实：Puppeteer 看不到浏览器原生 alert 弹窗，依赖这类弹窗的功能更容易出 bug——视觉验证的盲区就是新 bug 的藏身处。

## 落到你手上

1. **给 HanaAgent 的长任务引入"交接协议"**：你现在的定时任务和 16 周课程推进都是跨 session 场景，正是这篇的主场。当天可动手：在工作区建一个 `progress/` 目录，放 `course-progress.txt`（日志体）+ `course-tasks.json`（每条任务带 `passes` 状态，禁止改结构只许翻状态）。以后任何新 session 接手前先读这两个文件再动手，比翻聊天记录可靠。
2. **Ubuntu 服务器的日常维护套用 get-up-to-speed 四步**：写一个 `init.sh`（起服务 + curl 推理接口冒烟测试），每次要动服务器前先看 git log/日志、跑冒烟、确认没坏再开工。你昨天 503 的排查就是这个缺口的反面教材——不知道状态就开干。
3. **结构化状态文件改用 JSON**：机器要修改的文件（任务清单、状态追踪）用 JSON，给人看的文档用 Markdown。你今天的这个仓库本身就可以试试：把"已收录文章清单"维护成一个 JSON（slug + 状态），而不是靠 ls 目录猜。

## 原文摘录

> compaction isn't sufficient. Out of the box, even a frontier coding model like Opus 4.5 running on the Claude Agent SDK in a loop across multiple context windows will fall short of building a production-quality web app.

> By "clean state" we mean the kind of code that would be appropriate for merging to a main branch: there are no major bugs, the code is orderly and well-documented, and in general, a developer could easily begin work on a new feature without first having to clean up an unrelated mess.

> We prompt coding agents to edit this file only by changing the status of a passes field, and we use strongly-worded instructions like "It is unacceptable to remove or edit tests because this could lead to missing or buggy functionality."
