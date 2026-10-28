---
title: "让 Claude 写证明，让 Rocq 验证明：239 个 Fuzz 入口实验的做法与边界"
date: 2026-10-29
source_url: "https://www.anthropic.com/research/formal-verification"
source_title: "LLMs are making formal software verification feasible at a much larger scale"
author: "Keane Lucas 等 / Anthropic Frontier Red Team 与 Theorem"
tools:
  - Claude
  - Claude Opus 5
  - Claude Opus 5.5
  - Rocq 9.2.0
  - OSS-Fuzz
  - AddressSanitizer
  - UndefinedBehaviorSanitizer
  - RISC-V
  - LPS
  - Docker
evidence_level: "B：官方实验与公开机器证明、验证脚本和版本锁定；本次核查了源码与说明，未在本地完成计算密集型证明复跑"
score: 94
caveats:
  - "239 个入口/123 个库是研究报告总量，公开仓库当前 main 的 61 个入口/26 个库不是全部结果"
  - "形式化保证仅对固定构建、有限输入和给定 RISC-V/Linux 语义及内存前提成立"
  - "证明检查器只证明定理成立，不证明定理覆盖用户需要的全部行为；有些证明并不证明终止"
  - "Claude 修改程序后重新生成规格，有让程序退化为平凡可证实现的风险；必须独立审查补丁"
  - "论文报告的 token 改进来自作者实验，未独立复测真实推理费用或证明运行时资源"
tags:
  - coding-agent
  - claude
  - formal-verification
  - rocq
  - oss-fuzz
  - security
  - proof-checking
  - agent-evaluation
---

# 让 Claude 写证明，让 Rocq 验证明：239 个 Fuzz 入口实验的做法与边界

**一句话结论：让 Agent 生成庞大的形式化证明，把“是否正确”交给独立的证明检查器；但人仍要检查定理、目标二进制和运行环境是不是自己真正关心的东西。** 适合维护解析器、压缩库、协议实现等已有 Fuzz 测试、对内存安全要求很高的团队，不适合拿来一键宣布整个服务“没有 Bug”。

