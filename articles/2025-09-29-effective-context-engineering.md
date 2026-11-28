# Effective context engineering for AI agents

- 日期：2025-09-29
- 链接：https://www.anthropic.com/engineering/effective-context-engineering-for-ai-agents
- 分类：Engineering / Agent 上下文工程（Applied AI 团队：Prithvi Rajasekaran, Ethan Dixon, Carly Ryan, Jeremy Hadfield）

## 【讲了什么】

这是 Anthropic Applied AI 团队在 Sonnet 4.5 发布同日发的文章，把"上下文工程"（context engineering）从概念变成了一套可操作的方法论。核心定义：上下文工程不是写好一个 system prompt，而是管理 LLM 推理时"整套 token 状态"的策略集合——system prompt、工具、MCP、外部数据、消息历史全部算在内。

文章分三层展开。第一层解释为什么上下文必须被当作有限资源：transformer 注意力是 n 个 token 两两配对（n² 关系），上下文越长，模型分配给每对关系的注意力越薄；加上训练语料里短序列远多于长序列，模型对跨长距离依赖"经验不足"，所以所有模型都会出现 context rot——上下文越长，召回准确率越低，而且是渐变下降而非断崖。第二层讲静态部分怎么调：system prompt 要"高度适中"（不硬编码脆弱的 if-else，也不写得含糊假设共享上下文），工具集要最小化且职责不重叠，示例要少而典型而不是穷举边缘情况。第三层讲动态部分：用"just-in-time"检索代替预计算 RAG——agent 只持有轻量标识符（文件路径、查询、链接），运行时按需把数据拉进上下文；对长任务给出三个武器：compaction（压缩旧对话重启上下文）、结构化笔记（NOTES.md 写进上下文之外再取回）、子 agent 架构（每个子 agent 用干净窗口干活，只回传 1000-2000 token 的摘要）。

## 【关键洞察】

1. **Context rot 是架构层面的必然，不是某个模型的缺陷。** 底层原理是注意力的 n² 摊薄：每个新 token 进来，所有已有 token 的"被关注度"都被稀释一分。类比工作记忆：你同时盯 5 件事时每件都清楚，盯 50 件时每件都模糊——不是变笨了，是注意力预算被摊薄了。工程推论：窗口大小永远不等于可用大小，真正的设计单位是"高信号 token 密度"而不是 token 配额。

2. **信息该走两条通道：常驻的进上下文，随用的走外部存储。** just-in-time 检索的本质是把文件系统当"工作记忆之外的外存"，agent 只缓存指针（路径/链接），用时再 deref。这和虚拟内存、缓存行是同一个思想：CPU 再快也不把所有数据塞进寄存器，只留地址。附带收益是渐进披露——文件名、目录层级、时间戳本身就是元数据信号，探索过程会逐层过滤掉无关信息，模型不会被一次性灌入的数据淹没。

3. **长任务的三个技巧对应三种不同的"状态"。** compaction 处理的是"对话流"状态（要保持交互连续性，就定期蒸馏重启）；结构化笔记处理的是"里程碑"状态（关键决策、未解 bug 写进 NOTES.md，跨上下文窗口存活）；子 agent 处理的是"探索"状态（脏活累活隔离在一次性窗口里，只把结论带回主线程）。选哪个取决于任务里哪种状态占主导，不是越高级越好——一个 NOTES.md 就能救活的任务，别上 multi-agent。

4. **Compaction 的调优顺序是反直觉的：先召回，后精度。** 摘要丢掉的细节，重要性往往到后面才暴露。所以先把压缩 prompt 调到"什么都留住"（高召回），再逐步砍掉冗余提精度。这和信息检索里"先宁滥勿缺再排重"的逻辑一致：不可逆操作的代价不对称，丢掉的上下文永远找不回来。

## 【落到你手上】

1. **给 114 上的本地 Qwen 模型做一次 context rot 实测。** 小模型的注意力预算比大模型更紧，降级曲线可能陡得多。写个 needle-in-a-haystack 脚本（固定问句，把答案埋进 2k/4k/8k/16k/32k 不同长度上下文中，测召回准确率），半天就能跑完。产出一张"实际可用上下文上限"的表，之后 llama.cpp 服务按这个上限而不是标称 max context 来设服务参数，避免长会话静默降智。

2. **给 HanaAgent 的长会话加"工具结果清理"。** 文章说这是最轻、最安全的 compaction 形式：历史深处的工具原始返回（比如几十 KB 的网页抓取）用过就没用了，定期清掉只留一句话摘要。openhanako 里可以做个实验：对比同一个长任务在"保留全部工具输出"和"超过 N 轮后清理旧输出"两种模式下的答案质量和 token 消耗，数据说话再决定是否默认开启。

3. **把定时任务改成"自记笔记"模式。** 你现在的一日一读、巡检类定时任务每次都是无状态重跑。按结构化笔记的做法，给每个定时任务配一个 state 文件（如 anthropic-reading/state.md），任务结束时把"已覆盖到哪篇、踩过的坑、下次的起点"写进去，下次开工先读它。成本是几行代码，收益是长周期任务不再靠外部记忆续命。

## 【原文摘录】

> "Context, therefore, must be treated as a finite resource with diminishing marginal returns."

> "Good context engineering means finding the smallest possible set of high-signal tokens that maximize the likelihood of some desired outcome."

> "Start by maximizing recall to ensure your compaction prompt captures every relevant piece of information from the trace, then iterate to improve precision by eliminating superfluous content."
