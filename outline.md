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

## 第四章 How（续）— 把零件组装成整体：工作流、协调与自动化

**本章核心问题：** Ch3的解法是针对单个问题的"零件"，但实际的长时运行任务需要同时应对多个问题。你需要选择正确的零件组合，并定义它们之间的协作方式。

### 4.1 三种核心工作流模式——零件怎么串起来

| 模式 | 适用场景 | 解决哪些难题 | 来源 |
|------|---------|------------|------|
| 顺序工作流 | 任务有依赖关系：步骤B需要步骤A的输出 | 难题三（退化）：每阶段专注一件事 | Workflow模式 |
| 并行工作流 | 独立任务可同时执行；需要多维度评估 | 难题一（焦虑）+难题三（退化）：隔离上下文 | Workflow模式 |
| 评估器-优化器 | 首次尝试质量不够，需要迭代改进 | 难题二（过早完成）：生成与评估分离 | Workflow模式 |

决策框架：先用单Agent试试，不行再加结构 [来源: Workflow模式]

### 4.2 五种多智能体协调模式——多个Agent怎么协作

| 模式 | 关键特征 | 解决哪些难题 | 来源 |
|------|---------|------------|------|
| Generator-Verifier | 生成-验证循环 | 难题二（过早完成） | Multi-Agent协调 |
| Orchestrator-Subagent | 编排器派发子任务 | 难题一（焦虑）+难题三（退化）+难题四（断层） | Multi-Agent协调 |
| Agent Teams | 持久工作者自主领取任务 | 难题三（退化）：每个Teammate积累领域专长 | Multi-Agent协调 |
| Message Bus | 发布/订阅 | 难题五（安全）：事件驱动、可审计 | Multi-Agent协调 |
| Shared State | 共享持久存储 | 难题四（断层）：跨Agent共享发现 | Multi-Agent协调 |

关键区分：**持久性**是Orchestrator-Subagent vs Agent Teams的分界线 [来源: Multi-Agent协调]
推荐起点：Orchestrator-Subagent [来源: Multi-Agent协调 — "Start with orchestrator-subagent"]

### 4.3 Routines——无人值守的持续执行

**Ch3的零件需要人在场才能运作。** 但长时运行Agent的一个核心需求是：人不看着的时候也能干。Routines解决的就是这个问题。

- 三种触发方式：计划调度 / API调用 / Webhook（当前仅GitHub） [来源: Routines]
- 运行在Claude Code的web基础设施上，不依赖本地电脑 [来源: Routines]
- GitHub Webhook模式：每个PR创建一个会话，持续接收PR更新 [来源: Routines]
- 典型场景：夜间自动修bug并开draft PR、部署后自动冒烟测试、PR自动代码审查 [来源: Routines]

**Routines如何组合Ch3的零件：**
- 需要3.4的记忆（CLAUDE.md）来保持跨会话一致性
- 需要3.5的安全（Auto Mode / 隔离环境）来安全自主运行
- 需要4.1的工作流来定义执行步骤

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
          零件需要组装 → 工作流/协调模式/Routines
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
