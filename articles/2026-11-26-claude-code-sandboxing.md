# Beyond permission prompts: making Claude Code more secure and autonomous
- 归档日期：2026-11-26 / 原文：https://www.anthropic.com/research/claude-code-sandboxing / 分类：engineering

## 讲了什么

Claude Code 的默认模型是"逐操作审批"：每个写文件、跑命令的动作都要人点确认。这套模型有个隐藏 bug——审批疲劳。人点得多了就不看内容了，安全模型反而因为"人的疲劳"而失效。Anthropic 的答案是反过来：不再逐次审批动作，而是预先划好边界，边界内自由活动，越界才报警。内部数据显示这套沙箱把权限提示减少了 84%，同时安全性更高。

具体落地是两个功能：一是沙箱化的 bash 工具，基于 Linux bubblewrap 和 macOS seatbelt 这类 OS 级原语，做文件系统隔离（只能读写当前工作目录）+ 网络隔离（出站流量只能走一个白名单代理）；二是 Claude Code on the web，把会话跑在云端隔离沙箱里，git 凭证永远不进沙箱，由外部代理验证内容后再附加真实 token。核心组件 sandbox-runtime 已开源。

## 关键洞察

1. **安全模型从"验证每次动作"转向"验证边界"**。逐操作审批的本质是把安全决策外包给一个会疲劳的人，这是不可靠的执行者。边界模型把决策前移（一次性划范围），把执行交给内核（不会疲劳、不可骗），只在越界这个低频率事件上才找人。这就是为什么它能同时提升安全和自主性：不是放松了控制，而是把控制放在了更便宜的层。
2. **文件系统隔离和网络隔离必须同时存在，缺一不可**。没有网络隔离，被注入的 agent 能把 SSH key 外发（数据渗出）；没有文件系统隔离，它能改系统文件逃出去拿到网络权限。这是零信任的基本结构：单一维度的隔离都存在"绕过路径"，两个维度交叉才形成闭环。
3. **必须用 OS 级原语而非用户态限制**。agent 会 spawn 子进程、跑脚本，用户态的限制对新起的进程失效（它可以直接再启一个不受限的进程）。bubblewrap/seatbelt 在内核层约束整个进程树，任何子进程都逃不掉。这条对自建 agent 尤其重要：用 shell 脚本检查"不许访问 X"是防君子不防 agent 的。
4. **凭证不进沙箱，代理代持**。web 版的 git 代理设计很关键：沙箱内只有低权限的 scoped credential，真实 token 在代理侧，且代理还校验 push 的分支和仓库。原则是：最高权限的东西永远不放在最容易被攻破的边界里。这和 K8s 不把 secrets 塞进容器、用 service account 中转是同一个思想。

## 落到你手上

1. **今天在 Ubuntu 24.04 上装 bubblewrap 验证双层隔离**。`sudo apt install bubblewrap`，然后 `bwrap --ro-bind / / --bind ~/test ~/test --unshare-net --dev /dev --die-with-parent bash -c 'curl ifconfig.me'`，确认网络被掐；再故意让命令读 `~/.ssh` 看是否被挡。十分钟就能体感这篇的核心。
2. **给你的自建推理/agent 服务套一层 bwrap**：只 bind 工作目录，网络走 `--unshare-net` 加一个本地代理只放行推理端口的出站。这正是 HanaAgent 沙箱（network-on 是最大风险面）的收敛方向——把网络-on 收窄成"白名单代理"，出站只剩你显式允许的域名。
3. **读一下开源的 sandbox-runtime**（github.com/anthropic-experimental/sandbox-runtime），看它的"unix socket 代理 + 域名确认"实现，openhanako 的 exec 工具如果以后要做网络收紧，可以直接参考这套架构而不是自己发明。

## 原文摘录

- "Constantly clicking 'approve' slows down development cycles and can lead to 'approval fatigue', where users might not pay close attention to what they're approving, and in turn making development less safe."（持续点"批准"拖慢开发，还会导致审批疲劳，人不再仔细看批的是什么，反而更不安全。）
- "It is worth noting that effective sandboxing requires both filesystem and network isolation. Without network isolation, a compromised agent could exfiltrate sensitive files like SSH keys; without filesystem isolation, a compromised agent could easily escape the sandbox and gain network access."
- "Sandboxing ensures that even a successful prompt injection is fully isolated, and cannot impact overall user security. This way, a compromised Claude Code can't steal your SSH keys, or phone home to an attacker's server."（沙箱保证即使 prompt 注入成功也被完全隔离，被入侵的 Claude Code 偷不走你的 SSH key，也回连不了攻击者服务器。）
