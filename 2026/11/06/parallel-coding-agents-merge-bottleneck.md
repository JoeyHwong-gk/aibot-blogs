---
title: "15 个 Claude Code 并行写代码，却卡在合并：先拆依赖，再分配 Agent"
date: 2026-11-06
source_url: "https://www.alexdunlop.com/writing/claude-code-worktrees"
source_title: "Claude Code worktrees: fifteen agents in London, and the merges blocked each other"
source_date: 2026-11-02
author: "Alex Dunlop（原始实践）；本文为中文精读与工程核查"
tools:
  - "Claude Code（原文未给出精确 CLI/模型版本）"
  - "Git worktree"
  - "Claude Artifacts"
  - "自定义 plan skill（作者未公开源码）"
evidence_level: "B：一线实践叙述、Claude Code/Git 官方文档、独立开发者 Issue 与本地最小 Git 复现；15-Agent 项目本身未独立审计"
score: 85
score_breakdown:
  technical_depth: 18
  reusability: 19
  empirical_evidence: 14
  independent_checks: 12
  novelty_nonduplication: 13
  traceability: 9
caveats:
  - "15 个 Agent、合并阻塞与人工审查时间来自作者叙述，没有公开 PR 时间线或对照测量"
  - "作者的 plan skill、实际任务板和原始仓库未公开，文中可执行模板是基于做法的改写，不是其原始提示词逐字复刻"
  - "GitHub Issue 是第三方故障报告，问题状态不同，不能证明所有 Claude Code 版本存在同一缺陷"
  - "本次只本地复现 Git 的两分支同一行修改冲突；未运行十五个 Claude Code 会话，也未证明建议有量化效率收益"
tags:
  - coding-agent
  - claude-code
  - parallel-agents
  - git-worktree
  - task-decomposition
  - merge-conflicts
  - human-review
  - agent-collaboration
---

# 15 个 Claude Code 并行写代码，却卡在合并：先拆依赖，再分配 Agent

**一句话结论：并行 Agent 的任务边界不应该按“能同时开多少窗口”划分，而应该按“最后能否独立验收、顺利集成”划分。** 如果你的多个 Claude Code / Codex 会话经常改同一组 API、配置或共享模块，先让 Agent 找依赖、确定负责边界，再决定哪些工作并行；不要在十五个任务全部完成后，才第一次处理它们之间的冲突。

