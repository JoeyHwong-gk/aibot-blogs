---
title: "Anthropic 为什么把子 Agent 折回主循环：一次包含失败样本的迁移实验"
date: 2026-10-28
source_url: "https://claude.com/resources/articles/building-agents-when-to-fold-subagents-into-your-main-agent-loop"
source_title: "Building agents: When to fold subagents into your main agent loop"
tools:
  - Claude Code
  - Claude Platform
  - Claude Opus 5
  - Skills
  - MCP
  - Tool Search
evidence_level: "B：Anthropic 一手工程实践、公开源码与真实模型轨迹；社区讨论独立，企业效果未独立复现"
score: 92
caveats:
  - "企业部署中约 2 倍加速和系统提示词缩短 80% 均为厂商报告，缺少公开原始对照"
  - "公开示例的第一轮只有 12 道任务、每题两次，不能推断普遍成功率"
  - "参考实现目前不维护，部分更新只通过离线测试，未在真实模型上重跑"
  - "本次核查直接读取 GitHub 记录并计算指标，未实际运行需要模型密钥的评测"
tags:
  - coding-agent
  - claude-code
  - single-agent-loop
  - subagents
  - context-engineering
  - agent-evaluation
  - tool-use
  - agent-security
---

# Anthropic 为什么把子 Agent 折回主循环：一次包含失败样本的迁移实验

**一句话结论：** 多个专家 Agent 不一定比一个能按需加载 Skill、查找工具、持续持有上下文的 Agent 更好。正确做法不是立即删除子 Agent，而是固定任务、模型和验收标准，每次只折叠一个角色，查看真实交付是否改善，特别是有没有悄悄破坏原来依赖跨 Agent 交接的路径。

**适用场景：** 已有 Router / Orchestrator，后面连接多个领域子 Agent；经常出现重复查询、冗余转述、会话交接丢信息、延迟和费用增长。对高度并行、独立验证或有严格权限隔离的任务，先不要套用这一结论。

