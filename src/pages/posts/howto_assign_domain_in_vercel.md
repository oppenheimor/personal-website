---
title: 如何为 Vercel 中的多个应用程序分配路由
date: 2025-09-14T00:00:00.000+00:00
lang: zh
duration: 3min
author: 沈佳棋
---

## 背景

我有一个自己的域名：`oppenheimor.tech`，之前我给自己的博客应用绑定该域名，直接使用的是根路由。但是最近我做了一个新的应用程序并部署在 Vercel 上，我希望为这个新应用分配一个子路由（例如：`oppenheimor.tech/ai-prompt-card`），而不是使用根路由。因此研究了下应该如何做这样一件事情。

## 解决方案

1. 在 Vercel 上给主应用（需要使用根域名访问的应用）配置自定义域名；

<img src="/public/vercel_custom_domain_settings.png" />

2. 主应用程序的根路径新增 `vercel.json` 配置文件，并添加如下配置：

```json
{
  "rewrites": [
    {
      "source": "/(.*)",
      "destination": "/"
    }
  ]
}
```

因为我的主应用程序是一个 `SPA` 应用，因此需要将子路由都重定向到根路径，通过 rewrites 规则，​所有路径​（如 `/about`, `/contact`）都返回根路径 `/` 的 `index.html`，然后由`vue-router` 处理路由。

3. 仅有上述配置还不够，因为如果我们访问另一个新的应用程序的时候，例如 `oppenheimor.tech/ai-prompt-card` 还是会重定向到主应用程序的根路径，因此还需要在主应用程序中的 `vercel.json` 中添加新程序的重定向配置。

4. 在 Vercel 上先发布 `B 程序`（代指新应用），然后复制其生成的域名（例如：`ai-prompt-card-xxxx.vercel.app`），并将其添加到主应用程序的 `vercel.json` 中，如下所示：

```json
{
  "rewrites": [
    {
      "source": "/ai-prompt-card/(.*)",
      "destination": "https://random-prompt-card-3sst.vercel.app/"
    },
    {
      "source": "/(.*)",
      "destination": "/"
    }
  ]
}
```

这样就大工告成啦！
