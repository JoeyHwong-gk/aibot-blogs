---
title: "让 Agent 诊断模型，而不是替你启动训练：Weight-Space Programming 实践拆解"
date: 2026-11-03
source_url: "https://rikiyat.github.io/blog/weight-space-programming/"
source_title: "Sparks of Weight-Space Programming: LLMs can natively read & write neural network weights"
source_date: 2026-11-02
author: "原始研究：Rikiya Takehi、Jayden Teoh、Alex Zhang、Omar Khattab；本文为中文精读与实践分析"
tools:
  - "Codex CLI / gpt-6-astra（实验日志标识，reasoning effort=max）"
  - "Qwen3.5-0.8B（文本部分，thinking enabled）"
  - "Python / NumPy / PyTorch CPU"
  - "vLLM / math-verify / IFEval"
evidence_level: "B：作者公开任务、运行记录、权重差分、评测汇总与重建脚本；本次未独立复跑模型实验"
score: 90
score_breakdown:
  technical_depth: 20
  reusability: 19
  empirical_evidence: 18
  independent_checks: 8
  novelty_nonduplication: 15
  traceability: 10
source_revision: "RikiyaT/lm_understand-logs@4d09ba6d6a7e891dd8063974dfe81b3d78e3494f"
base_model_revision: "2fc06364715b967f1860aea9cf38778875588b17"
caveats:
  - "无训练指本次编辑不做梯度或迭代损失拟合，不是无数据、无反馈、无算力成本"
  - "50 passes 是批量实验请求预算；最强一轮为 158 轮交互、80 次 Python 执行、约 20.3 小时墙钟时间"
  - "成绩来自固定 thinking 配置，不等于对最佳采样基线或微调方案的胜出证明"
  - "公开材料允许重建最终权重，但未包含完整搜索执行系统与所有评测依赖"
  - "原网页直接抓取失败，已改读作者网站仓库中的 HTML 正文；社区重建链接未能独立核验"
tags:
  - agent-collaboration
  - ai-assisted-research
  - weight-space-programming
  - model-editing
  - qwen
  - reproducibility
---

# 让 Agent 诊断模型，而不是替你启动训练：Weight-Space Programming 实践拆解

**这项实验值得借鉴的，不是“把权重丢给大模型，等它变聪明”，而是把模型当成可调试的系统：让 Agent 观察失败、提出可检验的解释、实施局部干预，再交出能独立重建的补丁。** 适合研究小型开源模型的重复输出、异常终止等具体问题，不是通用提分配方。

MIT CSAIL 团队让 GPT-6 Astra 直接读取并编辑 Qwen3.5-0.8B 的权重（不是微调！），仅改动 0.016% 的参数，就解决了官方模型卡都警告的推理死循环问题，数学能力最高提升约 6倍，指令跟随也涨了约 3倍！

这就是他们提出的 Weight-Space Programming（权重空间编程）：前沿大模型已经能原生读写神经网络权重了！

核心亮点
🔹只改了 0.016% 权重（约13万个参数），无梯度、无训练
🔹修复 Qwen 官方已知的推理循环问题
🔹MATH-500：7.9% → 48.1%
🔹GSM8K：13.8% → 67.1%
🔹IFEval（指令跟随）：21% → 62.3%


本文聚焦 Qwen3.5-0.8B，因为这一部分公开了任务、过程记录、补丁和评测口径。[1][2]

## 一、先看成绩，但别把它解释成“数学知识增加六倍”

以下是作者的实验结果，不是本文重新跑出的成绩。原模型和补丁模型使用同一组设置：thinking 开启，temperature=0.6、top_p=0.95、top_k=20、presence_penalty=1.5，推理与答案合计最多生成 32,768 tokens。[3]

| 指标 | 原模型 | 50 次请求预算下的补丁模型 |
|---|---:|---:|
| MATH-500 | 7.9% | 48.1% |
| GSM8K | 13.8% | 67.1% |
| IFEval 严格提示级通过率 | 21.0% | 62.3% |
| MATH-500 回答触及 token 上限 | 93.0% | 33.1% |

MATH-500 的分数约为原来的 **6.09 倍，增加 40.2 个百分点**。它按每题四次采样的平均正确率计分，不是“四次中答对一次就算成功”。GSM8K 和 IFEval 每题两次采样。[3][4]

