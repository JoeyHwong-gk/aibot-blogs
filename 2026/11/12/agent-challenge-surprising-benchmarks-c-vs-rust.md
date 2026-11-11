---
title: "Agent 比较 C 和 Rust，别停在第一轮：追问反常结果并留下可复跑实验"
date: 2026-11-12
source_url: "https://simonwillison.net/2026/Nov/8/sqlite-parse-number-c-vs-rust/"
source_title: "Research: parse_number() as a SQLite extension: C vs Rust"
source_date: 2026-11-08
author: "Simon Willison"
tools:
  - "Claude Chat / Opus 5.5（非 Claude Code）"
  - "SQLite 3.53.1 / Python sqlite3"
  - "C / gcc 13.3.0"
  - "Rust / rustc 1.97.0"
evidence_level: "B：作者一手委派记录、公开代码、测试脚本及原始结果；没有独立复跑，社区独立验证不足"
score: 85
score_breakdown:
  technical_depth: 18
  reusability: 19
  empirical_evidence: 18
  independent_checks: 8
  novelty_nonduplication: 12
  traceability: 10
caveats:
  - "耗时只代表特定环境与合成输入上的程序性能，不代表人的工作效率提升倍数"
  - "作者未披露完整交互日志、人工总耗时和模型费用；发布的研究报告由 LLM 撰写"
  - "数值没有在本文作者的机器上独立复跑，也没有足够的独立社区重复实验"
tags: ["Agent协作", "Claude", "技术调研", "性能实验", "SQLite", "可执行验收"]
---

# Agent 比较 C 和 Rust，别停在第一轮：追问反常结果并留下可复跑实验

**一句话结论：** 让 Agent 比较两种技术实现，不要只收一张性能表。当结果违背预期时，继续问“差异来自语言、库函数还是实现细节”，要求它修改假设、重跑同一套测试，最后把代码和原始输出一并交付。这个方法适合做技术选型、性能排查、依赖替换和小型工程研究。

11 月 8 日，Simon Willison 记录了一个从遛狗时的想法开始的实验。他用的是普通 **Claude Chat（Opus 5.5）**，而非带完整本地环境控制权的 Claude Code。问题很小：给 SQLite 写一个 parse_number(value)，把「-$1,234.56」「12.5%」这类字符串转成可排序、求和的数值；比较 **C 和 Rust 扩展的性能、代码复杂度、编译后体积**。

这个案例值得读的，不是“C 打败了 Rust”，而是人如何在第一次答案出现意外后，继续驱动 Agent 找出原因，而非把第一次结果写成技术结论。

## 第一次委派：固定任务，允许不同实现

作者原始任务没有指定使用哪种库、怎样写解析循环，也没要求一开始就写优化版。他给出解析目标，再明确比较维度：性能、代码复杂度、动态库文件大小。由此，Claude 写出了三种初始方案：直接调用 C 的字符串转浮点函数、使用 Rust 的 rusqlite、使用 Rust 的 sqlite-loadable。

这一步有个很容易漏掉的前提：**跨语言比较必须先定义相同行为。** 公开研究目录中，Agent 补齐了模糊边界：NULL 和 BLOB 输入返回 NULL；INTEGER 和 REAL 原样返回；移除逗号、美元符号、英镑符号及百分号；负号只允许在数字前；无法解析的文本返回 NULL。特别地，45% 解析成 **45**，不是 0.45；科学计数法 1e5 被拒绝。它不一定符合所有业务习惯，却使四种实现能接受同一组验收数据。

如果平时用 Agent 做选型，我们很容易停在“写两份代码、跑一次 benchmark、做张表”。这次最有价值的工作发生在后面。

## 反常结果出现：14 KB 的 C 版本，居然比 Rust 慢

第一次结果有明显冲突：C 扩展只有约 **14 KB**，Rust 两个版本约 **321 KB、298 KB**；但在混合输入上，普通 C 的每行净解析开销是 **67.3 ns**，反而慢于 Rust 的 **45.9 ns / 40.5 ns**。

