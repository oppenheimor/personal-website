---
title: 把 web-access CDP 改造成稳定的定时任务浏览器
date: 2026-07-02T00:00:00.000+00:00
lang: zh
duration: 13min
author: 沈佳棋
---

## 背景

我最近在用 Hermes 搭一套内容创作 Agent 团队。里面有几个角色专门负责收集热点、竞品内容和后台数据，比如 `trendfox`、`sourcebee`、`dataloop`。

这些活儿有一个共同点：普通 HTTP 请求不太够用。小红书、公众号、创作后台这类页面经常需要登录态、真实浏览器环境，甚至还得等页面渲染完。Jina、curl、普通网页抓取能处理一部分文章页，但碰到平台后台和反爬页面，很快就露怯。

所以我一直用 `web-access` 的 CDP 模式。它直接接管本机 Chrome，能复用登录态，也能通过 `/eval` 读取 DOM、点击、滚动、截图。人坐在电脑前时，这套很好用。

问题出在定时任务上。

我遇到两个麻烦：

1. CDP 连接不稳定，有时 `9222` 端口还在，但 proxy 连不上。
2. Chrome 弹出远程调试授权窗口时，需要手动点允许。

这对自动化很要命。半夜定时任务醒来，Chrome 在等一个没人点的「允许」，Agent 就只能干站着。很礼貌，但没产出。

## 解决思路

这次最后没有继续修修补补 `chrome://inspect` 那条路，而是把浏览器环境拆成两层：

1. 独立浏览器：用 Chrome for Testing 或 Chromium，不和日常 Chrome 共用 app。
2. CDP Proxy：常驻在本机，只连接这个独立浏览器。

日常 Chrome 继续日常用。自动化浏览器专门跑任务。两个世界隔开，心里会踏实很多。

关键变化有两个。

第一，用命令行参数启动独立浏览器：

```bash
--remote-debugging-address=127.0.0.1
--remote-debugging-port=9333
--user-data-dir=<chrome_profile_dir>
```

这里我故意不用 9222。日常 Chrome 可能已经占着 9222，或者未来某天又被 `chrome://inspect` 勾选模式影响。自动化浏览器单独用 9333，少一层纠缠。

第二，修了 `web-access` 的 `cdp-proxy.mjs`。之前 proxy 会优先读 `DevToolsActivePort`，但这个文件可能指向旧的 Browser WebSocket UUID。Chrome 重启以后，UUID 会变，proxy 拿着旧地址去敲门，自然被拒。

更稳的做法是先读：

```bash
curl -s http://127.0.0.1:9333/json/version
```

这里会返回当前真实的 `webSocketDebuggerUrl`。拿这个地址连 Chrome，才是当下那扇门。

## 配置过程

下面的命令都做了脱敏。尖括号里的内容需要替换成你自己的。

需要你自己提供的信息：

```text
<chrome_app_path>       Chrome 可执行文件路径
<chrome_profile_dir>    专用浏览器 profile 目录
<start_url>             启动后默认打开的页面，例如某个平台创作中心
<node_bin>              Node.js 22+ 的路径
<cdp_proxy_path>        web-access 的 cdp-proxy.mjs 路径
<launch_agent_dir>      macOS 用户级 LaunchAgents 目录
```

不要提供这些东西：

```text
密码
Cookie
短信验证码
平台 token
浏览器 session 复制值
账号恢复手机号
```

登录态应该留在专用 Chrome profile 里，而不是写进脚本或文档。

这里的 `<chrome_app_path>` 最好指向独立浏览器，比如 `Google Chrome for Testing.app` 或 `Chromium.app`。不要直接拿日常的 `/Applications/Google Chrome.app` 去做常驻自动化。这个坑我已经踩过一次，不好看。

### 1. 启动独立浏览器

先创建一个专用 profile 目录：

```bash
mkdir -p <chrome_profile_dir>
```

然后启动浏览器：

```bash
<chrome_app_path> \
  --remote-debugging-address=127.0.0.1 \
  --remote-debugging-port=9333 \
  --user-data-dir=<chrome_profile_dir> \
  --no-first-run \
  --no-default-browser-check \
  <start_url>
```

第一次打开后，手动登录一次目标平台。登录完成后，关闭终端不应该影响登录态，因为它保存在 `<chrome_profile_dir>` 里。

### 2. 验证浏览器调试端口

执行：

```bash
curl -s http://127.0.0.1:9333/json/version
```

