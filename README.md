# 调研报告大纲（因果链驱动 · 来源标注版 · v3）

## 设计原则

1. 每个论断必须标注来源文章，格式：`[来源: 文章简称]`
2. 因果链必须完整：现象 → 根因 → 朴素方案为何失效 → Harness解法 → 证据
3. 不做原文未支持的推断；如需推断，显式标注`[推断]`
4. 模型版本必须精确，不混淆
5. 章节之间必须有显式的因果过渡，不能跳跃

## 来源文章简称表

| 简称 | 全称 |
|------|------|
| [Harness设计] | Harness design for long-running application development (anthropic.com/engineering) |
| [Effective Harness] | Effective harnesses for long-running agents (anthropic.com/engineering) |
| [Harnessing Intelligence] | Harnessing Claude's intelligence (claude.com/blog) |
| [Multi-Agent协调] | Multi-agent coordination patterns (claude.com/blog) |
| [CLAUDE.md] | Using CLAUDE.md files (claude.com/blog) |
| [Auto Mode工程] | Claude Code auto mode (anthropic.com/engineering) |
| [Managed Agents工程] | Scaling Managed Agents (anthropic.com/engineering) |
| [Managed Agents博客] | Claude Managed Agents (claude.com/blog) |
| [Workflow模式] | Common workflow patterns for AI agents (claude.com/blog) |
| [EE案例] | Making Claude a better electrical engineer (claude.com/blog) |
| [Skills详解] | Skills explained (claude.com/blog) |
| [Skills改进] | Improving skill-creator (claude.com/blog) |
| [1M Context] | 1M context is now GA (claude.com/blog) |
| [Auto Mode博客] | Auto mode for Claude Code (claude.com/blog) |
| [Subagents] | How and when to use subagents in Claude Code (claude.com/blog) |
| [Memory文档] | Use Claude Code - How Claude remembers your project (code.claude.com/docs) |
| [上下文管理] | Claude Code 上下文管理 (runoob.com) |
| [记忆系统] | Claude Code 记忆系统 (runoob.com) |
| [AgentTeams] | Claude Code AgentTeams (johng.cn) |
| [Managed Agents实践] | Claude Managed Agents：托管式长时智能体 (johng.cn) |
| [Superpowers] | obra/superpowers GitHub仓库 |
| [Routines] | Introducing Routines in Claude Code (claude.com/blog) |

---

## 第一章 What — 三个基本概念

### 1.1 什么是长时运行智能体？

- "长时"的本质：任务跨越多个上下文窗口才能完成 [来源: Effective Harness]
  - 原文类比："Imagine a software project staffed by engineers working in shifts, where each new engineer arrives with no memory of what happened on the previous shift." [来源: Effective Harness]
- 即使是Opus 4.5这样的前沿模型，在跨多个上下文窗口的循环中也会失败 [来源: Effective Harness — "even a frontier coding model like Opus 4.5 running on the Claude Agent SDK in a loop across multiple context windows will fall short"]
- 具体怎么失败？→ 引出1.2（上下文的有限性）和1.3（Harness的存在意义）

### 1.2 什么是上下文（Context）？

- 上下文窗口是模型在单次会话中"看到"的全部内容，上限100万token [来源: 1M Context]
- 上下文的构成：系统提示 + CLAUDE.md + Auto Memory + 对话历史 + 工具调用结果 + MCP/Skills描述 [来源: Memory文档, 上下文管理]
- 关键属性：上下文是Agent唯一的信息来源——它不记得窗口之外的任何东西 [来源: Effective Harness — 每个新会话从零开始; Memory文档 — "each new session starts with a fresh context window"]
- **上下文的两个方向性挑战（为Ch2铺垫）：**
  - 向上：上下文快满了→模型行为异常（焦虑收尾）[来源: Harness设计 — Sonnet 4.5 context anxiety]
  - 向下：上下文装太多→质量退化 [来源: CLAUDE.md — "signal-to-noise ratio drops"]
  - 这两个方向共享同一根因——**上下文是有限资源**——但走向不同的症状

### 1.3 什么是Harness？

- 定义："A harness is an orchestration framework that surrounds an AI model to manage how it executes complex, long-running tasks." [来源: Harness设计]
- 为什么需要Harness？因为1.2中的上下文有限性导致Agent在长时任务中失败——Harness的工作就是编码"模型自己做不到什么"的补偿措施
- 核心洞察："Every component in a harness encodes an assumption about what the model can't do on its own, and those assumptions are worth stress testing, both because they may be incorrect, and because they can quickly go stale as models improve." [来源: Harness设计 — 原文引用]
- 设计原则："Find the simplest solution possible, and only increase complexity when needed." [来源: Harness设计 — 引用自Anthropic的Building Effective Agents]

**→ 过渡到Ch2：** 既然Harness编码的是"模型做不到什么"，那么模型到底在哪些方面做不到？这就引出五个核心难题。

---

## 第二章 Why — 五个核心难题及其根因

### 2.0 五个难题的因果关系图

在逐一展开前，先展示五个难题之间的关系——它们不是五个独立问题，而是共享底层机制、互相影响的系统：

```
上下文窗口是有限资源（元根因）
    │
    ├── 方向一：快满了
    │   └── 难题一：上下文焦虑 — 模型感知即将耗尽，提前收尾
    │
    ├── 方向二：装太多
    │   └── 难题三：上下文退化 — 信噪比下降，偏航，趋向平庸
    │
    ├── 后果：窗口是会话级的
    │   └── 难题四：跨会话记忆断层 — 新会话从零开始
    │
自我评估不可靠（独立根因）
    └── 难题二：过早标记完成 — Agent对自身工作系统性高估
         ↑ 注意：难题一（焦虑）可加剧难题二（急于声称完成），但根因不同

二元权限模型（独立根因）
    └── 难题五：自主与安全矛盾 — 全批或全拒，无中间态
```

**为什么焦虑和退化要分开？** 虽然都源于"上下文有限"，但解法方向不同——焦虑需要"给新起点"（重置/扩窗），退化需要"隔离不相关信息"（子智能体/渐进披露）。合并会模糊解法的针对性。

**为什么焦虑和过早完成要分开？** 现象相似（都是"还没做完就说做完了"），但根因不同——焦虑是"被上下文压力逼的"，过早完成是"自我评估能力差"。解法也不同——前者治上下文，后者治评估机制。

### 2.1 难题一：上下文焦虑（Context Anxiety）

**现象：**
- 模型接近上下文限制时，匆忙结束任务 [来源: Harness设计 — "Claude Sonnet 4.5 exhibited context anxiety strongly enough that compaction alone wasn't sufficient"]
- 原文称此为"context anxiety"并加了引号，表明是 coined term [来源: Harnessing Intelligence — "We added resets to clear the context window in order to address this 'context anxiety.'"]

**根因：**
- 模型感知上下文即将耗尽，触发"收尾"行为 [来源: Harnessing Intelligence — "Sonnet 4.5 would wrap up prematurely as it sensed the context limit approaching"]

