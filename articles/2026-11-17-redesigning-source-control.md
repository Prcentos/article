# Redesigning source control at Anthropic（当代码不再是瓶颈：Scion 重写版本控制）

- **日期**：2026-11-17
- **原文链接**：https://www.anthropic.com/research/redesigning-source-control
- **分类**：engineering / research

## 讲了什么

Anthropic 说，现在写代码已经不是瓶颈了，瓶颈是为人手写代码的节奏而建的开发者基础设施：版本控制、构建、CI。2026 年 5 月他们日均合并约 2500 个变更，9 月已经超过 30000 个。为扛住这个量，他们花五个月自研了一套版本控制系统 **Scion**，把 monorepo 从 git 全量迁移过去，同时搭了代码评审界面 Antique、代码搜索、IAM 权限系统。

核心判断：git 是为"去中心化、离线可用"设计的，每个 clone 都是完整仓库。而他们几乎所有客户端都是 agent 和 CI job，**永远在线**。于是 git 最优秀的那些特性（离线自足、packfile、分支引用）反而成了规模化的枷锁：主分支历史近 100 GB，压缩工作副本超 25 GB，每天约 2500 万次 checkout，峰值跑过几百个内部 git 镜像，仍不够用。Scion 的基本哲学就是反过来：客户端永远在线、服务端负责所有计算、数据全内容寻址、每一层都可水平扩展。

## 关键洞察

1. **假设变了，架构就该换，不是把老架构撑大**。Anthropic 试过所有 git 生态的标准解药（浅克隆、稀疏检出、promisor remote、几百个镜像），"买到了时间但越来越难运营"。真正的转折点是识别出底层前提：git 假设客户端离线且需要完整副本，而 agent 天生在线。推翻这个前提，系统就能从"每个客户端都是一个潜在服务器"变成"客户端只是薄缓存 + 工作副本"。这不是性能优化，是把一个错误的世界观换掉。

2. **内容寻址 + 派生数据递归，是水平扩展的数学基础**。Scion 的数据单元是 changeset，只存"相对第一个父亲的变更文件列表"和指向 blob 的内容哈希指针。因为内容寻址，一段数据存下就永不改变，任何服务器都能无条件缓存它而无需互相协调。blame、路径历史、提交图 skip list 这些查询需要的都不是 changeset 本身，而是"派生数据"，且每种派生数据都按"从父提交的派生值递归算出"来定义。这个设计的三重好处讲得非常漂亮：加新索引不改客户端、可以懒计算避免物化、客户端看不到内部结构所以能随便优化。**提交成本只随变更大小增长，不随仓库大小增长**，这是 lock-free 高吞吐的根本。

3. **工作副本是可选的，稀疏是一等公民**。他们不想 checkout 就完全不 checkout：CI 要某个 diff，直接问服务端。要文件时才按需物化。协议不像 git 那样先告诉服务端"我有什么"再协商，而是"我要什么就递归请求什么"，每个请求都是一个永不变化的特定对象，于是任何层级的缓存都能答，整个架构像经典的多级缓存。客户端→daemon→区域 cache→权威 server，四个角色响应同一套 RPC，但严格分层隔离故障域。结果：每层缓存命中率 99%+，基础设施 CPU/内存占用比之前的 git 镜像方案**小 200 倍**，checkout 从分钟级到秒级，光 CI 就省下 25% CPU。

4. **最反直觉的一条：工程师的价值变成了"说不"**。七个人五个月干完了一整套版本控制 + 评审系统 + 迁移，同时每天合并量涨了 10 倍。团队里所有工程师现在都承认 Claude 写代码比他们强。但工程师仍在提供"系统间高层组件之间的关系"和"设计级取舍"的理解，最常见的工作是告诉 Claude **不要做**：agent 会为了迎合一个本不该存在的模式而违反核心系统约束；会为了防御一个假想中的风险而过度设计（比如对内部服务的信任过于保守，结果加了一堆校验和复杂度）。人类审每个 change，但看的是变更的**意图**，而不是逐行读代码。

5. **Claude 自己加速了 Scion 的诞生**。手写的 git→Scion 数据摄取管线，让早期 Mythos 预览版重写一遍，零额外指令，内存消耗少 20 倍、快 4 倍，还保住了所有隐含的不变式。还有一类日常操作：Claude 随手找出了祖先查询的算法优化（漏用了某个索引），专家要几天几周去调，它第二天又找了一个。

6. **运营形态也变了**：他们在 Slack 里给每个组件配一个"superfan" Claude，只盯一个组件的健康，有遥测和日志权限，像一个"没别的事可干的资深工程师"不停刷图找异常。传统告警抓宏观，superfan 抓细微的新问题。

## 落到你手上

1. **给 HanaAgent 的仓库做一次"客户端薄化"体检**。Stripe 那套太重，但原则能借：你日常让 agent 反复 checkout、clone、diff 同一个仓库，每次都是完整副本。可以在脚本里改成"能不 checkout 就不 checkout"：用 `git ls-tree`、`git diff <a> <b> --stat`、`git cat-file -p` 这类服务端直接回答的查询，只在真要改文件时才落盘。对你 GitHub 上那几个仓库（含 article 仓库），一次 CI 或 agent 任务少 checkout 几次，就是实打实的省时间。

2. **你现在每天 push 归档 article 仓库，可以加一个"只看意图"的 review 习惯**。文章里最有价值的工程判断是"审查变更的意图而非逐行读"。对 agent 产出的 commit，别纠结 diff 每行，先看 `git log --oneline` 和每个 commit 的 message 是否讲清了"为什么改"。这条能直接改善你现有 `anthropic-reading` 仓库的提交质量，让 `git log` 变成可读的日志而不是流水账。

3. **给自建服务配一个"superfan"盯健康度**。你有 EPYC 7452 跑 SGLang 推理、Tailscale 组网、Ubuntu 24.04 主机，可以让 HanaAgent 定时轮询几个关键指标（GPU 显存、请求队列长度、Tailscale 在线设备、磁盘），只在偏离基线时报警，而不是等阈值告警。这正是文章里"有遥测和日志权限、只盯一个组件"的轻量版，成本几乎为零。

4. **把"提交成本随变更大小增长"当成写脚本的准则**。任何你让 agent 做的批处理，尽量只处理变化的增量（`git diff --name-only` 驱动），别全量扫描整个仓库，尤其在 EPYC 那种 IO 不算极致快的盘上，增量策略会让重复任务快一个数量级。

## 原文摘录

> "At Anthropic, coding is no longer the bottleneck to building systems or doing research. ... The bottleneck, now, is all of the developer infrastructure built for human scale: source control, builds, and CI."

> "The properties that make git excellent for distributed, offline work were actively undermining our ability to scale as the repo grew. So we asked ourselves: if we designed source control with the fact that agents are always online in mind from the start, what would it look like?"

> "Very often, we found that the role of the engineers was to tell Claude 'no.' ... They now examine the *intent* of the change as opposed to reading every line of code."
