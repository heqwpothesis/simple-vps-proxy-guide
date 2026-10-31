---
layout: default
title: 07 · 故障排查
---

# 07 · 故障排查：从 TCP 到应用层

[← 上一章：客户端](06-clients.md) · [下一章：日常运维 →](08-maintenance.md)

代理故障最常见的原因不是“协议坏了”，而是某一层没通。这一章按 **网络 → 端口/防火墙 → TLS → 认证 → HTTP → 应用** 的顺序排查。每轮只改一个变量，改完重复同一个测试。

## 本章目标

- 用错误信息判断故障发生在哪一层。
- 掌握 SSH、TCP 443、UDP 443、TLS 和 HTTP 的最小检查命令。
- 知道哪些现象不能证明“已经成功”。
- 写出一份可复现、可脱敏的故障记录。

## 1. 先判断故障层级

```text
应用 / 浏览器      页面行为、登录状态、HTTP 状态码
客户端             本地端口、System Proxy/TUN、规则、日志
隧道               TCP/UDP 443、TLS/Reality、认证
服务器             进程、监听、防火墙、证书、出口
网络               路由、丢包、上游防火墙、DNS
```

读日志时从上往下找**第一个**错误；排查时从下往上，先确认底层通不通。

| 表现 | 说明 | 先查 |
|---|---|---|
| `Connection timed out`（连接目标端口） | TCP SYN 没有响应，或 UDP 没有回包 | 监听、防火墙、路由、丢包 |
| `Connection refused` | 到达了主机，但端口没有接受连接 | 服务是否启动、端口是否写对 |
| `Permission denied (publickey)` | SSH 已到认证层 | 密钥、`authorized_keys`、权限 |
| TLS `unexpected eof` | TCP 通了，但 TLS 握手被关闭或重置 | SNI、目标站点、上游出口 |
| HY2 `authentication failed` | 隧道/QUIC 已建立，但凭据不对 | `用户名:密码`、服务端认证配置 |
| `HTTP 401` | 到达 HTTP 服务，但未授权 | API Key、登录态、认证头 |
| `HTTP 403` | 到达 HTTP 服务，但被拒绝 | 访问策略、反自动化、账号问题 |
| 日志只有 `outbound connection` | 请求进入出站，尚不能确认完成 | 继续等 HTTP 结果，不要提前判定成功 |

最重要的区分是 **timeout 和 refused**：

- timeout：连接根本没建立，优先查网络、防火墙、监听。
- refused：连接到了主机，但端口没人接，优先查服务是否启动、端口是否正确。

## 2. SSH 直连超时

**本地（macOS）：**

```bash
ssh -vvv -o ConnectTimeout=8 root@203.0.113.10
nc -vz -G 8 203.0.113.10 22
```

Linux 上 `nc` 用 `-w 8` 代替 `-G 8`。`ssh -vvv` 会打印详细握手过程；`ConnectTimeout` 避免一直等。

**服务器（通过 Web Console）：**

```bash
systemctl is-active ssh
ss -tlnp | grep ':22 '
ufw status verbose
sshd -t
```

必要时抓包：

```bash
tcpdump -ni any 'tcp port 22'
```

`-n` 不解析域名，`-i any` 监听所有网卡。让客户端重试一次，观察输出，按 `Ctrl+C` 停止。

| 抓包现象 | 可能原因 |
|---|---|
| 完全看不到 SYN | 客户端到 VPS 的路径不通，或服务商上游防火墙拦截。 |
| 看到 SYN，但没有 SYN-ACK | 服务器本机防火墙拦截，或 SSH 没监听。 |
| 有 SYN-ACK，但客户端没有最终 ACK | 回程路径有问题；需要结合两端抓包。 |

**不要因为 TCP 超时就重建 SSH 密钥。** 密钥问题通常表现为 `Permission denied (publickey)`，而不是连接超时。

## 3. Reality TCP 443 不可达

**服务器：**

```bash
systemctl is-active xray
ss -tlnp | grep ':443 '
tcpdump -ni any 'tcp port 443'
```

**本地：**

```bash
nc -vz -w 5 203.0.113.10 443
```

从不同网络分别测（家宽、手机热点、公司网络）。只有 TCP 建立之后，才进入 Reality/TLS/认证排查。

- `active` 只代表进程在跑，不代表外部可达。
- TCP 443 通但客户端握手失败：重点核对 `sni`、`pbk`、`sid`、`flow`，再看服务端日志。
- TCP 443 超时：先查 UFW、上游安全组、路由；不要先重置 Reality 密钥。
- `ss -tlnp` 没有输出：Xray 没监听，回到 [03](03-vless-reality.md) 看配置和日志。

## 4. Hysteria2 UDP 443

**服务器：**

```bash
systemctl is-active hysteria-server
ss -ulnp | grep ':443 '
ufw status
journalctl -u hysteria-server -n 100 --no-pager
```