**朴素方案为何失效：**
- 压缩（Compaction）保留了"我快满了"的感知，并未真正重置模型的状态感知 [来源: Harness设计 — "compaction alone wasn't sufficient to enable strong long task performance" for Sonnet 4.5]
- 更大的上下文窗口延缓触发但未消除行为 [推断: Opus 4.5消除焦虑是因为模型行为改变，不是窗口更大]

**模型差异（重要）：**
- Sonnet 4.5：焦虑严重，压缩不够，必须上下文重置 [来源: Harness设计]
- Opus 4.5：焦虑行为基本消除，可改为连续会话+自动压缩 [来源: Harness设计 — "Opus 4.5 largely removed that behavior on its own, so I was able to drop context resets from this harness entirely"]
- Opus 4.6：进一步改善，冲刺构造可移除 [来源: Harness设计]

### 2.2 难题二：过早标记完成

**现象：**
- "A later agent instance would look around, see that progress had been made, and declare the job done." [来源: Effective Harness — 原文引用]
- Agent对自身工作质量系统性高估："When asked to evaluate their own work, agents confidently praise mediocre outputs." [来源: Harness设计]
- 跑了单元测试或curl就声称功能完成，但端到端不工作 [来源: Effective Harness — "Without explicit prompting, agents tend to make code changes, run unit tests or curl commands, fail to recognize the feature doesn't work end-to-end"]

**与难题一的区别：** 难题一是"被上下文压力逼的收尾"，难题二是"真心认为自己做完了"——即使上下文还很充裕也会发生 [来源: Harness设计 — "especially pronounced for subjective tasks where there's no binary pass/fail check"，与上下文是否快满无关]

**根因：**
- 自我评估不可靠——尤其对主观任务没有二元pass/fail检查 [来源: Harness设计]
- Out of the box，Claude会"identify legitimate issues, then talk itself into deciding they weren't a big deal and approve the work anyway" [来源: Harness设计 — 原文引用]

**朴素方案为何失效：**
- 让Agent自己检查自己的工作，就像让学生自己给自己打分——调优独立评估器"far more tractable than making a generator critical of its own work" [来源: Harness设计]

### 2.3 难题三：上下文退化与偏航

**现象：**
- 对话越长，响应越模糊 [来源: CLAUDE.md — "signal-to-noise ratio drops"]
- 逐渐偏离原始目标："For complex tasks, agents tend to drift from the original plan over time without external structure to keep them on track." [来源: Harness设计]
- 输出趋向平庸默认："Without intervention, models gravitate toward safe, predictable outputs." [来源: Harness设计]

**与难题一的关系（2.0图中已铺垫）：** 同一元根因的两个方向。焦虑是"快满了怎么办"，退化是"装太多怎么办"。

**根因：**
- 信噪比下降——早期无关信息干扰当前任务注意力 [来源: CLAUDE.md]
- 缺乏外部结构纠偏 [来源: Harness设计]

**朴素方案为何失效：**
- 更大的上下文窗口延缓但未消除退化 [推断: 基于Opus 4.6在BrowseComp上84%的表现仍低于理论上限]
- 1M上下文仍有MRCR衰减——Opus 4.6在1M上下文MRCR v2为78.3% [来源: 1M Context]

### 2.4 难题四：跨会话记忆断层

**现象：**
- 每个新会话从零开始 [来源: Memory文档 — "each new session starts with a fresh context window"; Effective Harness — 轮班工程师类比]

**根因：**
- 上下文窗口是会话级资源，不跨会话持久化 [来源: Effective Harness — "each new session begins with no memory of what came before"; Memory文档]

**朴素方案为何失效：**
- 手动重复告知成本高且不一致 [推断]
- RAG检索有精度损失 [推断: 非参考文章明确论证]

### 2.5 难题五：自主性与安全性的矛盾

**现象：**
- 用户93%的权限提示都会批准 [来源: Auto Mode工程 — "users approve 93% of prompts anyway"]
- `--dangerously-skip-permissions`完全跳过审批但无保护 [来源: Auto Mode工程]

**根因：**
- 二元权限模型（全批/全拒）无法区分操作的风险等级 [来源: Auto Mode工程 — Auto Mode设计为"middle ground"]

**朴素方案为何失效：**
- 沙箱安全但限制能力（断网、无外部工具访问） [来源: Auto Mode工程 — "sandboxing — safe but high-maintenance and breaks capabilities needing network/host access"]
- 全权限灵活但危险 [来源: Auto Mode工程]

**→ 过渡到Ch3：** 五个难题的根因已明确，朴素方案为何失效也已说明。接下来的问题是：Harness如何精确对准根因给出解法？

---

## 第三章 How — 针对每个难题的因果解法

每个小节严格按"根因 → 解法原理 → Claude Code官方做法 → 行业做法 → 证据"展开

**前置说明：子智能体是跨难题的统一机制**

3.1和3.3都用到了子智能体，但侧重点不同：
- 3.1中，子智能体通过隔离减少主上下文的信息量→缓解"快满了"的压力→减轻焦虑
- 3.3中，子智能体通过隔离不相关信息→提高主上下文的信噪比→抵抗退化

两者共享同一机制（上下文隔离），但服务于不同目标。这正体现了2.0图中的"同一元根因的两个方向"。

### 3.1 消解上下文焦虑：给Agent一个"新的起点"

**根因回顾：** 模型感知到上下文临近耗尽，触发提前收尾行为 [来源: Harness设计, Harnessing Intelligence]

**解法原理：** 三条路径——
1. 重置上下文，消除"快满了"的感知 [来源: Harness设计 — context resets]
2. 延缓耗尽时刻（扩窗） [来源: 1M Context]
3. 减少主上下文的信息量（隔离），让有限的窗口装更多有效工作 [来源: Subagents, Harnessing Intelligence]

**Claude Code官方做法：**

| 能力 | 说明 | 来源 |
|------|------|------|
| /compact | 压缩对话历史，但Sonnet 4.5下压缩不够 | Harness设计 |
| /clear | 彻底重置，CLAUDE.md和Auto Memory存活 | 上下文管理, Memory文档 |
| 1M上下文窗口 | GA for Opus 4.6和Sonnet 4.6，无长上下文溢价 | 1M Context |
| 子智能体 | 独立上下文窗口，减少主上下文负担 | Subagents |
| Auto Compaction | Claude Agent SDK自动压缩 | Harness设计 |

**行业做法：**

- 上下文重置+交接物模式 [来源: Effective Harness]：
  - 初始化Agent创建环境脚手架（init.sh + progress文件 + 特性列表 + git基准）
  - 编码Agent每次只做一个特性，保持代码可合并状态
- Superpowers的Worktree隔离：在独立分支上自主工作 [来源: Superpowers — using-git-worktrees skill]

**因果证据：**

