---
title: "让 Agent 自己验证“更好玩”：Battle Drones 的模拟对战与人工试玩"
date: 2026-11-07
source_url: "https://simonwillison.net/2026/Oct/20/battledrones/"
source_title: "Battle Drones—a vibe coded RTS with audio-based multiplayer"
source_date: 2026-10-20
author: "Simon Willison（原始实践）；中文审读与工程分析"
tools:
  - "Claude Code for Web"
  - "Claude Opus 5.5"
  - "Ultracode（原文运行模式）"
  - "TypeScript"
  - "Vitest"
  - "Playwright"
  - "vite-node"
evidence_level: "B：一手项目、公开历史提示词、已合并 PR 与可检查的测试代码；模拟指标未独立复跑，主观体验存在争议"
score: 89
score_breakdown:
  technical_depth: 18
  reusability: 19
  empirical_evidence: 18
  community_independent_checks: 11
  novelty_nonduplication: 14
  traceability: 9
caveats:
  - "1,200 场 AI 对战及指标来自项目发布说明，本文未独立复跑"
  - "节奏和逆风翻盘只是游戏体验的代理指标，不能证明玩家认为游戏更有趣"
  - "没有可信的总工时、人工返工和 token 成本对照"
  - "文中列举的测试命令基于核查时的公开 main，未认定它们与 10 月 7 日测试代码完全相同"
tags:
  - coding-agent
  - claude-code
  - agent-collaboration
  - iterative-development
  - simulation
  - playtesting
  - feedback-loops
  - user-experience
---

# 让 Agent 自己验证“更好玩”：Battle Drones 的模拟对战与人工试玩

**一句话结论：** 对「做得更好玩、更顺手、更自然」这类难以写成单元测试的需求，不必先由人把所有实现步骤和参数规定好。可以把体验目标交给能力足够的 Agent，让它提出方案、搭建可批量运行的反馈、做迭代；人保留对真实体验和关键设计选择的判断。**但别把代理指标的进步误认为用户满意。**