UDP 没有 TCP 那样的三次握手，所以不能用 `nc -z` 或浏览器直接判断端口是否可用。正确做法是：

1. 用真实的 Hysteria2 客户端发起连接。
2. 同时看服务端日志是否出现该用户认证成功。
3. 客户端完成一次 HTTPS 请求，确认返回 VPS IP。

常见判断：

- 服务端完全没有认证日志：UDP 路径、客户端地址/端口、防火墙、域名解析问题。
- 有认证日志但目标请求失败：VPS 到目标网站的出口问题。
- 一个网络能用、另一个不能用：该网络可能封 UDP/QUIC；临时切回 Reality。
- 只测 TCP 443 不能说明 UDP 443 可用，两者是独立端口空间。

## 5. 特定网站超时

例如日志里出现：

```text
remote error: dial tcp4 198.51.100.20:443: i/o timeout
```

`remote error` 通常表示**服务器侧**在连接目标时失败：VPS 收到了代理请求，但它到最终目标的 TCP 建连超时。需要结合日志里的连接 ID 和时间段判断，不能只看这一行。

排查步骤：

1. 在服务器上直接访问目标，区分“VPS 出口问题”和“隧道问题”：

```bash
curl -I --connect-timeout 8 https://chatgpt.com
dig +short chatgpt.com A
```

2. 如果服务器直连也超时，查 VPS 到目标的 DNS、路由和目标站点是否封锁机房 IP。
3. 如果服务器直连正常，但客户端经代理失败，查客户端、规则、隧道和目标域名。
4. 同时测试多个目标：IP 查询、工作 API、国内服务。不要因为一个网站失败就判定全局断线。

目标网站封锁机房 IP 很常见。遇到这种情况，换目标、换 VPS 地区或换出口，通常比反复重装 Xray 更有效。

## 6. ChatGPT / Claude 的 HTTP 测试

**本地：**

```bash
for url in https://chatgpt.com https://claude.ai https://api.anthropic.com/v1/models; do
  echo "Testing $url"
  curl --noproxy '' -x http://127.0.0.1:2080 \
    --connect-timeout 8 --max-time 20 -sS -o /dev/null \
    -w 'HTTP=%{http_code} TLS=%{time_appconnect}s Total=%{time_total}s\n' "$url"
done
```

参数解释：

- `-o /dev/null`：丢弃响应正文，只看状态和耗时。
- `-sS`：静默模式，但显示错误。
- `-w`：输出 HTTP 状态码、TLS 握手耗时和总耗时；`\n` 在 curl 的格式串里表示换行。
- 没有加 `-f`，所以 401/403 不会让 curl 以失败退出，方便观察状态码。

怎么读：

| 结果 | 含义 |
|---|---|
| `HTTP=200` | 请求成功。 |
| `HTTP=401` | 到达 HTTP 服务，但未授权；API 无密钥时常见。 |
| `HTTP=403` | 到达 HTTP 服务，但被拒绝；可能是访问策略或反自动化。 |
| `HTTP=000` | 没有拿到 HTTP 响应：超时、TLS、DNS 或连接失败。 |
| `HTTP=5xx` | 目标服务器自身出错。 |

401 或 403 可以证明“HTTP 层通了”，但**不能证明浏览器登录、会话和页面功能正常**。仍要打开实际页面操作一次。

## 7. 快速故障记录模板

| 字段 | 示例 |
|---|---|
| 时间 / 时区 | 2026-10-30 21:30 CST |
| 客户端网络 | 家宽 / 手机热点 / 公司网络 |
| 目标 | `https://chatgpt.com` |
| 方式 | 直连 / 经客户端代理 |
| 协议 / 端口 | HY2 / UDP 443；或 Reality / TCP 443 |
| 客户端日志 | 关键错误行、连接 ID |
| 服务端日志 | 同一时间段的 `journalctl` 输出 |
| 已排除 | TCP 443 可达、UFW 已放行、其他网站正常 |

分享前删除：凭据、订阅地址、用户名、个人路径、私密目标域名、完整节点 URI。日志里的 IP 和域名也可能暴露你的节点。

## 8. 三条纪律

1. **一次只改一个变量。** 同时改防火墙、UUID 和 SNI，出了问题无法判断是哪一项。
2. **优先做可回退的修改。** 改配置前备份；临时开放端口，测完关掉；不要长期 `ufw disable`。
3. **先复现，再修改。** 记录修改前的现象和命令输出，改完用同一个测试对比。

## 做完检查

- [ ] 能根据错误信息判断是 timeout、refused、认证还是 HTTP 层。
- [ ] 会查 SSH、TCP 443、UDP 443 的监听和日志。
- [ ] 知道 `active`、监听、HTTP 成功是三件不同的事。
- [ ] 能区分“单个网站失败”和“全局断线”。
- [ ] 有一份可复现、已脱敏的故障记录。

[← 上一章：客户端](06-clients.md) · [下一章：日常运维 →](08-maintenance.md)
