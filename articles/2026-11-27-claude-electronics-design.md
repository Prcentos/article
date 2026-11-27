# Claude can design electronic prototypes from start to finish（Claude 从零设计电子原型）
- 归档日期：2026-11-27 / 原文：https://www.anthropic.com/research/claude-electronics-design / 分类：research

## 讲了什么

Anthropic 和电路板制造商 Diode Computers 合作，让 Claude Fable 5.2 用极简 prompt 完整设计了三台真能用的设备：USB 音箱电话、家庭语音助手 hub、手持电子宠物。流程覆盖写规格、画原理图、PCB 布局布线、3D 外壳、固件、物料清单，最后 Diode 直接用设计文件打板，原型一周没修过电路。一年前同任务上，模型连板子都布不完。

文章把进步拆成四个能力：几何推理（先布高速差分对、等长走线、调谐锯齿）、自我修正（发现走线"乱到看不出"就整片撕掉重来，而不是打补丁）、写自己的工具（嫌 KiCad 内置 router 一次只画一条太慢，半小时写了个网格搜索路由器，长走线一秒一条）、约束追踪（外壳 25 度斜放才能装下、麦克风腔体 25mm 会在 3.4kHz 共振就改成楔形壳）。

## 关键洞察

1. **这一年的核心进步是"空间约束求解"，不是知识**。电子设计难的不是芯片参数（查 datasheet 就行），而是几何：走线不能交叉、等长、绕开天线区、避开触控电极下方的过孔。这类问题在 LLM 训练语料里几乎没有"标准解"，只能靠推理时逐步约束搜索。Fable 5.2 在 PCB Gym 上 69% 对上一代 25%，接近 3 倍，这是推理能力在物理空间问题上的直接体现。
2. **"放弃重做"比"修补"是更深的能力**。旧模型的典型失败模式是 95% 完成但不可用：局部修补会把错误锁死在结构里。Fable 5.2 在走线缠绕时选择整片删除重新布线，在背面拥挤时交换可互换引脚重新摆一整块区域。底层原理：修补是局部贪心，重做是重启全局搜索；能判断"我的方案结构性地走不通"并主动推倒，是 agent 摆脱 95% 陷阱的关键。
3. **工具粒度要匹配 agent 的决策粒度**。KiCad router 的粒度是"两条点之间画一条线"，但 Fable 5.2 的决策粒度是"整块区域怎么走"，于是它自己写了个批量 router。当外部工具比 agent 的决策单元更细时，agent 的最优策略是造自己的工具，而不是逐步操作。这和 code execution with MCP（用代码批量调工具）是同一逻辑的极端形态。
4. **没有验证就没有"一次打板成功"**。DFM 检查和物料检查是打板前的验证层，且关键细节是：Claude 把检查结果分成"必须修"和"可以忽略"两类（自动检查器不认识的接地桥接被正确判断为故意设计）。验证不只是跑检查，还要判断检查结果——否则 agent 会被误报淹没或过度修改。

## 落到你手上

1. **给 HanaAgent 的子任务加"重做协议"**：subagent/workflow 里任务卡住时，现在的默认行为是继续打补丁。参照这篇，在任务 prompt 里明确一条：当发现方案结构性地走不通（同一错误第三次出现、或需要补丁叠补丁），允许并鼓励 agent 整段推倒重做，而不是在坏结构上缝。今天就可以在下一个 workflow 的 agent() prompt 里加这一句，观察是否减少"95% 完成但不可用"的产出。
2. **openhanako 工具粒度试验**：找一个你现有的、需要连续 N 次调用才完成一件事的工具流程（比如多次 read+grep 定位一个改动点），让 agent 自己写一个小脚本把 N 次调用合并成一次。这篇的 router 例子说明：agent 造批量工具的成本，远低于它逐步操作的成本，而且批量操作更容易一次做对。
3. **给 chenliangorient.xyz 推理服务加"DFM 式预检"**：打板前跑 DFM 的本质是"发布前用检查器对比目标环境的限制"。今天可动手：写一个 deploy-check 脚本跑在推送配置前，检查项包括依赖包是否可装（apt 可解）、端口占用、模型文件大小 vs 磁盘余量、OOM 风险（模型大小 vs 可用内存），并把结果分成 must-fix / tolerable 两类输出，must-fix 才拦部署。

## 原文摘录

- "When Claude noticed one of its designs wasn't working, it would change course without being prompted to do so... It proceeded to delete and reroute copper connections dozens of times until it found an arrangement that worked."（Claude 发现设计走不通时会主动改变方向，删掉重布铜走线几十次，直到找到可行的方案。）
- "KiCad's router drew one track at a time between two points Claude chose. But the hub had long tracks running from a 100-pin connector, and Claude judged that routing these one by one would take too much step-by-step guidance. Instead, Claude wrote its own router."（KiCad 的 router 一次只能画两条点之间的线，Claude 判断这样太慢，于是自己写了一个路由器。）
- "Diode manufactured all three boards directly from Claude's design files, without any back-and-forth over adjustments."（Diode 直接用 Claude 的设计文件制造了全部三块板，没有任何来回修改。）
