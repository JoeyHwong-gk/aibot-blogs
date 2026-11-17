---
title: "别让 Agent 每次重学项目：先让它盘点真实环境，再复用同一套工作指令"
date: 2026-11-18
source_url: "https://github.blog/ai-and-ml/github-copilot/build-once-run-on-any-agent-a-practical-guide-to-agent-skills/"
source_title: "Build once, run on any agent: A practical guide to agent skills"
source_date: 2026-11-12
author: "Andrea Griffiths（GitHub）；本文为独立中文分析"
tools:
  - "Data Forge v0.1.0（演示仓库）"
  - "GitHub Copilot、Claude Code、Cursor（原文未限定版本）"
  - "DuckDB、GitHub CLI"
evidence_level: "B：一手可复现演示，公开 Issue #3、已合并 PR #4 和测试代码；尚无真实生产效率对照"
score: 86
score_breakdown:
  technical_depth: 18
  reusability: 19
  empirical_evidence: 17
  independent_checks: 9
  novelty_nonduplication: 14
  traceability: 9
caveats:
  - "Data Forge 的 support-desk 属于本地合成数据演示，不是公开生产部署；未独立复跑"
  - "PR #4 的数据对账和测试结果来自项目作者，不能当作跨团队生产率实验"
  - "没有披露人工工时、交互轮次、Token 与实施成本的前后对照"
  - "模型、客户端和权限不同，不能由可打包推断执行结果一致"
tags:
  - coding-agent
  - skills
  - context-management
  - reproducible-workflow
  - github-copilot
  - claude-code
  - duckdb
---

# 别让 Agent 每次重学项目：先让它盘点真实环境，再复用同一套工作指令

**一句话结论：** 当你反复让 Agent 做同一类工作，别在每个新会话里重讲表结构、命名习惯和操作步骤。先让它从真实代码和工具环境中生成一份**可由人核对的项目事实清单**，再让简短的 Skill 使用这份清单做事。适合数据处理、定期分析、迁移、发布等有稳定做法、但项目细节不断变化的任务。

这是对 GitHub 的 Andrea Griffiths 2026 年 11 月 12 日文章的精读。案例有公开源码、Issue 和已合并 PR，但使用的是本地合成数据；**不是“实测节省多少人工小时”的生产报告**。

## 真正浪费时间的不是输入提示词，而是每次重新解释项目

假设你负责一套客服数据仓库。Ticket 指标已经有了，运营又要求 Incident 的创建数、解决数。需求单写得很自然：“像 Ticket 一样按不同层级统计，两个区域都要有。”

