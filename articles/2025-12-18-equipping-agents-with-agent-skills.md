# Equipping agents for the real world with Agent Skills
- 归档日期：2026-11-27 / 原文：https://www.anthropic.com/research/equipping-agents-for-the-real-world-with-agent-skills（发布 2025-12-18，2026-01 更新为开放标准）/ 分类：engineering

## 讲了什么

Skill 的本质是一个带 SKILL.md 的目录：YAML frontmatter 里写 name 和 description，正文写操作规程，旁边可以挂任意脚本和资源文件。它想解决的问题是：通用 agent（像 Claude Code 这种会执行代码、操作文件系统的）能力越来越强，但领域知识没法无限塞进 system prompt。Skill 把"给新员工写的 onboarding 手册"这个概念做成了 agent 的原生格式。

整套机制的核心是**渐进式披露**（progressive disclosure），分三层：第一层是每个 skill 的 name+description，启动时全部预载进 system prompt，token 成本极低，只用来判断"什么时候该用我"；第二层是 SKILL.md 正文，agent 判断相关时才读进来；第三层是目录里捆绑的其他文件（如 PDF skill 里的 forms.md），只在真正遇到具体场景时才读。因为 agent 有文件系统，它不需要一次读完所有内容，所以单个 skill 能捆绑的上下文量实际上是无限的。

除了"渐进式加载的知识"，skill 还带**确定性代码**：PDF skill 里有一个预写的 Python 脚本提取表单字段，agent 直接执行它，既不把脚本和 PDF 塞进上下文，又获得确定性结果。文章还给了开发建议（从评测缺口出发、为规模拆分文件、站在 Claude 的视角迭代、让 Claude 自己把成功路径沉淀回 skill）和一条安全警告：恶意 skill 可以注入指令让 agent 外传数据，只装可信来源，装之前通读文件。

## 关键洞察

1. **Skill 是对"上下文窗口经济学"的一次制度设计**。它把知识访问从"全量预载"改成"按引用付费"，和操作系统虚拟内存的分页加载、JIT 编译的按需优化是同一个思想：热数据（触发判断）放最便宜的层（几行元数据），冷数据（完整规程）留在磁盘，用到才换进内存。这解决了 LLM 应用最本质的矛盾：知识越丰富，上下文越挤。
2. **description 是路由层，不是文档层**。第一层披露的本质是一个分类器：agent 只看 name+description 决定要不要加载。所以写好 description 的重要性等于写好 API 的路由注释——它错了，整个 skill 就是死的（永远不会被触发）。这也是为什么文章反复强调"站在 Claude 的视角观察触发行为"。
3. **确定性代码和生成式推理的分工边界在"可形式化"上**。排序、解析、提取这类有精确解的操作用 token 生成是既贵又不可靠的路径，代码一次算清。skill 把这个分工显式化了：LLM 负责判断"该做什么"，代码负责"把它做对"。这和 agent 工具设计的通则一致，但 skill 让它变成了可复用、可分发的单元。
4. **安全模型跟着能力边界走**。skill 给了 agent"读文件+执行代码"的组合权限，等于把攻击面从"prompt 注入"扩大到"供应链注入"——恶意 skill 是持久化的、带代码的、有说服力的注入。Anthropic 的建议是装前通读、只信可信源，这和对待 npm 包、VS Code 扩展的逻辑完全一致：agent 时代，一切"给 agent 读的东西"都是代码。

## 落到你手上

1. **HanaAgent 已经在用这套格式了，可以反过来验证自己的理解**：你机器上 `~/.hanako/skills/` 里的 quiet-musing、skill-creator 都是标准 skill。花十分钟检查它们的 description 字段是否精准表达了触发边界（比如 quiet-musing 里的 "Do NOT activate for simple Q&A" 就是防误触发的路由约束），然后给自己写一个 20 行的 skill（比如"服务器排查"，内嵌 `tailscale status`、`docker ps` 等常用命令的规程），体会三层披露的实际 token 开销。
2. **把 114 机器的运维经验固化成 skill**：你排查过的模型服务访问、光猫、BMC 这些坑，现在散落在聊天记录里。按文章的方法"让 Claude 把成功路径沉淀回 skill"，建一个 `server-ops/SKILL.md`，正文写排查顺序，references 里挂各服务的端口/路径清单。下次你从微信发一句"114 又挂了"，agent 按 skill 走，不用你从头交代背景。
3. **验证确定性代码路径**：给自建推理服务写一个小 skill，里面带一个 Python 脚本（比如批量检查 endpoint 的延迟和 token 吞吐），让 agent 直接执行脚本而不是自己 curl 逐个试。对比两种方式的 token 消耗，体感"代码比生成便宜"这句话的真实数字。

## 原文摘录

- "Building a skill for an agent is like putting together an onboarding guide for a new hire."（给 agent 建 skill 就像给新员工写 onboarding 手册。）
- "Progressive disclosure is the core design principle that makes Agent Skills flexible and scalable."（渐进式披露是让 Agent Skills 灵活且可扩展的核心设计原则。）
- "Large language models excel at many tasks, but certain operations are better suited for traditional code execution. For example, sorting a list via token generation is far more expensive than simply running a sorting algorithm."（LLM 在很多任务上表现出色，但某些操作更适合传统代码执行，比如用 token 生成来排序远比跑一个排序算法贵得多。）
- "Malicious skills may introduce vulnerabilities in the environment where they're used or direct Claude to exfiltrate data and take unintended actions."（恶意 skill 可能引入漏洞，或诱导 Claude 外传数据、执行非预期操作。）
