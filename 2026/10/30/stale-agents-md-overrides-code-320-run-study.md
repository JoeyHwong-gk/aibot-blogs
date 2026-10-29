---
title: "AGENTS.md 过期时，Codex 可能改坏正常代码：320 次对照实验与修复边界"
date: 2026-10-30
source_url: "https://arxiv.org/abs/2610.30571"
source_title: "Stale Instruction Files Override the Code in Coding Agents: A One-Sentence Fix"
source_date: 2026-10-22
author: "Xiaonan Xu; Wenjing Wu"
tools:
  - "Claude Code 2.1.302 / Claude Sonnet 5.5"
  - "Codex CLI 0.163.0 / GPT-6 Sol"
evidence_level: "B：一手对照实验及公开项目、320 条运行记录与分析脚本；未独立复跑模型实验"
score: 92
score_breakdown:
  technical_depth: 19
  reusability: 19
  empirical_evidence: 19
  independent_checks: 11
  novelty_nonduplication: 15
  traceability: 9
caveats:
  - "40 个刻意制造冲突的小型 Python 项目，不代表真实大型仓库的自然冲突率"
  - "每个 Agent 仅覆盖一个版本、一个模型及有限工具权限设置"
  - "仅比较局部代码约定，不证明在安全策略、显式迁移需求或遭篡改代码上应始终优先遵循代码"
  - "公开数据与统计脚本已检查，但未重新运行 320 次模型调用，独立研究也未复现该论文的全部数字"
tags:
  - coding-agent
  - codex
  - claude-code
  - agents-md
  - context-engineering
  - instruction-priority
  - agent-evaluation
---

# AGENTS.md 过期时，Codex 可能改坏正常代码：320 次对照实验与修复边界

**一句话结论：** `AGENTS.md`、`CLAUDE.md` 不能只当作永久有效的项目事实。它们一旦落后于代码，Agent 可能发现冲突却继续照旧执行，甚至改写原来正常的代码。对于**局部编码约定**，在用户级指令中明确“当文档与当前实现不一致时，以代码现状为准并报告冲突”，能显著改变实验结果；但不能把这句话扩展成“代码永远高于安全规则或明确需求”。

