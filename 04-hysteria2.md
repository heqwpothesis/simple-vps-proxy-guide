---
layout: default
title: 04 · Hysteria2
---

# 04 · Hysteria2：UDP、ACME 和用户认证

{% include page-toc.html %}

[← 上一章：VLESS + Reality](03-vless-reality.md) · [下一章：Cloudflare、TLS 与 CDN →](05-cloudflare-tls.md)

这一章配置 **UDP 443** 的 Hysteria2。它基于 QUIC，适合丢包、延迟抖动和移动网络；代价是**需要自己的域名和 TLS 证书**。

如果你只想先用 Reality，可以跳过本章，之后需要 UDP 线路时再回来。但建议至少读完“开始前”，理解为什么 Hysteria2 比 Reality 多一个域名依赖。

## 本章目标

- 准备域名 A 记录、UDP 443 和 ACME 需要的 TCP 80。
- 安装 Hysteria2 服务端。
- 生成 4 个用户凭据，并知道为什么不能重复生成。
- 写 ACME、认证和 masquerade 配置，处理文件权限。
- 启动服务、确认证书签发，生成分享链接。
- 完成一次端到端验证，确认出口是 VPS。

## 开始前

| 检查项 | 要求 |
|---|---|
| 域名 | `hy2.example.com` 的 **A 记录**指向 VPS 公网 IPv4，Cloudflare 设为 **DNS only（灰云）**。 |
| UDP 443 | 没有被其他程序占用。 |
| TCP 80 | 使用 HTTP-01 时，公网必须能访问；使用 DNS-01 则不需要。 |
| 防火墙 | UFW 和服务商上游防火墙都放行 UDP 443；HTTP-01 还要放行 TCP 80。 |
| 恢复手段 | Web Console 可用。 |
| 时间 | 首次签发证书可能需要几十秒到几分钟，取决于 DNS 和 CA。 |

检查 UDP 443 是否空闲：

```bash
ss -ulnp | grep ':443 '
```

输出为空才继续。`ss -tlnp` 只看 TCP，看不到 Hysteria2。

> 本指南使用域名证书。客户端会用 `sni=hy2.example.com` 校验证书；域名解析必须指向这台 VPS。不要用 `example.com` 或别人的域名做实验。

## 1. 安装 Hysteria2

