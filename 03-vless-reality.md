---
layout: default
title: 03 · VLESS + Reality
---

# 03 · Xray：VLESS + Reality + Vision

[← 上一章：服务器初始化](02-server-setup.md) · [下一章：Hysteria2 →](04-hysteria2.md)

这一章配置 **TCP 443** 的 VLESS + Reality + Vision。

Reality 的特点是：你不需要自己的域名，也不需要自己签发证书。它借用另一个真实 HTTPS 网站的 TLS 握手作为伪装。代价是客户端必须和服务端的 SNI、Reality 公钥、Short ID 完全一致；填错一个字段，通常表现为超时或握手失败，而不是明确的“密码错误”。

## 本章目标

- 安装 Xray-core，确认版本和 systemd 服务文件。
- 生成 Reality 密钥、用户 UUID 和 Short ID。
- 写服务端配置，校验并启动。
- 生成客户端 URI，知道每个参数从服务端的哪里来。
- 确认 Xray 正在监听 TCP 443（完整端到端验证在第 06 章）。

## 开始前

- 完成 [02](02-server-setup.md)，SSH 可登录，UFW 已放行 TCP 443。
- TCP 443 没有被其他程序占用：

```bash
ss -tlnp | grep ':443 '
```

输出为空才继续。如果被 nginx、Caddy 等占用，先决定是停掉它们还是换端口，不要直接 `kill` 不认识的进程。

- 选一个 **Reality 目标站点**。它不是你的网站，而是被借用的真实 HTTPS 站点：

| 要求 | 说明 |
|---|---|
| 真实、稳定、可访问 | VPS 要能连上它；不要填 `example.com`。 |
| 支持 TLS 1.3 和 HTTP/2 | 装好 Xray 后可用 `xray tls ping 目标域名` 检查。 |
| 与 VPS 网络较近 | 同一地区/ASN 通常更稳；目标站点太远会增加握手延迟。 |
| 尽量不选 Cloudflare 后面的站点 | Reality 会把未通过认证的流量转发到目标；目标在 CDN 后面时，你的服务器可能变成 CDN 的转发器。 |

> Reality 的目标域名和 SNI 必须根据实际服务端配置及兼容性选择；本文示例不构成“某个域名长期稳定可用”的保证。目标站点换证书、换 IP 或封锁机房 IP 都会影响效果。

## 1. 安装 Xray-core

