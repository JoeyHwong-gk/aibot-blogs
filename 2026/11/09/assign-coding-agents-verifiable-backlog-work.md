---
title: "别把整个 Backlog 扔给 Agent：.NET 团队从 878 个 PR 学到的委派方法"
date: 2026-11-09
source_url: "https://devblogs.microsoft.com/dotnet/ten-months-with-cca-in-dotnet-runtime/"
source_title: "Ten Months with Copilot Coding Agent in dotnet/runtime"
source_date: 2026-03-23
author: "Stephen Toub / Microsoft .NET"
tools:
  - "GitHub Copilot Coding Agent (CCA；实验期间模型与工具持续更新)"
  - "Copilot CLI"
  - "GitHub Pull Requests / Copilot Code Review"
  - ".github/copilot-instructions.md"
evidence_level: "B：真实开源仓库的长期一手记录，关键 PR、配置与独立批评可查；并非随机对照实验"
score: 90
score_breakdown:
  technical_depth: 18
  reproducibility: 20
  empirical_evidence: 19
  independent_checks: 13
  novelty_nonduplication: 11
  traceability: 9
caveats:
  - "原文发表于 2026-03-23，非新发布；本次选择是因为远程仓库尚无任务适配与长期 PR 成本分析的同题文章"
  - "878 个 PR 是 2025-05-19 至 2026-03-22 的历史样本，不代表今天工具的能力"
  - "成功率为已结束 PR 的 merged/(merged+closed)，排除 90 个仍开放的 CCA PR；非任务完成率"
  - "任务由维护者选择，模型与环境不断变化；配置前后 38.1%/69% 不能解释为单独配置的因果收益"
  - "作者未测量总人工工时、模型费用或 CI 消耗；回滚率也可能漏计非显式 revert"
tags:
  - coding-agent
  - real-world-workflow
  - github-copilot
  - task-selection
  - issue-triage
  - build-feedback
  - pull-request-review
  - evidence-based-delivery
---

# 别把整个 Backlog 扔给 Agent：.NET 团队从 878 个 PR 学到的委派方法

**一句话结论：** 想让编码 Agent 真正替你消化积压工作，先挑「问题可复现、变更边界明确、它能自己构建验证」的任务；给它项目真实的构建命令和限制，再用 PR 评审决定接受、返工或撤销。让 Agent 开 PR 很便宜，让人把不靠谱的 PR 收拾干净并不便宜。

**适用场景：** 长期维护的复杂代码仓、积压 Bug、测试补齐、重复性 API 清理，尤其适合工作项很多、但工程师时间被更紧急的事占据的团队。跨平台原生代码、性能瓶颈和架构改造，应先确认 Agent 能否实际验证，不要仅因任务描述看起来清楚就放手。

