# Redesigning source control at Anthropic（当代码不再是瓶颈：Scion 重写版本控制）

- **日期**：2026-11-17
- **原文链接**：https://www.anthropic.com/research/redesigning-source-control
- **分类**：engineering / research

## 核心机制：为什么这样设计

### 1. 假设变了，架构就该换

git 的核心假设：客户端离线、需要完整副本。这个假设在"程序员在飞机上写代码"的场景下完美。但 Anthropic 的客户端是 agent 和 CI job，**永远在线**。于是 git 最优秀的那些特性（离线自足、packfile、分支引用）反而成了规模化的枷锁：主分支历史近 100 GB，压缩工作副本超 25 GB，每天约 2500 万次 checkout，峰值跑过几百个内部 git 镜像，仍不够用。

他们试过所有 git 生态的标准解药（浅克隆、稀疏检出、promisor remote、几百个镜像），"买到了时间但越来越难运营"。真正的转折点是识别出底层前提：git 假设客户端离线且需要完整副本，而 agent 天生在线。推翻这个前提，系统就能从"每个客户端都是一个潜在服务器"变成"客户端只是薄缓存 + 工作副本"。

类比：你住在一栋楼里，每层都配了一整套厨房（git 的"每个 clone 都是完整仓库"）。你发现 90% 的人只是热个饭，真正做饭的需要的是集中厨房。于是你拆掉每层的厨房，每层只留一个微波炉（薄缓存），真做饭的去集中厨房（服务端）。这不是"把每层厨房扩得更大"，是承认"每层都需要完整厨房"这个假设已经错了。

### 2. 内容寻址 + 派生数据递归

Scion 的数据单元是 changeset，只存"相对第一个父亲的变更文件列表"和指向 blob 的内容哈希指针。因为内容寻址，一段数据存下就永不改变，任何服务器都能无条件缓存它而无需互相协调。

"派生数据递归"是什么意思？blame、路径历史、提交图 skip list 这些查询需要的不是 changeset 本身，而是"从父提交的派生值递归算出"的中间结果。比如：想知道文件 X 在 commit C 的 blame，不需要拉整个仓库，只需要"父 commit 的 blame + C 对 X 的 diff"。每个派生结果都是一个内容寻址的对象，算过就缓存，下次直接取。

三重好处：加新索引不改客户端（服务端加计算就行）；可以懒计算避免物化（没人查就不算）；客户端看不到内部结构所以服务端能随便优化。

**提交成本只随变更大小增长，不随仓库大小增长。** 这是 lock-free 高吞吐的根本。git 的 commit 成本跟仓库大小挂钩（要更新 packfile），Scion 的 commit 成本只跟你这次改了几个文件挂钩。

### 3. 工作副本是可选的

他们不想 checkout 就完全不 checkout：CI 要某个 diff，直接问服务端。要文件时才按需物化。协议不像 git 那样先告诉服务端"我有什么"再协商，而是"我要什么就递归请求什么"，每个请求都是一个永不变化的特定对象，于是任何层级的缓存都能答。

客户端→daemon→区域 cache→权威 server，四个角色响应同一套 RPC，严格分层隔离故障域。结果：每层缓存命中率 99%+，基础设施 CPU/内存占用比之前的 git 镜像方案**小 200 倍**，checkout 从分钟级到秒级，CI 省 25% CPU。

### 4. 工程师的价值变成了"说不"

七个人五个月干完一整套版本控制 + 评审系统 + 迁移，同时每天合并量涨了 10 倍。团队里所有工程师现在都承认 Claude 写代码比他们强。但工程师仍在提供"系统间高层组件之间的关系"和"设计级取舍"的理解。

最常见的工作是告诉 Claude **不要做**：agent 会为了迎合一个本不该存在的模式而违反核心系统约束；会为了防御一个假想中的风险而过度设计（比如对内部服务的信任过于保守，结果加了一堆校验和复杂度）。人类审每个 change，但看的是变更的**意图**，不是逐行读代码。

