# Scaling Managed Agents: Decoupling the brain from the hands（把大脑和手分开：Anthropic Managed Agents 的虚拟化架构）

- **日期**：2026-11-20
- **原文链接**：https://www.anthropic.com/research/managed-agents
- **分类**：engineering（长时程 agent 系统与架构）

## 【讲了什么】

这篇文章（Lance Martin、Gabe Cemaj、Michael Cohen）讲 Anthropic 托管式长时程 agent 服务 Managed Agents 的架构演进，主线是一个三段式踩坑史。

第一阶段，session、harness、sandbox 全部塞进同一个容器。好处是文件编辑就是直接 syscall，没有服务边界要设计；坏处是养出了"宠物"（pet）：容器挂了会话全丢，唯一的观察窗口是 WebSocket 事件流，分不清故障在 harness bug、网络丢包还是容器掉线，而容器里还装着用户数据，工程师进去开 shell 调试等于没得调。更糟的是 harness 默认"所有资源都在容器旁边"，客户想接自己的 VPC 只能网络对等或把整套搬走。

第二阶段是"解耦大脑和手"。harness 搬出容器，容器变成普通工具：`execute(name, input) → string`，容器死了就是一次工具调用报错，模型想重试就用 `provision({resources})` 造新的；harness 自己也变成牛群，因为会话日志在它外面，崩溃后用 `wake(sessionId)` + `getSession(id)` 从最后一条事件恢复。安全边界随之重划：git token 在初始化时焊进 remote，MCP 调用走专用代理加外部 vault，凭据从结构上不可达于 Claude 生成的代码运行的沙箱。

第三阶段是规模化。脑和手解耦后，容器按需供给（不需要沙箱的会话不再白等容器启动，p50 TTFT 降 60%、p95 降 90%），多个无状态 brain 连多个 hand 成为可能。全文的元论点引用操作系统：OS 靠把硬件虚拟化成进程、文件这类"为尚未想到的程序设计的抽象"活了几十年；agent 系统同样该固定接口形状、放任实现更换。

## 【关键洞察】

1. **harness 是"模型哪里不行"的假设快照，假设会过期。** 文中实例：Sonnet 4.5 有"上下文焦虑"（感觉快到上下文上限就草草收尾），他们加了 context reset 来补偿；换 Opus 4.5 后该行为消失，reset 变成死重。这正是 Bitter Lesson 在工程侧的翻版：为模型弱点手写补偿逻辑是技术债，模型一代代变强，这些补丁会静默地从"修复"变成"限制"。定期审计 harness 里的补偿逻辑，问"这是为哪一代模型写的"，是本文最可操作的一条方法论。

2. **虚拟化的本质是"接口比实现活得久"。** 底层原理：系统的价值在稳定抽象，不在具体实现。`read()` 不知道下面是 1970 年代的盘组还是 SSD；这里 agent 被虚拟成 session（append-only 事件日志）、harness、sandbox 三个抽象，接口形状固定，实现随意换。判断一个架构能不能"为尚未想到的程序设计"，就看它把多少东西锁死在了实现里而不是接口里。

3. **状态外置是"牛群化"的前提：一个组件可替换，当且仅当它的状态不在它自己身上。** 容器和 harness 变成 cattle，本质是把全部状态挪进 append-only session log，日志是唯一事实源，所有节点无状态，崩溃即重启而非抢救。这就是 event sourcing + 无状态服务的经典形态。可替换性不是运维口号，是状态归属问题：状态放哪，决定了谁是谁的宠物。

4. **安全边界要做结构性不可达，不做权限收窄。** "收窄 token 作用域"是"模型不会滥用有限凭据"的假设，而模型越来越聪明，假设必会过期。结构性修复是让 token 从构造上就进不了沙箱：git token 在环境层焊死，MCP 走代理+vault，harness 全程不接触凭据。这和昨天那篇"白名单是能力授予而非目的地过滤"互为表里：别赌模型对权限的"倾向"，设计权限够不着的结构。

## 【落到你手上】

1. **给 openhanako/HanaAgent 做一次"死重 harness 审计"。** 把 system prompt 和工具逻辑里为某代模型弱点写的补偿（强制 reset、截断重试、禁某行为、防上下文焦虑类指令）列一张清单，每条标注"为哪个模型写的"。用当前模型各跑两次验证该行为是否还存在，不存在的删掉。每条死重都在消耗上下文和推理路径，而且可能反过来限制新模型。这是当天就能动手、零成本的事。

2. **把自建会话变成"牛群"：事件日志外置。** 现在 Hana 的会话状态在进程里，服务挂掉会话就丢，503 上下文过大的问题也是同一根源。最小实现不用上消息队列：会话事件逐条 append 到一个 JSONL 文件（天然 append-only），重启后读日志从最后一条事件恢复（wake 逻辑）。先在一个 session 上试点，验证"崩溃→重启→无损续接"，再推广。这比任何压缩技巧都先解决"会话是宠物"的问题。

3. **验证 git push 的凭据边界。** 文章仓库和 .114 上的服务：检查 agent 执行的脚本能否看到 token（在 agent 进程里跑 `env | grep -i token`、看 git remote 是否用明文 token 或 credential helper）。Anthropic 的做法是 token 在初始化时焊进 remote、agent 全程不接触。验证标准很简单：agent 触发的任何进程里，凭据不可见。

## 【原文摘录】

- "Harnesses encode assumptions about what Claude can't do on its own. However, those assumptions need to be frequently questioned because they can go stale as models improve."
- "We virtualized the components of an agent: a session (the append-only log of everything that happened), a harness (the loop that calls Claude and routes Claude's tool calls to the relevant infrastructure), and a sandbox (an execution environment where Claude can run code and edit files)."
- "The structural fix was to make sure the tokens are never reachable from the sandbox where Claude's generated code runs."
