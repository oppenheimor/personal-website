---
title: 给国内云服务器配置 GitHub 代理
date: 2026-07-01T00:00:00.000+00:00
lang: zh
duration: 8min
author: 沈佳棋
---

## 背景

国内云服务器访问 GitHub 经常不稳定。小文件还能忍，大一点的 release 包基本就是龟速下载。我的服务器直连 GitHub 时，下载 Mihomo 的 17 MB release 包，60 秒只下了 1 MB 左右，速度大概 16 KB/s。

这台服务器主要需要解决的是拉代码、下载依赖、访问 GitHub release。没必要把整台机器都改成透明代理，也没必要把代理端口暴露给公网。目标很简单：在服务器本机启动一个代理端口，只给 `git`、`curl` 这类命令行工具使用。

## 选型

本机已经在用 Clash Verge，并且有可用的订阅地址。服务器上不需要安装 Clash Verge 这种图形客户端，只需要跑代理内核。

这里选 Mihomo，原因很实际：

1. Clash/Mihomo 格式的订阅可以直接用，省掉订阅转换。
2. Linux 上可以作为普通二进制运行。
3. 用 systemd 托管后，重启服务器也能自动恢复。
4. 只监听 `127.0.0.1`，不会变成公开代理。

Mihomo 这个名字看起来不太像严肃基础设施，但内核本身能干活。最后测到 GitHub release 下载速度从约 16 KB/s 提升到约 1.2 MB/s，`git ls-remote` 从 17 秒降到 0.65 秒。结果比名字可靠。

## 需要提前准备的信息

下面这些信息需要自己替换，文章里不会写真实值。

```text
<server_user>      服务器用户名，例如 ubuntu、root、heqi
<server_host>      服务器 IP 或域名
<subscription_url> Clash/Mihomo 订阅地址
```

如果订阅地址已经发到聊天、工单或公开文档里，建议部署完以后去服务商后台重置一次。订阅地址基本等同于账号钥匙。

## 第一步：测试服务器直连 GitHub

先看一下服务器当前访问 GitHub 的速度，后面方便对比。

```bash
curl -L --connect-timeout 10 --max-time 60 -o /tmp/mihomo-test.gz \
  -w 'http=%{http_code} total=%{time_total}s size=%{size_download} speed=%{speed_download}B/s\n' \
  https://github.com/MetaCubeX/mihomo/releases/download/v1.19.27/mihomo-linux-amd64-v1.19.27.gz

time git ls-remote https://github.com/git/git.git HEAD
```

如果 release 下载只有几十 KB/s，或者 `git ls-remote` 要十几秒，这台机器就很适合加本地代理。

## 第二步：安装 Mihomo

先确认服务器架构：

```bash
uname -m
```

`x86_64` 用 amd64 包，`aarch64` 或 `arm64` 用 arm64 包。

如果服务器拉 GitHub 很慢，可以在本机先下载，再传到服务器。下面以 amd64 为例：

```bash
curl -L -o mihomo.gz \
  https://github.com/MetaCubeX/mihomo/releases/download/v1.19.27/mihomo-linux-amd64-v1.19.27.gz

scp mihomo.gz <server_user>@<server_host>:/tmp/mihomo.gz
```

登录服务器后安装：

```bash
gzip -dc /tmp/mihomo.gz > /tmp/mihomo
sudo install -m 755 /tmp/mihomo /usr/local/bin/mihomo

mihomo -v
```

看到版本号就说明安装好了。

## 第三步：生成正式配置

这一步做几件事：下载订阅，清理不兼容字段，只监听本机端口，并把 GEO 数据源改成服务器能顺利访问的地址。

```bash
tmp=$(mktemp -d)

read -r -s -p "Paste subscription URL: " SUB_URL; echo
curl -fsSL --connect-timeout 10 --max-time 30 "$SUB_URL" -o "$tmp/raw.yaml"

{
  cat <<'EOF'
mixed-port: 7890
allow-lan: false
bind-address: 127.0.0.1
geo-auto-update: false
geox-url:
  geoip: "https://testingcf.jsdelivr.net/gh/MetaCubeX/meta-rules-dat@release/geoip.dat"
  geosite: "https://testingcf.jsdelivr.net/gh/MetaCubeX/meta-rules-dat@release/geosite.dat"
  mmdb: "https://testingcf.jsdelivr.net/gh/MetaCubeX/meta-rules-dat@release/country.mmdb"
  asn: "https://testingcf.jsdelivr.net/gh/MetaCubeX/meta-rules-dat@release/GeoLite2-ASN.mmdb"
EOF

  awk '
    /^[^[:space:]].*:/ { skip=0 }
    /^geox-url:/ { skip=1; next }
    /^(mixed-port|port|socks-port|redir-port|tproxy-port|allow-lan|bind-address|geo-auto-update|global-client-fingerprint):/ { next }
    skip && /^[[:space:]]/ { next }
    { print }
  ' "$tmp/raw.yaml"
} > "$tmp/config.yaml"

sudo mkdir -p /etc/mihomo
sudo chmod 700 /etc/mihomo
sudo install -m 600 "$tmp/config.yaml" /etc/mihomo/config.yaml

sudo /usr/local/bin/mihomo -t -d /etc/mihomo

rm -rf "$tmp"
```