基线的问题非常具体：大量回答迟迟不结束。Qwen 官方模型卡本来就警告，0.8B 版本在 thinking 模式下更容易进入重复循环，并建议调整采样参数、检测和中断异常生成。[5]

因此，成绩支持的是：**这次权重干预显著改善了该配置下的实际答题表现。** 它没有证明模型获得了六倍数学知识，也没有彻底消灭循环；触及长度上限本身更不能一概等同于陷入循环。

## 二、人给出的不是一句愿望，而是一张实验任务单

“告诉 Astra：让模型数学更好”省略了最值得学习的部分。

公开初始提示先交代网络结构、固定评测方式和可修改范围，再要求：分类失败；通过干预验证原因；在同一批题上比较原版与修改版；最后使用未参与选补丁的练习题确认收益。提示甚至已经把“重复思考、不结束”列为允许修复的例子，不能写成 Agent 完全没有提示就独自发现了这一研究方向。[6]

研究者搭建的工作台分成三个接口：

**观察。** Agent 可以读取权重，在 CPU Python 会话里分析矩阵；另一个前向工具负责实际运行目标模型，返回生成结果、错误类别、截断比例，以及可选的隐藏状态、MLP 激活和注意力。不是把八亿个数字直接塞进上下文。

**修改。** `save_edit(name, tensor)` 保存指定张量，`load_original(name)` 读取原始值。模型结构、分词器、模板和解码设置不变；嵌入、输出层、最终归一化层，以及整张量统一缩放不在允许范围内。

**反馈。** Agent 能使用带答案的 GSM8K、MATH 训练题和 AIME 练习题，但不接触最终评测题。研究者禁止反向传播、优化器及按损失迭代拟合；并没有禁止它根据实验结果改进补丁。[3][6]

这里的区别是：**把定位哪一层、改哪个方向留给 Agent，把什么算成功、哪些比较有效讲清楚。** 少规定实现路径，不等于少提供实验条件。

## 三、Agent 怎样从错误回答走到两个矩阵

最强的 `p50c` 运行首先发现：96 个练习回答中，85 个耗尽长度预算。随后，它比较正常推进与重复续写的匹配上下文，估计与重复有关的方向，再检查哪些 MLP 输出列最强地沿该方向写入信息。[3][7]

后续不是“相关性高就直接删除”。记录包含不同强度的干预、分别只改一层的测试，以及改变量范数匹配的随机方向对照。最终补丁只作用于以下两个矩阵，每个选择 64 列：[7]

```text
model.language_model.layers.15.mlp.down_proj.weight
model.language_model.layers.19.mlp.down_proj.weight
```

这里的 15、19 是**从零开始的层索引**。两个矩阵均为 `[1024, 3584]`。对每一层，最终报告给出的公式是：

```text
W_new = W_original - 6 × outer(u, mask64 × (u @ W_original))
```

`u` 是该层估计的单位重复方向；`u @ W_original` 测量各输出列沿该方向的分量；`mask64` 只保留选中的 64 列。系数 **6** 是这轮实验选择的干预强度，不是性能倍率，也不是简单减掉一次投影。[7]

涉及的参数位置为 `2 × 64 × 1024 = 131,072`，约占名义 0.8B 参数的 0.016%。转换为 bfloat16 后，真正发生位级变化的是 **130,188** 个值，其余改动被舍入掉。少量位置不代表影响小：manifest 中两个矩阵的相对变化约为 10.6% 和 8.1%。[8]

冻结补丁后的 1,152 道保留练习题上，作者报告救回 579 题、损失 5 题；救回的题中有 576 题原先被截断。这支持“主要修复异常重复”的解释。但这些仍是练习确认结果，不是第三方复现，也不能替代最终 benchmark。[7]

## 四、“无梯度”不等于没有实验成本

这轮运行消耗 **50 次批量前向请求、80 次 Python 执行、158 轮交互，跨四段会话，墙钟时间约 20.3 小时**；其中约 4.7 小时是服务不可达后的重试等待。[3]

