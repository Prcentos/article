# The components of agent memory（Agent 记忆的组件）

- **日期**：2026-11-16
- **原文链接**：https://www.anthropic.com/research/components-of-agent-memory
- **分类**：engineering / research

## 讲了什么

Anthropic 把 Claude 的跨会话记忆拆成四层：transcript（可搜索的会话日志）→ memory notes（agent 自己写的笔记）→ memory representations（如 wiki 这类更结构化的表示）→ injected context（每个会话预加载的上下文）。层与层之间靠一个叫 **dreaming**（做梦）的离线过程组织：在交互间隙，模型复盘日志、纠错、提炼新笔记。这是 sleep-time compute 思想的应用，类比人脑海马体在睡眠中把短期经验固化进新皮层。

## 关键洞察

1. **记忆本质上是预测**。写下一条规则，就是预测"未来的某个会话会再用到这个模式"。写下一条已验证的事实，就是预测未来可以直接依赖它而不必重新验证。这个视角让"记什么"从模糊的直觉变成了可以评估的行为。
2. **新模型已经学会判断 salience（重要性），且旧模型的老毛病正好是它的反面**。Pokémon 实验里：Sonnet 3.7 写满 413 个记忆文件（每场战斗、每个物品、每个 NPC 一个文件），把屏上每回合都能读到的状态（金钱、位置）也存进去，结果游戏进行 1 万步后，它 17 步里删 9 个文件建 6 个，完全在翻自己的记忆而不是玩游戏。Fable 5 只写了 9 个文件，存的全是"游戏不告诉你的东西"（策略、试出来的规则），还建了个 `global.md` 当仪表盘（当前目标 + 计划 + 自己的记忆使用规则），通关了。
   核心区别：**可重新获取的信息不值得存，只有"自己推出来的"才值得存。**
3. **给强模型定死记忆规则反而降分**。他们测试了"只记录失败"这类固定规则，Opus 5 和 4.7 的性能增益被显著削弱。正确的指导方式是只告诉 agent 记忆"用来干什么"，不告诉它"怎么写"，让模型自己演化出记录习惯。
4. **学习闭环是记忆的价值上限**。CL-Bench 数据库任务里，能"答错 → 记录失败 → 提出假设 → 验证 → 写成规则"的模型，分数从 0.3~0.56 拉到 0.83 左右。记忆的价值不在"存得多"，在"纠错回路闭不闭得上"。

## 落到你手上

1. **给 HanaAgent 加一个凌晨的 dreaming pass**。你现在已经有每日记忆编译（今天状态/每日状态文件），可以再加一个 03:00 的定时任务：复盘昨天的会话日志，做三件事：删过时事实、把"反复出现的操作"提炼成规则、把冲突的记忆以对话为准修正。这是本文最直接可抄的架构。
2. **检查你的记忆里有没有"屏上能读到的东西"**。对照 Fable 5 的标准：凡是能通过工具随时重新获取的（文件内容、服务器当前状态、在线设备列表），存了就是负担，会随时间腐烂；只存验证过的事实、踩过的坑、你自己推出来的结论。可以手动翻一遍现在的长期记忆做一次"减脂"。
3. **不要给记忆系统写死规则**。如果想在 Hana 里加记忆写入的约束，只描述目的（"让未来的你能更快接手"），别写"只记录 X"这种白名单，强模型自己会找到更好的 salience 标准。

## 原文摘录

> "Memory is, occasionally, a prediction. By writing down a rule, an agent predicts that the pattern it's describing will be relevant again."

> （Fable 5 在连丢 7 颗精灵球后写下的记忆）"If 3+ balls fail in a row, VARY the timing (insert extra 'wait' presses between menu steps) - the catch landed right after doing this."

> "We've found that it helps to tell Claude what memory is for, but not how to use it."
