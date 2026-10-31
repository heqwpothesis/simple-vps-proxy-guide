---
layout: default
title: VPS 自建部署指南
---

# VPS 自建部署指南

从买一台 VPS，到在手机和电脑上稳定使用，一步步搭好两条代理线路。文档同时照顾第一次接触服务器的读者和已经熟悉 Linux 的读者。

> **第一次接触服务器？** 先读 [00 · 零基础准备](00-basics.md)。它解释 IP、端口、TCP/UDP、DNS、TLS 这些后面反复出现的词，不要求任何 Linux 基础。

## 这份文档能帮你做什么

- 选一台线路和价格都合适的 VPS，并完成购买后的交付检查。
- 把刚买来的 Debian 系统初始化好：更新、防火墙、SSH 公钥登录。
- 配置 **TCP 443** 的 VLESS + Reality + Vision。
- 配置 **UDP 443** 的 Hysteria2：自动 HTTPS 证书、用户认证、端到端验证。
- 在桌面（Throne、Clash Verge / Mihomo）或手机（v2rayNG、Hiddify）上导入节点，并确认流量真的从 VPS 出去。
- 出问题时按层排查，平时知道该备份什么、怎么撤销用户。

它不会提供现成节点、账号或订阅。所有参数都要你自己生成；线路是否可用会随网络环境变化，文档只能给你可复现的配置和验证方法。

## 你需要准备

| 项目 | 是否必须 | 说明 |
|---|---|---|
| VPS | 必须 | Debian 12，公网 IPv4，服务商提供 Web Console。 |
| 域名 | 可选 | 只有 Hysteria2 的自动证书需要；只做 Reality 可以先不买。 |
| 本地终端 | 必须 | macOS“终端”、Windows Terminal/PowerShell，或 Linux 终端。 |
| 密码管理器 | 必须 | 保存密码、私钥、节点参数和续费信息。 |
| 时间 | — | 第一次建议留出 1–2 小时；每章末尾的验收不通过，不要急着进入下一章。 |

## 两条线路

| 协议 | 传输 | 特点 | 章节 |
|---|---|---|---|
| VLESS + Reality + Vision | TCP 443 | 不需要自己的域名或证书；借用真实网站的 TLS 握手 | [03](03-vless-reality.md) |
| Hysteria2 | UDP 443 | 基于 QUIC，弱网表现好；需要域名和 TLS 证书 | [04](04-hysteria2.md) |

TCP 和 UDP 是两套独立的端口空间，所以两个服务可以同时监听 443。它们共享同一台 VPS：整机、IP 或上游线路故障时，两者会一起不可用。**同机双协议不是容灾。**

## 参考架构

```text
你的设备（Throne / Clash Verge / v2rayNG / Hiddify）
  ├─ TCP 443 ──> Xray-core：VLESS + Reality + Vision ─┐
  └─ UDP 443 ──> Hysteria2 + TLS（hy2.example.com）───┴─> VPS 出口 ─> Internet
                        ↑
                  DNS only / ACME
```

- `hy2.example.com` 只负责 Hysteria2 的证书和寻址，必须使用 Cloudflare 的 **DNS only（灰云）**。
- Reality 不使用你自己的域名证书，客户端填的 SNI 是借用目标网站的域名。
- 服务商控制台的上游防火墙是独立的一层，UFW 放行不等于公网一定可达。

## 推荐阅读顺序

### 第一次接触服务器

1. [00 · 零基础准备](00-basics.md)：先弄懂 IP、端口、TCP/UDP、DNS、TLS。
2. [01 · VPS 选型与购买](01-vps-selection.md)：选机房、套餐，完成交付检查。
3. [02 · Debian 初始化、UFW 与 SSH](02-server-setup.md)：把机器变成可安全远程管理的服务器。
4. [03 · Xray：VLESS + Reality + Vision](03-vless-reality.md)：先配 TCP 线路。
5. [04 · Hysteria2](04-hysteria2.md)：再配 UDP 线路和自动证书。
6. [05 · Cloudflare、TLS 与 CDN](05-cloudflare-tls.md)：Hysteria2 的域名解析、证书和 CDN 边界。
7. [06 · 客户端](06-clients.md)：导入节点并验证出口 IP。
8. [07 · 故障排查](07-troubleshooting.md)：出问题时按层定位。
9. [08 · 日常运维](08-maintenance.md)：更新、备份、轮换和容灾。

### 已经熟悉 Linux / VPS

- 直接看 [01](01-vps-selection.md) → [02](02-server-setup.md) → [03](03-vless-reality.md) → [04](04-hysteria2.md) → [06](06-clients.md)。
- 用域名或 Cloudflare 时补 [05](05-cloudflare-tls.md)；[07](07-troubleshooting.md) 和 [08](08-maintenance.md) 当参考手册。

## 每章怎么用

- **本章目标 / 开始前**：告诉你这一章的范围、前置条件和需要准备的东西。
- **命令块**：上方标注 **服务器** 或 **本地**；没标时按章节开头说明。
- **做完检查**：硬性验收。不通过就停在当前章，按“卡住了”排查。
- **卡住了**：给最短的定位路径；完整的分层排查在 [07](07-troubleshooting.md)。

## 验收标准

| 层次 | 应验证的内容 | 怎么确认 |
|---|---|---|
| 系统 | SSH 可维护、UFW 按预期放行 | 新终端能登录；`ufw status verbose` 只放行必要端口 |
| 服务 | Xray 和 Hysteria2 正常启动并监听正确端口 | `systemctl is-active`；`ss -tlnp` / `ss -ulnp` |
| TLS/认证 | 客户端使用正确 SNI、证书和各自凭据 | 服务端日志出现成功认证 |
| 传输 | 经客户端代理的 HTTPS 请求能完成 | `curl -x ... https://api.ipify.org` 有响应 |
| 出口 | IP 查询显示预期 VPS 的公网 IP | 返回的是你的 VPS IP，不是家宽 IP |
| 业务 | 实际访问目标应用 | 浏览器能打开目标网站并完成登录/请求 |

只看代理软件显示“已启动”不算通过。进程活着、端口在听、请求成功是三件不同的事。

## 公开文档说明

文档中的 `203.0.113.10` 是专用于示例的 IPv4，`example.com` 是示例域名。所有密钥和口令均为占位符。

- 不要把真实 VPS IP、个人域名、UUID、私钥、密码、订阅 URI 或 Cloudflare Token 提交到公开仓库。
- 分享日志和截图前先脱敏。
- 不要直接运行未替换参数的配置。

机器可读的文档索引见 [llms.txt](llms.txt)，适合 AI 助手或自动化工具快速定位章节和约束。

[第一次接触服务器：从 00 开始 →](00-basics.md) · [已经熟悉 Linux：从 01 开始 →](01-vps-selection.md)
