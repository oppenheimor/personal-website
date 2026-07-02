---
title: 用 Hermes 搭一个会自我迭代的内容创作 Agent 团队
date: 2026-07-02T00:00:00.000+00:00
lang: zh
duration: 22min
author: 沈佳棋
---

这篇文章记录一次真实的搭建过程：我用 Hermes 组织了一个内容创作 Agent 团队，服务微信公众号、微信视频号、抖音和小红书，定位是 AI 技术讲解。

一开始我以为重点是“创建一批角色”。后来发现，角色只是最表层的东西。真正要搭的是一个内容工厂 Harness：素材从哪里来，谁负责判断，谁负责生产，谁负责合规，结果怎么收数，失败经验怎么进入下一轮。

换句话说，不是让 AI 写一篇文章，而是让一组 Agent 像一个小编辑部一样，能持续产出、交接、复盘和变强。

这次我们最后做到了三件事：

1. 搭出 16 个 Hermes profile 组成的内容团队。
2. 跑通一次真实 Hermes Kanban 生产，CYCLE-002 完成 16/16 个任务，产出 22 个文件。
3. 在此基础上补齐素材库、武器库、gbrain 记忆包、Kanban 模板、发布前检查和小红书图文实验。

它还不是一个可以全自动发号施令的账号运营机器。更准确地说，它是一个已经有骨架、有流水线、有审计记录的内容工作台。

## 背景

目标很直接：做一组 AI 技术讲解账号，主战场是微信公众号、视频号、抖音、小红书。原始目标定得很激进，三个月内每个账号都冲 10 万粉丝。

这个目标听上去很热血，但只靠手搓肯定不现实。每天追热点、找资料、写脚本、适配平台、做图、做视频、查版权、复盘数据，人会很快被榨干。

所以我先定了几条约束：

1. 四个平台都是新账号，前期先以稳定产出和测试为主。
2. 暂时不真人出镜，也不用我的声音。
3. 不自动发布，所有产物先落到本地文件夹。
4. 可以用 AI 生成图片、视频和 BGM，但版权和来源要记录。
5. 产物目录统一成 `日期 - 平台 - 产物`。

这些约束听起来不性感，但很关键。没有边界的 Agent 系统很容易变成“看起来很忙”，最后留下一堆聊天记录，真正能交付的文件却没几个。

## 思路

我最开始把系统拆成四层：

1. 账号策略：讲什么，给谁看，三个月分几个阶段推进。
2. 角色组织：谁做定位，谁找素材，谁写脚本，谁做平台适配，谁做合规。
3. 技能系统：每个角色不只靠人设 prompt，而是有自己的 SOP 和工具。
4. 自动化运行时：runner 生成产物，verifier 检查产物，Hermes cron 定时触发。

后面继续梳理时，我发现四层还不够。一个真正能长期跑的内容工厂，至少还要补五层：

1. 素材库：竞品账号、信息源、日常工作记录、灵感、用户反馈，都要有结构化入口。
2. 武器库：技能、MCP、CLI、API、框架和参考仓库，要知道哪个角色该用什么。
3. 记忆偏好层：用 gbrain 记录我的修改意见、偏好、历史决策和发布结果。
4. 治理层：事实、版权、平台规则、AI 披露、人工审批，不能靠临时想起来。
5. 实验层：发布、收数、归因、更新记忆，形成最小闭环。

这套结构后来被整理成一个 Harness：

```text
素材库决定系统能看见什么。
gbrain 让系统越来越像我。
武器库决定角色能做什么。
Hermes Kanban 决定谁在什么时候做什么。
治理层决定什么能离开工厂。
实验层决定下一轮怎么变好。
```

这句话基本就是整个项目的核心。

## 先把团队建出来

我先在 Hermes 里创建了一个 board：

```bash
ai-tech-content-studio
```

然后创建 16 个 profile。Hermes dashboard 里能看到这些角色，它们不是随便起的机器人名字，而是按内容生产链路拆出来的岗位。

<img src="/hermes_content_workflow/profiles.jpg" class="medium-zoom-image" />

这 16 个角色分别是：