正常情况下会看到类似这样的字段：

```json
{
  "Browser": "Chrome/148.0.x",
  "Protocol-Version": "1.3",
  "webSocketDebuggerUrl": "ws://127.0.0.1:9333/devtools/browser/<uuid>"
}
```

这里最重要的是 `webSocketDebuggerUrl`。如果这个接口返回 404，通常说明你还在用 `chrome://inspect` 勾选模式，或者启动的不是这套独立浏览器。

### 3. 修正 cdp-proxy 的连接逻辑

`cdp-proxy.mjs` 里需要优先走 `/json/version`：

```js
function discoverBrowserWsUrl(port) {
  return new Promise((resolve) => {
    const req = http.get({
      hostname: '127.0.0.1',
      port,
      path: '/json/version',
      timeout: 2000,
    }, (res) => {
      let data = ''
      res.setEncoding('utf8')
      res.on('data', chunk => data += chunk)
      res.on('end', () => {
        if (res.statusCode !== 200) {
          resolve(null)
          return
        }
        try {
          const json = JSON.parse(data)
          const wsUrl = json.webSocketDebuggerUrl
          resolve(typeof wsUrl === 'string' && wsUrl.startsWith('ws://') ? wsUrl : null)
        }
        catch {
          resolve(null)
        }
      })
    })
    req.on('timeout', () => { req.destroy(); resolve(null) })
    req.on('error', () => resolve(null))
  })
}
```

核心就一句：先相信 Chrome 自己告诉你的地址，再考虑 `DevToolsActivePort` 兜底。

### 4. 用 launchd 托管独立浏览器

创建：

```text
<launch_agent_dir>/com.hermes.chrome-xhs.plist
```

内容示例：

```xml
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE plist PUBLIC "-//Apple//DTD PLIST 1.0//EN" "http://www.apple.com/DTDs/PropertyList-1.0.dtd">
<plist version="1.0">
<dict>
  <key>Label</key>
  <string>com.hermes.chrome-xhs</string>

  <key>ProgramArguments</key>
  <array>
    <string>&lt;chrome_app_path&gt;</string>
    <string>--remote-debugging-address=127.0.0.1</string>
    <string>--remote-debugging-port=9333</string>
    <string>--user-data-dir=&lt;chrome_profile_dir&gt;</string>
    <string>--no-first-run</string>
    <string>--no-default-browser-check</string>
    <string>&lt;start_url&gt;</string>
  </array>

  <key>RunAtLoad</key>
  <true/>

  <key>KeepAlive</key>
  <true/>

  <key>StandardOutPath</key>
  <string>/tmp/hermes-chrome-xhs.log</string>

  <key>StandardErrorPath</key>
  <string>/tmp/hermes-chrome-xhs.err.log</string>
</dict>
</plist>
```

加载服务：

```bash
launchctl bootstrap gui/$(id -u) <launch_agent_dir>/com.hermes.chrome-xhs.plist
launchctl kickstart -k gui/$(id -u)/com.hermes.chrome-xhs
```

### 5. 用 launchd 托管 CDP Proxy

再创建：

```text
<launch_agent_dir>/com.hermes.cdp-proxy.plist
```

内容示例：

```xml
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE plist PUBLIC "-//Apple//DTD PLIST 1.0//EN" "http://www.apple.com/DTDs/PropertyList-1.0.dtd">
<plist version="1.0">
<dict>
  <key>Label</key>
  <string>com.hermes.cdp-proxy</string>

  <key>ProgramArguments</key>
  <array>
    <string>&lt;node_bin&gt;</string>
    <string>&lt;cdp_proxy_path&gt;</string>
  </array>

  <key>RunAtLoad</key>
  <true/>

  <key>KeepAlive</key>
  <true/>

  <key>EnvironmentVariables</key>
  <dict>
    <key>CDP_CHROME_PORT</key>
    <string>9333</string>
    <key>CDP_PROXY_PORT</key>
    <string>3456</string>
  </dict>

  <key>StandardOutPath</key>
  <string>/tmp/hermes-cdp-proxy.log</string>

  <key>StandardErrorPath</key>
  <string>/tmp/hermes-cdp-proxy.err.log</string>
</dict>
</plist>
```

加载服务：

```bash
launchctl bootstrap gui/$(id -u) <launch_agent_dir>/com.hermes.cdp-proxy.plist
launchctl kickstart -k gui/$(id -u)/com.hermes.cdp-proxy
```

