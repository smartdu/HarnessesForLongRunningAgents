# Superpowers 技能清单

> 基于 [obra/superpowers](https://github.com/obra/superpowers) v5.0.7 仓库分析整理

## 总览

Superpowers 包含 14 个技能，按职责分为四类：**流程管线**（5个）、**纪律执行**（3个）、**协作审查**（4个）、**元技能**（2个）。它们串联为一条强制管线，从想法到合并，覆盖长时运行Agent的每个关键环节。

---

## 一、流程管线技能（5个）

按执行顺序排列，形成 `brainstorming → worktrees → planning → development → finishing` 的强制流水线。

### 1. brainstorming — 头脑风暴

| 项目 | 内容 |
|------|------|
| **技能名** | `superpowers:brainstorming` |
| **触发条件** | 任何创造性工作之前——创建功能、构建组件、添加功能、修改行为 |
| **解决的问题** | Agent直接跳到写代码，不先理解需求；凭想象填补模糊需求导致方向错误 |
| **核心规则** | **HARD-GATE**：用户批准设计前不能写任何代码、不能调用任何实现技能 |
| **流程** | 探索项目上下文 → 一次问一个问题 → 提出2-3种方案 → 分段呈现设计 → 用户逐段批准 → 写设计文档 → 自查 → 用户审查 → 调用writing-plans |
| **反模式** | "这太简单不需要设计"——每个项目都必须走这个流程 |
| **终端状态** | 只能调用 `writing-plans`，不能跳到任何实现技能 |
| **文件路径** | `skills/brainstorming/SKILL.md` |

### 2. using-git-worktrees — Git Worktree隔离

| 项目 | 内容 |
|------|------|
| **技能名** | `superpowers:using-git-worktrees` |
| **触发条件** | 开始需要在隔离环境中进行的特性开发，或执行实现计划之前 |
| **解决的问题** | Agent在主分支上直接改代码，做坏了无法回退；在已有bug的代码上继续开发 |
| **核心规则** | 目录优先级：`.worktrees/` > `worktrees/` > CLAUDE.md偏好 > 问用户；必须验证目录已被gitignore |
| **流程** | 检测目录 → 验证gitignore → 创建worktree → 自动运行项目设置(npm install/cargo build等) → 验证测试基线干净 |
| **安全验证** | 创建前必须 `git check-ignore`；未忽略则先添加到.gitignore并提交 |
| **文件路径** | `skills/using-git-worktrees/SKILL.md` |

### 3. writing-plans — 编写计划

| 项目 | 内容 |
|------|------|
| **技能名** | `superpowers:writing-plans` |
| **触发条件** | 有规格或需求文档，准备写代码之前 |
| **解决的问题** | Agent一次性做太多（one-shotting）；计划模糊导致执行偏差；任务粒度太大导致上下文耗尽 |
| **核心规则** | 2-5分钟粒度；**禁止所有占位符**（TBD、TODO、"add appropriate error handling"、"write tests for the above"均视为计划失败） |
| **任务结构** | 每步包含：精确文件路径 + 完整代码 + 精确命令 + 预期输出 + git commit |
| **自检** | 规格覆盖率 → 占位符扫描 → 类型一致性 |
| **执行交接** | 提供两种选择：Subagent-Driven（推荐）或Inline Execution |
| **文件路径** | `skills/writing-plans/SKILL.md` |

### 4. subagent-driven-development — 子智能体驱动开发

| 项目 | 内容 |
|------|------|
| **技能名** | `superpowers:subagent-driven-development` |
| **触发条件** | 有实现计划且任务相互独立时 |
| **解决的问题** | 主Agent上下文被实现细节填满（焦虑+退化）；自我评估不可靠（过早标记完成） |
| **核心规则** | 每个任务派发新子智能体；**子智能体永不继承会话上下文**——控制器精确构造所需信息；两阶段审查（规格合规 → 代码质量） |
| **模型选择** | 机械实现任务（1-2文件，清晰规格）→ 便宜模型；集成/判断任务 → 标准模型；架构/设计/审查 → 最强模型 |
| **4种状态码** | DONE → 进入审查；DONE_WITH_CONCERNS → 读完关注点再决定；NEEDS_CONTEXT → 补充后重新派发；BLOCKED → 评估阻塞原因 |
| **Red Flags** | 不在主分支开发；不跳过审查；不让子智能体读计划文件（提供全文）；不在规格合规通过前启动代码质量审查 |
| **文件路径** | `skills/subagent-driven-development/SKILL.md` |

### 5. finishing-a-development-branch — 完成开发分支

| 项目 | 内容 |
|------|------|
| **技能名** | `superpowers:finishing-a-development-branch` |
| **触发条件** | 实现完成，所有测试通过，需要决定如何集成工作时 |
| **解决的问题** | Agent完成工作后直接推到主分支；测试未通过就合并 |
| **核心规则** | 验证测试通过 → 才能呈现选项；测试失败则停止，不允许继续 |
| **4个选项** | 1.本地合并 2.推送创建PR 3.保持现状 4.丢弃（需输入"discard"确认） |
| **清理** | 选项1和4清理worktree；选项2和3保留 |
| **文件路径** | `skills/finishing-a-development-branch/SKILL.md` |

---

## 二、纪律技能（3个）

每个都有 **Iron Law**（铁律），不可违反。

### 6. test-driven-development — 测试驱动开发

| 项目 | 内容 |
|------|------|
| **技能名** | `superpowers:test-driven-development` |
| **触发条件** | 实现任何功能或修复bug之前 |
| **解决的问题** | Agent先写代码后补测试，测试没有真正验证任何东西；测试通过但不代表功能正确 |
| **Iron Law** | **NO PRODUCTION CODE WITHOUT A FAILING TEST FIRST** |
| **违反后果** | 先写了代码？删掉重来。不留参考、不改编、不看——Delete means delete |
| **RED-GREEN-REFACTOR** | RED：写一个最小失败测试 → 验证它失败 → GREEN：写最小代码使其通过 → 验证通过 → REFACTOR：清理，保持测试通过 |
| **11条合理化反驳** | "太简单不需要测试" → 简单代码也会坏，测试只需30秒；"先写代码后补测试效果一样" → 后补测试回答"这做了什么"，先写测试回答"这应该做什么" |
| **文件路径** | `skills/test-driven-development/SKILL.md` |

### 7. verification-before-completion — 完成前验证

| 项目 | 内容 |
|------|------|
| **技能名** | `superpowers:verification-before-completion` |
| **触发条件** | 即将声称工作完成、已修复、或测试通过时；提交代码或创建PR之前 |
| **解决的问题** | Agent声称"搞定了"但实际没跑验证命令；"should work"、"probably fine"、"seems to" |
| **Iron Law** | **NO COMPLETION CLAIMS WITHOUT FRESH VERIFICATION EVIDENCE** |
| **5步门控函数** | 1. IDENTIFY：什么命令能证明？→ 2. RUN：运行完整命令 → 3. READ：读输出，检查退出码 → 4. VERIFY：输出是否确认？→ 5. ONLY THEN：才能声称 |
| **24次事故记录** | 来自真实会话：用户说"I don't believe you"信任破裂；未定义函数被提交；缺失需求被发布 |
| **8条合理化反驳** | "Should work now" → 运行验证；"I'm confident" → 信心 ≠ 证据；"Just this once" → 没有例外 |
| **文件路径** | `skills/verification-before-completion/SKILL.md` |

### 8. systematic-debugging — 系统化调试

| 项目 | 内容 |
|------|------|
| **技能名** | `superpowers:systematic-debugging` |
| **触发条件** | 遇到任何bug、测试失败、或意外行为时，在提出修复之前 |
| **解决的问题** | Agent猜着修bug，修一个引两个；症状修复而非根因修复；在压力下随意尝试 |
| **Iron Law** | **NO FIXES WITHOUT ROOT CAUSE INVESTIGATION FIRST** |
| **4阶段流程** | 1.根因调查（读错误、复现、检查变更、收集证据、追踪数据流）→ 2.模式分析（找正常例子、对比差异）→ 3.假设测试（单一假设、最小变更、验证）→ 4.实现（写失败测试、单次修复、验证） |
| **3次修复失败规则** | 如果连续3次修复失败 → 停止修复，质疑架构是否从根本上就有问题 |
| **8条合理化反驳** | "Issue is simple" → 简单问题也有根因；"Emergency, no time" → 系统化调试比乱猜更快 |
| **辅助技术** | `root-cause-tracing.md`（反向追踪调用栈）、`defense-in-depth.md`（多层验证）、`condition-based-waiting.md`（条件轮询替代超时） |
| **文件路径** | `skills/systematic-debugging/SKILL.md` |

---

## 三、协作技能（4个）

### 9. requesting-code-review — 请求代码审查

| 项目 | 内容 |
|------|------|
| **技能名** | `superpowers:requesting-code-review` |
| **触发条件** | 完成任务、实现主要功能、或合并前验证工作时 |
| **解决的问题** | Agent自己写自己审，确认偏差；问题在后续任务中累积 |
| **核心规则** | 派发 `superpowers:code-reviewer` 独立子智能体；审查者获取精确构造的上下文（永不继承会话历史） |
| **审查时机** | 强制：子智能体驱动开发中每个任务后、主要功能后、合并前。可选：卡住时、重构前、复杂bug修复后 |
| **三级严重度** | Critical（必须立即修）→ Important（修完再走）→ Minor（记录稍后修） |
| **文件路径** | `skills/requesting-code-review/SKILL.md` |

### 10. receiving-code-review — 接收代码审查

| 项目 | 内容 |
|------|------|
| **技能名** | `superpowers:receiving-code-review` |
| **触发条件** | 收到代码审查反馈时，特别是反馈看起来不清晰或技术上有疑问时 |
| **解决的问题** | Agent收到反馈后敷衍接受（"You're absolutely right!"）；不加验证就实现建议；不敢回推错误建议 |
| **核心规则** | 技术评估，不是情绪表演。验证后再实现，提问后再假设 |
| **禁止回应** | "You're absolutely right!" / "Great point!" / "Let me implement that now"（未经验证就行动） |
| **YAGNI检查** | 审查者建议"properly implement"某个功能 → grep代码库看是否有人用 → 没人用则回推"Remove it (YAGNI)?" |
| **回推信号** | 如果不好意思公开回推："Strange things are afoot at the Circle K" |
| **文件路径** | `skills/receiving-code-review/SKILL.md` |

### 11. dispatching-parallel-agents — 派发并行智能体

| 项目 | 内容 |
|------|------|
| **技能名** | `superpowers:dispatching-parallel-agents` |
| **触发条件** | 面对2+个独立任务，可以在无共享状态的情况下并行处理时 |
| **解决的问题** | 多个独立问题串行调查浪费时间 |
| **核心规则** | 一个智能体一个独立问题域；并行派发；每个智能体不继承会话上下文 |
| **适用条件** | 3+测试文件因不同根因失败；多个子系统独立损坏；问题间无共享状态 |
| **不适用条件** | 失败是相关的（修一个可能修好其他的）；需要理解完整系统状态；智能体之间会互相干扰 |
| **文件路径** | `skills/dispatching-parallel-agents/SKILL.md` |

### 12. executing-plans — 执行计划

| 项目 | 内容 |
|------|------|
| **技能名** | `superpowers:executing-plans` |
| **触发条件** | 有写好的实现计划，需要在单独会话中执行，并有审查检查点时 |
| **解决的问题** | 平台不支持子智能体时的替代执行方式 |
| **核心规则** | 加载计划 → 批判性审查 → 逐任务执行 → 遇到阻塞立即停止询问 |
| **与subagent-driven-development的关系** | 替代方案。明确告知："Superpowers works much better with access to subagents" |
| **文件路径** | `skills/executing-plans/SKILL.md` |

---

## 四、元技能（2个）

### 13. using-superpowers — 使用Superpowers（自举技能）

| 项目 | 内容 |
|------|------|
| **技能名** | `superpowers:using-superpowers` |
| **触发条件** | 任何对话开始时（通过session-start Hook自动注入） |
| **解决的问题** | Agent不知道或假装不知道有技能可用；找借口跳过技能 |
| **核心规则** | **"If you think there is even a 1% chance a skill might apply, you ABSOLUTELY MUST invoke the skill."** |
| **指令优先级** | 1.用户显式指令（CLAUDE.md等）> 2.Superpowers技能 > 3.默认系统提示 |
| **12条Red Flags** | 每条都是跳过技能的借口+反驳："This is just a simple question" → 问题是任务，检查技能；"I remember this skill" → 技能会更新，读当前版本 |
| **注入机制** | `hooks/session-start` 在每次会话启动（startup/clear/compact）时读取SKILL.md全文，包裹在 `<EXTREMELY_IMPORTANT>` 标签中注入 |
| **文件路径** | `skills/using-superpowers/SKILL.md` |

### 14. writing-skills — 编写技能

| 项目 | 内容 |
|------|------|
| **技能名** | `superpowers:writing-skills` |
| **触发条件** | 创建新技能、编辑现有技能、或部署前验证技能时 |
| **解决的问题** | 技能写得模糊导致Agent不遵循；措辞不当导致Agent偷懒走捷径 |
| **Iron Law** | **NO SKILL WITHOUT A FAILING TEST FIRST** — 与TDD相同的铁律应用于文档 |
| **TDD映射** | 测试用例=压力场景(子智能体)；生产代码=技能文档；RED=Agent无技能时违规；GREEN=Agent有技能时合规；REFACTOR=关闭漏洞 |
| **CSO原则（Claude搜索优化）** | description只写触发条件，**绝不摘要工作流**。原因：测试发现当description摘要了工作流时，Claude会跟着描述走而非读取完整技能正文（如"code review between tasks"导致只做一次审查，实际流程图要求两次） |
| **说服力研究** | 基于Meincke et al. (2025, N=28,000)：Authority("YOU MUST") + Commitment(公开声明) + Social Proof("Every time") 使合规率从33%提升到72% |
| **防合理化设计** | 关闭每个漏洞（不留"reference"选项）；禁止"spirit vs letter"论证；构建合理化表；创建Red Flags列表 |
| **文件路径** | `skills/writing-skills/SKILL.md` |

---

## 按长时运行难题归类

| 难题 | 对应技能 | 机制 |
|------|---------|------|
| **上下文焦虑** | subagent-driven-development, writing-plans | 子智能体隔离（不继承会话历史）；小粒度任务（每个任务上下文需求低） |
| **过早标记完成** | test-driven-development, verification-before-completion, subagent-driven-development, writing-plans, requesting-code-review, receiving-code-review | TDD Iron Law；5步验证门控；两阶段独立审查；禁止占位符；YAGNI检查 |
| **上下文退化与偏航** | using-superpowers, writing-skills(CSO), brainstorming | 强制技能检查（session-start Hook）；描述即触发器（防走捷径）；HARD-GATE（防跳过设计） |
| **跨会话记忆断层** | using-superpowers + hooks/session-start, writing-plans, brainstorming | 每次会话启动自动注入；计划/设计文档持久化到docs/superpowers/ |
| **自主与安全** | using-git-worktrees, finishing-a-development-branch | 隔离分支（主分支不受影响）；4选1确认机制（丢弃需输入"discard"） |

---

## 强制工作流管线

```
用户说"我想做一个X"
  │
  ├─ [brainstorming]     探索意图 → 设计 → 用户批准
  │   HARD-GATE: 批准前不能写代码
  │
  ├─ [using-git-worktrees] 创建隔离工作空间 → 验证测试基线
  │
  ├─ [writing-plans]      拆为2-5分钟任务 → 禁止占位符
  │
  ├─ [subagent-driven-development]  逐任务派发子智能体
  │   │
  │   │ 子智能体内部遵循:
  │   ├─ [test-driven-development]        RED-GREEN-REFACTOR
  │   ├─ [systematic-debugging]           4阶段根因调查
  │   └─ [verification-before-completion] 5步验证门控
  │   │
  │   └─ 每个任务完成后:
  │       [requesting-code-review] → [receiving-code-review]
  │
  ├─ [dispatching-parallel-agents]  独立任务可并行
  │
  └─ [finishing-a-development-branch]  验证测试 → 4选1 → 清理
```

**关键洞察：** Superpowers不是一组可选工具，而是一条强制管线。工具可以挑着用，管线必须走完。这是它对"零件怎么组装"这个问题的回答——不是让你选零件，而是帮你定义了唯一正确的顺序。

---

## 配套组件

### Agents

| 名称 | 文件 | 用途 |
|------|------|------|
| `superpowers:code-reviewer` | `agents/code-reviewer.md` | 高级代码审查者，对比实现与原始计划，按Critical/Important/Suggestion分级 |

### Hooks

| 事件 | 脚本 | 用途 |
|------|------|------|
| SessionStart (startup/clear/compact) | `hooks/session-start` | 读取using-superpowers技能全文，包裹在`<EXTREMELY_IMPORTANT>`标签中注入 |

### Commands（已废弃）

| 命令 | 替代方案 |
|------|---------|
| `/brainstorm` | `superpowers:brainstorming` 技能 |
| `/write-plan` | `superpowers:writing-plans` 技能 |
| `/execute-plan` | `superpowers:executing-plans` 技能 |
