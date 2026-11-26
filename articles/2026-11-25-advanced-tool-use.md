# Introducing advanced tool use on the Claude Developer Platform

- 日期：2026-11-25
- 链接：https://www.anthropic.com/research/advanced-tool-use
- 分类：Engineering / 工具使用（Anthropic Developer Platform 团队，Bin Wu 等）

## 【讲了什么】

这是 Claude Developer Platform 团队发布的三项工具能力：Tool Search Tool（工具搜索）、Programmatic Tool Calling（代码内编程式调用工具）、Tool Use Examples（工具使用示例）。三者分别对应 Agent 工具使用链路里的三个瓶颈：工具定义占满上下文、中间结果污染上下文、参数调用格式错误。

背景是：MCP 工具连接一多，工具定义的 JSON Schema 就要吃掉几万甚至十几万 token，Anthropic 自己见过 134K token 的纯工具定义开销，而且工具多了之后选错工具、填错参数成为最常见的失败模式。

Tool Search Tool 让 Claude 一开始只加载一个约 500 token 的搜索工具，其余工具标记 `defer_loading: true` 按需检索加载，工具定义开销从约 77K token 降到 8.7K（降 85%），同时大工具库下的准确率反而提升（Opus 4 从 49% 到 74%）。

Programmatic Tool Calling 让 Claude 写 Python 代码来编排工具调用：工具返回的结果先进入沙箱代码环境被处理、过滤、聚合，只有最终结果进入模型上下文。文中例子把 200KB 的原始账单数据压成 1KB 的超标名单，token 消耗平均降 37%，还消除了 19 次以上的模型推理往返。

Tool Use Examples 允许在工具定义里直接给 1~5 个真实调用示例，弥补 JSON Schema 只能表达"结构上合法"、表达不了"格式约定和参数搭配"的缺陷，复杂参数处理准确率从 72% 提到 90%。

## 【关键洞察】

1. **上下文是比算力更稀缺的资源，工具定义是一种"固定税"。** 传统架构把所有工具定义预先塞进上下文，等于每次会话开工前先交一笔与任务无关的税。Tool Search Tool 本质是操作系统里的懒加载（lazy loading）：按需取用、用多少付多少。更反直觉的是懒加载还提升了准确率——工具数量本身会让模型"选择过载"，候选集缩小了，决策反而更准。这和人的工作记忆容量有限是同一个原理。

2. **模型应该看"结论"而不是"过程数据"。** PTC 的核心区分是：中间数据（2000 条账单）和执行结论（3 个超标的人）应该走不同的通道。用自然语言逐个调用工具时，中间数据被迫走"推理通道"——模型得逐条"目测"、累加、比较，既慢又容易错。把循环、条件、聚合写成显式代码，等于把确定性逻辑从概率推理中剥离出来。这其实是老原则的新应用：能用代码表达的控制流，就不该让 LLM 用 token 去"想"。

3. **Schema 是类型系统，示例是归纳偏置。** JSON Schema 回答"什么合法"，但回答不了"该用哪个格式"（日期是 YYYY-MM-DD 还是 ISO8601、ID 是什么约定、可选参数之间什么搭配）。示例直接给模型看真实分布，让它归纳出约定。这比在 description 里写一堆文字说明高效得多，因为例子就是最紧凑的信息编码。

4. **每个优化都有阈值，不是无脑全开。** 文章很克制地给了适用边界：工具少于 10 个别上搜索；单工具简单调用别上 PTC；简单单参数工具别加示例。加搜索就多一次检索延迟，加示例就多吃 token，ROI 取决于瓶颈到底在哪一环。

## 【落到你手上】

1. **给 Qwen3.5-27B 的调用链路加"代码前置过滤"。** 你上周刚被"会话上下文过大导致服务满载 503"坑过，PTC 正是解法：凡是让模型分析日志、表格、多接口数据的任务，先用本地 Python 把数据过滤/聚合完，只把摘要喂给模型。今天就可以试：写个脚本汇总 .114 上 llama.cpp 服务的请求日志（按状态码/耗时分桶），对比"直接贴原始日志给模型"和"贴聚合结果"两种方式的 token 数和答案质量。

2. **给 HanaAgent（openhanako）做工具定义瘦身实验。** 数一下你 agent 当前全量工具 schema 占多少 token（把 tool definitions 拼起来跑一次 tokenizer 就行）。如果超过 10K，就参照 `defer_loading` 思路：高频工具常驻，低频工具（浏览器控制、workflow 等）走一个 search 工具按需加载。改完跑几个多工具任务对比准确率和上下文占用。

3. **给自己的工具/MCP 补 input_examples。** 挑一个你最常调错参数的工具（比如带很多可选字段的），在定义里加 1~3 个真实调用示例，观察调错率变化。成本几乎为零，属于白捡的准确率。

## 【原文摘录】

> "Claude's context receives only the final result: the two to three people who exceeded their budget. The 2,000+ line items, the intermediate sums, and the budget lookups do not affect Claude's context, reducing consumption from 200KB of raw expense data to just 1KB of results."

> "JSON schemas define what's structurally valid, but can't express usage patterns: when to include optional parameters, which combinations make sense, or what conventions your API expects."

> "This represents an 85% reduction in token usage while maintaining access to your full tool library."
