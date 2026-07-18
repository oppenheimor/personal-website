---
title: 把 Docker 构建从生产机上搬走
date: 2026-07-19T00:00:00.000+00:00
lang: zh
duration: 15min
author: 沈佳棋
---

这次事故发生在一次普通发布里。

项目跑在一台 2 核、4GB 内存的云服务器上。服务器同时承担线上流量、数据库、Nginx、Docker 容器和 GitHub Actions self-hosted Runner。代码 push 到 `main` 后，Runner 会在这台机器上拉代码、装依赖、构建镜像、推送仓库，再重建正式容器。

发布开始后，SSH 先失去响应，网站随后打不开。云控制台还能看到主机在线，但从外部已经很难处置。等了几分钟，服务也没有自己恢复，最后只能重启云主机。

这篇文章记录这次事故怎样把发布链路的问题暴露出来，以及我后来如何把构建迁到国内托管服务。文章里的域名、服务器路径和账号信息都做了替换。

## 旧发布链路

旧方案的优点很直接：搭建快，已有服务器就能跑 Runner，不需要额外购买构建机。GitHub Action 连接到生产机以后，一份 YAML 可以把构建和部署全部串起来。

![旧发布链路组件图](/posts/production-build-migration/old-deployment-components.png)

麻烦也藏在这张图里。

`pnpm install` 会消耗网络和磁盘 I/O，`docker build` 会同时吃 CPU、内存和磁盘。线上 Next.js、数据库和 Nginx 并不会在发布时暂停，它们仍要处理用户请求。构建任务与线上容器共享同一组资源，没有硬隔离，也没有可用的 Swap 兜底。

一旦构建进入高峰，服务器就会同时面对几件事：

- Node 构建和 Docker layer 占用内存；
- 依赖解包、镜像层写入和日志竞争磁盘；
- Runner 拉源码、装依赖、推镜像占用网络；
- 数据库与线上容器仍在响应请求；
- SSH 需要在同一台机器上创建进程和读取磁盘。

所以网站和 SSH 一起失联并不奇怪。它们处于同一个故障域里。

## 事故是怎样发生的

下面这张时序图把一次发布从 push 到 502 的路径拆开了。

![旧发布链路事故时序图](/posts/production-build-migration/old-deployment-incident-sequence.png)

资源压力并不是唯一风险。Docker Compose recreate 容器后，容器 IP 会改变。如果 Nginx 没有 reload，它可能继续代理旧 IP。这样即使新容器已经 healthy，公网仍会返回 502。

这次排查里还碰到了几个容易混在一起的问题：

1. Runner 和应用在同一台主机。主机出问题时，发布任务也失去执行节点。
2. 构建与运行争抢资源。发布动作本身可以拖垮被发布的服务。
3. 镜像使用 `latest`。标签可覆盖，很难从运行容器反查唯一源码和构建记录。
4. migration 放在容器启动流程里。数据库检查失败会让正式容器进入重启循环。
5. Nginx upstream 没有在容器切换后刷新。
6. GitHub Secrets、流水线变量、服务器 `.env` 的职责没有分清，修改变量时容易改错地方。

Runner 故障域、资源竞争、可变标签、启动期 migration、Nginx 旧 upstream 和变量来源，单独看都能补一个脚本继续跑。六项风险同时存在时，给生产机增加构建缓存或清理命令只能缓解症状。

## 为什么选择国内托管构建

我一开始考虑过三种方案。

第一种是继续在生产机构建，再给 Docker 和 Runner 增加资源限制。这能降低一次构建把内存吃光的概率，但构建与运行仍在同一个故障域。

第二种是购买一台国内构建机。网络稳定，控制权也高，不过低频发布的项目要长期支付机器费用，还要维护系统、磁盘、Runner 和安全更新。

第三种是使用国内按次计费的托管构建。构建时临时申请资源，完成后释放；镜像直接进入国内仓库；生产机只拉取并运行镜像。这个方案比较符合项目当前的发布频率。

最终采用了火山引擎持续交付和同区域镜像仓库。GitHub 代码源连接开启网络加速，构建资源使用托管环境，生产服务器保留 Docker Compose 和 Nginx。

## 新发布架构

![新发布链路组件与边界图](/posts/production-build-migration/new-deployment-components.png)

新链路把配置分成四组。

