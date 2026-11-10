---
title: "别再向 Agent 口述故障：给它接上真实环境的反馈"
date: 2026-11-11
source_url: "https://simonwillison.net/2026/Nov/9/mysterious-beeps/"
source_title: "Locating the source of a mysterious beep with Claude"
tools:
  - "Claude Code Cloud / Claude Opus 5.5 (Extra thinking)"
  - "Claude Code / Claude Fable 5.2"
  - "Python / WebAudio / WebSockets / pytest"
evidence_level: B
score: 90
caveats:
  - "约40厘米定位效果由作者在一次真实家庭场景中报告，不是多场景统计精度"
  - "没有人工总耗时的对照；31.04美元仅是后段会话的API等价估算"
  - "有公开源码与模拟测试，但未独立复跑，部分真实设备功能未覆盖自动化测试"
tags: ["Agent协作", "Claude Code", "真实环境调试", "反馈循环", "可执行证据"]
---

# 别再向 Agent 口述故障：给它接上真实环境的反馈

**一句话结论：** 当 Agent 的任务依赖真实世界或运行中软件的状态，与其让人反复转述「刚才又失败了」，不如先让 Agent 建立能持续观察、复现、试改、回看的反馈通道。人负责提供设备、纠正感知和确认最终结果，Agent 才能真正独立迭代。

