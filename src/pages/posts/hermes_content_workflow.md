---
title: 用 Hermes 搭一个会自动跑的内容创作 Agent 团队
date: 2026-07-02T00:00:00.000+00:00
lang: zh
duration: 18min
author: 沈佳棋
---

这篇文章记录一次真实的搭建过程：我用 Hermes 组织了一个内容创作 Agent 团队，目标是服务四个平台，微信公众号、微信视频号、抖音和小红书。

注意，这不是“写几个 prompt，然后宣布自己有了一个 AI 团队”。那种东西我现在有点免疫。真正麻烦的是后半段：角色怎么协作，skill 放在哪里，产物怎么落盘，定时任务怎么跑，跑完之后怎么知道它没有偷偷糊弄你。

最后我们跑通了一个本地 dry-run。它不会自动发布，但会定时生成完整的内容包，并做一次机器校验。

## 背景

一开始的目标很直接：做一组 AI 技术讲解账号，主战场是微信公众号、视频号、抖音、小红书，三个月内每个账号都冲 10 万粉丝。

这个目标非常激进，激进到不能靠手搓。我的判断是，先不要幻想一上来就有“爆款机器”。应该先做一个能每天稳定产出的内容工厂。

我给这个团队定了几个约束：

1. 四个平台都是新账号，先以产出为主。
2. 暂时不真人出镜，也不用我的声音。
3. 不自动发布，所有产物先落到本地文件夹。
4. 可以使用 AI 生成图片、视频和 BGM，但必须记录版权和来源。
5. 产物目录统一成 `日期 - 平台 - 产物`。

这几个约束很重要。没有它们，Agent 很容易把事情做成“看起来很忙”，但没有一个文件能直接交付。

## 思路

我把这件事拆成四层。

第一层是账号策略：账号讲什么，给谁看，三个阶段怎么推进。

第二层是组织结构：谁负责选题，谁负责找资料，谁写脚本，谁做平台适配，谁管剪辑，谁做合规检查。

第三层是技能系统：每个角色不能只靠一段人设 prompt 活着。它需要自己的 SOP，也需要共享的产物协议。

第四层是自动化运行时：用 runner 生成产物，用 verifier 检查产物，用 Hermes cron 定时触发。

Hermes 的 profile 页面里已经能看到这些角色。它们不是随便起的机器人名字，而是按工作流拆出来的岗位。

<img src="/hermes_content_workflow/profiles.jpg" class="medium-zoom-image" />

这次创建了 16 个 profile：

- `aichief`：总编制片人，负责拆任务和最终验收。
- `posimind`：定位与阶段策划。
- `trendfox`：竞品和趋势侦察。
- `sourcebee`：素材与事实库。
- `ideaforge`：选题灵感。
- `scriptfeyn`：AI 技术讲解脚本。
- `wechatmo`：微信公众号主笔。
- `videomax`：微信视频号导演。
- `douyineon`：抖音增长导演。
- `xhshana`：小红书策展人。
- `visualmira`：图片素材和视觉叙事。
- `soundmuse`：BGM 和声音设计。
- `editkai`：自动剪辑与后期。
- `opsmatrix`：排期和本地落盘。
- `dataloop`：数据分析和复盘。
- `guardrail`：事实、版权和平台合规。

Kanban 里能看到任务之间的依赖。一个内容任务不是从“写文章”开始的，而是从定位、趋势、事实、选题、脚本一路流到平台适配和合规。

<img src="/hermes_content_workflow/kanban.jpg" class="medium-zoom-image" />

## 执行步骤

### 1. 先把团队建出来

我先在 Hermes 里创建了一个 board：

```bash
ai-tech-content-studio
```

然后创建 16 个 profile，并把每个角色的职责写进对应的 `SOUL.md`。这一步不是为了好看，而是为了让 Hermes 在分配任务时知道谁该接什么活。

组织关系大概是这样：

```text
aichief
  -> posimind
  -> trendfox / sourcebee
  -> ideaforge
  -> scriptfeyn
  -> wechatmo / videomax / douyineon / xhshana
  -> visualmira / soundmuse / editkai
  -> guardrail
  -> opsmatrix
  -> dataloop
```

这里最容易犯的错是把角色做成“全能型 Agent”。我刻意避开了这个方向。内容生产里的上下文很脏，趋势、事实、脚本、视觉、合规混在一起，最后谁都说不清自己在负责什么。

### 2. 再给角色配 skill

角色有了以后，我做了一轮 skill 调研。已有的 Hermes skills 能复用一部分，比如 `web-access`、`humanizer-zh`、`video-pipeline`、`byted-ark-seedance-skill`、`byted-ark-seedream-skill`、`xhs-viral-post`。