官方文档：[Hysteria2](https://hy2.io/)。

**服务器：**

```bash
bash <(curl -fsSL https://get.hy2.sh/)
hysteria version
systemctl cat hysteria-server
```

安装脚本会安装二进制、创建 `hysteria-server` systemd 服务，并生成默认配置。`systemctl cat` 用来确认实际的服务用户、启动命令和配置路径。

配置通常是 `/etc/hysteria/config.yaml`，但**以 `systemctl cat hysteria-server` 里的 `--config` 参数为准**。

## 2. 生成用户凭据

**服务器：**

```bash
umask 077
install -d -m 700 /root/.secrets
printf 'user1: %s\n' "$(openssl rand -hex 24)" > /root/.secrets/hy2-users.txt
chmod 600 /root/.secrets/hy2-users.txt
```

这条命令在做什么：

- `umask 077`：当前 shell 新建的文件默认只有属主可读写。
- `install -d -m 700`：创建目录并设为 `700`，其他人无法进入。
- `printf 'user1: %s\n'`：写入一行 `user1: 随机密码`。用户名可以改成便于识别的名字，但要和服务端配置、客户端保持一致。
- `openssl rand -hex 24`：生成 24 字节随机数，输出 48 个十六进制字符，足够长。
- `> /root/.secrets/hy2-users.txt`：把结果写入文件；文件已存在时会**覆盖**。
- `chmod 600`：只有 root 能读写。

查看生成结果：

```bash
cat /root/.secrets/hy2-users.txt
```

**只首次生成时执行。** 再次运行会覆盖旧密码，已经导入客户端的用户会立刻认证失败。不要一次生成多份用不到的凭据。

> 生成的密码只包含 `0-9a-f`，不需要 URL 编码。如果你后来改成包含 `@`、`:`、`/`、`?`、`#`、`%` 的密码，写进 URI 时必须正确编码。

## 3. 写服务端配置

先备份：

```bash
[ -f /etc/hysteria/config.yaml ] && cp /etc/hysteria/config.yaml /etc/hysteria/config.yaml.bak.$(date +%F)
```

编辑：

```bash
nano /etc/hysteria/config.yaml
```

模板：

```yaml
listen: :443

acme:
  domains:
    - hy2.example.com
  ca: letsencrypt
  type: http
  dir: /var/lib/hysteria

auth:
  type: userpass
  userpass:
    user1: "USER1_PASSWORD"

masquerade:
  type: proxy
  proxy:
    url: https://www.example.com/
    rewriteHost: true
```

把 `hy2.example.com` 换成你自己的域名，`www.example.com` 换成你想伪装的正常站点，`USER1_PASSWORD` 换成第 2 节生成的密码。

### 3.1 每个配置块在做什么

| 配置 | 含义 | 注意 |
|---|---|---|
| `listen: :443` | 监听所有网卡的 UDP 443 | 省略 IP 时同时监听 IPv4 和 IPv6；只监听 IPv4 可写 `0.0.0.0:443`。 |
| `acme.domains` | 申请证书的域名 | 必须和 A 记录一致，例如 `hy2.example.com`。 |
| `acme.ca` | CA 机构 | 示例用 `letsencrypt`；也可按官方文档换 ZeroSSL。 |
| `acme.type` | ACME 验证方式 | `http` 表示 HTTP-01，需要 TCP 80。 |
| `acme.dir` | 证书和 ACME 状态目录 | 服务用户必须可写。 |
| `auth.type` | 认证方式 | `userpass` 表示用户名 + 密码。 |
| `auth.userpass` | 用户表 | 把示例密码替换成刚生成的值；**只放服务端**。 |
| `masquerade` | 伪装响应 | 浏览器/探测者访问时转发到一个正常网站，不是安全保证。 |
| `rewriteHost` | 改写 Host 头 | 让上游网站按自己的域名响应。 |

关于 `type: http` 和 `type: dns`：

- HTTP-01 要求 CA 能从公网访问 `http://hy2.example.com/.well-known/acme-challenge/...`，所以 TCP 80 必须可达，A 记录必须正确。
- DNS-01 通过 DNS TXT 记录验证，不需要开放 80；但它需要额外的 DNS Provider 配置和 API Token，**不能只把 `type` 改成 `dns` 就完事**。本模板只覆盖 HTTP-01。

YAML 对缩进敏感：用空格，不要用 Tab；冒号后要有空格；密码建议加引号。粘贴后检查缩进是否被编辑器改变。

### 3.2 权限

先确认服务实际使用的用户和组：

```bash
systemctl show hysteria-server -p User -p Group
```

如果输出是 `User=hysteria`、`Group=hysteria`，执行：

```bash
install -d -o hysteria -g hysteria -m 700 /var/lib/hysteria
chown -R hysteria:hysteria /var/lib/hysteria
chown root:hysteria /etc/hysteria /etc/hysteria/config.yaml
chmod 750 /etc/hysteria
chmod 640 /etc/hysteria/config.yaml
```

为什么需要这些权限：

- 如果服务以 `hysteria` 用户运行，它必须能写 ACME 目录。
- 配置里只有明文密码，应该让 root 拥有、`hysteria` 组可读，其他用户不可读。
- 不要用 `chmod 777` 图省事；那会让本机任何用户都能读到密码。

如果 `systemctl show` 显示的用户不同，把命令里的 `hysteria` 换成实际用户名。

## 4. 启动与证书

**服务器：**

```bash
systemctl enable --now hysteria-server
systemctl status hysteria-server --no-pager
journalctl -u hysteria-server -n 80 --no-pager
ss -ulnp | grep ':443 '
```

首次启动会触发 ACME 申请证书，可能需要等待。`systemctl status` 显示 `active` 只代表进程在跑，不代表证书已经签发成功；以日志为准。

查看日志时关注：

- 证书申请/续期是否成功；
- 是否有 `authentication failed`；
- 是否有权限错误；
- 是否成功监听 UDP 443。

持续观察日志：

```bash
journalctl -u hysteria-server -f
```

按 `Ctrl+C` 退出。

| 现象 | 常见原因 |
|---|---|
| ACME challenge failed | DNS 没指向这台 VPS、Cloudflare 不是灰云、TCP 80 被挡、CA 速率限制。 |
| `permission denied` | `/var/lib/hysteria` 不属于服务用户，或配置不可读。 |
| `address already in use` | UDP 443 被其他程序占用；`ss -ulnp` 查。 |
| `no such host` | 域名还没解析或拼写错误。 |
| 服务 active 但 UDP 443 没监听 | 配置解析失败或监听地址错误；看日志。 |

证书签发有速率限制。失败后不要反复重启服务，先看日志确认原因，修好再试。

## 5. 分享链接（URI）

```text
hysteria2://user1:USER1_PASSWORD@hy2.example.com:443/?sni=hy2.example.com#VPS-HY2-user1
```

| 部分 | 含义 |
|---|---|
| `hysteria2://` | 协议标识；客户端要支持 Hysteria **2**，不是 1。 |
| `user1:USER1_PASSWORD` | 用户名和密码；`userpass` 认证需要两者。 |
| `hy2.example.com:443` | 域名和端口；域名必须解析到 VPS。 |
| `sni=hy2.example.com` | TLS SNI，必须和证书域名一致。 |
| `#VPS-HY2-user1` | 客户端显示名，只影响显示。 |

客户端填写时注意：

- 使用 `userpass` 时，Password/Auth 通常填 `user1:密码`；如果界面把用户名和密码分开，就分别填。
- 证书正常时**不要**启用 `insecure=1` / `Allow insecure`。
- 服务端没有配置 `obfs` 时，客户端也不要启用 Salamander/Gecko 混淆；两边不一致会直接连不上。
- 每个用户用自己的链接，不要四个人共用一个密码，否则无法单独撤销。

如果密码包含 URL 保留字符，需要先编码再放进 URI。示例生成的十六进制密码不需要编码。

## 6. 端到端测试

先在客户端导入 URI 并启动节点。Throne 的本地混合代理端口默认可能是 `127.0.0.1:2080`，以客户端日志里的 `mixed-in` 监听地址为准。

**本地：**

```bash
curl --noproxy '' -x http://127.0.0.1:2080 --connect-timeout 5 --max-time 15 https://api.ipify.org
```

参数解释：

- `--noproxy ''`：清空 `NO_PROXY` 环境变量，避免 curl 对 localhost 或某些地址自动直连。测试代理时很重要。
- `-x http://127.0.0.1:2080`：使用本地混合代理；端口按你的客户端实际值改。
- `--connect-timeout 5 --max-time 15`：避免网络不通时一直挂着。

正确结果应该是**你自己的 VPS 出口 IP**。本文档的 `203.0.113.10` 是示例，不能用于真实连接。

同时观察服务端日志：

```bash
journalctl -u hysteria-server -f
```

基本通过的标准：

1. 客户端能完成 HTTPS 请求；
2. 返回的 IP 是 VPS 的公网 IP；
3. 服务端日志出现该用户成功认证的记录。

进一步验证：

- 换一个网络（例如手机热点）再测，排除本地网络封 UDP 的可能。
- 打开一个实际目标网站，不要只测试 IP 查询接口。

如果 curl 超时，但服务端没有任何认证日志，优先查 UDP 路径、客户端配置和防火墙；如果服务端有认证记录但目标请求失败，问题更可能在 VPS 到目标网站的出口。

## 7. 做完检查

- [ ] `dig +short hy2.example.com A @1.1.1.1` 返回 VPS 公网 IP。
- [ ] Cloudflare 记录是 **DNS only（灰云）**。
- [ ] UFW 和服务商上游防火墙放行 UDP 443；HTTP-01 还放行 TCP 80。
- [ ] `systemctl is-active hysteria-server` 为 `active`。
- [ ] `ss -ulnp | grep ':443 '` 显示 Hysteria2 监听。
- [ ] 日志显示证书签发成功。
- [ ] 客户端认证成功，curl 返回 VPS 的 IP。
- [ ] 用户密码和完整 URI 没有出现在公开文件里。

## 卡住了

| 现象 | 先查 |
|---|---|
| 客户端一直超时，服务端没有认证日志 | UDP 443 是否被本机/上游防火墙挡住；客户端域名和端口是否正确。 |
| `authentication failed` | 用户名:密码是否与服务端 `userpass` 一致；是否用了其他用户的密码。 |
| 证书申请失败 | A 记录、灰云、TCP 80、日志中的 ACME 错误。 |
| 服务起不来 | `journalctl -u hysteria-server -n 100 --no-pager`；检查 YAML 缩进和权限。 |
| 一个网络能用，另一个不能用 | 该网络可能封 UDP/QUIC；临时切回 Reality（TCP 443）。 |
| 客户端显示已连接但网页打不开 | 用第 06 章的 curl 验证出口；区分隧道问题和目标网站问题。 |

[← 上一章：VLESS + Reality](03-vless-reality.md) · [下一章：Cloudflare、TLS 与 CDN →](05-cloudflare-tls.md)