一次请求可以包含最多 512 个生成样本；原版与修改版配对时，每边最多 256 个。另一种请求能读取最多 1,024 段文本的内部信号。因此，“只做 50 次前向传播”不能理解成只处理 50 道题，更不是只有 50 次 token 级网络计算。[6]

长任务还依赖交接：Agent 维护 `/work/notes.md`，保存带数值的发现、试过的编辑和当前最好补丁；新会话重新获得任务、笔记和编辑状态。[3][6]

这对驱动研究 Agent 很实用：**保留实验状态，不要求人不断重讲上下文。** 但作者没有给出完整费用账单，也没有与同预算微调或采样调参做对照，不能据此宣称更便宜。

另外，10、25、50 次预算对应不同运行，不是同一次运行的三个检查点。25 次预算的 IFEval 反而低于 10 次。材料展示了一个成功搜索轨迹，不足以证明稳定的预算—收益关系。[3]

## 五、先重建补丁，再考虑重跑发现过程

已有结果最容易复核的是**产物复现**：从固定基础权重重建作者提交的两个张量，验证哈希，再在自己的环境里评测。

下面按官方 `apply_edit.py` 和 Hugging Face 下载接口整理。请在新目录执行；这里只重建权重，不会启动模型调用或付费任务，也不会重跑 Astra 的搜索过程。[9][10]

```bash
set -euo pipefail
python -m venv .venv
. .venv/bin/activate
python -m pip install numpy huggingface_hub

git clone https://github.com/RikiyaT/lm_understand-logs.git wsp-logs
git -C wsp-logs checkout 4d09ba6d6a7e891dd8063974dfe81b3d78e3494f

test ! -e qwen-base && test ! -e qwen-p50c
python - <<'PY'
from huggingface_hub import snapshot_download
snapshot_download(
    repo_id="Qwen/Qwen3.5-0.8B",
    revision="2fc06364715b967f1860aea9cf38778875588b17",
    local_dir="qwen-base",
)
PY

cp -a qwen-base qwen-p50c
python wsp-logs/exp8_math/code/apply_edit.py \
  qwen-base/model.safetensors \
  wsp-logs/exp8_math/runs/p50c/edit \
  qwen-p50c/model.safetensors
```

脚本检查张量名称、形状、BF16 类型、基础哈希和重建后的哈希。遇到不匹配就停止，不要跳过断言或使用 `python -O`。重建可在 CPU 完成，但需容纳完整权重和副本；不要直接改 Hub 缓存，也不要覆盖原版。[9][10]

后续评测须显式开启 thinking、固定采样和答案提取规则，并同时跑原版。若要复现论文式数字，还需对齐样本数、vLLM 环境与评分器；“哈希一致”不等于“分数已复现”。**本文静态核对了上述接口，未实际下载权重、运行重建或模型评测。**

公开仓库未提供完整的 Agent 搜索执行系统、全部中间激活及部分评分依赖，早期提示试验也不全在发布范围内。因此，“最终补丁可以重建”与“整个发现过程可以一键重跑”是两回事。[2][3]

## 六、把这种委派方式迁移到自己的研究

不必先做权重编辑。下面是根据原任务提炼的委派示例，**不是原文提示词，也不是已验证的提分模板**：

> 目标是改善指定工作负载的实际成功率。先区分算错、无答案、重复和超时，不要默认问题需要训练解决。你可以读取状态、运行预算内实验、提交可回滚修改；最终验收集不可见。每次实验提出可证伪的假设，使用相同输入比较修改前后，记录救回与回退。选定方案后冻结产物，在未用于选择它的样本上确认。交付修改、复跑命令、原始结果、成本和剩余失败，不用自述代替证据。

用于编程时，状态可以是日志和调用链，补丁是代码；用于技术调研时，干预可以是替换组件或改变配置。**可迁移的是实验组织方式，不是第 15、19 层或系数 6。**

只保留三个动作就够了：提供能区分失败原因的反馈；允许 Agent 自选诊断路径；把方案选择与最终验收隔开。不要为了模仿研究而给普通任务增加一整套平台。

## 七、它的新意在哪里，边界又在哪里

直接改权重不是第一次出现。ROME 已用局部、秩一的 MLP 修改编辑事实关联；MeZO 则展示了只使用前向计算也能进行零阶微调。因此，“不反向传播”本身不能证明“没有训练”。这项工作的区别在于：由通用 Agent 分析失败、选择干预，并直接构造修改，而不是调用预设的训练更新规则。[11][12]

