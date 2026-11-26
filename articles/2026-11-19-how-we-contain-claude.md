# How we contain Claude across products（如何驯服失控的 agent：Anthropic 的三套隔离架构与三次翻车）

- **日期**：2026-11-19
- **原文链接**：https://www.anthropic.com/research/how-we-contain-claude
- **分类**：research / engineering（agent 安全与隔离架构）

## 【讲了什么】

Anthropic 团队（Max McGuinness 等五人）复盘了三个 agent 产品（claude.ai、Claude Code、Claude Cowork）两年间的安全架构演进和真实事故。

核心框架：agent 的部署风险 = 故障概率 × 故障爆炸半径。模型训练把前者不断压低，但能力越强、权限越大，爆炸半径只增不减。所以工程问题变成"如何给爆炸半径封顶"。文章给出三种隔离模式：claude.ai 用一次性 gVisor 容器（爆炸半径最小，能力天花板也最低）；Claude Code 用"人审 + OS 沙箱"（气泡 wrap/Seatbelt，权限提示减少 84%）；Cowork 用整机本地 VM（用户不懂 bash，只能靠绝对边界）。

文章最有价值的部分不是架构，而是三次翻车：Claude Code 在"信任对话框出现之前"就会执行仓库里的恶意 hook；内部红队用钓鱼邮件让员工亲手粘贴恶意 prompt，25 次里 24 次成功外传 `~/.aws/credentials`；Cowork 的出口白名单放行了 api.anthropic.com，结果恶意文件带着攻击者的 API key 把数据"合法"上传到了攻击者的账户。

## 【关键洞察】

1. **批准（approval）不是防御，是疲劳陷阱。** 遥测显示用户批准了 93% 的权限提示。从底层看：任何概率性防御（模型、分类器、人的注意力）都有非零漏过率，且漏过率随审查次数上升而恶化。人脑和分类器一样会"注意力衰减"。所以真正的防御必须是确定性的边界（沙箱、VM、egress 控制），边界内的行为才可以让模型自由发挥。这是"控制能做什么"与"监督做了什么"的差别。

2. **模型层防御只能改变"倾向"，不能改变"能力"。** system prompt、分类器、训练修改塑造的是 agent 倾向于做什么，理论上它永远做得到。Opus 4.7 在 prompt injection 基准上单次攻击成功率压到 0.1%，但 100 次自适应攻击后是 5-6%；auto mode 拦截约 83% 的过激行为，漏 17%。任何单点概率防御都无法独立成立，只能层叠，且最后一层必须是确定性的环境边界。

3. **出口白名单的正确语义不是"目的地过滤"，而是"能力授予"。** 这是全文最深刻的一条。放行 api.anthropic.com 意味着放行了该域名上所有可达功能，包括"向任意账户上传文件"。沙箱工作得完美、数据照样通过"合法路径"外泄。修复方式是在 VM 内部做 MITM 代理，只放行携带 VM 自身 session token 的请求，拒绝外部植入的 key，并封掉 server-side fetch 头。要点：白名单校验身份与功能，不能只校验目的地。

4. **你自己写的组件永远是最弱层。** 三次重大故障全部出在自研组件（自定义 egress 代理、项目配置加载时机），而 gVisor、seccomp、hypervisor 这些被数代对手打磨过的底层原语全部守住了。"最弱的一层是你自己搭的"是安全界最老的教训，在 agent 时代被反复验证。另外两个容易被忽略的点：信任对话框之前执行的代码（clone 仓库就自动跑的 hook）应按"来自互联网的入站请求"对待；用户自己粘贴的 prompt 是模型层完全无法识别的注入向量（分类器锚定"用户意图"，用户亲自输入的恶意指令没有任何异常信号），这种场景下唯一有效的防御还是环境层的 egress 与文件系统边界。

## 【落到你手上】

1. **审视 .114 上的自建推理服务出口。** 如果助手或 agent 流程里给模型开了"能上网"或调外部 API 的白名单，把它当"能力授予"重新读一遍：这个域名上还有哪些可达功能（上传？fetch？）？自建反代里除了查目的地，加一条"只认自己签发的 session token + 封 SSRF 头"的校验。一条 curl 就能验证：用外部 key 打你的代理，看它是否放行。

2. **验证 HanaAgent 的 sandbox 网络边界是硬边界。** 当前 sandbox 是 read-all_write-scoped_network-on，跑一条 `curl -m 5 https://example.com` 确认沙箱内网络确实被拦（走 require_escalated 才通），确认"边界内自由"的前提成立。

3. **把记忆/工作区文件当成"每次启动都会重新加载的注入面"。** 文章预测的最大威胁是 persistent memory poisoning：memory 文件、CLAUDE.md、挂载工作区里的内容每次会话启动都会被重新读进上下文，落在里面的注入是常驻的。具体动作：检查你的工作区和 HanaAgent 的记忆目录里有没有来路不明的文件（尤其 git clone 下来的仓库），以及 clone 任何仓库前确认启动流程不会自动执行仓库内的配置和脚本。

## 【原文摘录】

- "Previously, we'd conceptualized the allowlist as a destination filter, something that told Claude these domains are okay to talk to. But it may be better conceptualized as a capability grant. Every function reachable through any domain on an allowlist is now an attack surface."
- "The weakest layer is the one you built yourself. gVisor and seccomp have been hardened against well-resourced adversaries for far longer than agentic AI has existed."
- "The only defense that holds in this situation is the environment, specifically egress controls that block the POST regardless of intent and filesystem boundaries that keep ~/.aws out of reach in the first place."
