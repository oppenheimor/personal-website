---
title: 拆开 Pixelle-Video：一个开源 AI 短视频引擎到底怎么跑起来
date: 2026-07-03T00:00:00.000+00:00
lang: zh
duration: 24min
author: 沈佳棋
---

这篇文章是一次竞品拆解。对象是 [Pixelle-Video](https://github.com/ATH-MaaS/Pixelle-Video)，一个开源的 AI 全自动短视频引擎。

我一开始只想看它怎么把“输入主题，生成视频”串起来。拆完之后，感觉它更像一个开源短视频生产工坊的样板间：LLM 写文案，TTS 读旁白，图片或视频模型出素材，HTML 模板做画面，最后用 FFmpeg 合成。

它没有把商业短视频平台需要的所有东西都做完。比如账号定位、选题策略、素材治理、质量评估、发布排期、数据回流，这些都还很薄。但它把最底层、最容易被低估的生产链路跑通了，这点很有参考价值。

先放几个核对过的外部信号。到 2026-07-03，我通过 GitHub API 看到它的仓库是 `ATH-MaaS/Pixelle-Video`，Apache-2.0 协议，主语言是 Python，star 约 24k，fork 约 3.4k，open issue 140 个。最新 release 是 `v0.1.15`，Windows 一键包下载量已经 5.4 万多次。

这个数据说明两件事：一是需求确实旺，大家都想要一个“自动生成短视频”的工具；二是这类项目的门槛不是模型调用本身，而是把一堆不稳定的媒体能力编排成一条相对稳定的生产线。

## 简介

Pixelle-Video 对外讲得很直接：输入一个主题，自动生成一条短视频。

实际链路大概是这样：

```text
主题或脚本
  -> LLM 生成旁白、标题、图片/视频 prompt
  -> TTS 生成每一段旁白音频
  -> 图片模型、视频模型或 ComfyUI 工作流生成媒体素材
  -> HTML 模板渲染成画面
  -> FFmpeg 合成分镜片段
  -> 拼接、加 BGM、输出 final.mp4
```

这条链路听起来朴素，但很实用。短视频创作里有很多脏活：每段文案对应几秒音频，画面时长要对齐音频，字幕卡不能挡住主体，竖屏和横屏要分开处理，BGM 要混音，视频片段长短不一还要 pad 或 trim。Pixelle-Video 没有用一个巨大黑盒解决这些问题，而是把它们拆成了几个能替换的服务。

我更愿意把它看成三件东西的组合：

1. 一个视频生成流水线。
2. 一个模型和工作流适配器集合。
3. 一个基于 HTML 模板的轻量视频画面系统。

它的产品野心不小，但工程风格很务实。

## 功能

从 README 看，它承诺的是一键生成短视频：写文案、生成 AI 配图或视频、合成语音、添加背景音乐、导出视频。

从代码看，Web 里真正暴露出来的是五个产品入口：

| 入口 | 注册名 | 作用 |
|---|---|---|
| 快速生成 | `quick_create` | 从主题或固定脚本生成完整视频 |
| 自定义素材 | `custom_media` | 上传图片/视频，分析素材，再生成营销视频 |
| 图生视频 | `image_to_video` | 上传图片，加 prompt 生成动态视频 |
| 数字人口播 | `digital_human` | 使用参考图、参考音频或文案做口播视频 |
| 动作迁移 | `action_transfer` | 把参考视频里的动作迁移到目标形象 |

这五个入口对应了现在 AI 视频工具最常见的几个需求：一句话成片、素材成片、图生视频、数字人、动作迁移。它们不是五套完全独立的系统，而是共享底层的 LLM、TTS、媒体生成、HTML 渲染和 FFmpeg 合成能力。

更细一点，项目里有三类“生产资源”：

1. 31 个 HTML 模板。
2. 29 个媒体或 TTS 工作流。
3. 多个直连 API 模型适配器。

31 个 HTML 模板决定视频长什么样。29 个工作流决定图片、视频、语音怎么生成。直连 API 适配器负责把 DashScope、Kling、Seedance、Seedream、OpenAI image 这类模型接进来。

### 31 个 HTML 模板

模板目录按画布尺寸组织，主要是 `1080x1920` 竖屏，少量 `1920x1080` 横屏和 `1080x1080` 方图。

文件名前缀也很直接：

| 前缀 | 含义 |
|---|---|
| `image_` | 需要图片素材，最后用图片加旁白生成视频片段 |
| `video_` | 需要视频素材，最后叠加模板画面和旁白 |
| `static_` | 主要靠文字和背景，不一定生成媒体 |
| `asset_` | 面向用户上传素材的模板 |

完整清单如下：

| # | 模板 | 画布 | 类型 | 媒体尺寸 | 变量 |
|---:|---|---|---|---|---|
| 1 | `1080x1080/image_minimal_framed.html` | 1080x1080 | image | 1024x1024 | `image,text,title` |
| 2 | `1080x1920/asset_default.html` | 1080x1920 | asset | 1024x1024 | `image,text,title` |
| 3 | `1080x1920/image_blur_card.html` | 1080x1920 | image | 1024x1024 | `author,brand,describe,image,text,title` |
| 4 | `1080x1920/image_book.html` | 1080x1920 | image | 1024x1024 | `author,image,subtitle,text,title` |
| 5 | `1080x1920/image_cartoon.html` | 1080x1920 | image | 1024x1024 | `image,text,title` |
| 6 | `1080x1920/image_default.html` | 1080x1920 | image | 1024x1024 | `author,brand,describe,image,text,title` |
| 7 | `1080x1920/image_elegant.html` | 1080x1920 | image | 1024x1024 | `image,text,title` |
| 8 | `1080x1920/image_excerpt.html` | 1080x1920 | image | 1024x1024 | `author,image,signature,text,title` |
| 9 | `1080x1920/image_fashion_vintage.html` | 1080x1920 | image | 1024x1024 | `image,text,title` |
| 10 | `1080x1920/image_full.html` | 1080x1920 | image | 1024x1024 | `author,brand,describe,image,text,title` |
| 11 | `1080x1920/image_healing.html` | 1080x1920 | image | 1024x1024 | `image,index,signature,text,title` |
| 12 | `1080x1920/image_health_preservation.html` | 1080x1920 | image | 未标注 | `image,signature,text,title` |
| 13 | `1080x1920/image_life_insights.html` | 1080x1920 | image | 1024x1024 | `image,text,title` |
| 14 | `1080x1920/image_life_insights_light.html` | 1080x1920 | image | 1024x1024 | `author,image,text,title` |
| 15 | `1080x1920/image_long_text.html` | 1080x1920 | image | 1024x1024 | `author,image,text,title` |
| 16 | `1080x1920/image_modern.html` | 1080x1920 | image | 1024x1024 | `accent_color,author,brand,describe,image,text,title,title_font_size` |
| 17 | `1080x1920/image_neon.html` | 1080x1920 | image | 1024x1024 | `author,brand,describe,image,text,title` |
| 18 | `1080x1920/image_psychology_card.html` | 1080x1920 | image | 1024x1024 | `image,text,title` |
| 19 | `1080x1920/image_purple.html` | 1080x1920 | image | 1024x1024 | `author,brand,describe,image,text,title` |
| 20 | `1080x1920/image_satirical_cartoon.html` | 1080x1920 | image | 1024x1024 | `author,brand,describe,image,title` |
| 21 | `1080x1920/image_simple_black.html` | 1080x1920 | image | 1024x1024 | `author,brand,describe,image,title` |
| 22 | `1080x1920/image_simple_line_drawing.html` | 1080x1920 | image | 1024x1024 | `author,brand,describe,image,text` |
| 23 | `1080x1920/static_default.html` | 1080x1920 | static | 1024x1024 | `author,background,brand,describe,text,title` |
| 24 | `1080x1920/static_excerpt.html` | 1080x1920 | static | 1024x1024 | `author,signature,text,title` |
| 25 | `1080x1920/video_default.html` | 1080x1920 | video | 512x288 | `author,brand,describe,text,title` |
| 26 | `1080x1920/video_healing.html` | 1080x1920 | video | 1024x1024 | `index,signature,text,title` |
| 27 | `1920x1080/image_book.html` | 1920x1080 | image | 1024x1024 | `author,image,text,title` |
| 28 | `1920x1080/image_film.html` | 1920x1080 | image | 1024x1024 | `author,brand,describe,image,text,title` |
| 29 | `1920x1080/image_full.html` | 1920x1080 | image | 1024x1024 | `author,brand,describe,image,text,title` |
| 30 | `1920x1080/image_ultrawide_minimal.html` | 1920x1080 | image | 1024x1024 | `image,text,title` |
| 31 | `1920x1080/image_wide_darktech.html` | 1920x1080 | image | 1024x1024 | `image,text,title` |

这里有个小细节值得注意：模板里的 `template:media-width` 和 `template:media-height` 不一定等于视频画布尺寸。比如竖屏视频是 1080x1920，但很多 AI 图像只生成 1024x1024，再由模板负责裁切、排版和叠字。`video_default.html` 的媒体尺寸甚至是 512x288，这说明它不是直接生成满屏视频，而是把视频当作模板里的一个元素。

### 29 个工作流

项目有两类工作流。

第一类是 RunningHub 工作流，一共 21 个。它们本地只是很薄的一层 wrapper，基本就是记录 `source=runninghub` 和一个 `workflow_id`。真正的 ComfyUI 图在 RunningHub 云端。

| # | 文件 | workflow_id | 用途 |
|---:|---|---|---|
| 1 | `runninghub/af_scail.json` | `2013073105194852353` | 放大或增强 |
| 2 | `runninghub/analyse_image.json` | `1996069253201739777` | 图片理解 |
| 3 | `runninghub/digital_combination.json` | `2003717471859294210` | 数字人组合 |
| 4 | `runninghub/digital_customize.json` | `2010608838151507970` | 数字人定制 |
| 5 | `runninghub/digital_image.json` | `2004120336125861890` | 数字人图像 |
| 6 | `runninghub/i2v_LTX2.json` | `2011258580393009153` | 图生视频 LTX2 |
| 7 | `runninghub/image_Z-image.json` | `1995319131513794562` | Z-image 生图 |
| 8 | `runninghub/image_flux.json` | `1983427617984585729` | Flux 生图 |
| 9 | `runninghub/image_flux2.json` | `1996872017192308738` | Flux2 生图 |
| 10 | `runninghub/image_qwen.json` | `1984140002701574146` | Qwen Image 生图 |
| 11 | `runninghub/image_qwen_chinese_cartoon.json` | `1988434426705133569` | Qwen 中文漫画风 |
| 12 | `runninghub/image_sd3.5.json` | `1983932442484604929` | SD 3.5 生图 |
| 13 | `runninghub/image_sdxl.json` | `1983925934648655874` | SDXL 生图 |
| 14 | `runninghub/tts_edge.json` | `1983513964837543938` | Edge TTS |
| 15 | `runninghub/tts_index2.json` | `1983718528991862786` | IndexTTS2 |
| 16 | `runninghub/tts_spark.json` | `1983921902282539009` | Spark TTS |
| 17 | `runninghub/video_Z_image_wan2.2.json` | `1993931250872369154` | Z-image + Wan2.2 |
| 18 | `runninghub/video_qwen_wan2.2.json` | `1993608528969531394` | Qwen + Wan2.2 |
| 19 | `runninghub/video_understanding.json` | `1996419135271747586` | 视频理解 |
| 20 | `runninghub/video_wan2.1_fusionx.json` | `1985909483975188481` | Wan2.1 FusionX |
| 21 | `runninghub/video_wan2.2.json` | `1991693844100100097` | Wan2.2 视频 |

第二类是 selfhost 工作流，一共 8 个。它们是完整的本地 ComfyUI graph。

| # | 文件 | 节点数 | 关键节点 |
|---:|---|---:|---|
| 1 | `selfhost/analyse_image.json` | 4 | `LoadImage`, `ImageResize+`, `AILab_QwenVL` |
| 2 | `selfhost/analyse_video.json` | 4 | `VHS_LoadVideo`, `ImageResize+`, `AILab_QwenVL` |
| 3 | `selfhost/image_flux.json` | 13 | `UNETLoader`, `DualCLIPLoader`, `KSampler`, `FluxGuidance` |
| 4 | `selfhost/image_nano_banana.json` | 3 | `GeminiImageNode`, `PrimitiveStringMultiline`, `SaveImage` |
| 5 | `selfhost/image_qwen.json` | 13 | `CLIPLoader`, `LoraLoader`, `KSampler`, `EmptySD3LatentImage` |
| 6 | `selfhost/tts_edge.json` | 6 | `EdgeTTS`, `SaveAudioMP3` |
| 7 | `selfhost/tts_index2.json` | 4 | `IndexTTS2BaseNode`, `VHS_LoadAudioUpload` |
| 8 | `selfhost/video_wan2.1_fusionx.json` | 13 | `UNETLoader`, `CLIPLoader`, `VHS_VideoCombine` |

这套设计很聪明。普通用户可以用 RunningHub 省掉本地显卡和工作流调试，高级用户可以用 selfhost 控制细节。项目本身只需要识别文件名和工作流来源。

不过这也埋下了平台化问题：RunningHub 的 `workflow_id` 对项目来说是黑盒，平台不知道里面的节点、显存、耗时、失败原因和质量差异。它能调用，但还不太能治理。

### 直连 API 模型

除了 ComfyUI 和 RunningHub，项目还把一些模型 API 包装成统一的 workflow key：

```text
api/{provider}/{model}
```

图片模型包括：

| provider | models |
|---|---|
| dashscope | `wan2.7-image`, `wan2.7-image-pro`, `wan2.6-t2i` |
| openai | `gpt-image-2` |
| seedream | `doubao-seedream-5-0-260128`, `doubao-seedream-4-5-251128`, `doubao-seedream-4-0-250828` |

视频模型包括：

| provider | models |
|---|---|
| dashscope | `wan2.7-t2v`, `happyhorse-1.0-t2v`, `wan2.7-i2v`, `wan2.7-r2v`, `wan2.7-videoedit`, `wan2.6-i2v-flash`, `happyhorse-1.0-i2v`, `happyhorse-1.0-r2v`, `happyhorse-1.0-video-edit` |
| kling | `kling-v3`, `kling-v2-6`, `kling-v2-5-turbo` |
| seedance | `doubao-seedance-2-0-260128`, `doubao-seedance-2-0-fast-260128`, `seedance-1-0-pro`, `seedance-1-0-lite` |

这个抽象比表面上更重要。视频模型不是“同一种能力的不同供应商”。有的支持文生视频，有的支持首帧图生视频，有的支持音频驱动，有的支持数字人参考图，有的支持动作迁移。Pixelle-Video 已经开始给模型记录 `ability_type`、`adapter_ability_types`、`duration`、`resolution`、`ratios`、`api_contract_verified` 这些元数据。

这就是模型路由的雏形。

## 分层

我把它拆成七层会比较清楚。

第一层是入口层。它有 Streamlit Web，也有 FastAPI。Web 面向本地用户，FastAPI 面向集成和异步任务。

第二层是 Core 门面层。`PixelleVideoCore` 初始化并持有 LLM、TTS、Media、API Media、图片分析、视频分析、FrameProcessor、VideoService、Persistence、History 和 Pipelines。它像一个总插线板，业务入口不用到处找服务。

第三层是 Pipeline 编排层。`LinearVideoPipeline` 定义通用生命周期，`StandardPipeline`、`asset_based`、`digital_human` 这些 pipeline 再填具体流程。

第四层是内容规划层。这里负责生成旁白、标题、图片 prompt、视频 prompt，并把文案切成多段 storyboard frame。

第五层是单帧生产层。`FrameProcessor` 是关键对象，它把一段旁白变成一个可合成的视频片段。

第六层是能力适配层。包括 LLMService、TTSService、MediaService、APIProviderMediaService、ComfyKit、RunningHub、本地 ComfyUI 和各类 provider adapter。

第七层是渲染、合成和持久化层。HTMLFrameGenerator 用 Playwright 截图，VideoService 用 FFmpeg 合成，PersistenceService 把产物写到 `output/{task_id}`。

画成文字图就是：

```text
Streamlit / FastAPI
  -> PixelleVideoCore
    -> Pipeline
      -> Content Planning
        -> Storyboard / Frame
          -> TTS / Media / API Provider / ComfyUI / RunningHub
          -> HTMLFrameGenerator
          -> VideoService
          -> PersistenceService / History
```

这个分层不花哨，但边界挺干净。尤其是 `Pipeline` 和 `FrameProcessor` 的拆法，值得自研项目参考。

## 架构

### Web 和 API

Web 侧更像一个本地控制台。快速生成页大概是三栏：

| 区域 | 内容 |
|---|---|
| 左栏 | 内容输入、BGM、版本信息 |
| 中栏 | 风格配置、TTS、模板、工作流 |
| 右栏 | 生成按钮、进度条、视频预览、下载 |

FastAPI 则提供原子能力和完整视频生成能力。主要接口包括：

| method | path | 用途 |
|---|---|---|
| GET | `/health` | 健康检查 |
| GET | `/version` | 版本信息 |
| POST | `/api/llm/chat` | 原子 LLM 调用 |
| POST | `/api/tts/synthesize` | 原子 TTS 合成 |
| POST | `/api/image/generate` | 原子图片生成 |
| POST | `/api/content/narration` | 生成旁白 |
| POST | `/api/content/image-prompt` | 生成图片 prompt |
| POST | `/api/content/title` | 生成标题 |
| POST | `/api/video/generate/sync` | 同步生成视频 |
| POST | `/api/video/generate/async` | 异步生成视频 |
| GET | `/api/tasks` | 列出任务 |
| GET | `/api/tasks/{task_id}` | 查询任务 |
| DELETE | `/api/tasks/{task_id}` | 取消任务 |
| GET | `/api/files/{file_path:path}` | 访问 output 文件 |
| GET | `/api/resources/workflows/tts` | 列出 TTS 工作流 |
| GET | `/api/resources/workflows/media` | 列出媒体工作流 |
| GET | `/api/resources/templates` | 列出模板 |
| GET | `/api/resources/bgm` | 列出 BGM |
| POST | `/api/frame/render` | 单帧 HTML 渲染 |
| GET | `/api/frame/template/params` | 解析模板参数 |

API 的请求模型里有几个字段很能说明它的产品形态：`mode`、`n_scenes`、`tts_workflow`、`media_workflow`、`frame_template`、`template_params`、`prompt_prefix`、`bgm_path`。也就是说，用户既可以让系统自动生成，也可以逐项指定声音、媒体模型、模板和 BGM。

### Core 门面

`PixelleVideoCore` 做的事很简单：把所有服务初始化好，并给 pipeline 统一使用。

它持有的对象包括：

```text
llm
tts
media
api_media
image_analysis
video_analysis
api_asset_analysis
video
frame_processor
persistence
history
pipelines
```

一个值得注意的点是 ComfyKit 的 lazy 初始化。项目不会启动时就连接所有工作流，而是在真正需要时创建 ComfyKit。如果配置变了，还会根据配置 hash 重建。这对本地工具很实用，因为用户经常改 API key、RunningHub key、ComfyUI 地址。

### Pipeline

`LinearVideoPipeline` 是模板方法，生命周期固定：

```text
setup_environment
generate_content
determine_title
plan_visuals
initialize_storyboard
produce_assets
post_production
finalize
```

`StandardPipeline` 则把这个抽象落到具体步骤：

1. 创建 `output/{task_id}`。
2. 如果 `mode=generate`，用 LLM 根据主题生成旁白。
3. 如果 `mode=fixed`，按段落、行或句子切分已有脚本。
4. 自动或手动确定标题。
5. 根据模板类型判断是否需要图片或视频素材。
6. 为每段旁白生成 image prompt 或 video prompt。
7. 初始化 `Storyboard` 和 `StoryboardFrame`。
8. 对每个 frame 执行 TTS、媒体生成、HTML 渲染和片段生成。
9. 如果走 RunningHub，按 `runninghub_concurrent_limit` 控制并发。
10. 拼接片段，加 BGM，保存 metadata 和 storyboard。

这里的重点不是它用了什么模型，而是它把“整条视频”拆成了“多段 frame”。短视频生产的可控性基本都来自这里。

## 实现细节

这一节细一点讲。

### 一个 frame 是怎么被做出来的

`FrameProcessor` 是整套系统最关键的类之一。它处理的不是整条视频，而是一段旁白对应的画面。

流程是：

```text
frame.narration
  -> TTS 生成 audio
  -> 读取 audio duration
  -> 生成 image 或 video media
  -> HTML 模板合成 composed image
  -> 生成 video segment
```

第一步是 `_step_generate_audio`。它拿到 `frame.narration`，生成类似 `frames/xx_audio.mp3` 的音频文件，并读取时长。

第二步是 `_step_generate_media`。如果模板是 `image_`，就生成图片；如果模板是 `video_`，就生成视频。这里有一个很实际的设计：视频工作流会把 TTS 音频时长传给模型，作为目标视频时长。否则就会出现旁白 8 秒、视频 5 秒的尴尬。

第三步是 `_step_compose_frame`。HTML 模板会被填入标题、正文、图片、序号、自定义变量，然后用 Playwright 截图成一张合成图。

第四步是 `_step_create_video_segment`。如果是图片模板，就用静态图加旁白音频生成视频片段；如果是视频模板，就把 AI 视频和 HTML overlay、旁白音频合在一起。

这套 frame 机制很适合图文口播、知识卡片、情绪短句、图书摘抄这类短视频。它不适合特别复杂的剪辑，但足够覆盖大量自动化内容场景。

### HTML 模板怎么工作

`HTMLFrameGenerator` 用 Playwright 渲染 HTML，再截图。

模板里会有两类信息。

一类是媒体尺寸：

```html
<meta name="template:media-width" content="1024">
<meta name="template:media-height" content="1024">
```

这决定 AI 图片或视频要生成多大。模板所在目录，比如 `1080x1920`，决定最终视频画布。

另一类是变量：

```text
{{title}}
{{text}}
{{image}}
{{index}}
{{accent_color:color=#ffffff}}
{{title_font_size:number=64}}
```

项目会解析这些变量，把它们暴露给 UI 或 API。于是一个 HTML 文件既是渲染模板，也是一个简易的配置表单。

这个方案很土，也很好用。HTML/CSS 的表达力足够强，截图之后又很容易接进 FFmpeg。对于自研平台来说，我甚至会建议先沿用这个路线，不要急着发明视频模板 DSL。DSL 很容易让人兴奋，也很容易把项目拖进泥里。

但 Pixelle-Video 这里还少一层模板 manifest。现在模板的类型、变量、媒体尺寸、适用场景主要散落在文件名、meta 和 HTML 变量里。等模板多起来之后，应该有一个统一的模板注册表，记录适用平台、内容类型、品牌约束、默认模型、质量评分和版本。

### 工作流怎么被发现

工作流由 `ComfyBaseService` 扫描。

默认路径是：

```text
workflows/<source>/*.json
```

用户自定义路径是：

```text
data/workflows/<source>/*.json
```

自定义目录优先于默认目录。TTS 只扫 `tts_*.json`，媒体生成扫 `image_*.json` 和 `video_*.json`。

所以文件名不是装饰，它直接参与路由。

```text
tts_edge.json       -> TTS
image_flux.json     -> image
video_wan2.2.json   -> video
```

RunningHub 工作流会把 `workflow_id` 交给 ComfyKit。Selfhost 工作流会把本地 JSON 路径交给 ComfyKit。直连 API 则走 `api/{provider}/{model}` 这条分支。

这让扩展变得很快：加一个 JSON 文件，起一个符合规则的名字，UI 就能发现它。代价也很明显：它缺少正式 schema。一个工作流的输入参数、输出类型、耗时、成本、GPU 需求、失败重试策略，都不在一个强约束的元数据里。

### 直连 API 怎么适配

直连 API 的入口是 `APIProviderMediaService`。

调用路径大概是：

```text
UI / Pipeline
  -> media_workflow = api/dashscope/wan2.7-t2v
  -> MediaService
  -> APIProviderMediaService.resolve_workflow()
  -> _generate_image() 或 _generate_video()
  -> provider adapter
```

它还会做几件实际但容易漏的事：

1. 根据模型能力限制 duration。
2. 根据 provider 映射 resolution、ratio、seed、watermark 等参数。
3. 如果命中内容审核错误，调用 LLM 把 prompt 改写得更中性，然后重试一次。

第三点挺有意思。很多 AI 视频产品在 demo 时看起来顺滑，真正跑批量任务时会遇到各种审核失败、参数不合法、结果超时。Pixelle-Video 至少开始处理这类边角料了。

但它还没有完整的模型路由。一个成熟平台里，模型选择应该考虑成本、速度、质量、平台风格、可用性、历史成功率、fallback 和预算。现在 Pixelle-Video 更像是“用户选哪个，我就调哪个”。

### Prompt 模块

Prompt 被拆成了多个文件：

| 文件 | 用途 |
|---|---|
| `topic_narration.py` | 根据主题生成旁白 |
| `content_narration.py` | 根据已有内容生成旁白 |
| `title_generation.py` | 生成标题 |
| `image_generation.py` | 生成图片 prompt 和风格预设 |
| `video_generation.py` | 生成视频 prompt |
| `style_conversion.py` | 做风格改写 |
| `asset_script_generation.py` | 根据素材分析结果生成脚本 |

这已经比把 prompt 全塞进业务代码好很多。

不过它还不是完整的 prompt 工程。缺的东西包括版本、评测集、失败样例、行业策略、平台策略、违规案例、模型差异适配。对个人开源工具来说可以接受；对公司自研平台来说，这一层迟早要产品化。

### TTS

TTS 有两种模式。

`local` 模式走 Edge TTS，默认 voice 是 `zh-CN-YunjianNeural`，默认 speed 是 `1.2`。这很适合开箱即用。

`comfyui` 模式通过 ComfyKit 执行 `tts_*.json`，可以接 IndexTTS2、Spark TTS 这类工作流。

TTS 参数里有 `text`、`workflow`、`voice`、`speed`、`ref_audio`、`output_path`。这说明它不只想做普通旁白，也为参考音频、数字人声音、声音克隆这类能力留了入口。

### FFmpeg 合成

`VideoService` 基本承包了最后一公里。

它负责：

```text
concat_videos
merge_audio_video
overlay_image_on_video
create_video_from_image
add_bgm
trim_video_to_duration
pad_video_to_duration
```

多片段拼接默认用 demuxer，速度快，但要求格式一致。格式不一致时可以走 filter concat。图片加音频会生成静态视频，视频加音频会根据时长 pad 或 trim。BGM 用 `amix` 混音，并 loop 到视频时长。

这部分没什么玄学，就是大量媒体工程里的苦活。一个短视频工具如果没有把这里封好，前面的模型效果再惊艳，最后也会被音画不同步、BGM 太响、片段黑屏这些问题拖垮。

### 任务和历史

API 异步任务管理是内存态：

```text
_tasks: Dict[str, Task]
_task_futures: Dict[str, asyncio.Task]
```

状态包括：

```text
pending
running
completed
failed
cancelled
```

产物持久化是文件态：

```text
output/{task_id}/
  final.mp4
  metadata.json
  storyboard.json
  frames/
```

`PersistenceService` 还会维护 `.index.json`，支持任务分页、统计、删除和重建索引。

这里是一个明显短板：任务状态和历史产物不是同一个可靠系统。服务重启后，内存里的异步任务会丢。对本地工具问题不大，对平台就不行。平台至少要有队列、worker、DB 状态、重试、幂等、超时、取消、成本记录和 trace。

## 优缺点

先说优点。

第一，它真的把链路跑通了。很多 AI 视频项目停在“调一个模型”层面，Pixelle-Video 往后走了一步，把文案、声音、画面、模板、合成都串起来了。

第二，抽象边界比较清楚。Core、Pipeline、FrameProcessor、MediaService、APIProviderMediaService、VideoService、PersistenceService 各管一段。以后想换模型、换模板、换合成策略，不至于全项目大出血。

第三，HTML 模板路线很务实。做知识卡片、口播字幕、图文视频时，HTML/CSS 比很多复杂编辑器 SDK 更轻。前端同学也能参与模板生产。

第四，它没有只押一个供应商。ComfyUI、RunningHub、直连 API 都能接。对开源项目来说，这比“绑定某一家模型”更耐用。

第五，API provider metadata 已经露出模型路由的雏形。它意识到视频模型有不同能力，而不只是不同名字。

缺点也很清楚。

第一，任务系统弱。内存任务适合 demo，不适合生产。没有可靠队列、持久状态、分布式 worker、重试、幂等和成本审计。

第二，内容策略弱。它能生成视频，但不回答“这个账号为什么要发这条”。选题、受众、平台、账号历史、竞品差异，这些都不在核心系统里。

第三，素材资产层弱。`asset_based` 能上传素材并分析，但它不是素材库。没有版权字段、标签体系、人物/品牌实体、去重、复用、质量评分和权限。

第四，模板治理弱。31 个模板能撑起风格墙，但缺少 manifest、版本、适配规则、质量评估和品牌一致性约束。

第五，模型路由还薄。现在更多是“用户选模型”，不是系统按成本、速度、质量、失败率和平台风格自动调度。

第六，质量评估不足。脚本好不好，画面是否一致，字幕是否可读，口播是否自然，节奏是否拖，审核风险高不高，这些都没有形成系统性的 eval。

第七，没有发布和数据闭环。它负责生成 `final.mp4`，但不负责排期发布、账号管理、评论反馈、播放数据和下一轮创作调整。

所以我的判断是：Pixelle-Video 是一个很好的生成引擎样板，但还不是完整的商业短视频创作平台。

## 自研类似工具的实现思路

如果我们要自研一个类似工具，我不会从“再做一个一键生成视频”开始。这个入口大家都会做，差异不大。

我会把系统拆成九层。

### 1. 账号策略层

每个账号要有自己的 profile：

```text
账号定位
目标人群
内容边界
语气风格
禁区
历史爆款
历史失败样例
平台规则
商业目标
```

没有这一层，系统只能生成“像短视频的东西”，但不知道为什么要生成它。

### 2. 选题策略层

短视频不是从脚本开始的，是从选题开始的。

选题层应该接入：

```text
热点
竞品账号
行业新闻
用户问题
历史数据
内容日历
老板或运营临时需求
```

它的输出不是一句主题，而是一份 brief：为什么做、给谁看、核心观点是什么、平台怎么切、风险在哪里。

### 3. 素材资产层

素材库要比文件夹强得多。

至少需要：

```text
素材类型：图片、视频、音频、文档、网页、竞品链接
版权状态：自有、授权、可商用、待确认、禁用
实体识别：人物、品牌、产品、地点
标签：主题、行业、风格、情绪、平台
质量分：清晰度、可用性、是否过期
引用关系：哪条视频用过，效果如何
```

素材资产层是很多 AI 创作平台的分水岭。没有素材库，系统每次都像失忆一样从 prompt 开始；有了素材库，它才像一个在积累经验的编辑部。

### 4. 创作规划层

这里负责把 brief 变成可生产的 storyboard。

输出应该包括：

```text
标题候选
开头钩子
完整脚本
分镜列表
每镜旁白
每镜画面意图
模板选择
模型选择
字幕策略
BGM 策略
```

Pixelle-Video 的 `StoryboardFrame` 很值得借鉴。我们可以把它加强成更完整的创作中间表示。

### 5. 生成能力层

这一层可以吸收 Pixelle-Video 的做法：

```text
LLM
TTS
图片模型
视频模型
ComfyUI
RunningHub
HTML 模板
FFmpeg
```

但 provider metadata 要更完整：

```text
能力类型
输入模态
输出模态
支持时长
支持比例
平均耗时
平均成本
失败率
审核风险
适合内容类型
fallback 模型
```

模型路由不要只做配置项，要做成策略系统。

### 6. 模板和品牌层

HTML 模板可以继续用，但要加治理。

每个模板应该有 manifest：

```text
模板 ID
版本
适用平台
适用内容类型
必填变量
可选变量
媒体尺寸
安全区域
品牌色
字体
字幕规则
示例截图
质量评分
```

模板不是越多越好。真正有价值的是“知道什么时候该用哪个模板”。

### 7. 质量评估层

每条视频生成后都要过评估。

可以分几类：

```text
脚本评估：是否清楚、有钩子、有信息密度
画面评估：是否相关、主体是否清晰、风格是否一致
音频评估：音量、语速、停顿、发音
字幕评估：是否遮挡、是否可读、是否错字
平台风险：违规词、版权、医疗金融等高风险表达
账号一致性：是否符合账号人设和历史风格
```

这里不一定全靠 LLM。规则、OCR、VLM、音频分析、人工抽检都应该混用。

### 8. 任务调度层

生产系统要当成 job system 做。

最低配置：

```text
任务表
队列
worker
重试
幂等 key
超时控制
取消任务
分段进度
成本记录
日志和 trace
产物索引
```

Pixelle-Video 的内存任务可以作为本地版起点，但平台版必须换成可靠任务系统。

### 9. 发布和反馈层

最后一层是发布运营。

包括：

```text
多平台发布
排期
标题和封面 A/B 测试
播放、完播、互动、转粉数据回流
评论和私信摘要
复盘报告
下一轮选题建议
```

这是自研平台最应该拉开差距的地方。生成一条视频已经不稀奇了，难的是持续知道下一条应该怎么做。

我的结论很简单：Pixelle-Video 值得学，但不要只学它的“生成”。它真正有价值的是那条能跑通的生产链路。

如果我们自研，底层可以借鉴它的 frame pipeline、HTML 模板、工作流适配和 FFmpeg 合成；上层要补账号策略、素材资产、模型路由、质量评估、任务治理和数据反馈。这样做出来的东西才不是一个更漂亮的 demo，而是能被运营团队长期使用的内容生产系统。
