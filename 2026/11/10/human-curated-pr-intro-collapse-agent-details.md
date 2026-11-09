---
title: "Agent 写的 PR 描述太长？保留完整证据，只让人审核开头"
date: 2026-11-10
source_url: "https://feeds.simonwillison.net/2026/Nov/9/details-summary/"
source_title: "Hide your slop PR descriptions behind <details><summary>"
source_author: "Simon Willison"
source_date: 2026-11-09
tools:
  - Claude Code
  - Codex
  - GitHub
evidence_level: B
score: 87
caveats:
  - "作者的一手工作习惯和五个公开 PR 可核查，但没有审核时间或阅读率的对照实验。"
  - "作者认为自己项目的 Agent 描述通常准确，不能推导为所有 Agent 描述都可靠。"
  - "效果是减轻阅读负担的主观报告；并非已量化的交付效率收益。"
tags:
  - AI-Agent
  - Coding-Agent
  - PR-Review
  - Human-in-the-loop
  - Claude-Code
  - Codex
---

# Agent 写的 PR 描述太长？保留完整证据，只让人审核开头

> **一句话结论：** 让 Claude Code 或 Codex 继续留下完整的改动、测试与风险说明，但把人亲自认可的一小段「为什么改、改了什么」放在 PR 顶部，把机器生成的长报告折叠起来。适合 Agent 已经能独立产出 PR、而审阅者开始被长描述淹没的团队。

Simon Willison 2026 年 11 月 9 日写了一篇不足千词的短文，主题也很小：怎么写 Agent 生成的 Pull Request 描述。但它击中一个实际问题：生成解释的成本越来越低，**阅读解释的成本却没有跟着下降**。如果每个 PR 都附上一份结构完整的长报告，读者可能连第一页都不想看。

作者的原话很直白：他看到这些大段 AI 文字，第一反应是「我不打算全读」。尴尬的是，这些 PR 往往还是他本人驱动 Agent 写出来的。不是它们一定错了，而是机器太擅长把每一步都写进去。

## 他遇到的不是「Agent 不会解释」，而是「解释过剩」

Willison 维护 Datasette、llm 等开源项目。他现在把 **PR 当作一项工作的基本单位**：Agent 开始改动，提交几次 commit，反复测试、修改，最后通过 PR 留下决策和验证记录再合并。他实际使用 Claude Code 和 Codex；本文公开的具体示例多数标注 Claude Opus 5.5，个别标有 Medium 推理档位，不能据此推断所有 PR 都使用同一配置。

旧做法的困难并非缺少文档。一个 Agent 可以在 PR 描述里准确列出改了哪些模块、为何这样实现、如何测试。读者却需要先阅读很多实现细节，才能弄明白这次变更到底解决什么问题。写作端的「全面」，变成阅读端的「先筛选再理解」。

他没有规定 Agent 每次必须把描述压到五行，也没有删掉技术细节。他选择**让信息分两层出现**：顶部放经过人确认的简短叙述，完整技术记录保留在可展开区域。

## 一个小到可以立即复制的工作方式

GitHub 的 Markdown 支持 HTML 的 `<details>` 和 `<summary>`。下面是依据作者做法整理的 PR 正文示例，并非他的逐字模板：

```markdown
原来每个表格页面都会加载约 220KB 的 edit-tools.js，
包括没有编辑权限的用户。现在改为用户触发编辑功能时按需加载。

<details>
<summary>Claude Opus 5.5 · 实现与验证明细</summary>

### 改动
- 引入按需加载脚本
- 调整编辑入口的加载时机

### 验证
- 相关权限与浏览器测试
- 回归测试结果

</details>
```

其中有三个决定值得保留：

**第一，顶部不是「AI 摘要」，而是「人认可的判断」。** 作者不一定逐字亲写。他有时直接把 Agent 已经写好的那句好话摘出来。关键不是谁敲了键盘，而是有人看过并认可：读者首先应该知道这一点。

**第二，详细文本不用丢。** 发生回归、要追溯兼容性决定，或审阅者对测试范围有疑问时，可以展开全文。折叠解决的是默认展示顺序，不是证据归档。

**第三，明确标出机器生成部分。** 作者用 `Claude Opus 5.5`、`Claude Opus 5.5 Medium` 这样的标题，告诉读者下面是哪一类说明，而不把整份报告伪装成人类写作。标签不是质量担保。