官方项目：[XTLS/Xray-core](https://github.com/XTLS/Xray-core)；[安装脚本](https://github.com/XTLS/Xray-install)。

**服务器：**

```bash
bash -c "$(curl -L https://github.com/XTLS/Xray-install/raw/main/install-release.sh)" @ install
xray version
systemctl cat xray
```

这条安装脚本会从官方仓库下载最新版本，安装二进制、默认配置和 systemd 服务。`xray version` 确认版本；`systemctl cat xray` 查看服务实际使用的启动命令和配置文件路径。

主配置通常位于 `/usr/local/etc/xray/config.json`，但**以 `systemctl cat xray` 里的 `-config` 参数为准**。不要凭教程假设路径。

> 脚本来自官方仓库。如果介意“下载后直接执行”，可以先下载脚本阅读，再手动执行；不要从不明镜像或网盘获取安装包。

## 2. 生成密钥、UUID 和 Short ID

**服务器：**

```bash
xray x25519
xray uuid
openssl rand -hex 8
```

### 2.1 `xray x25519` 的输出怎么读

新版 Xray 的输出通常长这样：

```text
PrivateKey: ...
Password: ...
Hash32: ...
```

- `PrivateKey`：Reality 私钥，**只填服务端**的 `privateKey`。
- `Password`：这里就是 Reality **公钥**，填客户端的 `pbk`。旧版 Xray 和很多教程把它写作 `Public key`。
- `Hash32`：本配置不需要填。

这里最容易踩的坑：`Password` 不是登录密码，也不要把它当成 Hysteria2 的用户密码。它是公钥，只是 Xray 改了名字。

### 2.2 UUID 和 Short ID

- `xray uuid`：生成一个 VLESS 用户 ID。**每个用户单独一个**，方便撤销和轮换。
- `openssl rand -hex 8`：生成 16 个十六进制字符，可作为 Short ID。Short ID 长度必须是偶数，最多 16 个十六进制字符；`openssl rand -hex 8` 正好满足。

| 生成物 | 填到哪里 | 敏感性 |
|---|---|---|
| `PrivateKey` | 服务端 `privateKey` | 高，只留服务器 |
| `Password` / 公钥 | 客户端 URI 的 `pbk` | 不要连同完整 URI 公开 |
| UUID | 服务端 `clients[].id`、客户端 URI | 等同账号，别公开 |
| Short ID | 服务端 `shortIds`、客户端 `sid` | 不是强密码，但也不要公开 |

## 3. 写服务端配置

先备份原配置（如果存在）：

```bash
[ -f /usr/local/etc/xray/config.json ] && cp /usr/local/etc/xray/config.json /usr/local/etc/xray/config.json.bak.$(date +%F)
```

用 `nano` 编辑：

```bash
nano /usr/local/etc/xray/config.json
```

把下面模板粘贴进去，替换所有 `YOUR_...` 占位符，以及 `target`/`serverNames` 里的示例域名：

```json
{
  "log": {"loglevel": "warning"},
  "inbounds": [{
    "listen": "0.0.0.0",
    "port": 443,
    "protocol": "vless",
    "settings": {
      "clients": [{"id": "YOUR_UUID", "flow": "xtls-rprx-vision"}],
      "decryption": "none"
    },
    "streamSettings": {
      "network": "tcp",
      "security": "reality",
      "realitySettings": {
        "target": "www.example.com:443",
        "serverNames": ["www.example.com"],
        "privateKey": "YOUR_REALITY_PRIVATE_KEY",
        "shortIds": ["YOUR_SHORT_ID"]
      }
    }
  }],
  "outbounds": [{"protocol": "freedom"}]
}
```

> 这个模板是演示用的最小配置，不是完整生产审计结果。编辑前先备份；JSON 不支持注释，也不能有末尾多余的逗号。

### 3.1 每个字段在做什么

| 字段 | 含义 | 怎么填 |
|---|---|---|
| `log.loglevel` | 日志级别 | `warning` 只记录警告和错误，适合日常。 |
| `listen` | 监听地址 | `0.0.0.0` 表示所有 IPv4 网卡。 |
| `port` | 监听端口 | `443`。 |
| `protocol` | 入站协议 | `vless`。 |
| `clients[].id` | 用户 UUID | 填 `xray uuid` 的结果。 |
| `clients[].flow` | 流控模式 | `xtls-rprx-vision`，客户端必须一致。 |
| `decryption` | VLESS 解密方式 | 固定 `none`；加密由 Reality/TLS 负责。 |
| `network` | 传输层 | `tcp`。 |
| `security` | 传输安全 | `reality`。 |
| `target` | 被借用的真实站点 | `域名:443`，VPS 必须能访问。 |
| `serverNames` | 允许的 SNI 列表 | 通常与 `target` 域名一致；客户端 `sni` 必须是其中之一。 |
| `privateKey` | Reality 私钥 | 填 `x25519` 输出里的 `PrivateKey`。 |
| `shortIds` | 允许的 Short ID 列表 | 填 `openssl rand -hex 8` 的结果；客户端 `sid` 必须是其中之一。 |
| `outbounds` | 出站 | `freedom` 表示直连目标。 |

多人使用时，把配置里 `clients` 的值整体替换成数组，为每个人加一个对象，每人一个 UUID：

```json
[
  {"id": "UUID_USER_1", "flow": "xtls-rprx-vision"},
  {"id": "UUID_USER_2", "flow": "xtls-rprx-vision"}
]
```

### 3.2 保存文件

如果用的是 `nano`：`Ctrl+O` 保存，回车确认文件名，`Ctrl+X` 退出。其他编辑器按自己的方式保存。

## 4. 校验并启动

**服务器：**

```bash
xray run -test -config /usr/local/etc/xray/config.json
systemctl enable --now xray
systemctl status xray --no-pager
ss -tlnp | grep ':443 '
journalctl -u xray -n 50 --no-pager
```

每条命令的作用：

- `xray run -test`：只解析和校验配置，不启动服务、不接收流量。没有报错才继续。
- `systemctl enable --now xray`：设为开机自启，并立即启动。
- `systemctl status`：看服务是否 `active (running)`。
- `ss -tlnp`：确认有进程监听 TCP 443。
- `journalctl`：服务启动失败时，日志里通常有具体原因。

常见错误：

| 报错/现象 | 常见原因 |
|---|---|
| `invalid character` | JSON 少了引号、逗号，或多了末尾逗号。 |
| `address already in use` | 443 已被其他服务占用；用 `ss -tlnp` 找出来。 |
| `failed to load config` | 路径写错或文件不可读。 |
| 服务 `active` 但 443 没监听 | 看 `journalctl -u xray`，通常是配置解析失败或监听地址错误。 |
| 启动后立刻退出 | 先用 `xray run -test` 排除配置问题。 |

服务 `active` 且监听 TCP 443，只能证明**本机配置和运行状态**，不证明外部 TCP 路径或 Reality 握手可用。后者要在第 06 章用客户端验证。

## 5. 客户端 URI

把下面的模板替换成你的参数：

```text
vless://YOUR_UUID@203.0.113.10:443?encryption=none&security=reality&sni=www.example.com&fp=chrome&pbk=YOUR_PUBLIC_KEY&sid=YOUR_SHORT_ID&type=tcp&flow=xtls-rprx-vision#VPS-Reality
```

| 参数 | 含义 | 从服务端哪里来 |
|---|---|---|
| `YOUR_UUID` | 用户 ID | `clients[].id` |
| `203.0.113.10` | VPS 地址 | 你的公网 IP（也可以填解析到它的域名） |
| `encryption=none` | VLESS 不再额外加密 | 固定值；TLS/Reality 负责传输加密 |
| `security=reality` | 使用 Reality | 固定值 |
| `sni` | 客户端声明的目标域名 | `serverNames` 中的一个 |
| `fp=chrome` | 模拟的 TLS 指纹 | 常用 `chrome`，需客户端支持 |
| `pbk` | Reality 公钥 | `x25519` 输出里的 `Password`（旧版叫 `Public key`） |
| `sid` | Short ID | `shortIds` 中的一个 |
| `type=tcp` | 传输方式 | 与服务端 `network` 一致 |
| `flow=xtls-rprx-vision` | 流控 | 与服务端 `clients[].flow` 一致 |
| `#VPS-Reality` | 客户端显示名 | 只影响显示，可改 |

客户端与服务端必须一致的项目：

- UUID
- 端口 443
- SNI 属于 `serverNames`
- `pbk` 是公钥，不是私钥
- `sid` 属于 `shortIds`
- `flow` 是 `xtls-rprx-vision`
- 客户端支持 `fp=chrome` 对应的指纹

完整的 URI 等同凭据。不要提交到公开仓库、issue 或群聊；截图前先遮住 UUID、`pbk` 和 `sid`。

## 6. 与 Hysteria2 共存

- Reality 监听 **TCP 443**。
- Hysteria2 监听 **UDP 443**。
- TCP 和 UDP 是独立端口空间，数字相同不会冲突。

如果客户端连 TCP 443 超时，先排查防火墙、上游安全组、路由和 Xray 是否监听，**不要直接重置密钥**。密钥错误通常不会表现为 TCP 连接超时；超时说明 TCP 三次握手都没完成。

查看监听时也要区分：

```bash
ss -tlnp | grep ':443 '   # Reality，TCP
ss -ulnp | grep ':443 '   # Hysteria2，UDP
```

不要用浏览器直接打开 `https://你的IP:443` 来判断 Reality 是否正常。普通 HTTPS 客户端不会带 Reality 需要的 SNI、公钥和 Short ID，结果没有参考意义。

## 7. 做完检查

- [ ] `xray version` 有输出。
- [ ] `xray run -test` 通过。
- [ ] `systemctl is-active xray` 为 `active`。
- [ ] `ss -tlnp | grep ':443 '` 显示 Xray 监听。
- [ ] UFW 和服务商上游防火墙都放行 TCP 443。
- [ ] 客户端 URI 的 UUID、SNI、`pbk`、`sid`、`flow` 与服务端一致。
- [ ] 完整 URI 没有出现在公开文件或聊天记录里。

## 卡住了

| 现象 | 先查 |
|---|---|
| `xray run -test` 报 JSON 错误 | 用 JSON 校验器检查引号、逗号和括号；模板里不能有注释。 |
| 443 被占用 | `ss -tlnp \| grep ':443 '` 找进程，决定停服务还是换方案。 |
| 服务启动失败 | `journalctl -u xray -n 100 --no-pager`。 |
| 本地监听正常但客户端超时 | 防火墙/上游安全组/路由；见 [07](07-troubleshooting.md)。 |
| 客户端握手失败 | 重点核对 `sni`、`pbk`、`sid`、`flow`，不要先改服务端。 |
| `pbk` 填了私钥 | 重新看 `x25519` 输出：客户端要 `Password`/`Public key`，不是 `PrivateKey`。 |

[← 上一章：服务器初始化](02-server-setup.md) · [下一章：Hysteria2 →](04-hysteria2.md)