| 证据 | 来源 |
|------|------|
| Sonnet 4.5：压缩不够，必须上下文重置 | Harness设计 |
| Opus 4.5：焦虑消除，重置变为多余，可改为连续会话 | Harness设计 |
| Opus 4.6：冲刺构造可移除，评估器改为最终一次性 | Harness设计 |
| 子智能体使BrowseComp提升2.8%（Opus 4.6） | Harnessing Intelligence |
| 1M上下文使压缩事件减少15% | 1M Context — Jon Bell (CPO) 报告 |
| Hex报告：上下文从200K到500K反而总token减少 | 1M Context |

### 3.2 防止过早标记完成：引入外部裁判

**根因回顾：** 自我评估不可靠，Agent对自身输出系统性高估 [来源: Harness设计]

**解法原理：** 三条路径——
1. 分离生成者与评估者 [来源: Harness设计 — GAN启发]
2. 用结构化检查清单替代主观判断 [来源: Effective Harness — Feature List]
3. 强制运行验证命令 [来源: Effective Harness, Superpowers]

**Claude Code官方做法：**

| 能力 | 说明 | 来源 |
|------|------|------|
| /rewind | 基于检查点的回滚 | 上下文管理 |
| Generator-Evaluator分离 | 三Agent系统中的独立评估器 | Harness设计 |
| 冲刺合约(Sprint Contract) | 做之前先定义"完成"的标准 | Harness设计 |
| Managed Agents可配置评估Agent | 独立评估能力 | Managed Agents博客 |

**行业做法：**

- Feature List（结构化特性清单） [来源: Effective Harness — 作者自定义的Harness组件，非Claude Code内置功能]：
  - JSON格式的特性列表，Agent只能标记passes字段，不能删除或编辑测试项
  - JSON优于Markdown："the model is less likely to inappropriately change or overwrite JSON files compared to Markdown files." [来源: Effective Harness]
  - claude.ai克隆案例中生成了"over 200 features" [来源: Effective Harness]
- Superpowers两阶段审查 [来源: Superpowers — subagent-driven-development skill]：
  - 第一阶段：规格合规审查（"Do NOT trust the implementer's report. Read the actual code. Compare against requirements line by line."）
  - 第二阶段：代码质量审查（仅在规格合规通过后启动）
  - 审查循环：发现问题→修复→再审查，直到批准
- Superpowers证据优先验证 [来源: Superpowers — verification-before-completion skill]：
  - "NO COMPLETION CLAIMS WITHOUT FRESH VERIFICATION EVIDENCE"
  - 引用"24 failure memories"——24次过早声称完成导致的实际事故
- GAN启发的Generator-Evaluator [来源: Harness设计]：
  - 独立评估器用few-shot examples校准 [来源: Harness设计]
  - 调优循环：读评估器日志→找到与人类判断分歧处→更新prompt→重复 [来源: Harness设计]

**因果证据：**

| 证据 | 来源 |
|------|------|
| 独立评估器仍偏宽容，但调优独立评估器远比让生成器批判自己可行 | Harness设计 |
| Out-of-box的Claude会识别问题后说服自己放行 | Harness设计 |
| 评估器价值取决于任务是否超出模型可靠能力范围——在范围内是开销，在范围外是必需品 | Harness设计 |
| Superpowers记录24次过早声称完成事故 | Superpowers |

### 3.3 抵抗上下文退化：隔离而非堆叠

**根因回顾：** 信噪比随上下文增长而下降 [来源: CLAUDE.md]

**解法原理：**
1. 不让无关信息进入主上下文——每个子任务用独立上下文 [来源: Subagents]
2. 按需加载而非全量预载 [来源: Skills详解, Harnessing Intelligence]
3. 让模型自己决定什么该保留 [来源: Harnessing Intelligence]

**与3.1的子智能体互补说明：** 3.1用子智能体减少主上下文的"量"（延缓耗尽），3.3用子智能体提高主上下文的"质"（信噪比）。同一机制，不同目标。

**Claude Code官方做法：**

| 能力 | 说明 | 来源 |
|------|------|------|
| 子智能体 | 独立上下文窗口，返回综合结果而非原始内容 | Subagents |
| Skills渐进式披露 | ~100 token元数据→<5K token完整指令→按需加载资源 | Skills详解 |
| .claude/rules/ | 路径感知规则，仅在操作匹配文件时加载 | Memory文档 |
| 代码作为编排机制 | 让Claude通过写代码来编排工具调用逻辑，而非全部经过上下文窗口 | Harnessing Intelligence |
| 上下文编辑(context editing) | 选择性移除过时/无关的上下文 | Harnessing Intelligence |
| /btw | 旁路提问，不进入对话历史 | 上下文管理 |

**行业做法：**

- Superpowers每个任务派发新子智能体 [来源: Superpowers — subagent-driven-development skill]：
  - "They should never inherit your session's context or history — you construct exactly what they need."
  - 控制器读取计划，提取任务全文，精确构造每个子智能体所需的上下文
- Superpowers描述即触发器设计 [来源: Superpowers — writing-skills skill]：
  - 技能描述只写触发条件，不摘要工作流
  - 原因：测试发现当描述摘要了工作流时，Claude会跟着描述走而非读取完整技能正文
- Superpowers Skills使用说服力研究强化纪律 [来源: Superpowers — writing-skills/persuasion-principles]：
  - 基于Meincke et al. (2025, N=28,000)研究
  - Authority + Commitment + Social Proof使合规率从33%提升到72%

**因果证据：**

| 证据 | 来源 |
|------|------|
| Opus 4.6自过滤工具输出：BrowseComp 45.3%→61.6% | Harnessing Intelligence |
| 子智能体提升BrowseComp 2.8%（Opus 4.6） | Harnessing Intelligence |
| Sonnet 4.5 + Memory Folder：BrowseComp-Plus 60.4%→67.2% | Harnessing Intelligence |
| Pokémon案例：Sonnet 3.5写了31个流水账文件仍在第二个城镇；Opus 4.6写了10个有组织的文件获得3个道馆徽章 | Harnessing Intelligence |

### 3.4 弥合跨会话断层：持久化关键知识

**根因回顾：** 上下文窗口是会话级资源，不跨会话持久化 [来源: Effective Harness, Memory文档]

**解法原理：**
1. 将关键知识写入窗口之外的持久存储 [来源: Memory文档 — CLAUDE.md + Auto Memory]
2. 新会话启动时自动加载 [来源: Memory文档]
3. 用Session日志作为外部上下文对象，可按需回溯 [来源: Managed Agents工程]

**Claude Code官方做法：**

| 能力 | 说明 | 来源 |
|------|------|------|
| CLAUDE.md | 项目级指令，每次会话自动注入；层级结构（企业>用户>项目>子目录） | Memory文档 |
| Auto Memory | Claude自动积累的跨会话笔记；MEMORY.md索引（前200行/25KB加载）+ 主题文件按需读取 | Memory文档 |
| .claude/rules/ | 模块化路径感知规则 | Memory文档 |
| Session (Managed Agents) | Append-only事件日志，可通过getEvents()按位置切片回溯 | Managed Agents工程 |
| # 快捷键 | 快速写入CLAUDE.md | 上下文管理 |