如果最后看到：

```text
configuration file /etc/mihomo/config.yaml test is successful
```

说明配置可以用。

这里有两个小处理值得保留：

1. `bind-address: 127.0.0.1` 和 `allow-lan: false`：只允许服务器本机访问代理，不开放公网。
2. 删除 `global-client-fingerprint`：Mihomo 新版本已经移除了这个全局字段，订阅里如果还带着它，会导致配置测试失败。

## 第四步：用 systemd 托管 Mihomo

手动运行 `mihomo -d /etc/mihomo` 也可以，但 SSH 一断、进程一挂、服务器一重启，它就没了。systemd 的作用就是让系统帮你照看这个进程。

创建服务文件：

```bash
sudo tee /etc/systemd/system/mihomo.service >/dev/null <<'EOF'
[Unit]
Description=mihomo Daemon
After=network.target

[Service]
Type=simple
ExecStart=/usr/local/bin/mihomo -d /etc/mihomo
Restart=always
RestartSec=5
LimitNOFILE=1048576

[Install]
WantedBy=multi-user.target
EOF
```

启动并设置开机自启：

```bash
sudo systemctl daemon-reload
sudo systemctl enable --now mihomo
sudo systemctl status mihomo --no-pager
```

状态里看到 `active (running)` 就可以继续。

常用维护命令：

```bash
sudo systemctl restart mihomo
sudo systemctl status mihomo --no-pager
sudo journalctl -u mihomo -f
```

## 第五步：确认代理端口没有暴露到公网

```bash
ss -ltnp | grep 7890 || true
```

期望看到类似：

```text
127.0.0.1:7890
```

如果是 `0.0.0.0:7890`，先停下来检查配置。这个端口不应该对外开放。

## 第六步：测试 GitHub 代理效果

```bash
curl -I -x http://127.0.0.1:7890 https://github.com
```

看到下面这种结果，说明代理通了：

```text
HTTP/1.1 200 Connection established
HTTP/2 200
```

再测一下 release 下载速度：

```bash
curl -L --connect-timeout 10 --max-time 60 -o /tmp/github-test.gz \
  -x http://127.0.0.1:7890 \
  -w 'http=%{http_code} total=%{time_total}s size=%{size_download} speed=%{speed_download}B/s\n' \
  https://github.com/MetaCubeX/mihomo/releases/download/v1.19.27/mihomo-linux-amd64-v1.19.27.gz
```

实测代理后速度约 1.2 MB/s。和直连的十几 KB/s 比，已经不是一个量级。

## 第七步：让 Git 使用本机代理

```bash
git config --global http.proxy http://127.0.0.1:7890
git config --global https.proxy http://127.0.0.1:7890
```

验证：

```bash
time git ls-remote https://github.com/git/git.git HEAD
```

实测从 17 秒左右降到了 0.65 秒左右。这个结果已经足够说明问题。

如果以后要撤销：

```bash
git config --global --unset http.proxy
git config --global --unset https.proxy
```

## 总结

这套方案没有把服务器改成全局透明代理，只是在本机开了一个 `127.0.0.1:7890`。Git、curl、其他命令行工具需要时走它，不需要时继续直连。

最后的状态是：

```text
Mihomo 二进制：/usr/local/bin/mihomo
配置文件：/etc/mihomo/config.yaml
代理端口：127.0.0.1:7890
托管方式：systemd
Git 代理：http://127.0.0.1:7890
```

对国内云服务器来说，这样已经够用了。结构简单，出问题也好查。真正要注意的只有两件事：订阅地址别泄露，代理端口别监听公网。

参考文档：

1. [Mihomo 配置文档](https://wiki.metacubex.one/en/config/general/)
2. [Mihomo systemd 运行文档](https://wiki.metacubex.one/en/startup/service/)
