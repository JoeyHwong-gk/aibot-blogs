---
title: "别让 Agent 读完 README 就下结论：用原型和失败探针做技术调研"
date: 2026-11-03
source_url: "https://simonwillison.net/2026/Oct/26/"
source_title: "webframe: evaluated as a scraper and a sandbox"
source_date: 2026-10-26
author: "Simon Willison"
tools:
  - "Claude Opus 5.5"
  - "webframe @ 03155e2"
  - "shot-scraper"
  - "Go 1.27.1"
  - "WebKitGTK 2.52"
evidence_level: "B：一手委派实践、公开的可运行原型与实验记录、上游独立 Issue/PR 交叉核查；未独立复跑"
score: 87
score_breakdown:
  technical_depth: 18
  reusability: 19
  empirical_evidence: 16
  independent_checks: 11
  novelty_nonduplication: 14
  traceability: 9
caveats:
  - "作者公开的研究报告、测试脚本和工作笔记均由 AI 生成，不代表已经由人逐项验证"
  - "原文所链接的完整 Claude 对话本次无法访问；本文依据公开文章和代码审查"
  - "对比计时每站点仅三次；未测量人类投入、总完成时间或生产环境收益"
  - "问题定位针对 webframe 03155e2，部分缺陷于 2026-10-29 已在上游修复，不能当作最新版现状"
tags:
  - coding-agent
  - agent-research
  - technical-investigation
  - reproducible-experiments
  - tool-evaluation
  - agent-collaboration
---

# 别让 Agent 读完 README 就下结论：用原型和失败探针做技术调研

**下次评估一个开源工具，别只让 Agent 整理功能表。让它固定版本、实际安装、复现核心用法、造一个替代方案，再故意寻找失败路径。最终交付的是能复跑的调查现场，而不是一份看起来已经得出结论的报告。**