**非新发布。** 主文是 Stephen Toub 于 **2026 年 3 月 23 日**写的 [Ten Months with Copilot Coding Agent in dotnet/runtime](https://devblogs.microsoft.com/dotnet/ten-months-with-cca-in-dotnet-runtime/)。它追踪的是 **2025 年 5 月 19 日—2026 年 3 月 22 日**的 GitHub Copilot Coding Agent（CCA）实践。选它不是因为今年十一月又有了什么新模型，而是因为它把「到底该把什么活交给 Agent」从口号变成了可检查的真实 PR 样本。

## 一个生日派对，二十多个工单

2025 年 10 月的一个周六，Toub 在孩子的生日派对上翻手机里的 dotnet/runtime Backlog。很多标着 help wanted 的问题早已被人确认，却始终排不上时间。他花了约一小时挑选并委派了 **20 多个 Issue**；到当天晚上，已经查看了 **22 个 PR**。

重点并不是他能在手机上点击「Assign to Copilot」。他挑工单时做了三次判断：问题表述够不够清楚？按照当时的 Claude Sonnet 4.0/4.5 和工具环境，Agent 有能力解决吗？改动范围是不是可控？这一步仍由懂项目的人负责。之后 Agent 独立查代码、提交补丁和测试，人才进入审查。

其中三件事特别说明这种分工的价值和风险。

**线程安全 Bug，快速合并。** [PR #120619](https://github.com/dotnet/runtime/pull/120619) 处理 `JsonObject.InitializeDictionary` 的并发竞争导致 `GetPath` 间歇性失败。Toub 已有修复假设，在委派时一并提供；Copilot 验证假设、使用 `Interlocked.CompareExchange` 修复并添加回归测试。作者报告人工审查约 10 分钟，主要是两个小修改意见。PR 于 2025 年 10 月 13 日合并。这里省掉的是人从头写补丁、反复运行测试的劳动；不是把工程判断交给模型。

**正确地写出了一个不该合并的改动。** [PR #120638](https://github.com/dotnet/runtime/pull/120638) 试图禁止对正则锚点进行量词修饰，比如 `^*`。Agent 找到解析器、修改代码、同步更新测试，看起来十分完整。审查测试变化时，维护者发现既有行为原本就是有意设计的；其他主流引擎也并非一律拒绝。PR 最终关闭、未合并。这是一次有价值的调查，但不能把「Agent 完成要求」等同于「原需求值得实现」。**修改现有测试以迎合新逻辑，往往正是应该停下审查的信号。**

**环境无法验证，反馈往返膨胀。** [PR #120633](https://github.com/dotnet/runtime/pull/120633) 希望让 Windows 使用 BCrypt；Agent 最初堆砌 `#if` 条件，维护者要求改成 Windows/Unix 独立源文件。多轮修改后，还暴露了 NativeAOT 二进制体积问题。它不能在当时的 Linux 沙箱中验证 Windows 路径。这个 PR 最终没有合并。Agent 能写跨平台代码，并不意味着它能测试跨平台代码；未闭合的验证回路会把节省的编写时间变成昂贵的审查轮次。

## 878 个 PR，不是 878 次成功

研究覆盖 878 个 CCA PR：**535 个合并、253 个关闭、90 个当时仍开放**。作者给出的 **67.9%** 是 535/(535+253)，只在已结束的 PR 里计算，不是所有委派任务的一次成功率，也不是与人类随机分配同样任务后的胜率。团队报告 535 个已合并 PR 中 3 个随后被显式回滚，但这种检索也可能漏掉其他形式的撤销。

比总量更值得关注的是任务类型。

| 委派类型 | 样本 PR 数 | 已结束 PR 合并比例 |
| --- | ---: | ---: |
| 删除与清理 | 77 | 84.7% |
| 测试 | 99 | 75.6% |
| Bug 修复 | 317 | 69.4% |
| 新功能 | 131 | 64.5% |
| 性能优化 | 46 | 54.5% |

这不是证明「清理任务天然更容易」的受控实验，分类和任务分配本身就有选择偏差。但它提供了一个可操作的优先次序：**优先选有稳定回归条件、明确局部边界的活；对于需要解释系统行为、测量性能或跨平台验证的任务，先补证据和工具。**

在 464 个关联源 Issue 的 CCA PR 中，**93 个（20%）**来自创建超过两年的旧工单。它们不是突然变得更重要，而是当实现边际成本下降后，过去「值得修但不值得现在腾时间」的问题有了新处理方式。团队同时也把新报的、具备复现条件的 Bug 当天派给 Agent。

Toub 还发现，清理任务并非绝对不用管。[PR #122953](https://github.com/dotnet/runtime/pull/122953) 最初搜索范围过窄，遗漏测试目录中的条件编译标记；经人要求全仓搜索后才补齐。即使任务是机械替换，也应在验收里写明**搜索范围与剩余命中数**，而不是只问「测试是否通过」。

## 第一轮做错的事：让 Agent 在不能构建的环境里写代码

团队于 2025 年 5 月启用 CCA 时，仓库缺少 `.github/copilot-instructions.md`，沙箱网络规则又阻止它访问必需的 NuGet 包源。Agent 不知道该用什么参数构建，有时执行一个二十多分钟的大构建，有时连依赖都拉不下来。最糟糕的状态是：**它交出了代码，但自己根本没办法确认代码能不能编译。**

后来团队补了可访问的依赖源、明确的构建命令和平台限制，并持续维护说明。作者统计，初次环境准备调整前的 PR 合并比例为 **38.1%**，调整后为 **69%**。这是很强的关联信号，但同时期模型、任务和用户经验也在变化，不能据此宣称「多写一个说明文件让能力提高 30.9 个百分点」。

说明文件里有几条真正来自项目经验的细节：库代码的构建入口和参数不是通用 `dotnet build`；特定平台测试应使用 `[ConditionalFact]` / `[ConditionalTheory]`；不能把本地化异常消息的英文全文写死在断言里；不要为了测试方便随意添加 `InternalsVisibleTo`。这些规则不是为了控制模型每一步怎么想，而是把**项目中容易导致返工的事实**暴露给它。

还有一个反向改变：早期为避免失败，Agent 每接一个任务先做完整基础构建，至少多耗二十分钟。到了 2026 年 1 月，团队又取消了「每次必做全量构建」，让它根据任务选择需要的构建。说明文件并非越厚、步骤越多越安全；一条过去合理的规则，也可能变成今天拖慢交互的主要开销。

当前仓库的 [copilot-instructions.md](https://github.com/dotnet/runtime/blob/main/.github/copilot-instructions.md) 已继续演进。不要把现在的内容误认为 2025 年实验开始时的完整配置。

## 人的瓶颈从写代码移到了审核

作者还有一次在旅途中快速委派了九个 PR 的经历。每个都需要认真审查，复杂变更可能需要 **30—60 分钟人工检查**，于是一个人很快制造了约 **5—9 小时**的团队审查队列。甚至一周后仍有三项没处理。这是原作者根据工作经验的估计，不是工时系统的精确测量。

已合并 CCA PR 平均有 **16.5 条评审与讨论评论**，同期人类 PR 平均 **12.4 条**；在 529 个历史完整的已合并 CCA PR 中，**158 个（29.9%）**初始提交后不需要再追加代码提交，其他大多仍需反馈与修改。更快生产 PR 不自动等于更快交付。

因此，批量委派不能只按「能启动多少 Agent」来规划。先筛选工单，再按评审容量排队；遇到不清晰的业务契约或无法验证的平台，不妨交给本地有工具链的 Agent，或先委派它调查而非直接修复。

## 我会怎样明天就用起来

以下是从上述案例提炼的**实践建议，不是作者公开的逐字提示词**。面对一个积压 Bug，可以这样交代：

> 请先用当前代码和测试复现 Issue 描述的问题，指出现有行为是否可能是刻意设计；如果与已有测试或兼容性约束冲突，先报告证据，不要直接改测试来迎合 Issue。若确认需要修复，限定到相关模块，添加行为回归用例，运行仓库指定的构建与测试命令。最终交付：失败原因、改动范围、运行命令及结果、未覆盖的平台/情形、全仓同类问题的搜索结果。不要自行扩大 API 或跨平台改造范围。

最小的行动清单：

1. **先选题，再开 Agent：** 优先可复现 Bug、明确清理、缺失测试；不要把含糊架构任务与批量替换一起扔进去。
2. **把「怎样验证」交给 Agent：** 提供正确构建入口、依赖访问和测试限制；做不到就让它写清无法验证的边界。
3. **检查测试是不是在证明错误行为：** 当 Agent 改写旧测试或改动兼容语义时，先复核需求是否成立。
4. **限制在途 PR 数量：** 依据团队可用的审核时间委派，不用「今天启动了几个 Agent」作为产出指标。

## 证据、反对意见和边界

这份长期记录难得的地方在于可点开具体 PR，看到维护者如何纠正模型，甚至撤销整个任务。对照 [#120619](https://github.com/dotnet/runtime/pull/120619) 与 [#120638](https://github.com/dotnet/runtime/pull/120638)，可以直接看到「写得对」和「该不该写」是两回事。

独立的早期批评也不能省略。Remy van Duijkeren 曾以 [#115762](https://github.com/dotnet/runtime/pull/115762) 为例，批评 Copilot 反复宣称修好却需要专家纠错，认为审查成本可能吞噬速度收益（[原文](https://automatevalue.com/short/copilot-just-sent-a-pr-to-net-runtime/)）。这个 PR 确实没有合并。Toub 后来解释当时 Agent 连依赖和构建环境都拿不到；两种观点并不矛盾：**改善环境能减少某类无效失败，但不意味着所有代码审查都能交给 Agent。**

本文核查了作者文章、公开 PR 状态及评论、仓库当前指令和独立批评；没有重新统计全部 878 个 PR，没有复测不同模型，也没有作者未披露的人工工时、计算费用与完整回归质量数据。**PR 合并比例仅能描述该团队在那个时期的选择与流程，不能直接推算另一支团队会省下多少时间。**

## 来源

- [主文：Stephen Toub，Ten Months with Copilot Coding Agent in dotnet/runtime（2026-03-23）](https://devblogs.microsoft.com/dotnet/ten-months-with-cca-in-dotnet-runtime/)
- [成功的线程安全修复：PR #120619](https://github.com/dotnet/runtime/pull/120619)
- [正确实现、最终不应合并的正则解析器改动：PR #120638](https://github.com/dotnet/runtime/pull/120638)
- [平台限制导致反复返工：PR #120633](https://github.com/dotnet/runtime/pull/120633)
- [全仓搜索遗漏：PR #122953](https://github.com/dotnet/runtime/pull/122953)
- [仓库现行 Copilot 指令（持续演进，不等于历史快照）](https://github.com/dotnet/runtime/blob/main/.github/copilot-instructions.md)
- [独立早期批评：Remy van Duijkeren（2025-05-26）](https://automatevalue.com/short/copilot-just-sent-a-pr-to-net-runtime/)