- `aichief`：总编制片人，负责拆任务、把控目标和最终验收。
- `posimind`：定位与阶段策划，负责账号方向和阶段目标。
- `trendfox`：竞品和趋势侦察，负责找机会。
- `sourcebee`：素材与事实库，负责证据和资料。
- `ideaforge`：选题灵感，负责把信号变成选题。
- `scriptfeyn`：AI 技术讲解脚本，负责解释骨架。
- `wechatmo`：微信公众号主笔。
- `videomax`：微信视频号导演。
- `douyineon`：抖音增长导演。
- `xhshana`：小红书策展人。
- `visualmira`：图片素材和视觉叙事。
- `soundmuse`：BGM 和声音设计。
- `editkai`：自动剪辑与后期。
- `opsmatrix`：排期、本地落盘和打包。
- `dataloop`：数据分析和复盘。
- `guardrail`：事实、版权和平台合规。

这里最容易犯的错，是把角色做成“全能型 Agent”。我刻意避开了这个方向。

内容生产里的上下文很脏。趋势、事实、脚本、视觉、版权、平台规则混在一起，最后谁都说不清自己负责什么。把角色拆细，不是为了热闹，而是为了让每个交付物都有明确 owner。

## 用 Kanban 串起协作

角色有了以后，下一步不是让它们开始聊天，而是把任务流放进 Kanban。

<img src="/hermes_content_workflow/kanban.jpg" class="medium-zoom-image" />

一个内容任务不是从“写文章”开始的。它应该从定位、趋势、事实、选题、脚本一路流到平台适配、视觉、声音、剪辑、合规和复盘。

这套 DAG 大概长这样：

```text
aichief
  -> posimind
  -> trendfox
  -> sourcebee
  -> ideaforge
  -> scriptfeyn
  -> wechatmo / videomax / douyineon / xhshana
  -> visualmira / soundmuse / editkai
  -> guardrail
  -> opsmatrix
  -> dataloop
```

Kanban 的价值不是“有个看板真好看”，而是它能表达依赖关系：

- 趋势没有跑完，素材整理不应该开工。
- 素材和事实没有完成，脚本不应该开始。
- 脚本骨架完成后，四个平台可以并行适配。
- 合规没过，发布就不能进入下一步。

这比把 16 个 Agent 拉进一个群聊靠谱得多。

## 给角色配 skill

角色只是 who，skill 才是 how。

我先做了一轮 skill 调研，复用了已有 Hermes skills，比如：

- `web-access`：当前网页调研和来源验证。
- `humanizer-zh`：中文文案去 AI 味。
- `video-pipeline`：视频生成和剪辑链路参考。
- `byted-ark-seedream-skill`：图片生成。
- `byted-ark-seedance-skill`：视频生成。
- `xhs-viral-post`：小红书标题、正文和封面策略。

但这些只是工具箱，还不是这个团队自己的工作习惯。所以我补了两类 skill：

```bash
~/.hermes/skills/
~/.hermes/profiles/<role>/skills/
```

共享 skill 放全局，比如本地落盘协议、平台打包、最终文案润色。

角色专属 skill 放 profile 目录，比如 `ai-tech-signal-scout` 给 `trendfox`，`content-rights-and-safety-gate` 给 `guardrail`。

这个边界很小，但很重要。全局 skill 越堆越多，每个 Agent 都背着一整个杂货铺出门，最后反而不知道该用什么。

## 先跑通 dry-run

为了不让 Agent 只产出一段聊天记录，我先定义了本地产物协议：

```text
YYYY-MM-DD - 平台 - 产物/
  manifest.json
  source-pack.md
  copy.md
  script.md
  visual-brief.md
  audio-brief.md
  edit-plan.md
  compliance.md
  metrics-template.csv
  run-report.md
  assets/
```

其中最重要的是 `manifest.json`。它记录主题、平台、角色状态、产物路径、质量门禁和缺失输入。

有了 manifest，后面才能做校验、复盘和自动化。没有 manifest，就只能靠人肉翻文件夹。

我写了几个本地脚本：

```bash
tools/install_content_studio_skills.py
tools/github_signal_scan.py
tools/run_content_workflow.py
tools/verify_content_run.py
tools/hermes_cron_content_studio_daily.py
```

`run_content_workflow.py` 负责生成内容包。

`verify_content_run.py` 负责检查：

- 产物目录是否存在。
- 必要文件是否齐全。
- `manifest.json` 是否能解析。
- 16 个角色是否都有状态。
- 四个平台的内容包是否都生成。
- no-auto-publish 边界是否存在。
- shared skills 和 role skills 是否装在正确位置。