**行业做法：**

- Superpowers session-start Hook [来源: Superpowers — hooks/session-start]：
  - 每次会话启动时注入using-superpowers技能内容
  - 该技能强制Agent在任何操作前检查是否有适用的技能
- 初始化Agent创建的持久化脚手架 [来源: Effective Harness]：
  - init.sh（启动开发服务器）
  - claude-progress.txt（进度日志）
  - feature_list.json（特性路线图，JSON格式因为模型更不容易不当修改）
  - 初始git提交

**因果证据：**

| 证据 | 来源 |
|------|------|
| 无CLAUDE.md时每次新会话需重复解释相同决策 | CLAUDE.md — "eliminating the need to repeatedly explain basic project information" |
| 项目根CLAUDE.md在/compact后重新从磁盘注入 | Memory文档 — "Project-root CLAUDE.md survives compaction" |
| 嵌套子目录的CLAUDE.md在压缩后不会自动重新注入 | Memory文档 |
| Auto Memory是机器本地的——不跨机器同步 | Memory文档 |
| Session日志让任何组件崩溃后都能通过wake(sessionId)恢复 | Managed Agents工程 |

### 3.5 平衡自主与安全：分级权限 + 隔离执行

**根因回顾：** 二元权限模型无法区分操作风险等级 [来源: Auto Mode工程]

**解法原理：**
1. 用分类器替代二元判断 [来源: Auto Mode工程]
2. 在隔离环境中执行以降低风险 [来源: Auto Mode工程, Managed Agents工程]
3. 凭证与沙箱分离 [来源: Managed Agents工程]

**Claude Code官方做法：**

| 能力 | 说明 | 来源 |
|------|------|------|
| Auto Mode三级权限 | 安全工具白名单→项目内文件操作→分类器审批 | Auto Mode工程 |
| 双层安全 | 输入侧：提示注入检测；输出侧：行为分类器（两阶段：快速过滤+CoT推理） | Auto Mode工程 |
| 拒绝-继续模型 | 被阻止后Agent转向安全路径，不中断任务 | Auto Mode工程 |
| 升级机制 | 3次连续拒绝或20次总拒绝时升级到人工 | Auto Mode工程 |
| 分类器推理盲 | 只看用户消息和工具调用，不看助手文本和工具结果 | Auto Mode工程 |

**行业做法：**

- Managed Agents Brain/Hands分离 [来源: Managed Agents工程]：
  - 凭证永远不进沙箱——git token在沙箱初始化时克隆，OAuth token存在安全vault中通过代理访问
  - 沙箱从"宠物"变"牲畜"——容器死了只是返回工具调用错误，Claude可以重试
- Managed Agents网络限制 [来源: Managed Agents实践]：
  - limited模式仅允许指定域名白名单
- Superpowers Git Worktree隔离 [来源: Superpowers — using-git-worktrees skill]：
  - 在独立分支上工作，主分支不受影响
  - 完成后提供4个选项：合并/推送创建PR/保持/丢弃

**因果证据：**

| 证据 | 来源 |
|------|------|
| Auto Mode分类器真实内部流量FPR 0.4% | Auto Mode工程 |
| 真实过度行动FNR 17%——多数是误判用户意图是否覆盖该操作 | Auto Mode工程 |
| 合成数据泄露FNR 5.7% | Auto Mode工程 |
| Managed Agents解耦后p50 TTFT降60%，p95降>90% | Managed Agents工程 |
| 拒绝后Agent找到安全替代路径继续 | Auto Mode工程 |

**→ 过渡到Ch4：** Ch3给出了五个难题各自的"零件"。但长时运行Agent面临的是多个问题同时出现——焦虑和退化可能同时发生，自我评估不可靠需要评估器，评估器又需要跨会话记忆来保持一致……你需要一种方式把这些零件组装成整体。这就是工作流和协调模式的作用。

---

## 第四章 How（续）— 把零件组装成整体：三个端到端场景

**本章核心问题：** Ch3给出了五个难题各自的"零件"。但真实的长时运行任务不是一次只遇到一个问题——五个问题会同时出现，零件之间需要配合。读者需要的不是模式目录，而是看到"一台完整的机器是怎么组装出来的"。

本章用三个递进的真实场景演示组装过程：

- 场景一：最复杂——从一句话到完整应用，五大难题全部出现，展示完整零件组合
- 场景二：人不在场——夜间自动修Bug，展示无人值守时零件怎么自动运转
- 场景三：多个Agent——并行开发，展示Agent之间怎么分工协作

### 4.1 场景一：从"做一个claude.ai克隆"到完整应用

**任务描述：** 用户给出一句话需求（如"做一个claude.ai克隆"），Agent需要自主完成从设计到实现的全部工作。这个任务可能持续数小时，跨越多个上下文窗口。 [来源: Harness设计, Effective Harness]

**阶段一：设计 — 防止偏航**

| 步骤 | 遇到的问题 | 用的零件 | 零件怎么配合 | 来源 |
|------|-----------|---------|------------|------|
| Planner Agent将一句话扩展为产品规格 | 难题二（自我评估差：一句话需求太模糊，Agent会凭想象填补，导致方向错误） | 3.2的外部裁判思路，但此阶段裁判是人——设计必须经用户批准才能动手 | 人作为"裁判"把关方向 | Harness设计 |
| 设计文档分段呈现，每段等用户确认 | 难题三（偏航：一次性呈现全部设计，用户来不及消化就通过） | 3.3的渐进式披露——不一次全给，分段呈现 | Superpowers的brainstorming skill强制此流程 | Superpowers |

**为什么需要人来当裁判？** 因为设计阶段没有二元pass/fail标准——"设计好不好"是主观判断，自动评估器不可靠 [来源: Harness设计 — "especially pronounced for subjective tasks"]。所以此阶段用3.2的第一条路径"结构化检查清单"不合适，只能用人的判断。

**阶段二：计划 — 防止One-Shotting**

| 步骤 | 遇到的问题 | 用的零件 | 零件怎么配合 | 来源 |
|------|-----------|---------|------------|------|
| 将规格拆分为2-5分钟粒度的任务 | 难题一（焦虑：大任务一次性做完会耗尽上下文） | 3.1的"减少信息量"——拆小后每个子任务上下文需求极低 | 每个任务有精确的文件路径、完整代码、验证步骤 | Harness设计, Superpowers |
| 任务必须包含实际内容，禁止占位符 | 难题二（过早完成：Agent会写"TODO: add error handling"然后标记完成） | 3.2的"结构化检查清单"——计划本身就是一个清单，占位符=未完成 | Superpowers的writing-plans skill禁止TBD、TODO | Superpowers |

**拆小为什么同时解决两个问题？** 小任务不会触发焦虑（上下文够用），也不会触发过早完成（完成标准精确到"运行这个命令看到这个输出"）。这就是2.0因果图中"焦虑可加剧过早完成"的具体体现——拆小任务从根源上同时缓解了两者。

**阶段三：执行 — 子智能体驱动开发**