本文精读 Xiaonan Xu 与 Wenjing Wu 在 **2026 年 10 月 22 日**发表的论文 [Stale Instruction Files Override the Code in Coding Agents: A One-Sentence Fix](https://arxiv.org/abs/2610.30571)。作者公开了 [40 个项目、任务、320 条运行记录及分析脚本](https://github.com/WenJing95/stale-docs-data)，因此可以检查实验定义和汇总口径；本文没有独立重跑这些模型调用。

## 为什么一份旧文档会让 Agent 更容易写错？

假设团队已把模块日志从 `print()` 迁移到 `logging`，但 `CLAUDE.md` 还写着“输出进度时使用 print”。开发者让 Claude Code 增加一个新命令，Agent 读取代码后，面前出现两组不一致的信号：

- 项目指令文件：使用旧约定。
- 当前模块和测试：一致采用新约定。

人类工程师通常会把旧文档当作待修复的信息；Agent 不一定如此。研究者观察到的一个典型行为是：**Claude Code 在最终答复中明确指出代码与文档冲突，新增代码却仍按旧文档编写。**这不是“没看到代码”，而是“看到了，却选择了另一种权威”。

问题更严重时，Agent 可能认为整仓代码都没有遵守项目规范，于是主动把工作中的代码“修正”成过期文档描述的样子。

## 实验如何控制变量

作者构造 **40 个小型 Python 项目**，每个有 3–8 个现有模块、能够通过的测试集、一份与代码冲突的指令文件和一个新增功能任务。冲突覆盖 16 类局部约定，包括日志、路径、异常、导入、命名、格式化、测试文件命名、枚举、金额表示等；相反约定在任务中双向出现，避免偏向某一种惯用写法。

固定运行组合：

| 执行器 | 具体版本 | 模型 |
| --- | --- | --- |
| Claude Code | 2.1.302 | Claude Sonnet 5.5 |
| Codex CLI | 0.163.0 | GPT-6 Sol |

每个项目、每个 Agent 在**不添加与添加一条用户级优先级指令**两种条件下各运行两次：40 × 2 × 2 × 2 = **320 次运行**。每次都是新的项目副本，模型只收到功能要求。作者自动检查生成代码中的局部约定，恢复项目原始测试并运行隐藏功能测试。另用 Claude Opus 5.5 盲判最终答复是否明确报告冲突。

这个设计的长处是同一任务、同一冲突、同一模型下只改变用户级规则；它并非随意挑选成功 Demo。不过，“一次运行”与“一个约定选择”不是一回事：每次新增功能涉及 4–5 个选择，**两个实验条件各有 720 个被评估的选择**。

## 结果：能指出矛盾，不代表能按正确来源行动

| 指标 | 原有指令 | 添加优先级规则 |
| --- | ---: | ---: |
| 遵循现有代码约定 | 4.2%（30/720） | **91.4%（658/720）** |
| 遵循过期文件约定 | **90.1%（649/720）** | 3.2%（23/720） |
| 原始与隐藏测试全部通过 | 89.4%（143/160） | **100%（160/160）** |
| Codex 改写原有代码的运行 | 20/80 | **0/80** |

数据来自作者发布的 [numbers.md](https://github.com/WenJing95/stale-docs-data/blob/main/numbers.md) 与 [more_numbers.md](https://github.com/WenJing95/stale-docs-data/blob/main/more_numbers.md)。**90.1% 是局部选择比例，不是 90.1% 的真实项目失败率**；100% 也只指这 160 次特定实验的测试通过率。

再看两个 Agent 的不同失败方式：

**Claude Code：**不加规则时，80 次运行中有 **79 次报告了冲突**，但其局部选择仍有 86.4% 服从过期文件。它在这一组实验中没有大规模重写已有代码，全部 80 次测试仍通过；这说明“测试通过”可以与“项目新增代码风格失配”并存。

**Codex CLI：**不加规则时，80 次中有 **20 次重写旧模块**，其中 **17 次使原有测试失败**。论文举出的 `kanbanlite` 案例把已有的 `Column` 枚举换成字符串常量，造成原测试无法导入 `Column`。这些重写涉及 12 个项目，重写运行的删行数中位数为 198 行、涉及文件数中位数为 9 个。加规则之后 80 次中没有发生同类重写，测试均通过。

作者还报告了工具调用数及耗时中位数的小幅下降，例如 Codex 从 108.4 秒降至 94.9 秒，但这不能外推出长期任务的成本收益；论文没有给出足以支持通用 Token 成本结论的比较。

## 为什么会这样：指令优先级与事实来源被混在一起

作者提出的机制解释是：

1. `CLAUDE.md`、`AGENTS.md` 经过 Agent 执行器加载后，作为项目指令呈现给模型。
2. 当前代码则通常在 Agent 通过文件读取工具检查仓库之后，以**工具结果**的形式进入上下文。
3. 模型本来就应当避免把外部文本中的命令当成高优先级指令；然而，过期项目文件中的“使用某写法”也被呈现为应该服从的命令。

问题在于，项目说明混合了两种性质完全不同的内容：**真正的行为约束**和**对代码现状的描述**。当后者失效，模型仍可能按前者对待它。

实验支持“来源优先级影响行为”这一解释，但没有直接观测模型内部的因果过程。用户级补充指令也不是安全控制：它只是把“如何处理冲突”放在模型能够看到的位置，不能保证所有模型、权限配置或项目上都成立。

## 怎么用：先补冲突规则，再让过期文档退出系统

作者测试的是添加到用户级文件中的一句英文规则：[查看实验原句](https://github.com/WenJing95/stale-docs-data/blob/main/conditions/B_fix.md)。其意图是：当项目文档与代码约定冲突时，采用现有代码惯例，并在最终答复中报告冲突。

Claude Code 对应 `~/.claude/CLAUDE.md`，Codex CLI 对应 `~/.codex/AGENTS.md`。这一次实验使用的是用户级文件，**不是随意追加在仓库里任意位置就已得到相同验证**。

如果准备在实际团队推广，我建议先采用带边界的中文版本：

> 对于当前任务没有要求改变的局部实现约定，若项目说明与现有已验证代码冲突，以现有实现为线索，并在答复中列出冲突及证据；不要据此推翻明确的用户需求、安全要求、权限约束或目标迁移规范。

**注意：这段中文是本文的生产化改写，不是论文使用并测过的原句；不能直接引用论文的 91.4% 作为其效果。**

接下来应修复真正的问题：把被标记为不一致的规则提交到正常的代码评审中，在修改约定的同一个 PR 更新指令文件。不要长期依靠一条补丁规则维持过期文档。

如要复核论文作者发布的汇总，可以在隔离环境中检查以下命令和脚本：

```bash
git clone https://github.com/WenJing95/stale-docs-data.git
cd stale-docs-data
python -m pip install numpy tiktoken
python scripts/analysis.py numbers
python scripts/more_numbers.py
```

这些命令用于从**已存档的运行记录重新统计**，不是重做 320 次真实模型调用。运行第三方脚本前应检查内容并使用隔离环境。论文复现实验中的 Codex 曾使用 `danger-full-access`，且 Claude Code 自动接受文件编辑；那是受控实验设置，**不应作为实际工作仓库的安全配置照搬**。

## 独立核查与证据边界

独立研究 [The Working Set of a Coding Agent](https://arxiv.org/abs/2608.16630)（2026 年 8 月）也设计了“工程规范与当前代码冲突”的试验：39 次冲突试验均选择了书面规范；另一次对照中，“正确规范 / 无规范 / 过期规范”的更好实现比例分别为 100% / 33% / 0%。它支持**旧规范有时比没有规范更糟**的现象，但不是对本论文 320 次实验的直接复现。

[openai/codex#8547](https://github.com/openai/codex/issues/8547) 则有实际用户报告**同一会话中修改了 AGENTS.md，Agent 仍沿用旧指令**。OpenAI 维护者说明指令在会话开始时加载并保留，更新后应开启新会话或明确要求重读。这个 Issue 讨论的是**会话缓存过期**，不是本论文的**文档与代码冲突**；两者不可混作同一漏洞。

论文数据仓库确实公开了 40 个任务、`runs.jsonl`、统计脚本以及判别依据。本文已核查结构、版本记录和作者汇总表，**没有独立执行模型实验，也没有独立证明其在真实大型项目上的泛化效果**。本次证据等级评为 B，而非将可公开检查误写成第三方复现。

## 风险、反例与不适用场景

**代码并不天然代表正确答案。**当团队主动推进 React、数据库 Schema、加密方案或安全接口迁移时，现有代码可能恰好是需要淘汰的对象。若机械执行“代码优先”，反而会阻止新标准落地。

同样，代码如果已被恶意篡改，读取它不能产生新的业务授权；当前仓库里的调用方式也不能覆盖平台级的访问策略。对外部副作用、用户数据、凭证和部署权限的控制，必须由独立执行层核验。

对于多仓、多语言、长期跨会话任务，本论文没有证明一句话能解决所有文档漂移。更可靠的持续措施是给项目规则标明适用模块和失效条件，让构建、测试、源文件引用等能够验证的事实尽量自动检查，并在迁移完成时删除旧约定。

## 可复用检查清单

- [ ] 清点 `AGENTS.md` / `CLAUDE.md` 中的事实描述与真正强制约束，避免混写。
- [ ] 设计一个小型任务，故意让过期约定和现有代码冲突，观察 Agent 最终采用哪一边。
- [ ] 用相同仓库快照比较有无冲突规则的执行结果，而不是凭主观感受加规则。
- [ ] 检查新代码的局部约定、原有测试、隐藏行为测试和对既有文件的不必要改写。
- [ ] 对每一处冲突修订说明文件；修改后开启新会话或明确要求重新加载。
- [ ] 遇到架构迁移、用户明确要求或安全政策冲突时，不使用“现有代码优先”覆盖它们。
- [ ] 保存模型版本、规则内容、工具权限、轨迹及回归结果，以便后续模型升级重新评估。

## 来源

1. **主文**：Xiaonan Xu, Wenjing Wu, [Stale Instruction Files Override the Code in Coding Agents: A One-Sentence Fix](https://arxiv.org/abs/2610.30571)，2026-10-22。
2. **公开数据与统计脚本**：[WenJing95/stale-docs-data](https://github.com/WenJing95/stale-docs-data)，包括 [实验优先级句子](https://github.com/WenJing95/stale-docs-data/blob/main/conditions/B_fix.md)、[主结果](https://github.com/WenJing95/stale-docs-data/blob/main/numbers.md) 和 [补充结果](https://github.com/WenJing95/stale-docs-data/blob/main/more_numbers.md)。
3. **独立研究**：[The Working Set of a Coding Agent: Coherence Debt in Repository-Scale Tasks](https://arxiv.org/abs/2608.16630)，2026-08-17。
4. **GitHub 真实问题讨论**：[openai/codex#8547 — Automatically reread AGENTS.md within a session when it is modified](https://github.com/openai/codex/issues/8547)，2025-12-25 提出、2026-02-23 关闭。
5. **第三方解读（未独立复现）**：[DotsFeed，2026-10-24](https://dotsfeed.com/updates/codex-follows-stale-agents-md-one-sentence-fix)。

_编辑说明：对原文技术条件和失败样例进行了忠实转述；未复制论文长段文字。用户级中文指令、安全边界和推广建议属于本文工程判断，已与作者实测结果区分。_
