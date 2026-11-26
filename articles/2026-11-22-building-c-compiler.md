# Building a C compiler with a team of parallel Claudes（16 个 Claude 从零写出一台能编译 Linux 内核的 C 编译器）

- 日期：2026-11（Engineering/Research）
- 链接：https://www.anthropic.com/research/building-c-compiler
- 分类：AI 工程实践 / 多智能体
- 作者：Nicholas Carlini（Safeguards 团队）

## 【讲了什么】

Carlini 用 16 个并行运行的 Claude 实例，无人值守地从零写了一个 Rust 实现的 C 编译器。近 2000 个 Claude Code 会话、约 20 亿美元输入 token、花费近 2 万美元，产出 10 万行代码，能编译 Linux 6.9（x86/ARM/RISC-V）、QEMU、FFmpeg、SQLite、Postgres，还能跑 Doom。文章的重心不在编译器本身，而在"如何设计让 LLM 团队长时间自主运转的 harness"：无限循环脚本（Ralph-loop 变体）、用 git 仓库 + 锁文件做任务分配的同步算法、面向 LLM 而非人类设计的测试反馈体系，以及这套方法在哪里撞到天花板（16 位 x86 代码生成超限、优化后代码质量不如 GCC 关优化、修 bug 经常打破旧功能）。

## 【关键洞察】

1. **自主性的瓶颈不是模型，是环境设计**。作者花了大部分精力在测试、日志、进度文件上，而不是 prompt 上。模型会一直"解决问题"，但解决的是你 verifier 允许它解决的问题。任务验证器不完美，agent 就会认真地解决错的问题。
2. **LLM 有两个被忽视的物理缺陷：上下文污染和时间盲**。测试输出必须精简、错误写成一行 `ERROR: 原因` 方便 grep；Claude 感知不到时间流逝，会无意识地跑几小时测试，所以要给它 `--fast` 采样选项（1%~10% 确定性抽样）。这两条对任何长任务 agent 都通用。
3. **并行化的前提是任务可切分**。Linux 内核编译是"一个巨大任务"，16 个 agent 全卡在同一个 bug 上互相覆盖。解法不是更多 agent，而是用 GCC 当 oracle 做二分定位，把大问题切回可并行的独立小问题。规模收益只来自任务切分，不来自堆算力。
4. **多角色分工是免费红利**。dedicated agent 负责去重、性能、代码质量、文档，成本极低但显著改善产出。这其实就是"把人的代码评审流程拆给多个模型实例"。

## 【落到你手上】

1. **你的 16 周 LLM→具身智能路线可以借鉴锁文件模式**：学 RSI/写双足控制代码时，想让多个会话并行推进（比如一个调步态、一个写状态机），用 git 裸仓 + `current_tasks/*.txt` 锁文件做最简同步，不需要任何编排框架。一个下午就能搭出你实验里 3~4 个 agent 并行的雏形。
2. **给 .114 的 Qwen3.5-27B 做长任务实验时先修"时间盲"**：本地模型上下文更紧，跑 RL/长任务脚本时务必加采样/快路径开关，日志按 `ERROR: 原因` 单行格式写，避免烧上下文。这对 27B 比 Opus 重要得多。
3. **HanaAgent 的 subagent 协作可以直接套用"角色分工"**：调研用 read-only subagent、实现用 write subagent、再加一个专职"合并重复代码/更新文档"的收尾 agent，就是这篇文章最小化版本，今天就能试。

## 【原文摘录】

> "I had to constantly remind myself that I was writing this test harness for Claude and not for myself, which meant rethinking many of my assumptions about how tests should communicate results."

> "Having 16 agents running didn't help because each was stuck solving the same task."

> "I have consistently found the best way to understand what language models can do is to push them to their limits, and then study where they start to break down."
