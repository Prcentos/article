# Code execution with MCP: Building more efficient agents
- 日期：2026-11-23
- 链接：https://www.anthropic.com/research/code-execution-with-mcp
- 分类：engineering
- 作者：Adam Jones & Conor Kelly（Anthropic）

## 讲了什么

MCP 让 Agent 接上成百上千个工具，但带来两个隐性成本：一是所有工具定义预先灌进上下文窗口，几千个工具意味着模型还没读用户请求就要处理几十万 token；二是工具调用的中间结果必须过模型，比如从 Google Drive 拉一份两小时的会议纪要再写进 Salesforce，全文要流经上下文两次，轻松多耗 5 万 token，还容易在"复制粘贴"环节出错。

解法是把 MCP server 暴露成代码 API 而不是直接的 tool call：把每个工具生成一个文件，组织成文件系统树（`servers/google-drive/getDocument.ts`），Agent 通过浏览文件系统按需加载工具定义，并写代码完成"拉取 → 过滤 → 写入"的整条链路。例子里 token 用量从 15 万降到 2 千，省 98.7%。

进一步的好处：数据过滤发生在执行环境而不是模型脑子里（1 万行表格只回显 5 行）；循环、条件、轮询用代码写而不是反复调工具，省掉每一步的模型推理等待；中间结果默认不进上下文，隐私数据天然被隔离；Agent 还能把自己的代码存成可复用的 skill，慢慢积累工具箱。Cloudflare 的 "Code Mode" 是同一思路的独立验证。文章最后也提醒：代码执行需要沙箱、资源限制和监控，这些运维成本要算进去。

## 关键洞察

1. **上下文才是最贵的硬件**。token 是模型世界里唯一的稀缺资源，所有"低效"最终都折算成 token。工具定义占位是固定开销（每个会话都交），中间结果过模型是可变开销（任务越长越贵）。这两个问题在传统软件里对应的是"预加载全部依赖"和"数据经过不必要的中转层"，软件工程里早就不这么干了。
2. **文件系统是最便宜的渐进披露机制**。LLM 天生会读目录、读文件、grep——这套能力是免费的。把工具注册表变成目录树，"发现工具"就从"全量注入"变成"按需检索"，搜索成本从 O(全部工具) 降到 O(任务相关的几个文件)。这跟人类工程师查 API 文档是同一个行为模式。
3. **让数据尽量不过模型，是能力问题也是隐私问题**。模型看到的数据 = 模型可能被污染、可能泄露、可能记错的数据。过滤、聚合、字段映射这些确定性操作交给代码，模型只负责"决定做什么"，执行环境负责"确定地做"。这是控制权边界的划分：模糊的判断归模型，精确的操作归代码。
4. **旧问题新马甲**。文章最后的点睛之笔：上下文管理、工具组合、状态持久化，在软件工程里都有成熟答案（模块化、管道、持久化存储）。Agent 工程的很多"创新"本质上是把既有模式重新应用到一个新运行时上。

## 落到你手上

1. **HanaAgent/openhanako 的 MCP 工具可以改成"工具目录树 + search_tools"**。现在你接的 MCP 工具定义是随会话常驻的，每次新会话都在交固定 token 税。可以把工具定义生成到本地 `servers/` 目录，会话里只保留一个 `search_tools` 入口，按任务加载。对你跑 Qwen 本地推理的场景，262k 上下文里省下的几万 token 直接就是"一个长会话能多撑多久"。
2. **本地 Qwen3.5-27B 的长任务测试**：挑一个多步任务（比如"从本地 NAS 目录读一份日志 → 过滤错误行 → 生成摘要"），对比两种跑法——直接工具调用链 vs 让模型写一段 shell/python 一次跑完。记录两边的耗时和上下文消耗。27B 级别的模型在"写代码执行"上比小模型更稳，这个实验当天就能做，也正好验证你说的"代码执行对小模型是降维"。
3. **隐私数据不过模型的隔离模式**：你自建推理服务在公网上（chenliangorient.xyz），请求是走 API 的。凡是要处理含密码、域名凭据、个人信息的任务，可以用"中间结果留在执行环境、只回显计数和路径"的模式——比如批量改配置文件的任务，让代码直接改，模型只看到"改了 N 个文件"。

## 原文摘录

> "These problems that feel novel — context management, tool composition, state persistence — have known solutions from software engineering."

> "The agent discovers tools by exploring the filesystem... This reduces the token usage from 150,000 tokens to 2,000 tokens — a time and cost saving of 98.7%."

> "Intermediate results stay in the execution environment by default. This way, the agent only sees what you explicitly log or return."