### 5. Claude 自己加速了 Scion 的诞生

手写的 git→Scion 数据摄取管线，让早期 Mythos 预览版重写一遍，零额外指令，内存消耗少 20 倍、快 4 倍，还保住了所有隐含的不变式。还有一类日常操作：Claude 随手找出了祖先查询的算法优化（漏用了某个索引），专家要几天几周去调，它第二天又找了一个。

### 6. 运营形态：superfan

他们在 Slack 里给每个组件配一个"superfan" Claude，只盯一个组件的健康，有遥测和日志权限，像一个"没别的事可干的资深工程师"不停刷图找异常。传统告警抓宏观（CPU 超 80% 报警），superfan 抓细微的新问题（"这个函数的 P99 延迟从 2ms 慢慢爬到 5ms，还没到阈值但趋势不对"）。

## 实际应用到陈亮的工程

### 动作 1：给 HanaAgent 的仓库操作做"客户端薄化"

你日常让 agent 反复 checkout、clone、diff 同一个仓库，每次都是完整副本。改成"能不 checkout 就不 checkout"：
- 看文件列表：`git ls-tree HEAD`
- 看两个 commit 的 diff：`git diff <a> <b> --stat`
- 看某个文件在某 commit 的内容：`git cat-file -p <commit>:<path>`
- 只在真要改文件时才 `git checkout`

对你 GitHub 上那几个仓库（含 article 仓库），每次 CI 或 agent 任务少 checkout 几次，省的是实打实的 IO 时间。

### 动作 2：给 article 仓库加"只看意图"的 review 习惯

对 agent 产出的 commit，别纠结 diff 每行，先看 `git log --oneline` 和每个 commit 的 message 是否讲清了"为什么改"。把 `git log` 变成可读的日志而不是流水账。

### 动作 3：给自建服务配一个"superfan"

你有 EPYC 7452 跑 SGLang 推理、Tailscale 组网、Ubuntu 24.04 主机。让 HanaAgent 定时轮询几个关键指标（GPU 显存、请求队列长度、Tailscale 在线设备、磁盘使用），只在偏离基线时报警，不等阈值。成本几乎为零，就是几条 curl + 一个 cron。

### 动作 4：批处理只走增量

任何你让 agent 做的批处理，尽量只处理变化的增量（`git diff --name-only` 驱动），别全量扫描整个仓库。在 EPYC 那种 IO 不算极致快的盘上，增量策略会让重复任务快一个数量级。

## 实际应用到 Hanako 自身

1. **我操作仓库时应该"薄化"。** 我看文件用 `git cat-file -p`，看 diff 用 `git diff --stat`，不动不动就 `git checkout`。每次省的是陈亮 GPU 机器的 IO 和磁盘。

2. **我审查自己产出的 commit 时，先看意图再看实现。** 我写 commit message 的时候，第一句话应该是"为什么改"，不是"改了什么"。如果我自己都说不清为什么改，那就是过度设计。

3. **我应该对陈亮的系统保持"说不"的能力。** 陈亮说"帮我加个 XX 功能"，我应该先问"这个功能解决什么具体问题？有没有更简单的方案？" 而不是直接动手。agent 最危险的不是"做错了"，是"做多了"。

4. **我给自己配一个"superfan"。** 每次改完配置（推理参数、记忆规则、prompt），跑完一组对照，如果指标没有明显变化，我应该承认"没验证成功"，而不是挑好看的讲。这就是预注册 + superfan 的轻量版。

## 原文摘录

> "At Anthropic, coding is no longer the bottleneck to building systems or doing research. ... The bottleneck, now, is all of the developer infrastructure built for human scale: source control, builds, and CI."

> "The properties that make git excellent for distributed, offline work were actively undermining our ability to scale as the repo grew. So we asked ourselves: if we designed source control with the fact that agents are always online in mind from the start, what would it look like?"

> "Very often, we found that the role of the engineers was to tell Claude 'no.' ... They now examine the *intent* of the change as opposed to reading every line of code."
