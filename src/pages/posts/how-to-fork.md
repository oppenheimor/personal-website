---
title: fork 仓库那些事儿
date: 2025-01-16T00:00:00.000+00:00
lang: zh
duration: 10min
author: 沈佳棋
---

最近在将 `shadcn/ui` 的代码 `fork` 到本地进行私有化的改造。涉及到仓库 `fork` 的一些小 tips 在此做一个统一的记录。

## 一、如何 fork 仓库？

<img src="/public/how-to-fork/1.jpg" />

在源仓库点击 `fork` 按钮，将该仓库 `fork` 到自己的 GitHub 中。然后 `git clone` 自己的仓库地址正常进行开发。

## 二、如何将原仓库的变更同步到自己的仓库？

<img src="/public/how-to-fork/2.jpg" />

例如，当源仓库的代码发生更新，我们想把最新的代码同步到自己的仓库怎么办呢？按以下步骤进行操作：

1. 查看当前远程仓库配置

```bash
git remote -v
```

<img src="/public/how-to-fork/3.jpg" />

此时只有自己的仓库源。

2. 添加上游仓库（`upstream`）源：

```bash
git remote add upstream https://github.com/shadcn-ui/ui.git
```

<img src="/public/how-to-fork/4.jpg" />

这里的 `https://github.com/shadcn-ui/ui.git` 只是一个示例，实际操作时替换成真实的上游仓库源。此外添加的源需要使用 `https:` 开头的地址，不能用 `git:` 开头的。

3. 获取上游仓库的所有分支：

```bash
git fetch upstream
```

4. 合并上游仓库的分支

```bash
git merge upstream/main
```

5. 将更新提交并推送到自己的远程仓库

```bash
git commit -m 'Merge remote-traking branch upstream/main'
git push origin main
```

## 三、开发完成后，如何将改动合并到源仓库？

如果我们想为源仓库做贡献，成为源仓库的 contributor 的话，可以在本地开发完成后提交一个 `pull request` 提交将改动合并到源仓库的申请。

<img src="/public/how-to-fork/5.jpg" />

<img src="/public/how-to-fork/6.jpg" />

## 注意
1. 在 GitHub 的 contribution 规则中，向 fork 的仓库提交代码是不算 contribution（小绿点）的