但这些还不够。它们更像工具箱，不是这个内容团队的工作习惯。

所以我补了两类 skill。

共享 skill 放到：

```bash
~/.hermes/skills/
```

角色专属 skill 放到：

```bash
~/.hermes/profiles/<role>/skills/
```

最后装了 23 个：

- 7 个共享 skill，比如 `local-content-studio-output`、`platform-packager`、`humanize-final-copy-zh`。
- 16 个角色专属 skill，比如 `ai-tech-studio-orchestrator`、`ai-tech-signal-scout`、`content-rights-and-safety-gate`。

这里有个细节：不要把所有东西都塞进全局 `~/.hermes/skills`。如果一个 skill 只属于某个角色，放到角色自己的 profile 目录里，后面维护会轻很多。

### 3. 定义本地产物协议

我不想让 Agent 只输出一段聊天记录。所以先定了一个产物目录：

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

其中最重要的是 `manifest.json`。它负责记录这次运行的主题、平台、角色状态、产物路径、质量门禁和缺失输入。

只要有了 manifest，后面就能做校验、复盘和自动化。没有 manifest，就只能靠人肉翻文件夹。

### 4. 写 runner 和 verifier

我写了几个本地脚本：

```bash
tools/install_content_studio_skills.py
tools/github_signal_scan.py
tools/run_content_workflow.py
tools/verify_content_run.py
tools/hermes_cron_content_studio_daily.py
```

`run_content_workflow.py` 负责生成完整内容包。

`verify_content_run.py` 负责检查：

- 产物目录是否存在。
- 必要文件是否齐全。
- `manifest.json` 是否能解析。
- 16 个角色是否都有状态。
- 四个平台的内容包是否都生成。
- no-auto-publish 边界是否存在。
- shared skills 和 role skills 是否装在正确位置。

这一步让我比较安心。Agent 生成东西不稀奇，生成后能不能被机器验收，才是分界线。

### 5. 接上 Hermes cron

最后把 daily dry-run 接到 Hermes cron。

```text
Job ID: c6bdb905a227
Name: ai-tech-content-daily-dry-run
Schedule: 30 9 * * *
Script: content_studio_daily_dry_run.py
Mode: no-agent
```

它每天 09:30 跑一次。逻辑很克制：

1. 刷新 GitHub 信号扫描。
2. 生成本地内容包。
3. 跑 verifier。
4. 输出运行结果。

手动触发方式：

```bash
hermes cron run c6bdb905a227 --accept-hooks
hermes cron tick --accept-hooks
```

最后一次验证结果是：

```json
{"passed": 46, "failed": 0}
```

也就是说，本地 dry-run 自动化已经通了。

## 注意点

第一，不要急着自动发布。

内容工作流里最危险的不是“没产出”，而是“错误地稳定产出”。尤其是 AI 技术讲解账号，事实错误、版权风险、平台敏感表达都可能把账号带偏。所以我现在只做本地落盘，不做自动发布。

第二，角色不是越多越好。

这次有 16 个角色，是因为我们确实覆盖了四个平台和完整生产链。如果你只做一个公众号，其实不需要这么多人。角色数量应该跟交付物数量走，不要跟想象力走。

第三，skill 要分层。

共享能力放全局，角色习惯放 profile。这个边界很小，但后面会救命。否则全局 skills 越堆越多，每个 Agent 都背着一整个杂货铺出门。

第四，dry-run 不是生产就绪。

现在还没做 live 平台调研、TTS、BGM、视频渲染、版权确认和真实指标回流。它已经能自动跑，但还不能说已经能替我运营账号。这个边界要讲清楚。

第五，未完成事项要写下来。

我单独留了 `unresolved-items.md`，记录哪些东西必须由人提供，比如平台账号、媒体生成服务、版权策略、指标导入方式。把这些写下来，比在脑子里“以后再说”靠谱。

## 总结

这次搭完之后，我对 Hermes 的感受是：它适合当一个“内容创作工作台”，但前提是你要把工作台上的零件摆清楚。

profile 解决“谁来做”。

skill 解决“怎么做”。

kanban 解决“做什么，以及谁等谁”。

cron 解决“什么时候自动跑”。

runner 和 verifier 解决“跑完以后算不算真的完成”。

少任何一个，都容易退回到聊天机器人。

下一步我会优先补三块：实时平台信号扫描、视频渲染链路、指标回流。等这三块接上，这个系统就不只是每天写一包文件，而是可以开始真正做内容实验了。
