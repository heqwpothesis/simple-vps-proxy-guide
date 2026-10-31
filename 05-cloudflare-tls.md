---
layout: default
title: 05 · Cloudflare、TLS 与 CDN
---

# 05 · Cloudflare 域名、DNS、TLS 与 CDN

{% include page-toc.html %}

[← 上一章：Hysteria2](04-hysteria2.md) · [下一章：客户端 →](06-clients.md)

这一章只在你要给 Hysteria2 配域名证书时需要（第 04 章）。

只使用 Reality 的话，可以跳过本章，之后需要域名时再回来。但第 4 节的 CDN 边界值得读一遍：它解释了为什么“把灰云改成橙云”通常会把 Hysteria2 弄坏。

## 本章目标

- 买一个域名，或复用已有域名创建子域名。
- 正确添加 A 记录，并确认它是 **DNS only（灰云）**。
- 理解 HTTP-01、DNS-01 和 TCP 80 的关系。
- 知道 Cloudflare 橙云能代理什么、不能代理什么，以及为什么 Hysteria2 必须用灰云。

## 开始前

- 一个 Cloudflare 账户（或其他支持 API/ACME 的 DNS 服务商）。
- 可用的支付方式。
- 决定好 Hysteria2 使用的主机名，例如 `hy2.example.com`。
- 如果要做 Hysteria2 证书，先完成 [04](04-hysteria2.md) 的服务器安装和防火墙配置。

## 1. 域名是什么，为什么要买

域名只是互联网上的**名字**，不是服务器本身。它通过 DNS 指向 IP；TLS 证书则绑定这个域名，用来证明“你连的确实是这个名字”。

购买时注意：

- **首年价和续费价可能差很多**，以结账页为准。
- 确认注册信息、邮箱验证和账户 MFA。
- 隐私保护（WHOIS privacy）是否包含，通常建议开启。
- 域名不需要和 VPS 在同一家服务商；DNS 可以放在 Cloudflare。
- 已有域名不必再买一个，直接创建子域名即可：`example.com` 下加 `hy2`，得到 `hy2.example.com`。

在 [Cloudflare Registrar](https://domains.cloudflare.com/) 或其他注册商搜索 `.com` 等域名，核对可注册性、价格和条款。价格会调整，不要依赖旧教程里的数字。

## 2. 添加 DNS 记录

假设你的域名是 `example.com`，要给 Hysteria2 用：

| 字段 | 示例值 |
|---|---|
| Type | `A` |
| Name | `hy2` |
| IPv4 address | `203.0.113.10`（替换成你的 VPS IP） |
| Proxy status | **DNS only（灰云）** |
| TTL | Auto |

结果：`hy2.example.com → 203.0.113.10`。文档里的 IP 只是示例，不是真实节点。

本指南只需要 **A 记录**（域名 → IPv4）。如果 VPS 也有固定 IPv6，可以再添加 AAAA 记录：

- **A**：域名 → IPv4。
- **AAAA**：域名 → IPv6。
- `Name` 填 `hy2` 表示 `hy2.example.com`；填 `@` 表示根域名 `example.com`。

**Proxy status 必须是 DNS only（灰云）。** 灰云表示 DNS 只负责解析，客户端直接连到 VPS；橙云会改变连接路径，原因见第 4 节。

验证解析：

```bash
dig +short hy2.example.com A @1.1.1.1
```

期望输出是你的 VPS 公网 IP。如果输出的是 Cloudflare 的 IP，说明记录还是橙云；如果为空，检查主机名、域名服务器和传播时间。DNS 生效可能需要几分钟。

## 3. HTTPS 证书和 ACME

TLS 证书做两件事：加密连接，并向客户端证明“这个域名确实属于这台服务器”。没有可信证书，客户端会报警告；Hysteria2 也会认证失败。

Let's Encrypt 提供免费证书，ACME 负责自动申请和续期。常见验证方式：

| 方式 | CA 怎么验证 | 你需要什么 |
|---|---|---|
| HTTP-01 | 访问 `http://你的域名/.well-known/acme-challenge/...` | 域名 A 记录正确，公网 TCP 80 可达。 |
| DNS-01 | 查询你域名下的 TXT 记录 | DNS Provider API Token，权限尽量小。 |

本指南的 Hysteria2 模板使用 HTTP-01：

- 域名 A 记录必须指向这台 VPS。
- 公网 TCP 80 必须可达。本指南要求 HY2 记录保持灰云，因此 CA 会直连 VPS 的 80。
- Hysteria2 会自动处理 challenge 文件，不需要你手动放文件。
- TCP 80 和 Reality 的 TCP 443 不冲突；两个端口可以同时使用。

如果 80 已经被 Nginx/Caddy 占用，需要让 Web 服务器代理 ACME challenge 路径，或改用 DNS-01；不要为了签证书直接停掉正在提供服务的网站。

关于 DNS-01：

- 不需要开放 80，适合 80 被封或不想暴露 Web 服务的场景。
- 需要在 Hysteria2 配置里增加 `dns` 段和 Provider 参数，不能只把 `type` 改成 `dns`。
- API Token 只给必要的 DNS 编辑权限，不要使用全局 API Key，也不要把 Token 写进公开仓库。

证书和域名绑定，不和 IP 绑定。换 VPS IP 时，只要改 A 记录，证书在有效期内可以继续使用。续期是自动的，但仍要定期看日志和到期时间；CA 有速率限制，失败后不要反复重试。

## 4. Cloudflare CDN 能做什么，不能做什么

| 模式 | 实际连接 | 适用 |
|---|---|---|
| **DNS only（灰云）** | 客户端 → VPS | Hysteria2，以及任何需要直连的协议 |
| **Proxied（橙云）** | 客户端 → Cloudflare → 源站 | 受支持的 HTTP(S)、WebSocket 等 Web 流量 |

具体来说：

- 灰云只负责解析，真实 IP 对外可见，但连接直达 VPS，延迟和协议行为可预期。
- 橙云是反向代理，主要处理 HTTP(S)/WebSocket；TLS 可能在 Cloudflare 边缘终止。
- 橙云**不会直接转发** Hysteria2 的原始 UDP。
- 把 Hysteria2 的灰云记录直接改成橙云，通常会导致当前协议不可用。

一句话：**橙云不是通用 VPN 中转，也不能给 Hysteria2 加速。**

## 5. 做完检查

- [ ] `dig +short hy2.example.com A @1.1.1.1` 返回 VPS 公网 IP。
- [ ] Cloudflare 记录是 **DNS only（灰云）**。
- [ ] 使用 HTTP-01 时，公网 TCP 80 可达。
- [ ] 第 04 章日志显示证书签发成功。
- [ ] DNS API Token、账户凭证没有出现在仓库或截图里。

## 卡住了

| 现象 | 先查 |
|---|---|
| `dig` 返回 Cloudflare IP | 记录还是橙云；改成 DNS only，等生效。 |
| `dig` 为空 | 主机名拼写、域名服务器、传播时间；确认记录加在正确的 zone。 |
| ACME 一直失败 | A 记录是否指向这台 VPS、80 是否可达、是否被上游防火墙挡住、是否触发速率限制。 |

[← 上一章：Hysteria2](04-hysteria2.md) · [下一章：客户端 →](06-clients.md)