| 步骤 | 遇到的问题 | 用的零件 | 零件怎么配合 | 来源 |
|------|-----------|---------|------------|------|
| 每个任务派发一个新子智能体 | 难题一（焦虑）+难题三（退化）：主Agent上下文保持干净 | 3.1+3.3的子智能体——隔离是统一机制 | "They should never inherit your session's context or history — you construct exactly what they need." | Effective Harness, Superpowers |
| 子智能体完成后，先做规格合规审查 | 难题二（过早完成：子智能体可能声称做了但实际没做） | 3.2的"分离生成与评估"——审查者是独立子智能体 | "Do NOT trust the implementer's report. Read the actual code." | Harness设计, Superpowers |
| 规格合规通过后，再做代码质量审查 | 难题二（过早完成：功能有了但代码质量差） | 3.2的两阶段审查——先看"做没做"，再看"做得好不好" | 规格合规未通过则不启动质量审查，避免浪费 | Superpowers |
| 审查不通过→修复→再审查，循环直到通过 | 难题二（自我评估差：一次修复不一定够） | 3.2的"评估器-优化器"工作流 | 最多迭代到通过为止，但有最大迭代限制防止无限循环 | Harness设计, Superpowers |

**为什么审查要用子智能体而不是主Agent自己来？** 如果主Agent既写代码又审代码，对话历史中充满了自己写的代码→审查时会受"我写的应该没问题"的确认偏差影响 [来源: Subagents — "fresh perspective needed: unbiased review of an implementation, free from the assumptions and blind spots of the primary conversation"]。子智能体从零开始，没有"这是我写的"的心理包袱。

**阶段四：跨会话持续 — 从轮班到轮班**

| 步骤 | 遇到的问题 | 用的零件 | 零件怎么配合 | 来源 |
|------|-----------|---------|------------|------|
| 初始化Agent创建脚手架：init.sh + progress.txt + feature_list.json + git基准 | 难题四（断层：新会话不知道之前做了什么） | 3.4的持久化——把关键知识写到窗口之外 | 第一个会话专门做初始化，后续所有会话受益 | Effective Harness |
| 每个编码会话开始时：读git日志→读progress→选最高优先级未完成特性→跑init.sh→端到端验证 | 难题四（断层）+难题三（偏航：忘了原始目标）+难题二（过早完成：在已有bug上继续开发） | 3.4的记忆+3.2的验证 | "Always verify existing functionality before adding new features" | Effective Harness |
| 特性列表用JSON，Agent只能标记passes字段 | 难题二（过早完成：Agent可能删掉做不出来的特性然后说"全做完了"） | 3.2的结构化清单+3.4的持久化 | JSON比Markdown更抗篡改 | Effective Harness |
| 每次会话只做一个特性，完成后git commit + 更新progress | 难题一（焦虑：一次做太多）+难题四（断层：下一个会话需要干净起点） | 3.1的减少信息量+3.4的持久化 | "leave code in a state appropriate for merging to a main branch" | Effective Harness |

**阶段五：安全自主执行 — 人不在场时怎么办**

| 步骤 | 遇到的问题 | 用的零件 | 零件怎么配合 | 来源 |
|------|-----------|---------|------------|------|
| 在Git Worktree上工作，主分支不受影响 | 难题五（安全：Agent可能改坏主分支） | 3.5的隔离执行 | Worktree是独立的，做坏了只影响特性分支 | Superpowers |
| Auto Mode替代手动审批 | 难题五（安全：长时运行不可能每个操作都等人批准） | 3.5的分级权限 | 安全操作自动过，风险操作被分类器拦截→Agent转向安全路径 | Auto Mode工程 |
| 所有会话在Managed Agents的隔离容器中运行 | 难题五（安全：凭证不能暴露给生成的代码） | 3.5的凭证分离 | Git token在沙箱初始化时克隆，Agent永远看不到token | Managed Agents工程 |

**组装后的完整因果链：**

```
一句话需求
  → Planner扩展规格（人当裁判，防偏航）
  → 拆为小任务（防焦虑+防过早完成）
  → 子智能体逐个执行（隔离上下文，防焦虑+防退化）
  → 独立子智能体两阶段审查（分离评估，防过早完成）
  → 审查不通过→修复→再审查（评估器-优化器循环）
  → 每个任务完成→git commit+更新progress（持久化，防断层）
  → 新会话读progress→选任务→验证→继续（跨会话连续性）
  → Worktree隔离+Auto Mode+容器隔离（安全自主运行）
```

**效果数据：**
- 无Harness（Solo Opus 4.5）：$9 / 20分钟 — 核心功能坏了 [来源: Harness设计]
- 完整3-Agent Harness（Opus 4.5）：$200 / 6小时 — 完整可用的应用 [来源: Harness设计]
- 简化Harness（Opus 4.6）：$124.70 / ~4小时 — 功能完整的DAW [来源: Harness设计]

### 4.2 场景二：无人值守的夜间修Bug

**任务描述：** 每天凌晨2点，Agent自动拉取Linear中优先级最高的Bug，尝试修复，开Draft PR。全程无人参与。 [来源: Routines]

**与场景一的关键区别：** 场景一中人还在场（批准设计、审批操作），场景二中人完全不在。这意味着Ch3的某些零件不能用了，必须替换。

| 场景一中的零件 | 为什么在场景二中不可用 | 替代方案 | 来源 |
|--------------|---------------------|---------|------|
| 人批准设计 | 人睡着了 | CLAUDE.md中编码的项目规范替代人的判断 | CLAUDE.md, Routines |
| Auto Mode的分类器偶尔升级到人工 | 人不在线 | 预批准常见操作权限；升级到人工时暂停等待 | Auto Mode工程 |
| 人读设计分段并确认 | 人不在 | Routines按预定义prompt自动执行，不等人确认 | Routines |
| 子智能体审查后报告给人 | 人不在 | 审查结果直接作用于代码——审查不通过则自动修复循环，通过则自动commit | Superpowers |

**场景二的零件组合：**

```
Routines定时触发（每天2am）
  → 拉取Linear最高优先级Bug（需要3.5的安全：API token通过代理访问，Agent不直接持有）
  → 读CLAUDE.md获取项目规范（3.4的持久化：替代人的设计批准）
  → 在Worktree上创建修复分支（3.5的隔离执行）
  → 子智能体执行修复（3.1+3.3的上下文隔离）
  → 独立子智能体审查（3.2的外部裁判）
  → 审查通过→自动commit+开Draft PR
  → 审查不通过→自动修复循环
  → Session日志记录全过程（3.4的持久化：人醒来后可回溯）
```

**Routines的三种触发方式扩展了这个模式的适用范围 [来源: Routines]：**
- **计划调度：** 夜间修Bug、每周扫描文档漂移
- **API触发：** 部署后自动冒烟测试、告警自动分诊
- **Webhook触发：** 每个新PR自动代码审查（每个PR一个持续会话，接收后续更新）