GitHub [官方文档](https://docs.github.com/en/get-started/writing-on-github/working-with-advanced-formatting/organizing-information-with-collapsed-sections)确认支持这种折叠写法。实际复制时，在 `</summary>` 后和 `</details>` 前留空行，保证内部的 Markdown 标题、列表和代码块正常渲染；到 PR 的 Preview 页看一眼即可。

## 看看他实际合并的 PR，而不是只看写作建议

作者提供了五个公开实例。我逐一核对了这些 PR 的正文；它们都呈现「简短开头 + 可选截图或命令输出 + 折叠详细解释」的结构。

| PR | 首屏让人知道什么 | 展开后才看到什么 |
|---|---|---|
| [Datasette #3052](https://github.com/simonw/datasette/pull/3052) | 46 列的单行数据页横向滚动，页面右侧还留着大片空白；改成纵向字段列表 | 字段复制、空值、长文本展开、模板兼容性与测试 |
| [Datasette #3043](https://github.com/simonw/datasette/pull/3043) | 分页时看不清自己在哪一段；顶部加入行范围与导航 | Keyset pagination 的位置计数、SQL 索引策略、边界测试 |
| [Datasette #3042](https://github.com/simonw/datasette/pull/3042) | 约 220KB 的编辑脚本连只读用户也加载；现在等需要编辑才取 | 约 6KB 的 loader、权限门控、Playwright 检查 |
| [datasette-agent #51](https://github.com/datasette/datasette-agent/pull/51) | 插件现在能为聊天框 `#` 引用注册新对象类型 | Hook 的数据结构、权限检查、异常处理、325 项测试的作者报告 |
| [llm #1838](https://github.com/simonw/llm/pull/1838) | OpenAI 兼容端点不必再手工编辑 YAML，可以用 CLI 管理 | 子命令设计、向后兼容、测试和示例配置 |

这些例子各有不同的「第一眼证据」。界面 PR 在开头附截图；CLI PR 贴真实的 `--help` 输出；性能与加载行为 PR 则先说明问题和改变。**不是统一套一句「本 PR 优化了用户体验」。** 作者把最能帮助读者判断的内容放到首屏，才让后面的技术细节各归其位。

我还核对了这些公开 PR 的状态：五个示例均已合并，并且正文确实包含对应的折叠块。这能证实该做法已被作者用于真实交付，而不是仅存在于一篇建议文章中。但「已合并」不能证明这种排版提高了代码质量。

## 为什么它有用，以及哪部分只是判断

我的工程解释是：审查有两种不同的认知任务。

第一种是**分流**：这个改动解决什么问题？我现在要不要投入注意力？第二种是**核验**：它到底改了哪些代码，哪些风险已经测过？把全部内容摊开，强迫读者在还没决定是否深读之前就承担第二种任务的成本。折叠描述允许先做第一种判断，再按需进入第二种。

这不是要求人少审查关键代码。若 PR 涉及支付、权限、数据迁移或用户数据，简短开头只能帮你定位风险，不能替代代码检查、日志和测试。**「少读机器解释」与「少核验实际改动」是两回事。**

还有一个反例：在 [r/devops 的独立讨论](https://www.reddit.com/r/devops/comments/1nf0zga) 中，有开发者抱怨其团队的 AI PR 描述会写出代码根本没有实现的功能。这与 Willison 对自己 PR 的评价不同，提醒我们不要把「描述完整」当成「内容可信」。如果顶部摘要本身没经过确认，折叠只是在更漂亮地隐藏错误。

Willison 报告的是他自己的主观感受：PR 读起来没那么令人疲惫。公开材料没有提供审阅耗时、阅读率、缺陷漏检率或人工参与成本的前后对照。本文不给出虚构的节省百分比，也不把五个 PR 视为受控实验。

## 明天改一次 PR，就能试

不需要换 Agent，也不用引入新的流程系统。下次 Claude Code 或 Codex 做完一个中等规模的改动，让它照常生成完整 PR 描述，然后自己做三件事：

1. **确认开头的一小段。** 它应该说清楚旧行为、用户或维护者遇到的问题，以及新行为。可以从 Agent 的长报告中摘句，但必须与最终 diff 一致。
2. **留下一个能快速理解变化的证据。** UI 用前后截图，CLI 用 `--help` 或输出，缺陷修复用最小复现与结果；不必为每个 PR 机械增加附件。
3. **折叠剩余说明，保留测试与限制。** 标注 Agent/模型来源；在后续改动使描述过时时，同步更正顶部与明细。最终合并仍以测试、实际改动和必要人工审查为依据。

值得试的不是某种 HTML 标签，而是一个编辑决定：**让 Agent 负责把工作记录完整，让人负责决定读者第一眼该看什么。** 当 Agent 每天能写出更多代码时，这种小小的注意力分配可能比再写十条 PR 模板规则更实用。后半句是工程判断，尚无独立量化收益。

## 来源与核查记录

- **主文：** Simon Willison，[Hide your slop PR descriptions behind <details><summary>](https://feeds.simonwillison.net/2026/Nov/9/details-summary/)，2026-11-09。
- **原始交付物：** [Datasette #3052](https://github.com/simonw/datasette/pull/3052)、[#3043](https://github.com/simonw/datasette/pull/3043)、[#3042](https://github.com/simonw/datasette/pull/3042)、[datasette-agent #51](https://github.com/datasette/datasette-agent/pull/51)、[llm #1838](https://github.com/simonw/llm/pull/1838)。
- **平台机制：** [GitHub Docs：Organizing information with collapsed sections](https://docs.github.com/en/get-started/writing-on-github/working-with-advanced-formatting/organizing-information-with-collapsed-sections)。
- **独立社区反例：** [r/devops：Malicious compliance（关于 AI PR 描述失真的讨论）](https://www.reddit.com/r/devops/comments/1nf0zga)，2025-09-12；这是相关风险反馈，不是针对 Willison 方案的效果评测。

**核查边界：** 已读主文和五份 PR 正文，并核对官方 Markdown 机制及独立社区的相反经验；未独立测量阅读效率，也没有将 PR 描述中的测试通过声明当作亲自复跑的结果。
