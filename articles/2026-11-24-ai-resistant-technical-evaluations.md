# Designing AI-resistant technical evaluations

- **日期**：2026-11-24
- **链接**：https://www.anthropic.com/research/AI-resistant-technical-evaluations
- **分类**：research / 工程实践（评测设计、性能工程）
- **作者**：Tristan Hume（Anthropic 性能优化团队负责人）

## 【讲了什么】

Tristan Hume 在 2023 年底为 Anthropic 性能工程团队设计了一道 take-home 测试：候选人在一个模拟加速器（VLIW、SIMD、scratchpad 内存、多核，类 TPU）上优化并行树遍历代码，配合 Perfetto 热重载 trace。这套题跑了两年多、筛出上千名候选人，撑起了整个性能工程团队的招聘。但每个新 Claude 模型都把它打败一次：Opus 4 在 4 小时时限内超过了大多数人类，Opus 4.5 在 2 小时内打平了最优秀的人类表现。

于是他迭代了三次。第一次换题：改成 TPU 寄存器数据转置 + bank conflict 优化，但 Opus 4.5 用 ultrathink 后照样解出，还发现了他本人没想到的"转置整个计算而不是转置数据"的技巧。根本原因是转置优化在训练语料里出现太多，模型的经验库大于任何单个工程师。第二次放弃"像真实工作"，转向"模拟新奇工作"：参考 Zachtronics 游戏（如 Shenzhen I/O）的极度受限指令集，做一组追求最少指令数的小谜题，并且刻意不提供可视化和调试工具。Opus 4.5 失败了，人类能稳定胜出。

最后他把原版题目开源为公开挑战：无限时间下人类最优解仍然大幅超过 Claude（人类历史最优 vs Claude 最好 1363 cycles），如果你能跑进 1487 cycles 以内就把代码和简历发给 Anthropic。

## 【关键洞察】

1. **模型的"经验库"是评测的阿喀琉斯之踵。** 第一版题被击败不是因为模型更聪明，而是因为树遍历加速、转置、bank conflict 这些技巧在公开语料里反复出现。抗 AI 的题必须出分布（out-of-distribution），即模型没怎么见过的新构造，而不是更难的老构造。这对所有评测设计都成立：凡是"经典题"，模型永远比你见过的工程师经验更多。

2. **瓶颈处的认知是人的护城河。** Opus 4.5 卡在"不可逾越的内存带宽瓶颈"上，多数人也会下同样结论。Tristan 告诉它"这个 cycle 数是可达的"，它就找出了利用问题结构的绕法。模型的失败模式不是算不出，而是过早接受"此路不通"。人类的价值在于对问题结构的直觉判断，以及知道何时该注入一个外部目标约束。

3. **"引导"（steering）可能变成旁观。** 人类通常要花一半时间读题，跟模型实时协作时人永远落后于模型的理解进度，最终策略会退化成"坐在那看"。长时限任务中人的优势依然存在，但 2 小时窗口的"人机协作"模式信号已经被模型本身稀释了。

4. **调优工具构建本身是信号。** 新版题不给任何可视化和调试器，因为"该不该花十分钟让模型写一个交互式调试器"这个判断，本身就能区分出真正的工程师。评测设计的颗粒度要从"解不解题"下沉到"怎么建工具链"。

## 【落到你手上】

1. **把开源挑战跑起来当实验场。** https://github.com/anthropics/original_performance_takehome 拉下来，用你 .114 上的 Qwen 本地模型和 Claude 各做一次 2 小时对比，正好是你"LLM 到具身智能、RSI"主线里一个绝佳的对照实验：本地模型在"卡瓶颈时会不会自我劝退"上和 Claude 差多少，比任何 benchmark 分数都直观。用 Tailscale 在手机上也能跟进结果。

2. **借鉴"注入目标约束"的技巧到你的推理调优。** 以后在 EPYC 7452 上调 GPU 推理性能时，如果助手（或你自己）卡在一个"瓶颈无解"的结论上，别顺着它走，直接给它一个可达的参考数字（比如参考同类硬件的 tokens/s），逼它找问题结构的绕路。这是文章里性价比最高的一招。

3. **给自己的 HanaAgent 工作留一道"新奇题"。** 你日常让 agent 干活，题越"经典"它越强、你的判断信号越弱。可以定期挑一个你系统里 out-of-distribution 的小问题（比如给 openhanako 加一个受限环境下必须手写的设计）亲手做一遍，保持自己对系统的理解领先于 agent 的经验。

## 【原文摘录】

> Then it stopped, convinced it had hit an insurmountable memory bandwidth bottleneck. Most humans reach the same conclusion. But there are clever tricks that exploit the problem structure to work around that bottleneck.

> The original worked because it resembled real work. The replacement works because it simulates novel work.

> If you optimize below 1487 cycles, beating Claude's best performance at launch, email us at performance-recruiting@anthropic.com with your code and a resume.