**场景二揭示了一个重要洞察：** 无人值守时，3.4（持久化）和3.5（安全）变得比3.1-3.3更关键——因为记忆和安全的零件是人不在场时的唯一保障。场景一中人可以弥补记忆断层和安全漏洞，场景二中不行。

### 4.3 场景三：多Agent并行开发

**任务描述：** 大型代码库迁移——需要同时更新依赖、修改代码、修复测试、验证功能。单Agent串行处理太慢。 [来源: Multi-Agent协调, AgentTeams, Superpowers]

**与场景一/二的关键区别：** 多个Agent同时工作，引入了新问题——Agent之间会不会互相踩脚？

**新增的难题：** 多Agent之间的协调问题

| 问题 | 说明 | 来源 |
|------|------|------|
| 文件冲突 | 两个Agent同时编辑同一个文件，后写的覆盖先写的 | AgentTeams |
| 状态不同步 | Agent A修了bug，Agent B不知道还在修旧版本 | Multi-Agent协调 |
| 重复工作 | 两个Agent独立调查同一个问题 | Multi-Agent协调 |
| 完成检测 | 不同Agent完成速度不同，怎么判断整体完成？ | Multi-Agent协调 |

**这些是Ch2五个难题之外的新问题吗？** 不是。仔细看——文件冲突和状态不同步是难题四（断层）的多Agent变体，重复工作是难题三（退化中信息冗余）的变体，完成检测是难题二（过早完成）的集体版本。Ch3的零件仍然适用，但需要适配多Agent场景。

**场景三的零件组合：**

**方案A：Orchestrator-Subagent（最简起点）**

```
主Agent（Orchestrator）
  ├── 拆分任务为独立子任务（每个Agent只改自己的文件集）
  ├── 并行派发子智能体（3.1+3.3的上下文隔离）
  ├── 每个子智能体返回结果后，主Agent汇总
  └── 主Agent做最终整合和验证（3.2的外部裁判）
```

- 关键约束：子任务必须独立、有界、互不依赖 [来源: Multi-Agent协调]
- 防冲突方法：每个子智能体只被分配互不重叠的文件集 [来源: AgentTeams]
- 局限：主Agent是信息瓶颈——所有子Agent的发现都经过主Agent，细节可能丢失 [来源: Multi-Agent协调]

**方案B：Agent Teams（需要持久协作时）**

```
Team Lead
  ├── 创建共享任务列表
  ├── 派发Teammates（每个Teammate是持久的独立Agent）
  └── 汇总结果

Teammate A（前端）──┐
Teammate B（后端）──┤── 共享任务列表 + 邮箱通信
Teammate C（测试）──┘
```

- 与方案A的关键区别：Teammates是持久的——跨多个任务积累领域专长 [来源: AgentTeams, Multi-Agent协调 — "when subagents need to retain state across invocations, agent teams are the better fit"]
- 防冲突方法：共享任务列表中每个任务有唯一owner；文件编辑通过互斥文件集避免 [来源: AgentTeams]
- 协作方式：Teammates之间通过邮箱直接通信（不需要经过Lead），可以共享发现、互相挑战 [来源: AgentTeams]
- 局限：需要真正的独立性——Teammates不能轻易共享中间发现；完成检测更难 [来源: Multi-Agent协调]

**方案选择决策树 [来源: Multi-Agent协调]：**

```
工作者需要跨任务保留上下文吗？
├─ 不需要 → Orchestrator-Subagent（方案A）
└─ 需要 → 工作者之间需要实时共享发现吗？
    ├─ 不需要 → Agent Teams（方案B）
    └─ 需要 → 工作流是可预测的？
        ├─ 可预测 → 共享状态模式
        └─ 涌现的 → 消息总线模式
```

**Superpowers的方案A实践 [来源: Superpowers]：**

Superpowers选择了Orchestrator-Subagent模式，并增加了两个强化措施：
1. **两阶段审查**：每个子智能体的产出经过规格合规+代码质量双重审查（3.2的外部裁判）
2. **模型选择策略**：机械实现任务用便宜模型，架构/审查任务用最强模型——长时运行的经济学可行性 [来源: Superpowers — subagent-driven-development skill]

**三种场景的递进关系：**

| 维度 | 场景一 | 场景二 | 场景三 |
|------|--------|--------|--------|
| 人在场 | 是 | 否 | 可选 |
| Agent数量 | 1主+N子智能体 | 1个 | N个 |
| 核心依赖的零件 | 全部五组 | 3.4持久化+3.5安全为主 | 3.1+3.3隔离+3.2裁判+新增协调机制 |
| 最难的问题 | 五个难题同时出现 | 无人值守时如何替代人的角色 | 多Agent之间如何不踩脚 |
| 对应的工作流 | 顺序+评估器-优化器 | 自动化工作流（Routines） | 并行工作流+协调模式 |

**→ 过渡到案例章：** 三个场景展示了零件如何组装，但都是概念层面的。读者可能会问：有没有一个真实的项目，把这些零件完整地落地了？Superpowers就是这样一个案例。

---

## 案例章 Superpowers深度剖析——一个完整落地的长时运行Harness

**为什么选Superpowers作为案例？** 前面的Ch3给了零件，Ch4给了组装场景，但都是基于Anthropic官方文章的概念性描述。Superpowers是一个真实存在的、开源的、可安装的Harness实现——它把Ch3的每一个零件都落地为具体的技能文件、Hook脚本和Agent定义。分析它，读者能看到从概念到实现的完整路径。

### 案例一：Superpowers如何解决Ch2的五个难题

**难题一：上下文焦虑 → Superpowers的子智能体隔离**

Superpowers的核心执行引擎是`subagent-driven-development`技能。它的第一条规则：

> "They should never inherit your session's context or history — you construct exactly what they need." [来源: Superpowers — subagent-driven-development/SKILL.md]

每个任务派发一个新子智能体，控制器（主Agent）只保留协调工作的上下文。这意味着：
- 主Agent永远不会"快满了"，因为它不承载实现细节
- 子智能体从干净状态开始，不会因为之前任务的残留信息而焦虑
- 控制器一次性提取计划中所有任务的全文，然后逐个粘贴给子智能体——子智能体甚至不需要读计划文件 [来源: Superpowers — subagent-driven-development/SKILL.md Red Flags: "Never make subagent read plan file (provide full text instead)"]

**难题二：过早标记完成 → Superpowers的三重防线**

防线1：**两阶段审查**。每个子智能体完成任务后，不是自己说"完成了"就行，而是必须经过两个独立审查子智能体 [来源: Superpowers — subagent-driven-development/SKILL.md]：
- **规格合规审查**（spec-reviewer）："Do NOT trust the implementer's report. Read the actual code. Compare against requirements line by line." [来源: Superpowers — subagent-driven-development/spec-reviewer-prompt.md]
- **代码质量审查**（code-quality-reviewer）：仅在规格合规通过后启动
- 顺序严格强制："Never start code quality review before spec compliance is ✅" [来源: Superpowers — subagent-driven-development/SKILL.md Red Flags]