本文精读 Anthropic 应用 AI 团队 Diane Sarkis 于 **2026 年 10 月 22 日**发表的 [Building agents: When to fold subagents into your main agent loop](https://claude.com/resources/articles/building-agents-when-to-fold-subagents-into-your-main-agent-loop)，并逐项检查官方公开的 [single-agent-loop-kit](https://github.com/anthropics/single-agent-loop-kit) 源码、评测记录和社区反例。核查时仓库 `release` 指向 `c43796287b83bec847db923f07131d1cc8de750d`。

## 多 Agent 怎样变成了额外负担

文章讨论了一种常见架构：前面一个意图分类器和协调 Agent，后面按部门或问题种类挂上订单、物流、退货、技术支持等专家。这个设计很容易随组织结构增长：一个团队拥有一个专家及自己的提示词、工具和测试集，中央团队再负责路由。

一开始看起来分工清晰，后来却可能出现五种问题：

- **信息损失：** 订单 Agent 查到订单号，物流 Agent 还要重新获取必要事实，最终主 Agent 只看到摘要。
- **角色膨胀：** 专家重复实现相似查询、校验和对话逻辑。
- **规则冲突：** 各团队不断往总提示词增加限制，形成难以维护的规则堆。
- **局部指标失真：** 各专家各自拿高分，但用户的完整任务不一定完成。
- **固定开销：** 每次委派都启动另一个模型循环，增加调用与等待。

Anthropic 报告，一位客户改成单主循环后，任务运行时间约快 **2 倍**；团队还称，为较强 Claude Code 模型精简了超过 **80%** 的系统提示词。这两项均是**官方内部经验**，缺乏公开的客户任务集、运行条件与独立对照，不应将其视为普遍倍率。

文章提到 Opus 4.5 之后的模型更能自行规划，并以 Opus 5.5 等近期模型说明某些旧式编排已不再必要。这是对模型能力变化的工程判断，不意味着取消所有外部验证。

## 新结构：一条主循环，Skill 和工具按需出现

目标架构并不是把所有知识复制进一个超长 Prompt，而是：

```text
用户任务
   ↓
主 Agent（同一个上下文持续规划与修正）
   ├─ Skill 索引 → 按需加载领域步骤和判断依据
   ├─ Tool Search → 按需发现工具定义
   ├─ 只读工具 → 返回可直接核查的原始结果
   └─ 写入工具 → 执行端校验权限、前置条件与批准
   ↓
外部验收 / 必要时向用户提问
```

核心是 **Progressive Disclosure（渐进式披露）**：系统提示词只保留 Skill 的简短描述；真正需要该领域知识时才读取完整内容。不常用的工具 Schema 也通过 Tool Search 延后加载。Anthropic 表示，当系统提示词携带约 **30–50 个工具定义**后，工具选择往往开始退化。这是经验观察，不是适用于所有模型的硬阈值。

更关键的是明确哪些东西不能只靠文字提醒：

| 旧子 Agent 持有的内容 | 折叠后应放哪里 |
| --- | --- |
| 领域工作步骤、业务判断方法 | Skill 正文 |
| 经常变化的事实、制度与产品资料 | 检索工具或按需参考文件 |
| 查询 API、业务动作及固定格式结果 | Tool / MCP |
| 不能越权、禁止重复写入、审批要求 | 工具执行端的强制检查 |
| 语气、问候、通用对话要求 | 主 Agent 的统一规范，重复内容删除 |
| 旧模型需要的过度分步和提醒 | 先做对照实验，再决定删除 |

这不是把复杂性藏进工具描述。**当一个规则必须始终成立，就让执行端拒绝违规调用，而不是祈祷模型记得它。**例如退货过期，`create_return` 应直接检查资格并拒绝，而不只在 Skill 里写“不要超过 30 天”。

官方 `reference_loop/loop.py` 提供了可读实现：每轮允许批量工具调用；设置轮数、Token 与并发批次上限；有写入能力的工具必须显式配备 `approve` 回调；子 Agent 的 Token 也可以计入统一预算。它是参考代码，不是生产级 Agent 平台。

## 迁移的关键：先做实验，再删角色

Anthropic 发布的 [single-agent-loop-kit](https://github.com/anthropics/single-agent-loop-kit) 并不假设单 Agent 一定赢。它要求按照以下顺序工作：

**第一步，清点实际调用链。** 从生产轨迹找出总是连续调用的角色、重复读取相同数据的角色，以及拥有独立上下文价值的角色。为每个子 Agent 给出 `merge`、`fold`、`as-tool` 或 `keep` 决策。

**第二步，冻结端到端验收集。** 先选约十道真实任务，覆盖多 Agent 连续交接和明确禁止的操作；每题运行两三次。必须固定模型 ID、代码版本、输入状态与验收标准。旧系统的路由准确率不能作为折叠后主指标，因为新系统可能没有路由；应测最终行为。

**第三步，一次只变动一个组件。** 可以合并经常相互依赖的专家，也可以将专家保留为返回结构化结果的工具；如果完全折叠，就把领域知识转为 Skill，把 API 调用转为工具。每次改动都重新执行相同验收集。

**第四步，不只看总分。** 逐题比较从通过变失败的任务、实际工具调用链、外部写操作、延迟、Token 和模型调用次数。特别要检查相邻领域：主 Agent 获得一个新查询工具之后，可能绕过另一个原本必经的专家。

官方源码可以从这里开始检查：

```bash
git clone https://github.com/anthropics/single-agent-loop-kit
cd single-agent-loop-kit
python3 -m unittest discover tests

cd examples/support-desk
KIT_MODEL=fake python3 evals/run_eval.py --variant smoke
python3 ../../skill/scripts/compare.py example-run/results baseline v1
```

其中 `KIT_MODEL=fake` 是**关键字规则构造的离线替身**，只能检查执行管线是否能跑通，不能给模型能力打分。真实模型调用会产生费用，需另外配置 Anthropic SDK 和 `KIT_MODEL`。把 Skill 加到自己仓库时，官方示例使用 `.claude/skills/single-agent-loop`；交互中会在审计、付费运行前、基线完成后和每次修改后暂停确认。

## 我重新计算了官方示例：总分变好，也可能丢掉原本会做的事

仓库保存了 **2026 年 10 月 8 日**用 `claude-opus-5` 执行的真实模型记录：12 道任务，每题两次，共 24 条计分记录；先运行原有多 Agent 路由，再将退货专家折进主循环。

我直接读取公开的 `results.jsonl`，重新统计了以下数据：

| 指标 | 原系统 | 折叠一名专家后 |
| --- | ---: | ---: |
| 通过次数 | 22/24 | 23/24 |
| 平均墙钟耗时 | 27.31 秒 | 19.36 秒 |
| 平均模型调用 | 6.33 次 | 4.83 次 |
| 平均输入 Token | 18,154 | 18,203 |
| 平均输出 Token | 1,539 | 1,051 |

**结果并非单向变好。** 原系统的 `t01`（查询订单状态和预计送达日期）两次均通过；折叠后一次通过、一次失败。失败轨迹中主 Agent 已拿到订单状态，却没有继续从物流专家获取送达日期，便告诉用户目前没有日期。

另一方面，原系统 `t11` 两次失败，折叠后两次通过。所以总成功数从 22 变为 23，却掩盖了一条用户原本能够顺利完成的任务发生回退。

这比厂商提到的“约 2 倍提速”更能说明问题：**当工具变得更直接，模型可能过早认定自己已经掌握了全部事实。**这是一种可以从轨迹观察到的路径变化，不能只依靠最终汇总得分判断是否安全迁移。

而且平均输入 Token 几乎没有下降。由此不能声称本次示例已经证明 Token 费用会按耗时同比减少；缓存、价格和模型调用分布仍需单独核算。

公开记录还包括 **10 月 9 日**的一次“保留诊断专家、但作为工具调用”的尝试。团队使用 17 道任务各两次作基线和新版本对照，又对两个有疑问的任务各追加四次运行。作者明确承认：6 次重复仍是小样本，而且部分多轮输入场景当时根本没有覆盖。样例的合并与长期维护步骤，以及经过后续修改的参考循环，并未得到完整真实模型复测。

这些限制由项目 README 主动披露。该仓库也明确声明：**参考实现不维护、不接受贡献**。应将它作为可拆解的实验样本，而非直接用于生产的完整框架。

## 为什么有效，也为什么会失效

**原理一：避免有损交接。** 每个独立 Agent 只把部分观察传回上层。多次摘要会丢失时间、标识、限制和不确定性。单循环能直接保留先前工具结果，让同一个模型修订计划。但长上下文仍可能遗忘细节；把所有日志一次性放入上下文并不能替代检索和结构化状态。

**原理二：减少固定调用开销。** Router、专家与最终汇总都可能产生新模型请求。折叠减少调用链，但单 Agent 也可能为了覆盖多个领域而反复查找工具。因此应测每个任务的 Token、模型调用和墙钟时间，而不能靠 Agent 数量估算成本。

**原理三：把判断与约束分开。** 模型决定下一步做什么；工具检查是否允许做。只有这样的隔离，才能在更换模型、修改 Prompt 或上下文压缩之后继续保障权限。Skill 文本不是授权边界，Tool Search 也不会自动防止恶意工具描述或不可信资料注入。

**反例：** 独立验证、探索多个互不相关的大型代码区域、需要不同凭证的操作，以及内容体量大到应当丢弃中间上下文的任务，仍然适合子 Agent。它们可以并行工作，返回精简证据，甚至使用不同模型。保留这些角色不是架构失败，而是对实际任务依赖关系的响应。

## 社区反馈与证据等级

[Reddit r/ClaudeAI 在 10 月 24 日的讨论](https://www.reddit.com/r/ClaudeAI/comments/1xdwquj/anthropics_new_guide_oct_22_argues_most_agents/)提供了两类反馈：一些使用者说自己减少了不必要的子 Agent，另一些人强调独立上下文在长文档阅读、不同模型成本控制、并行任务和交叉审查中的价值。这是**独立社区经验**，并非同条件复现实验，不能拿来证明 Anthropic 的企业加速幅度。

本次评定 **92/100，B 级证据**：技术深度 19/20、可复用性 19/20、实证 17/20、社区核查 12/15、新颖性 15/15、可追溯性 10/10。官方文章具有真实部署经验，源码、测试脚本和真实轨迹可公开检查；企业数据不可独立复现，示例样本较小，故不评为更高证据级别。

## 可复用清单

- [ ] 从真实任务和调用轨迹识别频繁串联、共享大量上下文的子 Agent。
- [ ] 先记录当前系统的成功率、成本、延迟、工具链及外部副作用。
- [ ] 评测覆盖每条常见专家调用链、多轮会话、禁止动作及相邻业务。
- [ ] 固定模型、仓库快照、测试数据和验收条件；每题至少重复运行。
- [ ] 领域步骤放入 Skill，实时数据与动作放入 Tool，硬约束放进执行端。
- [ ] 工具仅暴露必要能力，写操作校验当前用户权限和明确批准。
- [ ] 一次只折叠一个角色，对比逐题结果和轨迹，不只看总分。
- [ ] 单独试一次“去掉 Skill，仅保留工具”，确认 Skill 真的带来收益。
- [ ] 对显著变差或证据不足的场景，保留或恢复子 Agent。
- [ ] 统一管理主提示词与端到端评测，领域团队维护各自的 Skill 和工具。

**最后的判断：** 一套 Agent 架构的成熟度，不取决于它编排了多少角色。更好的问题是：哪个角色真的带来了不可替代的上下文隔离、并行能力或独立验证？如果回答不了，就让相同模型、相同任务和实际行为证据来决定是否需要它。

## 来源与复核入口

1. **原文：** Diane Sarkis, Anthropic, 2026-10-22：[Building agents: When to fold subagents into your main agent loop](https://claude.com/resources/articles/building-agents-when-to-fold-subagents-into-your-main-agent-loop)。
2. **官方源码：** [anthropics/single-agent-loop-kit](https://github.com/anthropics/single-agent-loop-kit)，核查 `release` commit `c43796287b83bec847db923f07131d1cc8de750d`。
3. **示例记录：** [10 月 8 日第一次真实运行](https://github.com/anthropics/single-agent-loop-kit/blob/release/examples/support-desk/example-run/README.md)；[10 月 9 日第二次运行](https://github.com/anthropics/single-agent-loop-kit/blob/release/examples/support-desk/example-run/as-tool/README.md)。
4. **参考代码：** [reference_loop/loop.py](https://github.com/anthropics/single-agent-loop-kit/blob/release/reference_loop/loop.py)；[评测比较脚本](https://github.com/anthropics/single-agent-loop-kit/blob/release/skill/scripts/compare.py)。
5. **社区反馈：** [r/ClaudeAI：关于减少子 Agent 的讨论](https://www.reddit.com/r/ClaudeAI/comments/1xdwquj/anthropics_new_guide_oct_22_argues_most_agents/)，2026-10-24。

_核查边界：上述示例统计从公开 JSONL 重新计算，但未在当前环境执行真实模型评测；不使用未公开的客户数据，也不将官方经验写成普遍定律。_