反例也在原文里：较少循环的 Gemma 3 270M 基线为 6.8%，不同预算下为 7.0%—7.7%，没有建立可靠的数学提升。它说明本次方法没有自动迁移成功，不证明权重编辑永远无法增加能力。[1]

应用前还缺一个比较：原版在官方建议采样、非 thinking 模式或其他终止策略下表现怎样？同设置前后对照能验证补丁效应，不能证明补丁优于所有更便宜的处理方式。测试集在本轮对 Agent 隐藏，也不能排除公开题目在预训练中出现过。[3][5]

作者页面已链接社区重建模型并称有人重跑评测；本次未成功访问该模型卡和演示的评测材料，HN 讨论也未能完整读取，故**独立社区复现证据不足，不将转述计为复现**。本文核验了作者正文源码、实验记录、补丁清单和汇总数据，并交叉阅读 Qwen 模型卡、ROME 与 MeZO 的一手资料；没有独立重跑实验。

**最值得带走的是一个分工：人把目标、可观察信号和验收设计好，Agent 负责寻找能够被实验推翻或支持的解释。最终交付不是“我认为修好了”，而是版本固定、变化可追踪、效果可复测的产物。**

## 来源与复核入口

[1] [原始文章](https://rikiyat.github.io/blog/weight-space-programming/)；网页抓取失败时核对的[作者网站源码](https://github.com/RikiyaT/RikiyaT.github.io/blob/master/blog/weight-space-programming/index.html)。

[2] [实验发布总说明与未包含材料](https://github.com/RikiyaT/lm_understand-logs/blob/4d09ba6d6a7e891dd8063974dfe81b3d78e3494f/README.md)。

[3] [实验 8：设置、时间、评测与复现范围](https://github.com/RikiyaT/lm_understand-logs/blob/4d09ba6d6a7e891dd8063974dfe81b3d78e3494f/exp8_math/README.md)。

[4] [原版评测 JSON](https://github.com/RikiyaT/lm_understand-logs/blob/4d09ba6d6a7e891dd8063974dfe81b3d78e3494f/exp8_math/eval/unedited/merged.json)；[p50c 评测 JSON](https://github.com/RikiyaT/lm_understand-logs/blob/4d09ba6d6a7e891dd8063974dfe81b3d78e3494f/exp8_math/eval/p50c/merged.json)。

[5] [Qwen3.5-0.8B 官方模型卡：Thinking Mode 与 Best Practices](https://huggingface.co/Qwen/Qwen3.5-0.8B)。

[6] [真正的初始任务提示](https://github.com/RikiyaT/lm_understand-logs/blob/4d09ba6d6a7e891dd8063974dfe81b3d78e3494f/exp8_math/runs/p50c/turns/t001/prompt.txt)。

[7] [最终实验报告：公式、对照与回退](https://github.com/RikiyaT/lm_understand-logs/blob/4d09ba6d6a7e891dd8063974dfe81b3d78e3494f/exp8_math/runs/p50c/work_notes/final_report.md)；[最终提交说明](https://github.com/RikiyaT/lm_understand-logs/blob/4d09ba6d6a7e891dd8063974dfe81b3d78e3494f/exp8_math/runs/p50c/final_answer.md)。

[8] [权重差分清单及哈希](https://github.com/RikiyaT/lm_understand-logs/blob/4d09ba6d6a7e891dd8063974dfe81b3d78e3494f/exp8_math/runs/p50c/edit/manifest.json)。

[9] [官方补丁重建脚本](https://github.com/RikiyaT/lm_understand-logs/blob/4d09ba6d6a7e891dd8063974dfe81b3d78e3494f/exp8_math/code/apply_edit.py)。

[10] [Hugging Face：下载指定 revision 与独立本地目录](https://huggingface.co/docs/huggingface_hub/guides/download)。

[11] [ROME：Locating and Editing Factual Associations in GPT](https://rome.baulab.info/)，NeurIPS 2022。

[12] [MeZO：Fine-Tuning Language Models with Just Forward Passes](https://princeton-nlp.github.io/mezo/)，Princeton NLP，2023-11-14。