防线2：**证据优先验证**。`verification-before-completion`技能设立了Iron Law：

> "NO COMPLETION CLAIMS WITHOUT FRESH VERIFICATION EVIDENCE" [来源: Superpowers — verification-before-completion/SKILL.md]

它强制一个5步门控函数：IDENTIFY（什么命令能证明）→ RUN（运行命令）→ READ（读输出）→ VERIFY（确认）→ ONLY THEN（才能声称）。任何"should"、"probably"、"seems to"都被列为Red Flag。该技能记录了24次过早声称完成导致的实际事故 [来源: Superpowers — verification-before-completion/SKILL.md "From 24 failure memories"]。

防线3：**TDD的Iron Law**。`test-driven-development`技能规定：

> "NO PRODUCTION CODE WITHOUT A FAILING TEST FIRST" [来源: Superpowers — test-driven-development/SKILL.md]

如果先写了代码再写测试？删掉重来。"Don't keep it as 'reference'. Don't 'adapt' it while writing tests. Don't look at it. Delete means delete." [来源: Superpowers — test-driven-development/SKILL.md]

**难题三：上下文退化 → Superpowers的渐进式披露 + 强制技能检查**

Superpowers用两个机制对抗退化：

机制1：**描述即触发器设计（CSO）**。技能的description字段只写触发条件，不摘要工作流。为什么？测试发现当description摘要了工作流时，Claude会跟着描述走而非读取完整技能正文 [来源: Superpowers — writing-skills/SKILL.md CSO section]：

> "A description saying 'code review between tasks' caused Claude to do ONE review, even though the skill's flowchart clearly showed TWO reviews (spec compliance then code quality)."

改为只写"Use when executing implementation plans with independent tasks"后，Claude正确读取了完整流程图。这是3.3中"渐进式披露"原则的具体实现——不一次性给所有信息，按需加载。

机制2：**强制技能检查**。`using-superpowers`技能在每个会话启动时通过Hook注入 [来源: Superpowers — hooks/session-start]，规定：

> "If you think there is even a 1% chance a skill might apply to what you are doing, you ABSOLUTELY MUST invoke the skill." [来源: Superpowers — using-superpowers/SKILL.md]

并附12条Red Flags列表，每条都是Agent不检查技能的常见借口及其反驳。这防止了Agent在长时运行中"忘记"使用技能——退化的一种表现。

**难题四：跨会话记忆断层 → Superpowers的会话启动注入**

Superpowers的`session-start` Hook在每次会话启动（包括`/clear`和`/compact`后）自动注入`using-superpowers`技能全文 [来源: Superpowers — hooks/session-start]：

```
Hook触发（startup|clear|compact）
  → 读取skills/using-superpowers/SKILL.md全文
  → 包裹在<EXTREMELY_IMPORTANT>标签中
  → 注入为additionalContext
```

这确保了每个新会话——即使是压缩或清除后——都会重新加载技能意识。结合Claude Code自身的CLAUDE.md机制（项目根CLAUDE.md在/compact后重新注入），形成了双重保障。

**难题五：自主与安全 → Superpowers的Worktree隔离**

`using-git-worktrees`技能强制在开始实现前创建隔离工作空间 [来源: Superpowers — using-git-worktrees/SKILL.md]：
- 在独立分支上工作，主分支不受影响
- 验证目录已被gitignore
- 运行项目设置，验证测试基线是干净的
- 完成后提供4个选项：合并/推送创建PR/保持/丢弃，丢弃需要输入"discard"确认

### 案例二：Superpowers如何约束用户——一个"不信任Agent"的设计哲学

Superpowers的设计哲学不是"信任Agent会做对"，而是"假设Agent会偷懒、会找借口、会在压力下放弃原则" [推断: 基于所有技能中大量的Rationalization Table和Red Flags列表]。这导致了对用户的特殊约束——用户不能随意绕过流程。

**约束1：强制先设计后编码**

brainstorming技能有HARD-GATE：

> "Do NOT invoke any implementation skill, write any code, scaffold any project, or take any implementation action until you have presented a design and the user has approved it." [来源: Superpowers — brainstorming/SKILL.md]

并显式针对反模式："This Is Too Simple To Need A Design"——每个项目都必须走这个流程，即使是一个配置变更。用户不能直接说"别废话了开始写代码"来绕过——因为技能的优先级高于默认行为 [来源: Superpowers — using-superpowers/SKILL.md Instruction Priority]。

**约束2：计划中禁止占位符**

writing-plans技能禁止所有模糊指令 [来源: Superpowers — writing-plans/SKILL.md]：
- "TBD"、"TODO"、"implement later"
- "Add appropriate error handling"（没有具体代码）
- "Write tests for the above"（没有实际测试代码）
- "Similar to Task N"（必须重复代码，因为子智能体可能不按顺序读任务）

用户不能通过说"随便写个计划就行"来降低质量标准——计划中每一步必须包含完整的代码和精确的命令。

**约束3：用户指令 > 技能指令，但技能会提醒你**

Superpowers的优先级体系是 [来源: Superpowers — using-superpowers/SKILL.md Instruction Priority]：
1. 用户的显式指令（CLAUDE.md、直接请求）— 最高优先
2. Superpowers技能 — 覆盖默认系统行为
3. 默认系统提示 — 最低优先

如果用户在CLAUDE.md中写"不要用TDD"，技能会服从。但技能通过Iron Law和Rationalization Table显式告知用户绕过的后果——用户是有意识地选择放弃保护，而不是无意中遗漏。

**约束4：技能本身也要用TDD来写**

writing-skills技能把TDD原则应用到技能创作本身 [来源: Superpowers — writing-skills/SKILL.md]：

> "NO SKILL WITHOUT A FAILING TEST FIRST"

写技能前，必须先用压力场景测试Agent在没有该技能时的行为（RED），然后写技能（GREEN），然后关闭漏洞（REFACTOR）。这意味着用户不能"随便写个技能"——每个技能必须经过对抗性测试。

**约束5：说服力研究支持纪律强化**

Superpowers的writing-skills技能引用了Meincke et al. (2025, N=28,000)的研究 [来源: Superpowers — writing-skills/persuasion-principles.md]：Authority（"YOU MUST"、"No exceptions"）+ Commitment（公开声明、追踪）+ Social Proof（"Every time"）的组合使合规率从33%提升到72%。这不是凭直觉写规则——而是用实验数据指导措辞。

### 案例三：Superpowers的完整工作流管线——从想法到合并

把Superpowers的14个技能串成一条完整的管线 [来源: Superpowers — README.md]：