然后接上 Hermes cron：

```text
Job ID: c6bdb905a227
Name: ai-tech-content-daily-dry-run
Schedule: 30 9 * * *
Script: content_studio_daily_dry_run.py
Mode: no-agent
```

手动触发方式：

```bash
hermes cron run c6bdb905a227 --accept-hooks
hermes cron tick --accept-hooks
```

那一轮 dry-run 的 verifier 结果是：

```json
{"passed": 46, "failed": 0}
```

到这里，系统已经能定时生成本地内容包。但我很快意识到，dry-run 只是合同测试，不是真正的生产。

## 再跑真实 Kanban 生产

后面我们又跑了一轮真实 Hermes Kanban 生产，编号 CYCLE-002。

这次不是本地脚本模拟，而是让 Hermes Kanban 里的角色按任务依赖完成交接。最终结果是：

```text
CYCLE-002 tasks: 16/16 done
Files: 22/22 present and non-empty
Manifest/disk alignment: exact
Compliance: PASS_WITH_LIMITATIONS, 0 fatal blockers
No auto publish: true
```

这轮产物包括：

- `mission-brief.md`
- `strategy-brief.md`
- `trend-scan.md`
- `source-pack.md`
- `topic-bank.md`
- `explanation-spine.md`
- `script.md`
- 四个平台的 copy
- 视觉、音频、剪辑 brief
- `compliance.md`
- `manifest.json`
- `metrics-template.csv`
- `run-report.md`
- `postmortem.md`

这一步的意义很大。它证明这个系统不只是能“生成文件”，而是能让多个 profile 通过 Kanban 完成角色间 handoff。

不过它也暴露了更关键的问题：我们已经能产出内容包，但还没有发布，没有数据，也就没法归因。

这就是所谓的零数据死循环。

## 升级成内容工厂 Harness

CYCLE-002 之后，我没有继续急着加角色，而是把系统补成六块基础设施。

### 1. materials 素材库

素材库放在：

```text
materials/
  inbox/
  selected/
  used/
  stale/
  sources/
  templates/
```

它不是一个“资料堆”。每条素材都要记录：

- 来源类型。
- 平台。
- 观察时间。
- 原始路径。
- 摘要。
- 为什么重要。
- 可复用场景。
- 事实强度。
- 版权风险。
- 是否要进入 gbrain。

这里借鉴了 Agent Memory 的一个原则：Raw 和 Derived 要分开。

Raw 是原始材料，Derived 是从材料里提炼出的判断。只有 Raw，没有判断，系统会钝；只有 Derived，没有来源，系统会漂。

### 2. weapons 武器库

武器库放在：

```text
weapons/
  registry.json
  schema.md
```

这里记录所有可用工具，包括 Hermes skill、MCP、CLI、API、框架、参考仓库。

每个 weapon 都要回答几个问题：

- 它能做什么？
- 适合给哪个角色用？
- 什么时候触发？
- 输入是什么？
- 输出是什么？
- 成本、版权、平台条款风险是什么？
- 下一步要安装、测试，还是暂时只作为参考？

这一步避免了一个常见问题：看到一个好工具就装，最后整个系统变成工具坟场。

### 3. gbrain 记忆包

gbrain 这一层不替代 Hermes memory。它更像外部大脑，用来保存长期偏好、发布结果、用户修改意见和流程经验。

本地落点是：

```text
gbrain/
  packets/inbox/
  packets/accepted/
  profiles/
  sources/
  templates/
```

我定义了一个 memory packet schema。每条记忆都要记录：

- 它来自哪里。
- 是用户直接说的，还是 Agent 推断的。
- 原始材料是什么。
- 派生结论是什么。
- 适用于哪些角色。
- 置信度是多少。
- 什么时候失效或复查。
- 是否建议更新 Hermes skill。

这就是记忆治理。不是把聊天记录塞进向量库，然后祈祷下次能搜到。

当前已经写了两条种子记忆：

1. 没有真实指标时，优先跑最低成本发布实验。
2. 小红书封面里少用 `合法合规` 这种法律感太强的词，优先改成 `真人主导版`。

### 4. Kanban template

CYCLE-002 不能只是一次战报，它应该变成模板。

所以我把它固化成：

```text
templates/kanban/cycle-production-template.md
```