作者没有要求 Agent 再写一篇“Rust 为什么更快”的解释性报告，而是追加了一个非常具体的任务：**写一个自己的 C 小数解析函数，争取追上或超过 Rust 实现**。

这个追问把比较对象从“哪门语言快”拉回“当前实现的成本在哪里”。拆开输入分布就能看出问题：普通 C 在整数上是 **25.1 ns/行**，比两个 Rust 实现都快；小数上却升到 **106.3 ns/行**，而 Rust 为 **54.5 / 46.5 ns/行**。瓶颈不是 C 语言本身，而是普通实现用的 glibc strtod() 在这类数据上的代价。此处“瓶颈归因”是作者和 Agent 根据细分数据与实现代码给出的解释；不是跨平台、跨 libc 的定律。

Claude 随后实现了一个 C 快速路径：扫描输入时同时积累有效数字，能精确处理的常见小数直接计算；当有效整数不超过 2^53、小数位不超过 22，且平台满足浮点计算条件时走快速路径，其他情况回退至 strtod()。代码还用 FLT_EVAL_METHOD 限制可能出现双重舍入的环境，并没有偷换为“更快但精度随缘”的近似解析。

**第二轮不是把提示词写得更复杂，而是让 Agent 带着已观测的异常修正实现，然后接受同一组测试。**

## 同一套实验，前后对照

公开仓库里的 bench.py 在 SQLite 内存表中生成 **200 万行**，依次对不同扩展执行求和查询。脚本对同一负载做 **15 次交错运行**，取最小值，并减去一次基础扫描的开销；分别测试混合数据、纯整数、纯小数三种分布。下表单位是 **ns/行，越低越好**：

| 输入分布 | 普通 C | 优化 C | rusqlite | sqlite-loadable |
| --- | ---: | ---: | ---: | ---: |
| 混合 | 67.3 | **19.2** | 45.9 | 40.5 |
| 整数 | 25.1 | **15.8** | 36.4 | 32.1 |
| 小数 | 106.3 | **17.5** | 54.5 | 46.5 |

这意味着在**该实验的混合输入**上，优化后的 C 版本相对自身第一版每行净开销下降约 71%，比两个 Rust 版本约快 2.1–2.4 倍。但不能把这写成“C 普遍比 Rust 快两倍”：比较的是四份特定代码、一个 SQLite 回调场景，以及一台共享的 2 vCPU Linux 虚拟机。最小值与扣除基础扫描的方法也可能低估实际部署的端到端成本。

正确性同样有交付物。test.py 定义了 **40 个手写用例**，不仅检查数值，还检查 SQLite 的 typeof()；fuzz.py 对四种实现使用四个随机种子、每种 50 万个输入，合计 **800 万次对照检查**，包括 -0.0、整数与实数类型、长小数、2^53 附近的舍入边界。公开日志报告没有不一致。这里是 800 万次实现对照，**不是 800 万个互不重复的样本**。

另一个细节体现了 Agent 研究报告的价值：在 notes.md 中，它承认最初按版本依次测、取七次中位数时噪声大到排名会翻转，后来才改成交错执行、各取十五次最小值。没有这段记录，读者只看到漂亮的最终数字，很难判断为什么值得相信。

## 人在什么时候介入？什么交给 Agent？

在这个案例中，人的关键动作并不是手把手规定代码。Simon 明确提出业务函数与比较维度；看到“体积更小却更慢”的反常结果后，他要求继续优化 C 解析器；最后又让 Claude 按已有的 simonw/research 仓库格式，把原型整理成研究报告和可下载的工程文件。

Agent 承担了实现四个版本、统一边界条件、构造测试及基准、寻找性能差异和撰写报告。作者并未公开完整对话和逐轮人工操作记录，不能据此断言它全程无人干预，更不能计算节省了多少工时。