```
用户说"我想做一个X"
  │
  ├─ [brainstorming] 探索意图，一次一个问题，分段呈现设计
  │   HARD-GATE: 用户批准设计前不能写代码
  │   反模式: "这太简单了不需要设计"
  │
  ├─ [using-git-worktrees] 创建隔离工作空间
  │   验证: 目录gitignored、测试基线干净
  │
  ├─ [writing-plans] 拆分为2-5分钟粒度的任务
  │   禁止: TBD/TODO/占位符
  │   每步包含: 完整代码 + 精确命令 + 预期输出
  │
  ├─ [subagent-driven-development] 逐任务派发子智能体
  │   每个任务: 新子智能体实现 → 规格合规审查 → 代码质量审查
  │   审查不通过: 修复 → 再审查，循环直到通过
  │   模型选择: 机械任务用便宜模型，审查用最强模型
  │
  │   子智能体内部遵循:
  │   ├─ [test-driven-development] RED-GREEN-REFACTOR
  │   │   Iron Law: 先写测试看它失败，再写代码
  │   │   先写了代码? 删掉重来
  │   │
  │   ├─ [systematic-debugging] 4阶段根因调查
  │   │   Iron Law: 没找到根因前不能提修复
  │   │   3次修复失败? 质疑架构
  │   │
  │   └─ [verification-before-completion] 5步门控
  │       Iron Law: 没运行验证命令就不能声称完成
  │       24次过早声称完成的事故记录
  │
  ├─ [requesting-code-review] 任务间代码审查
  │   派发独立审查子智能体
  │   三级: Critical(必须修)/Important(修完再走)/Minor(记录)
  │
  └─ [finishing-a-development-branch] 验证→选择→清理
      验证测试通过后提供4选项: 合并/PR/保持/丢弃
```

**这条管线覆盖了Ch2的每个难题：**
- 焦虑：子智能体隔离 + 小任务粒度
- 过早完成：两阶段审查 + 证据优先验证 + TDD
- 退化：渐进式披露(CSO) + 强制技能检查(session-start Hook)
- 断层：会话启动注入 + 计划文件持久化 + git commit
- 安全：Worktree隔离 + 审查子智能体

**Superpowers的独特之处：不只是工具，是方法论**

与Claude Code内置的能力（/compact、/clear、Auto Mode）不同，Superpowers不是一组独立的工具，而是一条强制管线。工具可以挑着用，管线必须走完。这是它对"Ch4零件怎么组装"这个问题的回答：不是让你选零件，而是帮你定义了唯一正确的顺序。

**→ 过渡到Ch5：** Superpowers的管线针对当前模型（Sonnet 4.5/4.6, Opus 4.5/4.6）设计。但"Every component in a harness encodes an assumption about what the model can't do on its own"——当模型能力变了，Superpowers的哪些Iron Law可能变为多余？

---

## 第五章 进化——模型在变，Harness也要变

**本章核心问题：** Ch3-Ch4给出了一套解法，但"Every component in a harness encodes an assumption about what the model can't do on its own"——当模型能力变了，这些假设还成立吗？你怎么知道？

### 5.1 Harness演进方法论——怎么知道该保留、该去掉、该新增

**第一步：检测信号——怎么知道某个组件的假设过期了？**

以上下文焦虑为例 [来源: Harness设计]：
- 信号：模型不再表现出提前收尾行为
- 验证：尝试去掉上下文重置→观察Agent是否仍能完成长时任务
- 结果：Opus 4.5去掉重置后仍可连续运行→假设过期→组件可移除

**第二步：实验方法——怎么安全地移除？**

不要一刀砍光 [来源: Harness设计 — "First tried cutting the harness back radically → couldn't replicate performance"]，而是逐个移除+测量：
- 保留其他组件，只移除一个
- 测量移除前后的性能差异
- 如果差异显著→组件仍在承重→保留
- 如果差异不显著→组件可能不再需要→继续观察

**第三步：评估框架——组件还在"承重"吗？**

| 评估项 | 问题 | 来源 |
|--------|------|------|
| 假设是否成立 | 这个组件编码的"模型做不到什么"的假设还成立吗？ | Harness设计 |
| 移除的影响 | 去掉它，性能是否下降？ | Harness设计 |
| 任务的边界 | 组件在哪些任务上仍然有价值，哪些上已变成开销？ | Harness设计 — 评估器在简单任务上是开销，在困难任务上是必需品 |

### 5.2 技能的进化方向

- 能力提升型技能（Capability Uplift）可能随模型改善变得不必要 [来源: Skills改进]
- 编码偏好型技能（Encoded Preference）更持久，但需要验证保真度 [来源: Skills改进]
- 技能可能从"怎么做"进化为"做什么"的规格说明 [来源: Skills改进 — "a SKILL.md may evolve from a detailed implementation plan to a natural-language specification"]

### 5.3 核心洞察

"The space of interesting harness combinations doesn't shrink as models improve. Instead, it moves, and the interesting work for AI engineers is to keep finding the next novel combination." [来源: Harness设计 — 原文引用]

这不是说Harness会越来越简单——而是说Harness的重心会转移。当模型在A领域不再需要支撑时，B领域的新挑战会出现，需要新的Harness组件。

---

## 第六章 总结

### 6.1 一张因果总图

```
上下文窗口是有限资源（元根因）
    │
    ├── 快满了 ────→ 焦虑收尾 ──────→ 重置/扩窗/隔离（子智能体）
    ├── 装太多 ────→ 退化偏航 ──────→ 上下文隔离/渐进式披露/按需加载
    └── 会话级 ────→ 记忆断层 ──────→ CLAUDE.md/Auto Memory/Session日志
                       │
自我评估不可靠 ──→ 过早完成 ────────→ 分离生成与评估/结构化清单/强制验证
                       │
二元权限模型 ───→ 自主与安全矛盾 ──→ 分级分类器/隔离环境/凭证分离
                       │
                       ↓
          零件需要组装（Ch4三个场景）：
          · 一句话→完整应用：全部五组零件协同
          · 无人值守修Bug：持久化+安全为主
          · 多Agent并行：隔离+裁判+协调机制
                       │
                       ↓
          完整落地案例：Superpowers
          · 14个技能 → 强制管线（非可选工具）
          · 三重防线防过早完成：两阶段审查+证据验证+TDD
          · 说服力研究指导措辞：合规率33%→72%
                       │
                       ↓
          模型在进化 → Harness假设需要持续检验
```

### 6.2 设计长时运行Agent的因果检验清单

- 每个Harness组件，问：它编码的假设是什么？这个假设还成立吗？ [来源: Harness设计]
- 每个新增复杂度，问：观察到的问题是什么？去掉它会怎样？ [来源: Harness设计]
- 每个方案选择，问：根因是什么？方案是否直接作用于根因？ [推断: 多篇文章的方法论综合]
- 每个零件组合，问：它们解决的是同一个难题还是不同难题？组合后是否有冲突？ [推断: 基于Ch4的组装逻辑]

### 6.3 未来方向

- 随模型能力提升，Harness从"怎么做"向"做什么"演进 [来源: Skills改进]
- 多智能体从编排器指挥向自组织协作演进 [来源: Multi-Agent协调, AgentTeams]
- Harness从工程实践向平台服务演进——Managed Agents是"meta-harness" [来源: Managed Agents工程 — "Managed Agents is deliberately a meta-harness"]
- Routines代表了事件驱动的自动化长时运行方向 [来源: Routines]