这不是一篇「如何制造智能麦克风阵列」的教程，而是 Simon Willison 在 2026 年 11 月 9 日公开的一次 Claude Code 协作实录：他让 Agent 调查储物间里隔几十秒才响一次的蜂鸣，最后找到了一个电池耗尽的老式一氧化碳报警器。[原文](https://simonwillison.net/2026/Nov/9/mysterious-beeps/)附有 [完整项目](https://github.com/simonw/beep-locator)、历史提交和实际提示词。

## 麻烦不在编程，而在 Agent 听不见

储物间塞满未拆封的纸箱。声音短、间隔长、反射严重。传统排查要么搬开箱子逐个听，要么由人蹲在房间里等待蜂鸣，再把听觉印象告诉程序员。

如果让 Agent「写一个定位蜂鸣的网页」，它可以很快产出程序，却无法知道麦克风是否工作、浏览器是否抵消回声、算法是否把反射音当成直达音。人仍得守着五台设备，反复刷新网页、截图、解释报错。这里浪费的不是模型的代码生成时间，而是**人充当传感器、测试员和日志转发器的时间**。

Willison 的第一步甚至不是写代码。他在 iPhone 上用 Claude Code Cloud（Opus 5.5，Extra thinking）要求先写 problem.md：房间里有未知蜂鸣源；用笔记本、手机和 iPad 作监听器；Python 网页协调；浏览器通过 WebAudio 和 WebSocket 连接；最好不必手工测量设备位置，并请 Agent 调查这种自动标定能否成立。

Agent 找到了 2007 年的 [BeepBeep 论文](https://www.cs.purdue.edu/homes/chunyi/pubs/sensys106-beepbeep.pdf)：设备互相播放短促扫频声，在各自录音里测时间间隔，借双向测量抵消跨设备时钟偏差和一部分音频延迟。作者由此有了可实施的方案，而不是凭空假设几部手机的时钟足够同步。

## 真正改变协作的，是第二次委派

Willison 接着让 Agent 开始实现，并加上一项明确要求：**让服务端能把新代码发送到监听设备并直接运行**。他的目的说得直白——把设备摆好，自己可以去做别的事，Agent 继续实验。

Opus 5.5 做出了初版，但**当时还没在真实设备上运行过**。这条边界很关键：生成一个看似完整的项目，离解决故障还差一次环境闭环。

他回到电脑，换用 Claude Code 和 Fable 5.2，启动服务，让它能读取真实事件、检索音频记录、下发新代码。项目将浏览器音频采集与 WebSocket 连接保留在相对稳定的 runtime 中；探测器、页面组件可以通过 WebSocket 热替换。某些设备重载页面后要重新点击授权麦克风，热替换避免了 Agent 每改一次代码，人就跑去点五次「开始监听」。

这次迭代里，人并没有消失，但从执行操作转向提出反例：

- **自动校准不准。** 作者让 Agent 加 Rough mode：人先把设备图标拖到大概位置，算法再根据声学测量修正，而不是为了坚持「全自动」继续空转。
- **地图上的点可信么？** 作者让它识别真实拍手声，用与蜂鸣不同的图形标记；又要求每个点有时分秒和逐台设备的录音回放。拍手成为可控的端到端测试，回放让人能复核事件分类。
- **找到的是目标声么？** 作者要求「This is the mystery beep」按钮。人确认一段录音后，系统才用它的频率、重复间隔和后续多次测量追踪同一声源。
- **要移动监听器怎么办？** 作者让 Agent 保留旧测量会话、重新粗定位，不用断开或重新授权；还请它建议把一台空闲设备放在哪儿能更好收窄范围。

这些请求都不是「第 15 行代码怎么改」。人只告诉 Agent **哪条现场证据不可置信、下一次应该如何验证**，具体实现交回 Agent。

## 它究竟看见了什么？

公开代码说明了这套反馈的颗粒度。监听端保留约 60 秒的滚动音频，用高通滤波和 FFT 检出候选突发声；服务端收到事件后选设备播放同步短音，要求各监听器上传蜂鸣前后的一段录音，再对齐音频并计算到达时间差，在地图上估计声源。

对 Agent 来说，反馈不是聊天窗口里的「还是不行」，而是能查询的 **设备状态、事件时间戳、录音片段、定位误差、模块加载错误**。源码中的 [CLAUDE.md](https://github.com/simonw/beep-locator/blob/main/CLAUDE.md) 还直接列出了怎么自检、校准、触发测试蜂鸣，提醒修改稳定 runtime 将需要人重新启动监听。

例如项目 README 公开了这些入口（需要事先安装 uv 和 cloudflared，并在自己的设备上授予麦克风权限）：

~~~bash
uv run beep-locator serve --tunnel
uv run beep-locator ctl state
uv run beep-locator ctl run selftest
uv run beep-locator ctl run calibrate
uv run beep-locator ctl run testbeep device=Blue
uv run beep-locator ctl events --follow
uv run pytest
~~~

同时有虚拟房间页面 /sim?n=5 和 [合成音频测试](https://github.com/simonw/beep-locator/blob/main/tests/test_dsp.py)，模拟不同采样率、设备时钟偏移、音频延迟和回声。测试代码确实包含定位误差断言；**这不等于所有浏览器、房间和硬件都通过了现场测试**。更有说服力的是 [真实设备修复提交](https://github.com/simonw/beep-locator/commit/d5b51485d697d74968073dac7bf657d63d422449)：实际回声让早期校准在不同峰值间跳动，作者与 Agent 改了首达声识别阈值、增加标定轮次，并引入人工粗定位。提交也承认，神秘蜂鸣匹配等部分行为尚无自动化测试。

## 验收不是漂亮的定位界面

作者最终报告，系统把目标收敛到**约 40 厘米范围**，他从对应纸箱堆里找出了旧一氧化碳报警器。这是一次真实世界的结果验证，而非仅有「程序已运行」的答复。公开项目提交和测试使操作路径可追溯；但约 40 厘米是作者的一次场景报告，不应当作普遍精度保证。

费用也得算完整。在 [Hacker News 讨论](https://news.ycombinator.com/item?id=50434310)中，Willison 称后半段 Fable 5.2 本机会话按 API 价格折算约 **31.04 美元**，前期 Opus 5.5 Cloud 阶段没有可靠估算；他使用的是每月 200 美元订阅。**没有总人工工时、等待时间或与「逐箱翻找」的对照**，所以不能宣称这一方案省下了多少小时。社区确实有人质疑：直接拆箱可能更省事。作者的回应是，他想解决反复出现的这一类间歇声源问题，而不止找到这一次蜂鸣。

从第一性原理看，真正值得迁移的不是定位算法，而是：**模型再聪明，也无法凭猜测补足缺失的观察；人如果每轮帮它观察，Agent 便永远无法自循环。** 把观察、可重复的刺激、变更部署和结果回读交给工具，Agent 才有条件自主提出假设并排除反例。

当然，这不意味着把所有权限都交给它。此项目的远程 JavaScript 执行及可选 --auto-update 会让新代码直接进入监听设备；只应在自己控制的环境启用，保护管理令牌，真实危险操作仍由人确认。

## 明天可以照着做的四件事

1. **先描述现场故障，不先规定实现。** 明确现象、可用设备、不可接受的人工动作，请 Agent 先检查可行性并形成 problem.md。
2. **把一次性观测变成可查询反馈。** 对 UI 是浏览器和截图，对服务是日志/指标，对设备是录音与状态；要求 Agent 自己能读。
3. **先设计一个会失败的测试动作。** 像拍手或测试蜂鸣那样，让 Agent 看到输入、预测、实际输出和偏差；不要用「看起来好了」代替验证。
4. **只在人掌握独有证据时介入。** 如粗略摆放设备、确认是哪一种声音、最后打开箱子。让 Agent 承担其余反复试验，并保存可回放证据。

这次实践里，最有价值的提示词不是「用最好的算法」，而是「**让我布置好环境，你就能自己继续实验**」。

## 来源与核查

- 主文：Simon Willison，[Locating the source of a mysterious beep with Claude](https://simonwillison.net/2026/Nov/9/mysterious-beeps/)，2026-11-09；作者公开了主要提示词和人工介入过程。
- 代码：[simonw/beep-locator](https://github.com/simonw/beep-locator)、[初始问题定义](https://github.com/simonw/beep-locator/commit/d270b4f05a2c7cd453cd297077bd4764ea8aa56f)、[首版实现](https://github.com/simonw/beep-locator/commit/44fea65c918fb31a7d2c0e17797ee1a106860e23)、[真实设备修复](https://github.com/simonw/beep-locator/commit/d5b51485d697d74968073dac7bf657d63d422449)、[测试](https://github.com/simonw/beep-locator/blob/main/tests/test_dsp.py)。
- 原理：Peng 等，[BeepBeep: A High Accuracy Acoustic Ranging System Using COTS Mobile Devices](https://www.cs.purdue.edu/homes/chunyi/pubs/sensys106-beepbeep.pdf)，SenSys 2007；证明测距思路的既有依据，不是该项目精度的独立复测。
- 独立讨论：[Hacker News](https://news.ycombinator.com/item?id=50434310)，包含「直接拆箱可能更省事」的反例与作者的费用补充。未发现独立用户对同一套硬件流程的完整复跑报告。