模板里定义了：

- 变量：`CYCLE_ID`、`DATE`、`PLATFORM_SCOPE`、`TOPIC`、`OUTPUT_DIR`。
- 16 个任务的 owner、依赖和产物。
- 每个任务的 handoff 格式。
- G1 到 G8 的质量门禁。
- 静态小红书实验和全平台生产两个变体。

下一步可以继续把这个模板变成 Hermes 任务生成器。

### 5. publish preflight

发布前检查单独放在：

```text
preflight/
  publish-preflight-checklist.md
  xiaohongshu-static-checklist.md
```

这里从 CYCLE-002 的 `compliance.md` 抽出检查项：

- 文件是否齐全。
- 事实是否可追溯。
- 动态事实是否有观察日期。
- 图片、BGM、SFX、TTS 是否有版权记录。
- 平台敏感词是否检查。
- AI 生成或 AI 辅助是否需要披露。
- 是否明确 no-auto-publish。
- 指标收集时间是否已经定好。

我现在的原则是：发布可以手动，但发布前检查要标准化。

### 6. 小红书最低成本实验

最后，我们选了第一个发布实验：小红书静态图文。

原因很简单：

- CYCLE-002 已经产出了 6 图滑动笔记结构。
- 不需要视频。
- 不需要 TTS。
- 不需要 BGM 和 SFX。
- 收藏、评论、关注转化都可以作为早期信号。

实验目录是：

```text
experiments/xiaohongshu-low-cost-001/
  publish-copy.md
  carousel-brief.md
  metrics.csv
  attribution.md
```

这一步是整个系统真正开始变聪明的起点。

如果不发布，就没有数据。如果没有数据，dataloop 只能做流程复盘，不能做效果归因。没有归因，gbrain 也只能记住“我们做过什么”，而不是记住“什么真的有效”。

## 注意点

第一，不要急着自动发布。

尤其是新账号阶段，自动发布不是第一优先级。更重要的是把产物、审核、收数、归因这条链跑顺。发布本身可以先手动，但发布前和发布后的流程必须结构化。

第二，dry-run 不是生产。

dry-run 适合做合同测试，检查脚本、目录、manifest、技能安装和文件完整性。真实生产要看 Kanban 任务有没有完成、handoff 有没有发生、质量门禁有没有过。

第三，记忆不是检索。

如果只是把所有材料塞进一个库，下次搜出来，那只是资料库。真正的 memory 要处理来源、派生、冲突、失效和复用。最后能沉淀成 SOP 或 skill，才算开始影响未来行为。

第四，工具要进武器库，不要进收藏夹。

好工具很多，但每个工具都要回答“给谁用、什么时候用、输入输出是什么、风险是什么”。否则工具越多，系统越笨。

第五，先打通最短闭环。

全平台全自动当然诱人，但第一步更应该是小红书静态图文这种低成本实验。先发布，先收数，先知道一个真实结果。

第六，文档要持续校准。

这次中间就出现过旧口径残留：有的文档还写 14 个角色、21 个文件，但最新 manifest 已经是 16 个角色、22 个文件。Agent 系统最怕这种“看起来差不多”的陈旧信息。它不会报错，但会慢慢污染后续决策。

## 总结

这次搭完以后，我对 Hermes 的理解更清楚了。

profile 解决“谁来做”。

skill 解决“怎么做”。

kanban 解决“谁等谁、谁交给谁”。

runner 和 verifier 解决“本地合同测试是否通过”。

manifest 解决“这次运行到底产出了什么”。

materials 解决“素材从哪里来”。

weapons 解决“角色能用什么工具”。

gbrain 解决“系统怎么记住我的偏好和实验教训”。

preflight 解决“什么东西能发布”。

experiment 解决“发布后怎么收数和归因”。

如果只停在前几层，它就是一个会写文件的 Agent 团队。补上后几层之后，它才开始像一个内容工厂。

下一步很具体：先把小红书 6 张图做出来，按 preflight 过一遍，手动发布，然后在 1 小时、24 小时、72 小时、7 天填指标。等第一批真实数据回来，再让 dataloop 写入 gbrain，决定下一轮到底该扩平台、换选题，还是改封面。

这才是我现在更相信的自动化：不是一键替我发一堆内容，而是让每一次内容实验，都能留下可复用的经验。