| 配置类型 | 例子 | 存放位置 |
| --- | --- | --- |
| 代码版本 | `SCM_COMMIT_ID`、`DATETIME` | 火山引擎预置变量 |
| SSH 连接 | `SERVER_HOST`、`SERVER_USER`、`SERVER_SSH_KEY` | 流水线变量；私钥为隐私变量 |
| 部署契约 | Compose、服务器脚本、必需变量名单 | Git 仓库 |
| 应用运行时配置 | 数据库地址、模型 API Key | 生产服务器 `.env`，权限 `600` |

GitHub Secrets 不再给生产容器注入配置。它们属于已经退役的 GitHub Actions 发布链路。

应用变量只有一个运行时来源：生产服务器项目目录中的 `.env`。Compose 通过 `env_file` 把它注入容器。流水线只检查必需变量是否存在和非空，不读取或打印值。

仓库新增了 `deploy/required-env.txt`，里面只写变量名。例如：

```text
DATABASE_URL
DEEPSEEK_API_KEY
TAVILY_API_KEY
```

新增必需变量时，要同时更新 `.env.example`、`required-env.txt` 和服务器 `.env`。如果只是可选开关，则不必加入启动门禁。

## 流水线控制台只留一行

刚迁移时，最终阶段里放了上百行 SSH 和部署命令。它能运行，但每次修改都只能在控制台点开编辑，代码审查和回滚都很麻烦。

后来把完整逻辑放回仓库，流水线最终阶段只执行：

```bash
DEPLOY_IMAGE_TAG="${SCM_COMMIT_ID}-${DATETIME}" \
  bash scripts/deploy-from-volcengine.sh
```

这条命令做两件事：确定本次不可变镜像版本，然后调用当前 commit 里的部署编排脚本。以后修改 SSH 参数、前置检查或上传文件，都可以走普通 Git diff 和测试。

