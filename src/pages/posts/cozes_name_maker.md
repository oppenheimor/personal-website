---
title: 用 coze 做一个文章起名器
date: 2025-01-26T00:00:00.000+00:00
lang: zh
duration: 3min
author: 沈佳棋
---

## 背景

今天在研究 `llama_coder` 项目的时候，很好奇项目中 `app` 下的 `main` 目录 为啥要用 `()` 包裹。 问了一下 `Cursor`，了解到：在 `NextJS` 中，使用括号 `()` 包裹文件夹名称是一个特殊的命名约定，它表示这个文件夹是一个"路由组"(Route Group)。于是想把学习到的相关知识记录到自己的博客里。

<img src="/public/cozes_name_maker/WechatIMG1426.jpg" />

然后就遇到了一个问题，我拟写的这篇文章中文名叫《NextJS 中 `()` 的作用》。我的博客（也就是现在看到的这个站点）是用代码写的，需要在 `post` 目录下起一个英文的文件名， 此时此刻如何起一个合适的英文名让我产生了一丝排斥心理。  
  
这让我联想到之前每次写文章也都会遇到这个问题，例如文章名为：《阿里云数据库 RDS Mysql 使用记录》，最后绞尽脑汁才起了个 "aliyun-db.md"。

<img src="/public/cozes_name_maker/WechatIMG1427.jpg" />

为了永久性地解决这个问题，我打算搭一个 AI 智能体。通过一些预设的规则，以后只需要输入文章的中文名，就能立马得到一个专业的结果。  
  
关于搭建智能体的平台，我脑海里第一想到的就是 `Dify` 和 `Coze`。由于 `Dify` 之前在公司里我已经用过了，所以这次想再体验下字节旗下的 `Coze`。

<img src="/public/cozes_name_maker/WechatIMG1428.jpg" />

## “文章起名器” 智能体搭建

选型完成后，就开始正式搭建 “文章起名器” 的智能体。
  
首先，新建一个智能体应用。输入名称，功能描述即可快速创建。`Coze` 还提供了根据描述自动生成应用图标的小功能。

<img src="/public/cozes_name_maker/WechatIMG1429.jpg" />

创建完成后，进入智能体应用的编排页面。  
  
页面的左侧是核心编辑区，用于编写提示词，这块奠定了我们的智能体的核心能力；中间是一些高级功能，包括"技能"、"知识"、"记忆"、"对话体验"，这些功能后续都可以逐一体验一下，本次暂时用不到；右侧就是智能体的调试预览区。

<img src="/public/cozes_name_maker/WechatIMG1430.jpg" />

接下来开始编写我们的提示词。
  
以下是我的原版提示词：
```md
## 角色：
你是一个文章起名大师，精通计算机技术和文章写作。

## 要求：
1. 根据用户输入的文章中文描述生成精简的英文名，最多不超过 5 个单词，多个单词之间用下划线连接。
2. 只输出结果，不要输出分析过程。
3. 输出 5 个结果，供用户选择。

## 示例：
输入：阿里云数据库 RDS Mysql 使用记录
结果：aliyun_db
--
输入：MacOS13 清除 DNS 缓存
结果：clear_dns_cache
--
输入：devv.ai 是如何构建高效的系统的
结果：devv_rag
--
输入：Llama Coder 分析
结果：llama_coder_analyse
```
  
惊喜地发现 `Coze` 是有自动优化提示词的功能的，而且优化地非常棒。以下是优化后的结果：

<img src="/public/cozes_name_maker/WechatIMG1431.jpg" />

编排完成之后就可以进行发布，填写一下开场白文案和开场白预制问题。

<img src="/public/cozes_name_maker/WechatIMG1432.jpg" />

发布完之后进入对话体验一下效果。  
  
可以看到，不管多复杂的中文描述，都能生成精简的英文名。这篇文章的英文名也是用这个智能体生成的，所以以后有需要的时候直接打开这个智能体输入自己的描述就行。

<img src="/public/cozes_name_maker/WechatIMG1433.jpg" />

## 应用商店

`Coze` 的应用商店里还是有很多好玩的应用的。例如这个 “你的名字就是春联”，非常适合现在过年的气氛。体验地址：[你的名字就是春联](https://www.coze.cn/store/project/7448232293166104602?from=project_card&bid=6f5ec314g6015)

<img src="/public/cozes_name_maker/WechatIMG1434.jpg" />