这恰好是 [Data Forge 示例 Issue #3](https://github.com/tewarishishir/dataforge/issues/3) 的内容。它**没写**粒度究竟是 account、workspace 还是 user，没规定日快照和累计快照的处理方式，也没说明软删除记录算不算。按传统的“把需求解释完整再扔给 Agent”方式，人得先翻已有 SQL、Python runner 和数据表，把隐含约定变成一大段提示词。下个类似任务又要解释一遍。

GitHub 文章提出的变化是：**不要把可发现的事实当作用户需要长期口述的知识**。人给目标和边界；Agent 先做一次环境侦察，把查到的事实落盘；只有查不出的业务决策才由人确认。

## 第一次委派：让 Agent 先发现事实，不急着写代码

作者配套的 [Data Forge 仓库](https://github.com/tewarishishir/dataforge) 提供了可在本机复现的 support-desk 练习。准备步骤是：

~~~bash
git clone https://github.com/tewarishishir/dataforge.git
cd dataforge/examples/support-desk
# 先安装 DuckDB CLI
python seed.py
~~~

然后在指向该目录的 Agent 会话中说：**initialize dataforge**。

触发的 forge-init 不靠猜测建立项目知识，而是按 Skill 定义检查实际环境：

- 读取 warehouse 下的 DuckDB 文件，确认 zone1、zone2 及执行顺序；查询 information_schema 获取源表字段。
- 观察已有 Ticket 的 Schema SQL 和 Code Python，提取 account/workspace/user 三种粒度、Daily/Cumulative 两种周期，以及指标名格式 total_{entity}_{verb}_count。
- 将发现写进 dataforge.manifest.resolved.yaml，交给后续工作读取；它发现 DuckDB 没有项目要求的作业调度器时，明确记为 engine-unsupported，而不是虚构一套调度。

这里有个必须保留的细节：示例种子文件仍需显式补充 Incident 的 **resolve** 动词。因为现有 Ticket 只展示 create/close，单凭旧代码推不出“解决事件”的准确业务含义。**发现事实不等于决定业务规则**；应把无法从环境证明的部分暴露给用户。

为什么这比在 Skill 里硬编码“我们公司有三种粒度”有用？因为下一套仓库的表、命名规则和平台可能不同，写死的知识反而会误导 Agent。事实清单可以重新生成和审阅，操作方法则保持稳定。

## 第二次委派：给任务入口，不再把整份项目背景复制进聊天

接着照 [DEMO.md](https://github.com/tewarishishir/dataforge/blob/main/demos/dataforge/examples/support-desk/DEMO.md) 创建需求单，再说：**build the metrics in issue #N**（N 是真实 Issue 编号）。

这次 forge-builder 用前面生成的清单协调三个专职步骤：forge-dbx 查询数据平台，forge-lint 检查命名约定，builder 自己负责从 Issue 到 PR 的工作。它先从现有数据判断三种粒度是否可用，再请人确认含糊的口径、软删除处理及周期，随后生成：

- 两个指标：total_incident_create_count、total_incident_resolve_count；
- 三种粒度乘两种周期的 **6 份 Schema SQL**；
- Daily 与 Cumulative **2 个 Python 管道**，以及 SQL 回归检查。

人没有被要求逐行指导 SQL 怎么写，但也没有退出。示例将工作分成七阶段，每阶段检查是否完成；在真正执行 DDL 前，Agent 展示六份 Schema 并等待确认。Python runner 先**打印 SQL**，再由单独的数据库执行工具运行。这个分离让“生成了什么”与“已经改了数据库”可以分别审查。

文章还强调，Skill 是指导文件，不是权限票据。它不会因为写了“查询数据库”就自动拥有 DuckDB 或 Databricks 的 CLI、账号和网络访问。操作能否成功，仍取决于使用者提供的真实环境。

## 验收不是“它说测试通过”，而是能对回源数据

这不是只停留在幻灯片上的流程。公开的 [Issue #3](https://github.com/tewarishishir/dataforge/issues/3) 最终关联到已合并的 [PR #4](https://github.com/tewarishishir/dataforge/pull/4)：**12 个变更文件，420 行新增、1 行删除**，包括上述 Schema、管道、SQL 测试和文档。

PR 列出六类检查：日汇总、累计汇总、不同粒度一致性、唯一性、行质量、表结构匹配。它还记录了一个具体返工：最初的日期覆盖测试把“当天没有事件”误判为错误，后来修正。这比“所有测试一次通过”的故事可信得多。

部署到本地两个 DuckDB 文件后，作者报告最新累计结果与源表直接计数一致：zone1 为 **261 created / 54 resolved**，zone2 为 **108 / 18**，软删除不计入。这里的数值来自项目演示记录；本文没有独立重跑。

对读者来说，最值得模仿的不是造出同样数量的文件，而是**先问验收证据长什么样**：同一日期、同一过滤条件下，指标表和源表能不能对账？如果异常是“0”，测试能不能区分“确实没有事件”和“任务没有运行”？

## 为什么有效，又在哪些地方不该照抄

这套做法减少的是**同一类任务中的知识重新发现和沟通损耗**，不是模型的神秘能力。Skill 保留“怎么执行和检查”，manifest 保留“这个项目此刻是什么样”，Issue 保留“本次要达成什么”。三者分开后，Agent 不必让旧聊天记录承担长期项目知识的职责，人也能审查那些会影响结果的假设。

不过，GitHub 并没有提供采用前后的总耗时、人工介入次数、Token 成本或缺陷率。示例 README 明确称 Data Forge v0.1.0 为 **Beta**，DuckDB 适配器通过示例验证而非生产实践。把 engine 从 duckdb 改为 databricks 是配置层的演示，**不能当作实际完成了生产迁移**。

独立社区也有相反方向的经验：[一位 Claude Code 用户](https://www.reddit.com/r/ClaudeCode/comments/1xhn8az/i_switched_off_every_builtin_claude_code_skill_i/) 清理了不需要的 Skills，原因是误触发和上下文负担；评论者追问实际节省多少，但没有可靠量化。[关于过期指令的研究](https://arxiv.org/abs/2611.09802) 还发现，Agent 即使绕过错误路径完成任务，也通常不会主动修订失效的指令文件。两者不否定 Data Forge 的示例，却提醒我们：**知识一旦持久化，就必须允许更新、删除和核对**。

## 明天直接试一次

选一个你已经向 Agent 解释过至少两遍的任务，**不要一上来开发四个 Skills**。先做一个小实验：

1. 让 Agent 列出它需要的项目事实，并标注每项来自哪个文件、数据库查询或命令；无法证明的不要猜。
2. 人只确认事实清单里影响业务结果的内容，然后让 Agent 从该清单完成一次真实需求。
3. 把稳定的执行步骤和验收命令写进一个可复用的 SKILL.md；下次换需求时，优先更新事实清单，而不是往 Skill 里不断加特例。
4. 比较两轮的解释次数、错误假设、实际测试和人工审核耗时。**没有对照，就不要宣布提效百分比。**

真正的转变是：**人不再反复当项目说明书，而是负责目标、不可推断的业务判断，以及对可执行证据签字。**

## 来源与核查

- 主文：Andrea Griffiths，[Build once, run on any agent: A practical guide to agent skills](https://github.blog/ai-and-ml/github-copilot/build-once-run-on-any-agent-a-practical-guide-to-agent-skills/)，2026-11-12。
- 开源工程：[Data Forge v0.1.0](https://github.com/tewarishishir/dataforge)、[复现说明](https://github.com/tewarishishir/dataforge/blob/main/examples/support-desk/DEMO.md)、[forge-init SKILL.md](https://github.com/tewarishishir/dataforge/blob/main/skills/forge-init/SKILL.md)。
- 交付证据：[Issue #3](https://github.com/tewarishishir/dataforge/issues/3)、[合并 PR #4](https://github.com/tewarishishir/dataforge/pull/4)（包含改动与测试说明）。
- 独立边界：[Claude Code 社区关于 Skills 误触发的讨论](https://www.reddit.com/r/ClaudeCode/comments/1xhn8az/i_switched_off_every_builtin_claude_code_skill_i/)、[When Instruction Files Go Stale](https://arxiv.org/abs/2611.09802)。
