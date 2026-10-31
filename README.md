# VPS 自建部署指南

基于 Debian、Xray-core、Hysteria2、Cloudflare DNS 和常见客户端的分章节中文操作手册。面向第一次接触服务器部署的读者，也保留熟悉 Linux 的读者需要的细节和排查命令。

**[开始阅读](index.md)** · [零基础准备](00-basics.md) · [服务器初始化](02-server-setup.md) · [Reality](03-vless-reality.md) · [Hysteria2](04-hysteria2.md) · [机器可读索引](llms.txt)

## 适合谁

- 没登录过 Linux 服务器，需要从“IP、端口、TCP/UDP”开始补概念。
- 已经会 SSH 和基本命令，想直接照配置搭建。
- 已经搭好，但需要一套按层排查和运维备份的参考。

文档不提供现成节点、账号或订阅；所有参数需要自己生成，示例 IP 和域名都是占位符。

## 目录

| 章节 | 内容 |
|---|---|
| [开始阅读](index.md) | 架构、准备工作、阅读路径、验收标准 |
| [00 零基础准备](00-basics.md) | IP、端口、TCP/UDP、DNS、TLS、Reality 等核心词 |
| [01 VPS 选择](01-vps-selection.md) | 机房、供应商和选购检查 |
| [02 服务器初始化](02-server-setup.md) | Debian、UFW、SSH 和安全基线 |
| [03 VLESS + Reality](03-vless-reality.md) | Xray 安装、密钥、服务端和客户端 |
| [04 Hysteria2](04-hysteria2.md) | UDP、ACME、用户认证、验证 |
| [05 Cloudflare 与 TLS](05-cloudflare-tls.md) | 域名、DNS、证书及 CDN 边界 |
| [06 客户端](06-clients.md) | 桌面与手机：Throne、Clash Verge / Mihomo、v2rayNG、Hiddify |
| [07 故障排查](07-troubleshooting.md) | TCP、UDP、TLS、HTTP 和抓包 |
| [08 运维](08-maintenance.md) | 更新、备份、监控、权限与轮换 |

## 快速路径

- **第一次做**：00 → 01 → 02 → 03 → 04 → 05 → 06 → 07 → 08。
- **有 Linux 经验**：01 → 02 → 03 → 04 → 06；用域名或 Cloudflare 时补 05，07/08 当参考。

## 环境

演示 TCP 443 的 VLESS + Reality + Vision 与 UDP 443 的 Hysteria2 共存。实际参数和软件版本以官方文档为准；每条命令执行前先理解用途。