镜像构建任务使用同一个版本表达式。火山官方文档明确支持在镜像版本字段里使用 [`${DATETIME}`](https://www.volcengine.com/docs/6461/189390)，所以最终产物类似：

```text
registry.example.com/team/app:<40位commit>-<14位构建时间>
```

我原先尝试过 `PIPELINE_RUN_ID`。它在 Shell 任务里存在，在镜像版本输入框里却没有按同样规则展开，产物最后只剩 `<commit>-`。托管平台不同输入框的变量语法可能不一致，不能看到环境变量名就默认每个任务都能使用。

## 生产部署门禁

服务器脚本没有直接 `docker compose up`。它先验证候选镜像，再动正式容器。

![生产发布与回滚状态机](/posts/production-build-migration/deployment-state-machine.png)

当前步骤如下：

1. 校验镜像版本、生产 `.env`、Docker、Compose、外部网络和镜像访问权限。
2. 拉取不可变镜像。
3. 启动候选容器，注入生产环境变量，但不挂载生产数据卷。
4. 等待 Eve 预热和 Next.js 健康检查通过。
5. 执行 `nginx -t`，确认反向代理配置可 reload。
6. 用一次性容器执行 `prisma migrate deploy`。
7. Compose recreate 正式容器，并等待 Docker HEALTHCHECK。
8. reload Nginx，再从 Nginx 容器访问新 upstream。
9. 只清理本应用超出保留数量的镜像，默认保留当前版本和最近两个历史版本。

前四个门禁失败时，旧正式容器没有被替换。migration 失败时，旧应用仍能服务。Compose recreate 之后如果健康检查或 upstream 验证失败，才需要执行应用镜像回滚。

候选容器不挂生产卷很重要。它可以验证镜像能否启动和完成预热，又不会修改用户工作流或沙盒持久化目录。

## migration 为什么要独立出来

旧容器入口里执行过数据库结构同步。这样应用每次重启都会先碰数据库，结构差异或数据丢失警告会让容器不停重启，Nginx 最后只能看到 502。

现在 entrypoint 只负责启动 Eve 和 Next.js。migration 由部署脚本在正式切换前单独执行，并且只允许 `prisma migrate deploy`。

这仍然不能自动解决所有数据库回滚。migration 一旦执行成功，切回旧镜像不会撤销数据库变化。相邻版本要遵守 expand/contract：先增加兼容字段和表，再迁移读写，确认旧版本不再需要后才能删除旧结构。

## 标准回滚

回滚不使用 `latest`，也不手工改 Compose。

先从成功记录中找到上一个不可变镜像，确认它与已经执行的 migration 兼容，然后在生产目录执行：

```bash
bash deploy-production.sh registry.example.com/team/app:<previous-tag>
```

回滚会重复候选检查、Nginx 配置检查、migration gate、正式容器健康检查和 Nginx reload。这样回滚与正常发布共用一条经过测试的路径。

如果旧镜像和当前数据库不兼容，应停止应用层切换，进入数据库恢复评估。此时继续来回换镜像只会扩大故障范围。

## 迁移时踩到的不可变标签冲突

第一次完整验证时，镜像仓库已经开启不可变标签，但构建任务仍使用纯 commit SHA。同一个 commit 被手工和事件触发各跑了一次，两个任务尝试推送同一个标签。先完成的任务占用了版本，后一个收到 `PRECONDITION` 拒绝。

仓库拒绝覆盖是正确行为。错误出在版本设计：commit 能定位源码，却不能区分同一源码的多次构建。

改成 `<commit>-<DATETIME>` 后，每次构建有独立身份。流水线还需要设置同一生产发布的并发锁为 1。唯一标签能防止制品覆盖，并发锁负责避免两个部署脚本同时操作同一个候选容器和正式容器。

## 新旧方案对比

| 维度 | 旧方案 | 新方案 |
| --- | --- | --- |
| 构建位置 | 生产服务器 | 托管临时资源 |
| 故障域 | 构建、运行、SSH 在一起 | 构建与生产运行分离 |
| 服务器负载 | 发布时明显升高 | 主要是拉镜像和切换容器 |
| 制品版本 | `latest` 或纯 commit | commit + 构建时间，不可变 |
| 运行时密钥 | 容易与 GitHub Secrets 混淆 | 只在服务器 `.env` |
| 发布逻辑 | 工作流和服务器脚本分散 | 仓库脚本版本化 |
| 上线前验证 | 构建完直接切换 | 候选预热 + migration gate |
| Nginx | 可能保留旧容器 IP | 正式健康后 reload 并检查 upstream |
| 回滚 | 找镜像、手工改命令 | 复用同一生产部署脚本 |
| 成本 | 省构建服务费用，占生产资源 | 按构建用量付费 |

## 清理旧链路

新流水线验证以后，我清理了旧工作流、Runner、退役项目文件和重复镜像：

- 删除旧 GitHub Actions 生产发布工作流；
- 卸载服务器上的 self-hosted Runner；
- 删除两个退役项目的 compose、旧 `.env` 和镜像压缩包；
- 删除重复构建镜像，保留当前和上一成功版本；
- 禁止生产发布脚本执行全局 `docker image prune -af`；
- 把镜像清理限制在当前应用仓库，并保留三个版本；
- 补充生产回滚手册和四项部署脚本测试。

清理后，生产服务器只保留仍在运行的容器、应用目录、Nginx、数据卷和少量回滚镜像。低配置服务器不会因为一条误伤其他项目的 prune 命令失去回滚材料。

## 给下一个项目的接入模板

后续项目复用这套链路时，我会先准备下面这组部署文件：

```text
Dockerfile
docker-compose.yml
deploy/required-env.txt
scripts/deploy-from-volcengine.sh
scripts/deploy-production.sh
tests/deploy-*.test.sh
docs/deployment/setup.md
docs/deployment/rollback.md
```

需要逐项替换的内容包括镜像仓库、生产目录、Compose 服务名、候选容器名、外部网络、健康检查、migration 命令、Nginx upstream 和数据卷。健康检查与 migration 跟应用耦合，不能复制以后不看。

首次发布完成不代表接入结束。至少还要验证一次旧镜像回滚，并记录镜像版本、migration 兼容性和公网冒烟结果。

## 还没做完的部分

新链路降低了生产机被构建拖死的风险，但服务器仍是 2 核、4GB、无 Swap，而且容器还没有完整的 CPU 与内存上限。应用自身的内存泄漏、数据库压力或磁盘写满仍然可能造成事故。

SSH 主机密钥目前采用任务内 `ssh-keyscan` 的首次信任方式。后面可以在流水线中增加固定主机公钥或指纹，减少中间人风险。

流水线控制台配置也没有全部保存成 YAML。最终命令已经进入 Git，构建任务的区域、仓库和资源规格仍需要靠文档复现。后续如果平台支持稳定导出，应把这部分也纳入版本管理。

这次迁移没有让发布系统一下变得豪华。它做的是把昂贵、波动大的构建工作移出生产机，再把上线拆成几个能够独立判断和停止的门禁。对于一台配置不高的云服务器，这已经能少掉一类很不划算的事故。
