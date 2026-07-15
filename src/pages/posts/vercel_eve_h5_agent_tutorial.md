---
title: 用 Vercel Eve 搭一个能做事的 H5 Agent
date: 2026-07-15T00:00:00.000+00:00
lang: zh
duration: 15min
author: 沈佳棋
---

## 背景

Vercel 在 2026 年 6 月开源了 Eve。它把 Agent 常用的模型循环、工具、Skill、沙盒、耐久会话、人工审批和前端协议放进了一套 TypeScript 框架里，Next.js 也有现成的同源集成。

我在 `eve@0.24.3` 上做了一轮源码调研，并搭了一个 H5 Demo。模型固定用 `deepseek-chat`，页面可以正常对话，也能看到工具调用。模型按需加载编码 Skill，在会话沙盒里写入并执行脚本；涉及“暂存产物”的动作会停下来，等用户批准后再继续。刷新页面后，工具结果和会话还能恢复。

这篇文章保留搭建过程中可以直接复用的代码，以及几处只看文档不容易发现的坑。Eve 目前仍是 beta，版本和行为变化很快，下面的内容都以 2026-07-15 的 `0.24.3` 为准。

官方资料：

- [Eve 文档](https://eve.dev/docs/introduction)
- [vercel/eve](https://github.com/vercel/eve)

## 我最后做了哪些能力

Demo 没有把 Eve 的每个目录都放一个空文件。我先选了一条完整任务链：

1. H5 发起多轮任务，界面显示流式消息和工具状态。
2. 一个自定义工具返回当前 Demo 的能力清单。
3. 编码任务先加载 `code-lab` Skill，再进入沙盒写文件、执行、读回结果。
4. 一个有副作用含义的工具使用 `always()` 审批，批准前不能执行。
5. 审批次数放进 `defineState`，沙盒文件按 session 保留。
6. 浏览器保存 Eve events 和 session cursor，刷新后恢复会话。
7. 用 Eve eval 检查模型是否调用过工具，避免只凭回复文案判断成功。

OAuth Connection、定时任务、Slack/Discord Channel 没有硬塞进 Demo。它们需要外部系统、身份和独立的验收场景，空接入只能增加文件数量。

## 初始化项目

Eve 0.24.3 要求 Node.js 24。机器上即使已经有 Node 22，CLI 也会在启动前拒绝运行。

```bash
node --version
# v24.x

npx eve@0.24.3 init demo --channel-web-nextjs
cd demo
npm install @ai-sdk/deepseek just-bash
```

我的默认 npm 镜像当时返回了 502，初始化命令可以临时指定官方 registry：

```bash
npm_config_registry=https://registry.npmjs.org \
  npx eve@0.24.3 init demo --channel-web-nextjs
```

环境变量只放服务端。示例文件可以提交，占位值不能换成真实 key：

```bash
# .env.local
DEEPSEEK_API_KEY=<your_deepseek_api_key>
```

Eve 还在 `0.x`，我把 `package.json` 里的版本锁成了 `"eve": "0.24.3"`。package-lock 能锁住一次安装，明确写死版本也能减少后来维护者误以为 caret 升级是安全操作。

## 接入 deepseek-chat

`agent/agent.ts` 只需要返回一个 Eve Agent 定义：

```ts
import { deepseek } from "@ai-sdk/deepseek";
import { defineAgent } from "eve";

// API key 由服务端 provider 读取，不进入浏览器和沙盒。
export default defineAgent({
  model: deepseek("deepseek-chat"),
  reasoning: "none",
  limits: {
    maxInputTokensPerSession: 200_000,
    maxOutputTokensPerSession: 20_000,
  },
});
```

这里走 AI SDK direct provider，没有经过 Vercel AI Gateway。运行 `eve info` 或访问 `/eve/v1/info` 时，Eve 会把模型识别成 external DeepSeek，并规范化为自己的模型目录 ID。这个目录 ID 和代码里传入的 `deepseek-chat` 名称不同，排查模型配置时需要区分两层名称。

## 自定义工具和人工审批

普通业务工具放在 `agent/tools`。下面是简化后的状态检查工具：

```ts
import { defineTool } from "eve/tools";
import { z } from "zod";

export default defineTool({
  description: "Inspect this demo agent's verified capabilities.",
  inputSchema: z.object({
    focus: z.enum(["all", "model", "tools", "skills", "sandbox"]).default("all"),
  }),
  async execute({ focus }) {
    // 清单由代码维护，模型不能靠猜测补全能力。
    return {
      focus,
      model: "deepseek-chat",
      authoredTools: ["demo_status", "stage_release"],
      skill: "code-lab",
    };
  },
});
```

有副作用含义的工具增加 `approval: always()`：

```ts
import { defineTool } from "eve/tools";
import { always } from "eve/tools/approval";

export default defineTool({
  description: "Stage a sandbox artifact after explicit approval.",
  inputSchema: artifactSchema,
  approval: always(),
  async execute({ artifactPath }, ctx) {
    // Eve 只有在用户批准后才会进入 execute。
    const sandbox = await ctx.getSandbox();
    const content = await sandbox.readTextFile({ path: artifactPath });
    return {
      artifactPath,
      bytes: new TextEncoder().encode(content ?? "").byteLength,
      publishedExternally: false,
    };
  },
});
```

真实运行时，模型请求这个工具后，事件流先出现 `input.requested`，session 进入 `waiting`。批准前没有 `action.result`，批准后工具才读取文件并返回结果。这个顺序很适合支付、发消息、创建工单一类需要人工把关的操作。

我还遇到一个容易误判的细节。批准已经完成，`action.result` 也返回成功，模型最后一句话却仍然说“等待人工审批”。工具事件证明执行成功，模型对状态的复述错了。界面、审计和业务状态机应读取结构化事件，不能解析助手文案来判断操作有没有发生。

## Skill 只负责加载规则

Skill 使用文件系统约定：

```text
agent/skills/code-lab/
├── SKILL.md
└── references/checklist.md
```

`SKILL.md` 的 frontmatter 描述触发场景，正文保存编码步骤：

```md
---
name: code-lab
description: 在沙盒中创建、修改、检查或执行代码时使用。
---

1. 先读取 references/checklist.md。
2. 用 todo 拆解任务。
3. 所有文件写入 /workspace/lab。
4. 执行脚本并读回产物，最后报告验证结果。
```

Eve 初始只把 Skill 描述交给模型。模型调用 `load_skill` 后，完整指令才进入当前任务上下文。Skill 不会自动增加 shell 或文件权限，执行能力仍来自 Agent 已经拥有的工具。

macOS 还有一处小坑：`agent/skills` 是严格 discovery 目录，Finder 生成的 `.DS_Store` 会被当成一个 Skill entry，随后启动直接报错。项目和 CI 都应清理并忽略它。

## 沙盒怎么选

Eve 的 `defaultBackend()` 在 Vercel 上选择 Vercel Sandbox，本地依次尝试 Docker、microsandbox 和 just-bash。我的配置对能设置网络策略的后端统一使用 `deny-all`：

```ts
import { defaultBackend, defineSandbox } from "eve/sandbox";
import { justbash } from "eve/sandbox/just-bash";

const backend =
  process.env.EVE_LAB_SANDBOX === "justbash"
    ? justbash({ autoInstall: false })
    : defaultBackend({
        docker: { networkPolicy: "deny-all" },
        microsandbox: { networkPolicy: "deny-all" },
        vercel: { networkPolicy: "deny-all" },
        justBash: { autoInstall: false },
      });

export default defineSandbox({ backend });
```

Docker 路径会拉取 `ghcr.io/vercel/eve:latest`。我这次在本机看到拉取和 template prewarm 启动，但到 GHCR 的网络没有让镜像下载完成，所以没有把 Docker 标成通过。为了继续验证 Eve 的工具链，我显式设置了：

```bash
EVE_LAB_SANDBOX=justbash npm run dev
```

just-bash 能提供虚拟文件系统和 shell 语义，`write_file`、`bash`、`read_file` 都跑通了，脚本输出为 `EVE_SANDBOX_OK`。它没有真实的 Node、git、包管理器和 VM 隔离，也不支持同等级网络策略。生产环境仍要单独验收 Vercel Sandbox、Docker 或 microsandbox。

## Next.js 页面保存什么

脚手架用 `withEve()` 把 Eve route 挂到 Next.js 同源地址，页面通过 `useEveAgent()` 收发事件：

```tsx
const STORAGE_KEY = "eve-lab-session-v1";

const [saved] = useState(() => readSavedChat());
const agent = useEveAgent({
  initialEvents: saved.events ?? [],
  initialSession: saved.session,
  onFinish(snapshot) {
    // cursor 负责继续远端 session，events 负责重建当前浏览器画面。
    localStorage.setItem(
      STORAGE_KEY,
      JSON.stringify({ events: snapshot.events, session: snapshot.session }),
    );
  },
});
```

只保存 `SessionState` 无法恢复聊天画面，它只是远端 session 的 cursor。Demo 在一次真实工具对话后保存了 176 条 events；刷新页面，工具输入、结果和 DeepSeek 的最终回复都恢复了。

localStorage 只适合单浏览器 Demo。多设备历史、审计检索和数据删除需要服务端 thread/event 存储。附件也可能迅速放大浏览器存储，生产页面应设置大小限制并把文件放到对象存储。

开发测试还发现一个 Next.js 配置问题。通过 `127.0.0.1` 打开页面时，SSR HTML 正常，按钮却完全没反应。日志提示开发资源来源被拦截，DOM 也没有 React fiber。加入下面的配置并重启后，页面完成水合：

```ts
const nextConfig = {
  allowedDevOrigins: ["127.0.0.1"],
};
```

移动端用 390×844 viewport 复测，页面 `scrollWidth` 和 `innerWidth` 都是 390，composer 左右各保留 12px，没有横向溢出。

## 用 eval 检查行为

聊天回复里出现“deepseek-chat”不能证明工具真的执行过。Eve eval 可以同时检查完成状态、工具事件和回复内容：

```ts
import { defineEval } from "eve/evals";
import { includes } from "eve/evals/expect";

export default defineEval({
  description: "Agent 调用状态工具并报告模型。",
  async test(t) {
    await t.send("请调用 demo_status 检查 model，并告诉我模型 ID。不要猜。");
    t.succeeded();
    t.calledTool("demo_status");
    t.check(t.reply, includes("deepseek-chat"));
  },
});
```

最后跑的校验命令如下：

```bash
npm run typecheck
npm run build
npm run build:eve
npm run eval
npm_config_registry=https://registry.npmjs.org npm audit
```

结果是 Next build 和 Eve build 通过，`eve info` 显示 0 errors、0 warnings，真实 DeepSeek eval 为 1/1、3/3 gates，依赖审计为 0。

Next 16.2.6 当时内嵌了 PostCSS 8.4.31，会命中一条 moderate XSS advisory。我在 npm overrides 里统一到 8.5.19，再重新构建和审计。升级 Next 后应该复查官方依赖，并在上游更新后删掉这个临时 override。

## Eve 能解决到哪里

这轮测试让我确认了 Eve 的几项价值：

- filesystem-first 的工具和 Skill 组织方式很适合 TypeScript 项目；
- Workflow SDK 提供 session、等待、恢复和事件回放，HITL 不需要自己拼一套轮询状态机；
- 默认 harness 已经带文件、shell、搜索、提问和子 Agent 工具；
- `withEve()` 与 `useEveAgent()` 缩短了 Next.js H5 的接入路径；
- Evals 直接走真实 session 协议，行为验证可以进入 CI。

它没有替应用完成账号体系、多租户 ACL、历史数据库、审计检索、数据保留与删除、prompt injection 防护、消息队列和自托管多副本 HA。默认 Agent 的权限也很强，shell、文件、web fetch 和 delegation 都需要按业务收口，沙盒出网策略不能留在默认值。

我会把 Eve 放在强能力 Agent 原型、内部工具和受控试点里使用，并固定版本、保留事件级测试。涉及监管数据、严格多租户或自托管高可用时，还需要在框架外补齐权限、存储和运维层。对这次 H5 Demo 来说，Eve 已经把最费时间的 session、工具循环、审批和沙盒协议串了起来，剩下的工作边界也足够清楚。
