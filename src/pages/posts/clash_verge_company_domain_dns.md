---
title: Clash Verge 里公司域名直连失败的排查
date: 2026-07-01T00:00:00.000+00:00
lang: zh
duration: 7min
author: 沈佳棋
---

## 背景

本机用了 Clash Verge，平时代理和分流都正常。公司内部服务有一批统一的域名，比如：

```text
*.corp.example.com
```

我希望这些公司域名全部直连，于是在规则里加了：

```yaml
- DOMAIN-SUFFIX,corp.example.com,DIRECT
```

看起来很合理，但访问公司服务时还是失败。Clash Verge 日志长这样：

```text
[TCP] dial DIRECT (match DomainSuffix/corp.example.com) 127.0.0.1:58226 --> mirror.corp.example.com:443 error: dns resolve failed: couldn't find ip
[TCP] dial DIRECT (match DomainSuffix/corp.example.com) 127.0.0.1:59511 --> hipilot.corp.example.com:443 error: dns resolve failed: couldn't find ip
```

第一眼很容易误判成“公司域名还是走代理了”。其实不是。

日志里的这段才是关键：

```text
dial DIRECT (match DomainSuffix/corp.example.com)
```

它说明规则已经命中，出站也是 `DIRECT`。真正的问题在后面：

```text
dns resolve failed: couldn't find ip
```

也就是说，Clash Verge 准备直连，但它自己没有解析出公司域名的 IP。

## 需要提前准备的信息

下面这些值需要按自己的环境替换，不要直接复制文中的占位符。

```text
<corp_domain>       公司域名后缀，例如 corp.example.com
<corp_dns_ip>       公司 DNS 服务器 IP，例如 10.x.x.x
<internal_host>     某个公司内部域名，例如 mirror.corp.example.com
```

如果你的 DNS 配置里有带 UUID、token 或随机路径的 DoH 地址，也别贴到公开文档里。它未必等同于代理订阅密钥，但仍然是敏感配置。

## 排查思路

这类问题不要一上来就改规则。先分层看。

第一层，看 Clash 规则有没有命中。

如果日志里出现：

```text
match DomainSuffix/<corp_domain>
dial DIRECT
```

那说明 `DOMAIN-SUFFIX` 规则没有问题。请求已经被分到直连。

第二层，看系统 DNS 能不能解析公司域名。

```bash
dig +short <internal_host>
```

如果能返回内网 IP，说明 Mac 系统 DNS 是通的。

第三层，看 Mac 用的是哪台 DNS 解析公司域名。

```bash
scutil --dns | grep -A8 -B2 <corp_domain>
```

可能会看到类似：

```text
resolver #1
  search domain[0] : <corp_domain>
  nameserver[0] : <corp_dns_ip>
  if_index : 14 (en0)
  flags    : Request A records
```

这里的 `<corp_dns_ip>` 就是要给 Clash Verge 配的公司 DNS。

第四层，直接问公司 DNS。

```bash
dig @<corp_dns_ip> <internal_host> +short
```

如果这里能解析，但 Clash Verge 日志仍然报 `dns resolve failed`，问题就很明确了：

```text
系统 DNS 正常，公司 DNS 正常，Clash 规则也命中了 DIRECT。
Clash Verge 内核解析公司域名时，没有使用公司 DNS。
```

## 解决过程

先看一份常见的 Clash Verge DNS 配置。这里做了脱敏：

```yaml
dns:
  ipv6: false
  enable: true
  listen: 0.0.0.0:1053
  use-hosts: false
  default-nameserver:
    - 119.29.29.29
    - 223.5.5.5
    - 8.8.4.4
    - 1.0.0.1
  nameserver:
    - https://dns-provider-a.example.com/dns-query/<token>
    - https://dns-provider-b.example.com/dns-query/<token>
    - https://dns-provider-c.example.com/dns-query/<token>
  fake-ip-range: 198.18.0.1/15
  fake-ip-filter:
    - "*.lan"
    - "*.local"
```

这份配置的问题是：Clash 开了自己的 DNS，却完全不知道公司 DNS。它会拿 `nameserver` 里的 DoH 去解析 `<internal_host>`。公共 DNS 或代理服务商的 DNS 通常不知道公司内网域名，于是解析失败。

修法是在同一个 `dns:` 块里加 `nameserver-policy`，不要新建第二个 `dns:`。

```yaml
dns:
  ipv6: false
  enable: true
  listen: 0.0.0.0:1053
  use-hosts: false
  default-nameserver:
    - 119.29.29.29
    - 223.5.5.5
    - 8.8.4.4
    - 1.0.0.1
  nameserver:
    - https://dns-provider-a.example.com/dns-query/<token>
    - https://dns-provider-b.example.com/dns-query/<token>
    - https://dns-provider-c.example.com/dns-query/<token>
  nameserver-policy:
    '+.<corp_domain>':
      - <corp_dns_ip>
  fake-ip-range: 198.18.0.1/15
  fake-ip-filter:
    - "*.<corp_domain>"
    - "*.lan"
    - "*.local"
```

`nameserver-policy` 的意思是：凡是匹配 `*.<corp_domain>` 的域名，都交给 `<corp_dns_ip>` 解析。

`fake-ip-filter` 也建议加上公司域名。很多公司内部服务、VPN、证书校验、内网网关对 fake IP 不一定友好。让公司域名尽量拿真实 IP，麻烦会少一点。

再把公司域名规则放到 `rules` 里靠前的位置：

```yaml
rules:
  - DOMAIN-SUFFIX,<corp_domain>,DIRECT
  - IP-CIDR,<corp_dns_ip>/32,DIRECT,no-resolve
```

如果原来已经有第一条，就不用重复加。第二条的作用是确保访问公司 DNS 本身也走直连。

改完后，在 Clash Verge 里做这几步：

1. 保存配置覆写。
2. 重新加载配置。
3. 重启 Clash 内核。
4. 再访问公司内部域名。

可以顺手清一下 Mac DNS 缓存：

```bash
sudo dscacheutil -flushcache
sudo killall -HUP mDNSResponder
```

如果问题修好，日志里不应该再出现：

```text
dns resolve failed: couldn't find ip
```

## 总结

这次问题不是代理规则失效。日志已经写得很清楚：

```text
dial DIRECT (match DomainSuffix/<corp_domain>)
```

请求已经直连了。失败点在 DNS。

本机系统 DNS 能解析公司域名，是因为 macOS 知道 `<corp_domain>` 应该问 `<corp_dns_ip>`。Clash Verge 接管 DNS 后，如果配置里没有这条策略，它就会拿默认 DoH 去问。默认 DoH 查不到公司内网域名，于是报 `dns resolve failed`。

最后保留三条配置就够了：

```yaml
dns:
  nameserver-policy:
    '+.<corp_domain>':
      - <corp_dns_ip>
  fake-ip-filter:
    - "*.<corp_domain>"

rules:
  - DOMAIN-SUFFIX,<corp_domain>,DIRECT
  - IP-CIDR,<corp_dns_ip>/32,DIRECT,no-resolve
```

排查这类问题时，别只看“是不是走了代理”。先看规则命中，再看 DNS，最后看连接。顺序对了，问题会安静很多。

参考文档：

1. [Mihomo DNS 配置文档](https://wiki.metacubex.one/en/config/dns/)
2. [Clash Verge Rev 自定义脚本文档](https://www.clashverge.dev/guide/script.html)