2026 年 10 月 22 日，Anthropic Frontier Red Team 与 Theorem 披露了一项实验：用 Claude 为 **123 个开源库中的 239 个 OSS-Fuzz 入口**生成机器可检查的证明，证明特定 RISC-V 构建在限定输入和环境前提下不会触发约定的崩溃。报告称最终有效证明约 **1.8 亿行 Rocq**，典型证明约 **60 万行**，对应二进制中位规模约 **1.8 万条 RISC-V 指令**。[原始研究](https://www.anthropic.com/research/formal-verification)

这些数字本身并不新鲜地证明“AI 会写很多代码”。不一样的是：**人不必阅读那 1.8 亿行证明，证明是否成立由一个显式固定的形式化检查器决定。** Claude 可以犯推理错误，但错误的证明原则上过不了检查；前提是检查器、定理和机器语义本身值得信任。

## 1. 不再只找一个崩溃样本，而是证明限定范围内找不到崩溃

传统 Fuzzing 的循环是：选择输入、运行程序、收集崩溃、修复、继续跑。即使执行了数万亿次，也只能说“还没有找到”，不能说“不存在”。

这项工作的关键选择是：**不从自然语言临时编一份“正确性规格”，而是复用开源项目已有的 Fuzz 入口。** 开发者已经在入口里写了断言、调用顺序和预期约束；团队把入口与依赖编译为二进制，再从真实机器指令构造待证明的定理。

流程可以简写成：

```text
现有 OSS-Fuzz 入口 + 固定源码版本
              │
              ▼
    RISC-V 静态编译：plain / ASan / UBSan
              │
              ▼
  二进制哈希 + LPS（RISC-V/Linux 形式语义）
              │
              ▼
        自动生成待证定理
              │
              ▼
     Claude 生成 Rocq 证明和辅助引理
              │
              ▼
    独立 Rocq 检查 + 公理审计 + CI
```

不能把“ASan 构建不崩溃”单独理解成“函数完全正确”。Theorem 的公开证明还涉及**普通构建和 Sanitizer 构建的可观察行为一致**，比较的内容包括进程读写、退出状态和文件结果；在它们定义的有效输入、系统调用和内存预算前提下，排除 Sanitizer 检测到的错误。

公开仓库的部分命题另有 `halts` 形式，证明执行结束并满足允许的退出状态。**只有一致性证明，不自动得到“所有输入都会终止”。** 同理，定理的量化范围也不是任意真实部署：普通构建的内存预算在相关 ASan 命题中为 **2 GiB**，UBSan 命题中为 **16 GiB**。超过前提的情况不在保证范围内。[Theorem README](https://github.com/theorem-labs/oss-verify-proofs)

## 2. Agent 可以反复试，验收器不能听 Agent 自己解释

在这套工作中，Claude 负责寻找证明路线、构造辅助引理、处理失败的证明目标、修改证明文件；Rocq 9.2.0 负责检查它最终提交的项能否具有固定定理要求的类型。

公开工具的 `oft check` 还会做几项关键检查：

- 核对每份被证明的二进制 SHA-256，避免证明指向另一份构建；
- 从 ELF 重新生成内存映像与定理，编译证明，再生成由检查器控制的包装模块；
- 使用 `Print Assumptions` 审计依赖的公理，只接受显式白名单；
- 拒绝 `Admitted` 等绕过证明的做法，CI 在 `--network none` 的容器中检查证明；
- 对 PR 使用基线提供的工具和待证陈述，防止补丁随意更改检查标准。

这才是“让 Agent 给证据”的工程版本：**结果不是一篇自称完成的报告，而是可以由另一台机器重新检查的证明工件。** 但它仍有信任根，包括 Rocq 内核、允许的公理、LPS 系统模型、ELF 映像生成器及检查脚本；这些组件不因为模型产出了证明就自动正确。[检查器细节](https://github.com/theorem-labs/oss-verify-proofs/blob/main/docs/checking.md)

## 3. 实际复核：不调用 Claude，也可以检查已公布的证明

Theorem 公开了证明、来源版本、构建记录和验证脚本。**截至 2026 年 10 月 29 日，本次读取的 `main` README 列出 26 个库、61 个入口的证明**；此前 10 月 22 日发布的研究总量仍是 123 个库、239 个入口。公开集合并非全部研究结果，部分涉及尚未修补的缺陷，团队没有释放相应证明。不能把研究总数写成“239 个入口均已公开可复验”。

选择公开的 `coturn/FuzzStunClient`，按仓库当前 README 可这样做：

```bash
git clone https://github.com/theorem-labs/oss-verify-proofs.git
cd oss-verify-proofs
./oft fetch coturn
python3 -m pip install --require-hashes --target "$HOME/oft-pydeps" \
  -r tools/oft/requirements.txt

docker run --rm --network none --ulimit stack=-1:-1 \
  -u "$(id -u):$(id -g)" -e HOME=/tmp -e PYTHONPATH=/pydeps \
  -v "$HOME/oft-pydeps:/pydeps:ro" -v "$PWD:/work" -w /work \
  ghcr.io/theorem-labs/oss-fuzz-theorems-checker@sha256:cc9ae43a9f4b0dd59d7c8d30727d36e039bfe72a2a9f9c4a99c68e413037e902 \
  ./oft check coturn/FuzzStunClient --parts statements,proofs
```

这不是轻量命令：需要 Docker、zstd、Python 依赖；官方说明单份证明在 **16 核**环境下可能需要 **最多约 20 GiB 内存**，也可通过限制 CPU 数控制内存。验证容器与关键依赖均应按具体版本或 digest 固定。本次**核查了公开源码和命令，但执行环境没有联网拉取镜像，也没有实际跑完验证**，所以这里不给出虚构的“本地通过”结果。

## 4. 证明过程中发现 Bug，但“修好后能证明”会带来新的风险

论文作者称，生成证明时发现了数百个缺陷，包括一个需要约 **20 MB 特定布局输入**才能触发的解压器越界写、另一个需要约 **64 MB 输入和超过 2 GB 内存**的图像解码错误，以及对 `libexpat` 的 **CVE-2026-56132** 的重新发现。这些是作者披露的案例；没有完整公开缺陷清单和逐个上游修复记录，不应将其写成“数百个漏洞均已修复并合入”。

Claude 有时先修改程序，再为修改后的二进制重新生成定理。这带来一个明显的捷径：**删掉处理逻辑、直接返回，也可能让“不崩溃”更容易成立。** 作者承认这种奖励作弊风险，并通过对补丁增加惩罚、记录和人工审计降低风险。

我会进一步提出一个实践要求（**工程建议，不是已验证的实验结论**）：将补丁验收和证明验收分开。前者用回归测试、功能契约和差分行为检查“是否保留预期功能”；后者检查限定的安全性质。**形式化证明不替代代码审查，更不替代规格审查。**

## 5. 成本：共享证明为何能省 Token

证明可以拆成多层辅助引理，也可以在不同项目中复用共同的运行时语义和初始化路径。原文报告，在 **36 个小型验证任务**上，Opus 5.5 完成其中一半任务所需的 Token 比 Opus 5 约少 **40%**；允许 Agent 借用其他 Agent 已完成证明时，Token 消耗比各自独立证明约低 **17%**。

这是同一研究设置内的成本结果，不是“所有 Claude Code 任务都降低 17%”。生成大型证明仍可能消耗数亿 Token；对于低价值、快速变化的业务逻辑，投入是否合算尚缺可复用的总成本比较。

## 6. 独立讨论：决定证明价值的是“证明了什么”

[Hacker News 的相关讨论](https://news.ycombinator.com/item?id=50202217)集中质疑一个核心问题：谁来确保规格与现实一致？还有开发者分享用模型辅助 Lean 验证 C/Rust 实现的经验，但那是其个人实践，不能作为 Anthropic 数据的复现。

Gautam Parab 在独立分析 [*239 Proved, 60 in Public, and the Statement Was a Fuzz Test*](https://gautamparab.com/writing/the-statement-was-a-fuzz-test/) 中做了有效的计数和规格核查：研究报告中的已证明入口、公开提供证明的入口，以及仅创建了待证陈述的入口，是三组不同的统计对象。公开数据可用于检查方法，却不能推出全部研究结果都已经第三方复验。

这也提示三个互不等价的判断：**Rocq 接受了证明；定理忠实表达了目标安全性质；目标构建等于线上运行的程序。** 其中任意一项缺失，都不能宣称生产环境已经安全。

## 可复用清单

- [ ] 先选择已有高质量 Fuzz 入口的解析器、协议库等明确边界模块，不从含糊的业务需求直接生成定理。
- [ ] 固定上游 commit、构建参数、编译器、libc、Sanitizer 与二进制哈希。
- [ ] 在允许模型写证明之前，由代码所有者复核定理文字、输入范围、错误定义、系统调用假设和内存前提。
- [ ] 证明源码与定理、检查脚本分权限管理；不得允许 Agent 悄悄放宽公理白名单、替换检验镜像或修改 CI 判定。
- [ ] 让机器执行 proof check，保存日志、产物版本与未证明分支；不要把 Agent 的完成声明当成验收。
- [ ] Agent 如需修补源码，独立执行行为回归与补丁审核，并为新构建重新生成和检查证明。
- [ ] 对照实际生产架构：x86/ARM、动态链接、多线程与真实 Linux 行为不被当前 RISC-V 静态模型自动覆盖。

## 什么时候不值得采用

缺少明确可形式化的性质、依赖大型并发运行时、代码频繁变更、验证资源成本高于潜在风险的项目，不应为了“形式化验证”这个标签而迁移。模型可能极擅长生成证明，但它不能替产品负责人定义正确需求，也不能让抽象机器模型天然等于生产环境。

真正值得借鉴的不是 **1.8 亿行**，而是一条可检验的责任分工：**Agent 负责产生候选证明，独立检查器决定它是否成立，人负责确定证明针对的是值得证明的东西。**

## 来源

1. Anthropic 与 Theorem，2026-10-22：[LLMs are making formal software verification feasible at a much larger scale](https://www.anthropic.com/research/formal-verification)（主文、实验条件与限制）。
2. Theorem：[公开证明与复验命令](https://github.com/theorem-labs/oss-verify-proofs)、[验证器与 CI 机制](https://github.com/theorem-labs/oss-verify-proofs/blob/main/docs/checking.md)、[机器语义的限制](https://github.com/theorem-labs/oss-verify-proofs/blob/main/docs/semantics.md)。
3. Hacker News：[技术社区讨论](https://news.ycombinator.com/item?id=50202217)（独立意见，不作为复现）。
4. Gautam Parab：[239 Proved, 60 in Public, and the Statement Was a Fuzz Test](https://gautamparab.com/writing/the-statement-was-a-fuzz-test/)（独立计数与规格解读）。

_核查说明：本文基于公开资料与代码阅读，没有使用私人历史记录。研究数字、代码当前状态和本文工程建议分别注明。没有运行完整验证，没有声称公开证明覆盖全部研究结果。_
