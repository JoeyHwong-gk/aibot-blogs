---
title: "不要让编码 Agent 凭感觉优化：先交付火焰图，再交付修复"
date: 2026-11-08
source_url: "https://simonwillison.net/2026/Oct/12/python-3-15-web-edit-profiling/"
source_title: "Research: Profiling web-edit with Python 3.15: flame graphs and source heatmaps"
source_date: 2026-10-12
author: "Simon Willison（原始实践）；本文为独立中文分析"
tools:
  - "Codex / GPT-6 Astra Extra High（性能剖析）"
  - "Claude Code / Claude Opus 5.5 Extra（修复）"
  - "CPython 3.15 profiling.sampling"
  - "pytest / uv / GitHub"
evidence_level: "B：一手实践、可重跑剖析脚本、原始采样数据和已合并代码 PR；效果数据为作者/Agent 报告，未独立完整复跑"
score: 90
score_breakdown:
  technical_depth: 18
  reusability: 20
  empirical_evidence: 19
  independent_checks: 9
  novelty_nonduplication: 14
  traceability: 10
caveats:
  - "原文发表于 2026-10-12，修复合并于 2026-10-13；不是过去 7 天的新文章"
  - "性能结果来自合成工作负载，不代表真实用户的延迟或前端交互"
  - "原始剖析与 PR 基准分别运行在 macOS ARM64/Python 3.15 和 Linux x86_64/Python 3.13.16，不能直接跨环境比较"
  - "没有公开人工投入、模型调用费用或完整任务耗时；未独立复跑全量基准"
  - "第三方报道没有独立复现实验；PR 中部分随机化测试脚本未提交"
tags:
  - coding-agent
  - agent-collaboration
  - performance-debugging
  - codex
  - claude-code
  - evidence-based-engineering
  - profiling
  - reproducibility
---

# 不要让编码 Agent 凭感觉优化：先交付火焰图，再交付修复

**下次遇到「这个服务很慢」，先别让 Agent 直接改代码。给它一个可复现的真实请求流程，让它找到 CPU 究竟花在哪里，保存火焰图、原始采样和复跑方法；然后把这份报告交给另一个编码 Agent 去修复。人不必替 Agent 猜热点，但必须确认测试负载和最终验收有没有意义。**