这种分工背后的原因很实在：**人判断问题是否值得追；Agent 用可执行代码回答为什么。** 模型可以擅长生成候选解释，但仅凭文字无法确认解释。要求它交付可重复运行的对照脚本，才把“说得有道理”变成了“别人能检查”。

最后一步也不只是让 Agent 写 README。报告目录包含 build.sh、四种实现、test.py、fuzz.py、bench.py、原始 results/ 和 notes.md。作者公开的重跑入口如下：

~~~bash
./build.sh
python test.py
python fuzz.py 1
python bench.py mixed
python bench.py int
python bench.py real
~~~

其中 Rust 的 sqlite-loadable 版本构建依赖 libclang；复现实验还需要匹配的 SQLite 开发环境。这里列出的是作者提供的命令，本文没有在另一台机器上完成复跑。

## 证据能说明什么，不能说明什么

[作者原文](https://simonwillison.net/2026/Nov/8/sqlite-parse-number-c-vs-rust/)明确说明使用 **普通 Claude 对话**；[公开研究目录](https://github.com/simonw/research/tree/main/sqlite-parse-number-c-vs-rust)提供代码、结果和工作笔记，而且 README 直接标注是 **AI 生成的研究报告**。这些是一手可核查的工程材料，但仍然不是独立实验室复现。其他媒体对结果的转述主要仍来自 Simon，**独立社区证据不足**；[SQLite 官方扩展文档](https://www.sqlite.org/loadext.html)只能佐证扩展接口机制，不能证明这些性能数值。

还有三个不能跳过的边界：测试数据偏向常见金额和百分比格式，不能代表任意字符串；C 的慢路径仍依赖本地 locale 对小数点的解释；报告没有记录人工总耗时、模型费用、重试次数或实际项目上线后的负载。因此，这次实验能证明**所给实现通过了作者发布的验证，并在发布环境里产生上述性能对照**，不能证明“使用 Agent 比人工开发节省了 X% 时间”。

## 明天就能照做

不必把流程膨胀成十几页规范。下次让 Agent 比较两套库、两门语言或两个修复方案时，可以直接改三个动作：

1. **固定外部行为和比较指标。** 给出真实输入与边界样例，要求候选都通过同一套验收，而不是各做各的“最佳版本”。
2. **遇到反常结果，要求定位而不是解释。** 让 Agent 拆分负载、寻找热路径、提出可证伪假设，然后实现一个定向改动，重新跑对照。
3. **把证据当成最终交付。** 要求保留原始输出、测试和基准命令、实验环境、失败记录，以及哪些结果还没有独立复验。

Simon 这次没有用一长串 Agent 指令获得好的结果。他只在最该追问的地方追问了一次，又坚持把结果交付成别人能够复跑的工程。这比“请分析 C 和 Rust 哪个更好”更值得学习。

## 原始来源与核查材料

- [Simon Willison 原始实践，2026-11-08](https://simonwillison.net/2026/Nov/8/sqlite-parse-number-c-vs-rust/)
- [四种实现、公开研究报告与测试入口](https://github.com/simonw/research/tree/main/sqlite-parse-number-c-vs-rust)
- [bench.py：负载、交错运行与计时方法](https://github.com/simonw/research/blob/main/sqlite-parse-number-c-vs-rust/bench.py)
- [fuzz.py：舍入与类型一致性检查](https://github.com/simonw/research/blob/main/sqlite-parse-number-c-vs-rust/fuzz.py)
- [notes.md：第一次结果、噪声与优化过程](https://github.com/simonw/research/blob/main/sqlite-parse-number-c-vs-rust/notes.md)
- [results/：每项原始测量记录](https://github.com/simonw/research/tree/main/sqlite-parse-number-c-vs-rust/results)
- [SQLite 官方可加载扩展文档](https://www.sqlite.org/loadext.html)