适用于技术选型、第三方库验证、Bug 根因排查和安全边界调查。本文精读 Simon Willison 于 **2026 年 10 月 26 日**公开的一次 Claude Opus 5.5 委派实践：[原文](https://simonwillison.net/2026/Oct/26/)，并检查了 [公开研究仓库](https://github.com/simonw/research/tree/main/webframe-evaluation)、上游 Issue 和合并 PR。Simon 的仓库明确说明：**该研究目录内所有文字与代码都由 LLM 写出**。这是可审计的 Agent 工作产物，不是独立复现的科学实验。

## 他没有问“webframe 好不好”，而是让 Agent 去试

调查对象是 [tractordev/webframe](https://github.com/tractordev/webframe)：一个通过原生 WebView 运行网页 JavaScript 的命令行工具。Simon 想知道两件很实际的事：能否替代成熟的 **shot-scraper javascript** 做网页采集？能否在隔离环境执行不可信的 JavaScript？

常见的低效做法是让 Agent 比较 README、API 和 Star 数，拿到一张功能表后仍要自己安装测试。更糟糕的是，把“没有 Node.js 的 fs/process 对象”误读为“可以安全执行不可信代码”。

Simon 的委派把动作讲得很清楚：阅读仓库和参考工具的文档，**实际运行 webframe、尽量试出边界、写几个探索性原型**，然后分别判断采集用途与沙箱用途。他没有替 Agent 指定全部测试步骤，也没有把“要证明它可用”当成预设结论。

这改变的是研究交付的单位：从“关于工具的回答”，变成“可以重新运行、比较并推翻的实验材料”。

## 一次调查留下了什么

公开记录固定了被评估的 **webframe 提交 03155e2**，环境包括 Ubuntu 24.04、Go 1.27.1、WebKitGTK 2.52 和 Xvfb。仓库记录的起点可以这样复现（依赖版本仍需按实际系统安装）：

~~~bash
git clone https://github.com/tractordev/webframe.git
cd webframe
git checkout 03155e2
make build
./webframe examples/hello.html
~~~

然后 Agent 没有停在 Hello World。它阅读了底层 JavaScript bridge 和 WebView 代码，留下四类材料：

| 交付物 | 能回答的问题 |
| --- | --- |
| README 与逐步笔记 | 环境、命令、观察结果以及已知缺口是什么？ |
| webframe.diff（约 +97/−17 行） | 针对已发现问题，最小改动能走到哪里？ |
| wfjs/（约 350 行 Go） | 复刻 shot-scraper javascript 常见调用后，实际效果怎样？ |
| sandbox/ 与 probes/ | 不可信输入、网络外联、持久化、无限循环是否能被控制？ |

这些都在 [研究目录](https://github.com/simonw/research/tree/main/webframe-evaluation)里；不是本文重新编造的实验。

### 不拿“能运行”冒充“能够替换”

研究中的 wfjs 原型跑了网页标题、异步表达式、等待条件、错误退出、Readability.js 导入等用例，并对同一网页进行了简单计时。

例如，访问 datasette.io 获取标题，记录为：

| 工具 | 三次耗时 |
| --- | --- |
| shot-scraper（Playwright 1.63） | 2.50 / 1.99 / 2.01 秒 |
| wfjs | 2.19 / 1.79 / 1.96 秒 |

这只能说明两者在该条件下大致同一量级，**不能证明 wfjs 更快**：每站点只有三次，网络抖动未控制；研究还报告 wfjs 相关进程常驻内存约 640 MB，而对照的 Chrome 与 Python 约 456 MB。

更要命的是负面结果。原版 webframe 的隐藏窗口尺寸为 0×0；页面 CSP 拒绝 eval 时调用失败；部分 DNS/代理加载失败可能一直等待；没有可用的 HTTP 状态码接口。作者让 Agent 写适配原型，才暴露出这些 README 功能表不容易说明的差异。

### 用反例戳穿“看起来像沙箱”

第二组原型对不可信 JavaScript 做了约二十项探针，包括本地网络外联、存储跨次保留、无限循环、输出伪造和文件读取。

结果里有一项很适合当作研究反例：严格 CSP 阻挡了 fetch 等请求，但**顶层页面导航仍能向本机 HTTP 服务发出请求**，测试服务记录了带参数的访问；把进程放入无网络的 namespace（unshare -rn）后，该路径才被阻断。

另一项反例更隐蔽：开启 bubblewrap 保护 WebKit 渲染进程，不意味着页面失去了网络能力，因为实际发请求的 WebKit NetworkProcess 在该隔离层之外。原版还默认保留 localStorage/cookie；沙箱外壳没有超时和内存上限；与网页共享的 JS 世界可能被页面干扰，导致假结果或卡住。

**结论是研究记录给出的，而不是工具宣传语：webframe 可以作为执行层的组件，不能单独充当安全沙箱。** 即使叠加网络隔离、临时 profile、进程超时及随机标识的结果校验，也还欠缺独立脚本世界和 CPU/内存配额等条件。

这正是值得让 Agent 做破坏性实验的场景：只要求“它能做什么”，很容易得到正面示例；要求“它在哪些具体输入下会失败”，才有机会发现决定选型的约束。

## 人如何直接复用这种协作方式

下面是根据上述任务改写的委派模板，**不是 Simon 的原始 Prompt**：

> 评估这个仓库是否能替代我们正在使用的某工具，用于指定的真实工作负载。先固定目标仓库提交与运行环境，读取当前方案的用法，再实际安装运行。不要只对比文档：做一个最小兼容原型，跑通几条典型成功路径，设计至少三种可能否定方案的失败探针。记录每条命令、环境、原始结果和无法验证的条件。最后给出适用、不可用及还需调查的范围；不得把模拟结果当作生产验证。

这种任务表达把 **What** 指定得足够具体：评估哪种用途、和谁比较、证据格式是什么；至于优先阅读哪些源码、选择什么探针、怎样修补原型，由 Agent 决定。

调研时我会要求把实验结果写进仓库文件，而不是只写在最终聊天里。这样另一位工程师或另一轮 Agent 可以从固定提交、运行日志与失败样例开始，省去重新猜测上下文。

验收也不必读完全部 Agent 生成的代码。优先检查三个问题：**原始命令能不能跑、关键负例是否真实触发、最终结论有没有超出实验覆盖范围。** 对高风险选型，必须自己抽样复跑关键探针；“Agent 已经测过”不是独立证据。

## 社区证据：研究有用，但也会漏掉 Bug

本次在 webframe 上游找到了独立问题报告：[Issue #10](https://github.com/tractordev/webframe/issues/10) 于 **10 月 29 日**指出，Linux 隐藏 WebView 的定时器会被限速，requestAnimationFrame 可能完全不触发。报告给出了三个可运行 HTML 复现页，问题于 **10 月 30 日**关闭。这说明 Simon 的大量探针仍可能漏掉其他关键失败模式；不能把一轮 Agent 调查当作穷尽证明。

同时，上游于 **10 月 29 日**合并了 [PR #11（临时 profile）](https://github.com/tractordev/webframe/pull/11)与 [PR #12（CSP 下执行表达式）](https://github.com/tractordev/webframe/pull/12)。因此，上面提到的部分问题属于 **03155e2 的历史状态**，不是对最新版 webframe 的实时缺陷清单；也不能据时间相近便声称这些合并由 Simon 的报告促成。

独立 Issue 与 PR 能核对工具确有相关问题及修复，却不能证明作者报告里每一个实验数字。公开的完整 Claude 对话链接本次无法访问，本次也未重新运行测试。至于“这种委派替人省了多少小时”，作者没有提供对照数据；本文不编造效率百分比。

## 下次调查前的简短清单

- [ ] 明确要替代的真实任务及现有对照工具，而非泛问“项目怎么样”。
- [ ] 固定 commit、依赖与环境，让结论能追溯。
- [ ] 让 Agent 做可运行原型、失败探针和对照，而不只是文档摘要。
- [ ] 保存运行命令、原始结果、失败记录和未覆盖场景。
- [ ] 人抽样复跑决定结论的关键路径；对外部副作用和凭证使用保留明确边界。
- [ ] 决策前检查上游近期 PR/Issue，防止拿旧版本问题评价新版本。

**要减少的不是研究深度，而是人替 Agent 手动试错、转述和重新整理证据的次数。** 把 Agent 当成能动手的研究助理，并要求它留下可推翻自己的实验，比让它快速写出一份自信的技术报告更值得尝试。

## 来源与核查

1. Simon Willison，[webframe: evaluated as a scraper and a sandbox](https://simonwillison.net/2026/Oct/26/)，2026-10-26。
2. [simonw/research — webframe-evaluation](https://github.com/simonw/research/tree/main/webframe-evaluation)：报告、逐步笔记、补丁、原型与攻击探针；仓库注明全部由 LLM 生成。
3. [tractordev/webframe @ 03155e2](https://github.com/tractordev/webframe/commit/03155e2ba67cb9f2825cca84c7c21134ed47add3)：原始研究评估版本。
4. [上游 Issue #10](https://github.com/tractordev/webframe/issues/10)：独立复现的 Linux 隐藏 WebView 定时器问题。
5. [上游 PR #11](https://github.com/tractordev/webframe/pull/11)与 [PR #12](https://github.com/tractordev/webframe/pull/12)：分别涉及临时 profile 和 CSP 兼容表达式执行。