## 效果

改完以后，检查链路变得很明确。

先看 Chrome：

```bash
curl -s http://127.0.0.1:9333/json/version
```

再看 proxy：

```bash
curl -s http://127.0.0.1:3456/health
```

正常结果里应该有：

```json
{
  "status": "ok",
  "connected": true,
  "chromePort": 9333
}
```

最后看页面：

```bash
curl -s http://127.0.0.1:3456/targets
```

我这次验证时，`targets` 能看到平台创作中心，`/eval` 也能读出后台页面里的账号信息、菜单和数据总览。到这里，才算不是“端口通了”，而是真正能干活。

## 日后的使用方式

以后 Agent 要用 `web-access`，不用再让人去点 Chrome 的远程调试授权。

日常只需要确认两件事：

```bash
curl -s http://127.0.0.1:9333/json/version
curl -s http://127.0.0.1:3456/health
```

需要打开新页面时，用 `web-access` proxy：

```bash
curl -s "http://127.0.0.1:3456/new?url=<target_url>"
```

需要读取页面内容时，用 `/eval`：

```bash
curl -s -X POST "http://127.0.0.1:3456/eval?target=<target_id>" \
  --data-binary 'JSON.stringify({ title: document.title, text: document.body.innerText.slice(0, 1000) })'
```

需要重启服务：

```bash
launchctl kickstart -k gui/$(id -u)/com.hermes.chrome-xhs
launchctl kickstart -k gui/$(id -u)/com.hermes.cdp-proxy
```

如果平台登录态过期，就打开那套独立浏览器，手动登录一次。不要把 Cookie 复制出来，也不要让 Agent 记住验证码。该人工的地方就人工，省得以后踩更大的坑。

## 日后的问题排查方式

我会按这个顺序查。

### 1. 独立浏览器是否在监听 9333

```bash
lsof -nP -iTCP:9333 -sTCP:LISTEN
curl -s http://127.0.0.1:9333/json/version
```

如果 `lsof` 没结果，说明独立浏览器没起来。重启：

```bash
launchctl kickstart -k gui/$(id -u)/com.hermes.chrome-xhs
```

如果 `lsof` 有结果，但 `/json/version` 是 404，大概率不是命令行启动的独立浏览器，而是又回到了 `chrome://inspect` 勾选模式。

### 2. proxy 是否连上 Chrome

```bash
curl -s http://127.0.0.1:3456/health
```

如果 `connected` 不是 `true`，看日志：

```bash
tail -80 /tmp/hermes-cdp-proxy.log
tail -80 /tmp/hermes-cdp-proxy.err.log
```

看到旧 UUID、`non-101 status code`、`连接断开` 这类信息时，先确认 `cdp-proxy.mjs` 是否已经优先读取 `/json/version.webSocketDebuggerUrl`。

### 3. 页面是否真的可读

只看 `health` 不够。继续看：

```bash
curl -s http://127.0.0.1:3456/targets
```

再用 `/eval` 读一次标题或正文：

```bash
curl -s -X POST "http://127.0.0.1:3456/eval?target=<target_id>" \
  --data-binary 'document.title'
```

如果页面标题能读到，但正文是登录页，那就是平台登录态过期。去独立浏览器里手动登录，不要试图把验证码自动化。

### 4. launchd 服务状态

```bash
launchctl print gui/$(id -u)/com.hermes.chrome-xhs
launchctl print gui/$(id -u)/com.hermes.cdp-proxy
```

重点看 `state`、`pid`、`last exit code`。`state = running` 且有 pid，说明服务至少活着。

## 总结

这次改造的重点不是把 Chrome 启动命令写得更长，而是换了一个心智模型。

临时网页操作可以接管日常 Chrome。定时任务不行。定时任务需要自己的浏览器 app、自己的 profile、自己的端口和自己的常驻进程。少隔离一层，早晚会从别的地方找回来。

最后这套结构很简单：

```text
Chrome for Testing profile 保存登录态
Chrome for Testing 监听 127.0.0.1:9333
cdp-proxy 监听 127.0.0.1:3456
launchd 负责守住它们
Hermes cron 只管调用 web-access
```

等这层稳定下来，`trendfox` 可以定时看趋势，`sourcebee` 可以收资料，`dataloop` 可以读后台数据。Agent 团队终于不用在半夜等人点按钮了。老实说，这一步比多写十个 prompt 都值。