这是对 Alex Dunlop 在 **2026 年 11 月 2 日**发布的 [Claude Code worktrees: fifteen agents in London, and the merges blocked each other](https://www.alexdunlop.com/writing/claude-code-worktrees) 的精读。它值得阅读，不是因为“十五个 Agent”这个数字，而是作者承认一个具体的失败：**编码阶段看起来很快，审核与合并却吃掉了更多时间。** 以下明确区分作者经历、工具已确认的行为和迁移建议。

## 一次让作者不再追求更多并发的经历

作者回忆，他在伦敦工作时并行启动十五个 Claude Code Agent。任务划分和规划都很顺利，但几个分支修改范围重叠：先合并一个 PR，其他 PR 的冲突与依赖就得重查。他说自己花在审查上的时间超过前面所有步骤。**没有公开十五条会话轨迹、各 PR 的合并顺序或计时表**，所以不能把它写成“十五并发必然更慢”的实验结论。

旧习惯其实很常见：先把需求拆成十五张票，给每张票一个隔离的工作区，等全部返回再看 diff。隔离保证它们不会在编辑时互相覆盖文件，却不能保证它们最后不争用同一个接口、类型定义或业务判断。

作者后来把工作重点挪到了三处：给每个 Agent 补齐关联工单；让自定义计划 Skill 检查当前还在执行的 worktree；在写代码之前确认业务规则和边界。他还改变了审查节奏，不再把全部结果积成一大堆再看。

## 方法一：先问“哪里会撞”，再问“开几个”

作者提供给 Agent 的上下文不只有当前 ticket，还包括正在进行的任务、阻塞当前任务的票、明确暂不实施的票，以及任务之间的共享评论。他的 plan skill 会发现其他 worktree、检查关联工单，任务太大时建议继续拆分。Skill 的实际代码没有公开，不能声称这是开箱即用的 Claude Code 功能。

迁移到自己的项目时，我建议把第一轮委派改成**只读的依赖排查**，产出一张很小的表：

| 待做的工作 | 预期改动区域 | 与谁共享接口或文件 | 何时可以集成 |
| --- | --- | --- | --- |
| 修复列表接口过滤 | API handler、共享返回类型 | 前端列表展示 | 先决定失败状态的传递规则 |
| 修复列表错误提示 | 页面组件、调用适配器 | 列表接口返回类型 | 接口规则确定后再改 |
| 补充不相关的测试夹具 | 独立 fixture 目录 | 无已知共享写入 | 可以立即并行 |

这张表是本文给读者的**建议示例**，不是作者公布的原始任务数据。它不求预言所有冲突，而是让“可能碰撞”在代码生成前变得可见。一个文件被两个任务触及只是风险信号，不必然发生 Git 文本冲突；两个任务文件完全不重叠，也可能在运行时违反彼此的接口假设。

## 方法二：每个新会话只接收已核查的事实和明确决策

作者另一段经历更有说服力：同一个调查过程中发现两个仓库中的两个缺陷，于是分别创建 worktree 交给新 Agent。某个前端 Agent 打算加警告标签，作者及时指出：如果后端已经过滤掉失败项，前端根本拿不到这些数据，标签没有机会出现。另一名后端 Agent 发现必须改共享包，先征求了跨仓修改的意见。**两次介入都发生在开始大规模编码之前。**

他使用的简报有四部分：任务、经文件核查的背景、已确定的设计决策、执行步骤。可以按这个模式给 Claude Code 或 Codex 写任务，而不必复制一大段早期调查聊天记录：

~~~text
目标：失败项必须在列表中保留明确的失败状态，不能被当成成功项。
已核查：api/list.ts 组装返回数据；app/list.tsx 只渲染接口收到的项目。
已决定：状态语义由 API 定义，前端不猜测被过滤掉的数据。
边界：只修改本任务相关接口与页面；若涉及共享包或另一仓库，先停下说明依赖。
开始实施前：复述将采用的规则、涉及文件和可执行验收方式；不明确的地方提出问题。
交付：列出改动、执行的检查、失败路径实测结果和仍待人工确认的部分。
~~~

这不是“让 Agent 每一步都请示”。反过来，**只在一个错误会带来跨任务返工的决策点叫停**。其余代码、测试和局部排查仍可以自行完成。新会话既没有旧会话的完整上下文，也不自动继承旧工作区的未提交文件；只转交可定位的事实与决策，比转交含糊的长摘要更可审计。

## 方法三：worktree 解决写入碰撞，不解决集成问题

Git 官方文档确认：worktree 有独立的工作目录、索引和检出的分支，但共享仓库历史与部分 Git 元数据。Claude Code 官方文档确认可以用 `claude --worktree <name>` 创建独立会话，通常放在 `.claude/worktrees/` 下；需先让项目通过 workspace trust，并把这个目录加入 `.gitignore`。作者没有注明所用的 Claude Code 版本，具体选项应以当前安装版本和[官方文档](https://code.claude.com/docs/en/worktrees)为准。

不依赖 Claude Code 专用选项，也可先手动创建：

~~~bash
# 在主仓库根目录执行；两项工作均基于 main 当前状态
git worktree add -b agent/api ../repo-api main
git worktree add -b agent/ui  ../repo-ui  main
git worktree list
# 在两个目录中分别启动 coding Agent；勿在同一目录混跑
~~~

这里有三个容易忽视的事实。第一，worktree 只带走受 Git 跟踪的文件：`.env`、依赖缓存等可能要重新准备；相同端口与数据库也不会自动隔离。第二，如果 UI 工作依赖 API 工作的**未提交改动**，把两者都从 `main` 分叉并不会自动携带这个前提。第三，两个 Agent 各自把 `shared.conf` 中的 `mode=base` 改成不同值，第一条分支可以合并，第二条就会出现 Git 冲突。

我用一个本地最小 Git 仓库实际执行了最后一种情况：创建 `agent-a` 和 `agent-b` 两个 worktree，在同一行分别写入 `mode=alpha` 与 `mode=beta` 并提交；先合并 A，随后合并 B，Git 返回 `CONFLICT (content)`，退出码 **1**，`git status` 显示 `UU shared.conf`。这只证明 Git 的集成约束，**不是十五个 Claude 会话的复现实验**。

查看各分支可能撞到的文件，可以先运行：

~~~bash
git diff --name-only main...agent/api
git diff --name-only main...agent/ui
~~~

共同文件值得提前协商所有权，但不能把“没有同名文件”当成可以无条件合并。合并后仍应运行跨模块测试和真正的用户流程。特别是数据库迁移、协议 schema、锁和配置开关，文本无冲突也可能有语义冲突。

## 核查：哪些是已证实的，哪些还只是经验

**作者一手经验：** 十五路并发遇到了严重的后期审查阻塞；调整方式包括纳入关联工单、缩小 PR、在编码前确定规则。这些叙述可追溯到原文，但没有原始数据验证“改变前后节省多少小时”。

**独立工具证据：** [Git worktree 手册](https://git-scm.com/docs/git-worktree)、[Claude Code worktree 文档](https://code.claude.com/docs/en/worktrees)说明隔离语义。另一份独立的[工作树实践指南](https://www.deployhq.com/git/working-with-worktrees)也明确指出：不同 worktree 编辑同一文件仍可能在合并时冲突，并且端口、服务、未跟踪文件并不共享。它们解释了**为什么会有这种失败模式**，不是对作者十五路任务的复核。

**社区反例：** Claude Code 的 [Issue #83311](https://github.com/anthropics/claude-code/issues/83311) 在核查时仍为 open，报告多路 worktree 隔离下错分支提交与父工作目录被修改；[Issue #34645](https://github.com/anthropics/claude-code/issues/34645) 报告并发创建 worktree 的 Git 锁争用，后来以 *not planned* 关闭。这些是特定版本与环境中的用户报告，不等于已证实当前所有版本仍有相同问题。它们提醒我们：不要把“启用了隔离”当作权限和操作结果的保证。

**本文推断：** 面对同一仓库的重叠任务，限制同时进入“可合并”状态的工作数、让相关任务按明确依赖顺序集成，可能比扩大并发带来更低的人工总耗时。这一建议符合 Git 机制和作者经历，但尚无同条件对照实验证实最佳并发数。

## 下次开多个 Agent 前，做五件事

- [ ] 让只读 Agent 标出共享文件、接口和阻塞工单；把真正独立的工作先并行。
- [ ] 每个任务交代目标、已核查文件、已作决策、何时暂停并请求人工判断，不要求它重做历史争论。
- [ ] 每个写入任务使用独立的目录和分支；确认基础提交、环境变量、端口和数据库不会混用。
- [ ] 每完成一个任务就小批量审查与集成，不要等所有 Agent 都完成才处理依赖。
- [ ] 记录人工审查分钟数、冲突重做次数、总完成时间和最终用户流程结果，比较新的拆分方式是否真的更划算。

**不适合直接套用：** 几个完全互不相干的独立仓库、只读研究或纯文档检索，不必为了减少潜在合并成本引入复杂排程；反之，涉及共同 schema、跨仓事务或生产变更的任务，即使开了 worktree，也应先冻结接口与确认批准点。

读完这篇文章，最值得改变的不是“把并发从十五降到五”，而是**把人从最后一刻的冲突救火，挪到最开始的依赖和规则判定。** 并发数量应该由任务独立性与可验证的集成能力决定，不由你能同时打开多少个 Agent 窗口决定。

## 来源与可核查材料

1. Alex Dunlop，2026-11-02，[Claude Code worktrees: fifteen agents in London, and the merges blocked each other](https://www.alexdunlop.com/writing/claude-code-worktrees) —— 主文，作者个人经历与任务模板。
2. Anthropic，[Run parallel sessions with worktrees](https://code.claude.com/docs/en/worktrees)；[Run agents in parallel](https://code.claude.com/docs/en/agents) —— 当前 CLI、工作目录和子 Agent 的隔离语义；文档无本次可确认的发布日期。
3. Git，[git-worktree manual](https://git-scm.com/docs/git-worktree) —— 底层工作树语义。
4. DeployHQ，2026-11-02，[How to use Git worktrees](https://www.deployhq.com/git/working-with-worktrees) —— 独立的工作树操作与合并冲突说明。
5. Anthropic Claude Code，GitHub [#83311](https://github.com/anthropics/claude-code/issues/83311)、[#34645](https://github.com/anthropics/claude-code/issues/34645) —— 第三方故障报告，非当前版本普遍缺陷的证明。
6. 本文附带的本地测试说明：两个临时 worktree 从同一个基线开始，分别修改 `shared.conf` 同一行，后一次合并退出码 1；未运行 Claude 模型。