适用场景不止游戏：交互界面、搜索结果排序、复杂业务流程、推荐策略，都存在「功能正确，却不好用」的鸿沟。这篇文章精读 Simon Willison 于 **2026 年 10 月 20 日**发表的 [Battle Drones 实践记录](https://simonwillison.net/2026/Oct/20/battledrones/)，并核对其公开提示词、PR、代码和后续社区反例。文章并非本周新发布。

## 看起来像游戏，玩起来却无聊

Simon 想做一款手机也能玩的即时战略游戏：采集资源、建设基地、控制地面和空中单位。早期他用 Claude Code for Web 搭出了原型，随后补了精灵图、地形和不同地图。第三版已有相当完整的视觉效果，他自己试玩后仍很快感到无聊。

这是一种编码 Agent 容易制造的「假完成」：需求清单、界面截图和测试可能都成立，但用户真正期待的体验没有出现。

他没有继续逐项要求「把战斗时间改成 X」「多加三种单位」。升级到 Claude Opus 5.5 后，他在手机上给出的关键指令是：

> This game isn't fun enough yet. Make it a LOT more fun.

当时使用的是 Claude Code for Web 的 **Ultracode** 模式。作者报告，这个任务从晚上运行到次日上午才交出可玩的结果，并不是一句指令后瞬间生成成品。完整的历史提示词和对应提交，公开在 [web-session-prompts.md](https://github.com/simonw/battledrones/blob/main/web-session-prompts.md)。

最值得学的不是「Prompt 越短越好」，而是**此时人已经有真实产品和明确的不满，Agent 可以接触完整项目、改动代码并获得运行反馈**。脱离这些前提，换一个空仓库重复这句话，信息量远远不够。

## Agent 改的不是颜色，而是玩家的决策与反馈

10 月 7 日合并的 [PR #3](https://github.com/simonw/battledrones/pull/3) 覆盖 155 个文件；提交记录证实这不是单纯的展示稿。Agent 引入了几组会互相影响的机制：

- **开局有选择：** Blitz、Industry 和 Air Wing 的不同初始资源与兵种，让玩家一开始就有策略承诺。
- **地图中间有争夺：** 可占领的 Uplink、补给箱和可回收残骸，迫使双方走出基地，减少只在角落堆兵的局面。
- **落后者仍有机会：** 受损和损失可以更快积累战术技能资源，设计上给逆风方创造翻盘窗口。
- **战况更容易理解：** 战斗提示、音效、动态音乐、时间控制和快捷操作让玩家知道发生了什么。

这些不是「让界面热闹一点」的随机加法。一个合理的*工程解释*是：先创造有意义的选择和接触机会，再让玩家看见这些选择的后果。但原文没有做逐功能消融实验，因此**不能断言哪一项单独造成了改善**。

## 从“感觉不对”到能批量运行的反馈

更有迁移价值的是，Agent 不是改完 UI 就宣布胜利。作者说它运行了超过一千局自动模拟，并据此调整兵种平衡和比赛节奏。

Agent 撰写、公开存档的 [The Fun Update 发布说明](https://github.com/simonw/battledrones/blob/main/Battle%20Drones%20The%20Fun%20Update%20%E2%80%94%20Release%20Notes.md) 给出 **1,200 局普通难度 AI 对 AI** 的统计：

| 观察口径 | 调整前 | 调整后 |
| --- | ---: | ---: |
| 超过 25 分钟仍未分胜负的比赛比例 | 14% | 0.9% |
| 曾落后超过 800 scrap 的一方最终获胜比例 | 24% | 46% |

**数字说明比赛节奏和逆转机会变了，不说明玩家一定更快乐。** 也不能把「翻盘更多」直接认定为平衡更好：如果逆风补偿过强，它也可能惩罚前期优势。这些数字来自项目自己的发布说明，本次没有独立重跑 1,200 局，也没有查到可用于推断人工效率提升的对照工时。

公开源码为这种反馈提供了可以检查的载体。游戏把可无界面运行的模拟逻辑与 three.js 绘制层分开；[tools/balance.ts](https://github.com/simonw/battledrones/blob/main/tools/balance.ts) 含 AI 对 AI 的比赛、单位对决及参数覆盖入口，[tools/smoke.mjs](https://github.com/simonw/battledrones/blob/main/tools/smoke.mjs) 则用 Playwright 驱动触屏操作，测试选择开局、造建筑、操作单位与回放一致性。

以**核查时仓库 main 的工具接口**为例，读者可以用下面的方式探索同类反馈（并非作者当晚 1,200 场实验的原样复现命令）：

```bash
git clone https://github.com/simonw/battledrones.git
cd battledrones
npm ci
npm run build
npm test

# 无需图形界面的 AI 对 AI 平衡观察
npx vite-node tools/balance.ts matches riverlands

# 修改一个参数做本地对照；当前工具支持 --tune
npx vite-node tools/balance.ts matches riverlands \
  --tune '{"UNITS.trooper.hp":200}'
```

这里有两个容易犯的错。其一，**只跑一个种子、一个地图或一种 AI 性格，就宣称更好**；随机性和地图机制会扭曲结论。其二，自动模拟正常，不代表真实玩家用触控设备也舒服。浏览器冒烟测试依赖本地 Chromium 环境，不能假设所有机器执行 `npm run smoke` 都会直接成功。

## 人什么时候该少说，什么时候必须插手？

Simon 在早期版本不断试玩、指出不便，实际提示词里出现过「页面白屏」「点不了按钮」「手机缩放后回不来」「某种 AI 总赢」。他还要求 Agent 尽早推送可玩的构建，自己从线上页面而不只是代码报告判断进展。这些交互都留在 [公开会话记录](https://github.com/simonw/battledrones/blob/main/web-session-prompts.md) 中。

后来他发现，新加入的运输机和步兵在旧地图上缺乏战术价值。到了 Archipelago 版本，他不再说「增加一些岛」，而是定义了一个因果目标：**让玩家必须通过运输机把步兵送上岛，才能争夺关键据点**。在正式编码前，他还要求 Agent 先研究和批判这个设计，双方同意后再实现。

这个对比很实用：

- **方向明确、改法开放时**，少指定文件、类名、实现路径，让 Agent 提出能兑现目标的机制。
- **规则之间存在关键依赖时**，先明确不可破坏的约束，要求 Agent 解释方案、识别冲突，别把设计决定伪装成实现细节。
- **效果无法靠机器完全判定时**，让 Agent 提供规模化证据，人再试玩、比较和决定是否接受。

以上三条是从案例抽象的工作方法，不是经过随机对照试验证明的通用规律。

## 这套方法也有明显的反例

作者后来尝试两台手机用声音同步游戏状态，甚至做了 Radio Probe 等调试工具。技术上曾经可行，但真实使用中高频音效让家人头痛，后来改用 WebSocket 中继。**工具验证能回答数据是否传过去，却回答不了人在房间里听着难不难受。** 这是由物理体验推翻技术可行性的案例。

社区对「好玩」也没有形成共识。2026 年 10 月 28 日的 [Hacker News 讨论](https://news.ycombinator.com/item?id=50273753) 中，有实际试玩者认为游戏令人厌倦，也有人认为补上常见玩法不等于设计出好游戏；Simon 则强调自己确实玩得开心。双方反馈有参考价值，但都不是代表性用户研究。**代理指标、作者个人判断和独立用户评价必须并列呈现，不能互相冒充。**

成本同样无法被浪漫化：作者承认 Ultracode 运行时间长、token 消耗很大；完整的人力、计算费用与传统开发对照未公布。尤其是这类巨型改动，PR #3 一次覆盖 155 个文件，维护者需要额外考虑回滚范围、集成失败与未来维护成本。

## 给下一次 Agent 任务的可复用做法

当一个功能「逻辑上没坏，但用起来不对」，可以尝试下面这个**根据案例改写的委派模板；不是 Simon 的原始提示词**：

```text
目标：改进当前功能的真实使用体验，不预先指定实现方案。

先亲自运行现有版本，描述最影响用户的 2–3 个体验问题。
针对每个问题给出可验证的假设，不要只罗列新功能。
优先建立低成本、可反复运行的反馈方式：模拟、测试数据、
浏览器操作或前后对比；记录基线、条件、失败样本。
自行实现并迭代低风险改动；不要为了让指标好看而破坏原始目标。
交付：可用构建、实际操作路径、测试命令、改善与退化的证据、
仍需人判断的体验问题。
涉及产品规则、用户资产或不可逆变更时，先提交方案等待确认。
```

实际使用时只需检查四件事：**有没有真实可运行环境；反馈是否和用户目标相关；有没有尝试找反例；人是否真的体验过最终结果。** 其中任一项缺失，就不要因为 Agent 给了一张漂亮截图或一段「测试全绿」的总结而停止验收。

## 来源与核查边界

1. **原始实践：** Simon Willison，2026-10-20，[Battle Drones—a vibe coded RTS with audio-based multiplayer](https://simonwillison.net/2026/Oct/20/battledrones/)。
2. **原始任务与交互：** [web-session-prompts.md](https://github.com/simonw/battledrones/blob/main/web-session-prompts.md)，含会话链接和提交号。
3. **实际代码交付：** [PR #3，2026-10-07 合并](https://github.com/simonw/battledrones/pull/3)、[发布说明](https://github.com/simonw/battledrones/blob/main/Battle%20Drones%20The%20Fun%20Update%20%E2%80%94%20Release%20Notes.md)。
4. **可检验实现：** [平衡测试入口](https://github.com/simonw/battledrones/blob/main/tools/balance.ts)、[浏览器冒烟测试](https://github.com/simonw/battledrones/blob/main/tools/smoke.mjs)。
5. **独立质疑：** [Hacker News，2026-10-28 相关讨论](https://news.ycombinator.com/item?id=50273753)。
6. **延伸一手反思：** [The October inflection point for vibe game development，2026-10-28](https://simonwillison.net/2026/Oct/28/vibe-game-development/)。

**已核查文章、历史提示词、公开 PR、测试入口及社区反例；未在本次运行中编译游戏、重跑模拟或组织真人试玩。** 没有公开的统一人力基线，故本文没有计算「Agent 节省了多少工时」。