这是 Simon Willison 在自己开源的 `web-edit` 项目上真正完成的一次协作。**2026 年 10 月 12 日**，他让 Codex 中的 GPT-6 Astra Extra High 研究 Python 3.15 的采样分析器，并为应用制作性能演示；**10 月 13 日**，他把报告链接交给 Claude Opus 5.5，后者产出优化 PR，随后合并和发布 `0.1a1`。[原始记录][simon]、[剖析报告与代码][research]、[PR #1][pr]

## 原先最费劲的环节，不是写优化代码

`web-edit` 是一个可通过浏览器操作本地文件夹的轻量应用，提供浏览、读取、保存、上传和删除文件的接口。假设只给 Agent 一句话「帮我加速」，它可能逐段阅读代码，猜测路径构造、正则、文件哈希或者数据库是瓶颈；即使改出更漂亮的代码，也不能证明用户操作变快。

Willison 第一条指令却不是「优化」：

> I need a Python 3.15 flame graph and heatmap demo of the "web-edit" app from PyPI - research first then make me those demos

他把探索工具、构造负载、采样、展示结果都交给 Codex。这里的关键不是模型品牌，而是**把“先证明哪里慢”变成第一份可以独立交付的工作**，避免在大段源代码上做无依据的猜测。[一手提示词][simon]

## 第一棒：让 Agent 产出别人也能核查的性能证据

Codex 最后交出的不是「某函数可能慢」的口头结论，而是一个独立的 [研究目录][research]：`README.md`、`demo.py`、`workload.py`、`uv.lock`、采样二进制、`stacks.txt`、`analysis.json`，以及可点开查看源码位置的[火焰图][flame]与[热力图][heat]。

它用原版 `web-edit 0.1a0`，生成 **12,000 行、混合 LF/CRLF 换行符**的文档及 **800 个文件**的目录。一个循环包含浏览、读取、改动每第 500 行、保存、下载并校验 SHA-256、上传、再次浏览、删除等八次 HTTP 请求。请求由独立客户端打到本地 Uvicorn 服务，不只是脱离实际接口的函数微基准。客户端同时校验内容和换行符，出错就中止。这样测到的热点至少对应一次有明确语义的完整操作。

在 Python 3.15 上采样约 20 秒，得到 **15,296 条成功记录的 CPU 调用栈**。分析表明：

| 调用路径 | 出现在采样调用栈中的比例 |
| --- | ---: |
| AnyIO `WorkerThread.run` | 95.1% |
| `FileStore.save_text` | 77.0% |
| `preserve_endings` | 64.8% |
| `SequenceMatcher.find_longest_match` | 38.3% |

**这不是四块可相加的 CPU 时间。** 一个栈同时经过 worker、保存和字符串比对，就会计入多行。真正关键的是：95.1% 的样本来自工作线程，若只抓主线程，很可能误判。保存文本时，为保留原文件混合换行符而执行的全文 `SequenceMatcher` 比对，才是最有价值的优化入口。[原始采样说明][research]

复跑入口写得很具体：

```bash
# 进入 research 仓库的 python-3-15-web-edit-profiling 目录
uv sync --locked
uv run demo.py --output fresh-run
# 打开 fresh-run/index.html
```

分析器核心调用也留在报告里：

```bash
python -m profiling.sampling attach -a --mode cpu -r 1khz -d 20 --binary -o recording.bin SERVER_PID
```

`-a` 是全线程采集；`SERVER_PID` 必须是实际运行且已预热的服务 PID。原实验在 macOS 上还遇到采样进程的调试权限问题，最终用单独复制、签名的 Python 解释器运行。不能把上述通用命令说成在所有系统上都能直接成功。研究报告记录 **13.40% 采样错误率**，并没有把它掩盖成零误差。

## 第二棒：把“诊断报告”而不是整段历史对话交给修复 Agent

接下来 Willison 向 Claude Opus 5.5 Extra 给出的指令只有一条核心要求：

> Read [the profiling report] and optimize this app

实际 PR 保留了[准确的报告 URL 和提示词][pr]。Claude 从报告中的已测热点出发，改动 `web_edit/files.py`，并在 `tests/test_app.py` 补充行为测试。这里出现了一次有价值的**跨模型、跨会话交接**：不要求两个 Agent 共享全部上下文，只要求第一棒给第二棒留下足够可靠的实验记录。

Claude 的主要修复不是微调一个常量，而是针对失败机制：

- **单一种类换行符：** 不再为每次保存做全文 diff，直接统一并替换换行符。
- **混合换行符：** 用在新旧文档中各出现一次的行作为锚点、求顺序一致的最长子序列，再让 `SequenceMatcher` 只处理锚点间的区段；针对散布式修改避免反复扫描整篇文件。
- **次要热点：** 用 `bytes.translate` 检查控制字符、`os.scandir` 遍历文件夹、减少文件重新读取与哈希；同时检查保存后的版本令牌语义。[代码差异][pr]

这比「让 Agent 自己做完」更具体：**每个改动都能回指原始热点或操作流程**。如果优化代理突然大规模重写前端，就应问它测量证据在哪里，因为第一棒根本没测浏览器端。

## 第三棒：让 Agent 用原负载验证，也让它找退化

PR 用 Linux x86_64、4 CPU、Python 3.13.16 在前后两个版本上跑同一套负载，报告了如下对比：

| 指标 | 修改前 | 修改后 |
| --- | ---: | ---: |
| 8 请求操作循环/秒 | 6.5 | 12.7 |
| 混合换行符保存的 HTTP 延迟 | 90.8 ms | 39.8 ms |
| 800 文件目录浏览的 HTTP 延迟 | 28.5 ms | 8.6 ms |
| 混合换行符且每第 5 行改动的函数耗时 | 3,963 ms | 29 ms |

所以，在**这组合成场景**中，操作循环吞吐约为 **1.95 倍**；最后一行约 **137 倍**只属于特定极端文本比对微基准，绝不能外推成「整套应用快 137 倍」。而且采样初测在 macOS ARM64/Python 3.15 上，PR 对比在 Linux/Python 3.13 上；有效的前后比较来自 PR **同环境**两版测量，不是把两台机器的数字拼在一起。

更值得学的是它没有只看速度。PR 报告 **130 个测试在 Python 3.11–3.15 通过**，另做 **20,000 个随机化换行符对比案例**。600 个构造了混合换行符的 Python 标准库文件实验里，新算法整体保留正确换行符的比例从 99.80% 到 99.93%，但其中 **25 个文件反而少保留了 1–6 行**。作者把这项语义差异列为审查要点，而不是用平均值盖过去。[PR 的测试与边界][pr]

还有一个隐藏收益：新版本令牌来自实际写入文件的 SHA-256，避免旧代码在并发外部修改时返回错误版本而可能导致后续覆盖；这属于**正确性变化**，应独立审查，不能简单归为性能优化。

## 为什么这种委派方式比“再写一段更长 Prompt”有价值

从 Agent 的工作机制看，第一棒把大范围、模糊的「查慢」缩成了一个可反驳的假设：**这项负载的大头是换行符比对，不是整体 Python 太慢。** 第二棒不必继承第一棒所有令牌，只要读取包含负载、环境、采样和反例的报告，就能在有限范围内选方法、修复、验证。

这种交接也让人更容易介入。人要决定的是：`12,000` 行混合换行的模拟是否贴合用户实际操作？换行符轻微变化能否接受？有没有前端体验、I/O 等其他瓶颈？而不是逐轮指挥 Agent 该改哪一行。

但原记录**没有给出人工工时、总任务用时、两次模型调用的真实成本**。因此可以说它展示了高质量的证据交接，不能据此宣称一定降低了全流程成本。第三方 [AT Pulse 的分析][third]也只是对原实验的转述，没有独立复跑这些性能数字。

## 明天就能照着试的一次委派

如果手头有一个 Python API 或后台「明显慢但找不到原因」，可以先让编码 Agent 做下面这件事，而不是让它立即提交优化 PR：

> 先不要修改生产代码。请在隔离环境复现一个具体的慢操作，使用真实 HTTP/命令入口构造可持续负载，采样所有实际执行线程，交付火焰图、热点对应的源码位置、原始数据、复跑命令与不确定性。负载必须检查操作结果正确；如果数据不足，不要猜最慢函数。

拿到报告后再开第二次任务：

> 根据这份报告针对最主要的已证实热点做最小改动。固定输入与机器条件测量修改前后结果；运行已有测试，增加针对行为退化的反例，列出哪些输入变快、哪些可能变慢。交付代码、性能数据和剩余风险，不要只给「优化完成」的文字总结。

**适用边界：** 当瓶颈主要是等待网络、数据库、锁或磁盘时，纯 CPU 采样可能看不到真正耗时，必须改用墙钟、I/O 或分布式追踪；真实用户体验还需浏览器端测量。小到只需查一行日志的任务，也不必额外建立完整剖析工程。

这次实践的可复用重点，不是所有任务都要两位 Agent，也不是盲目追求更强模型。**是把“定位、修复、验收”拆成可独立核查的交付，让下一位 Agent 继承证据，而不是继承前一位 Agent 的自信。**

## 原始来源与核查

- [Simon Willison：Python 3.15 对 web-edit 的性能剖析，2026-10-12][simon]（原始委派）
- [公开研究：脚本、`README`、负载、录制数据、HTML 图表][research]
- [PR #1：Claude Opus 5.5 修复、基准与测试，2026-10-13][pr]（已合并）
- [web-edit 0.1a1 Release](https://github.com/simonw/web-edit/releases/tag/0.1a1)
- [第三方报道：AT Pulse，2026-10-15][third]（非独立复现）

**核查方式：** 阅读了原文、报告、`workload.py`、PR 原文及文件补丁，并核对合并状态；没有在本次环境中部署 `web-edit` 或重跑 CPython 3.15 的原始基准。

[simon]: https://simonwillison.net/2026/Oct/12/python-3-15-web-edit-profiling/
[research]: https://github.com/simonw/research/tree/main/python-3-15-web-edit-profiling
[pr]: https://github.com/simonw/web-edit/pull/1
[flame]: https://simonw.github.io/research/python-3-15-web-edit-profiling/artifacts/flamegraph.html
[heat]: https://simonw.github.io/research/python-3-15-web-edit-profiling/artifacts/heatmap/index.html
[third]: https://www.at-pulse.com/stories/2667/
