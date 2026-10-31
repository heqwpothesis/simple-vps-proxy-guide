---
layout: default
title: 02 · Debian 初始化
---

# 02 · Debian 初始化、UFW 与 SSH

{% include page-toc.html %}

[← 上一章：VPS 选型](01-vps-selection.md) · [下一章：VLESS + Reality →](03-vless-reality.md)

这一章把“刚买来的机器”变成“可以安全远程管理、且不会把自己锁在外面”的服务器。

> **执行环境**：除特别标注“本地”的步骤外，命令都在**服务器**上执行。开始前先打开服务商的 Web Console，确认它能登录。改防火墙或 SSH 时，这是你的恢复入口。

## 本章目标

- 更新系统，安装后面要用的工具。
- 用 UFW 只放行必要端口。
- 在本地生成 SSH 密钥，把公钥上传到服务器。
- 在新终端验证公钥登录，确认后再关闭旧会话。
- 知道被防火墙锁在外面时怎么用 Console 恢复。

## 开始前

- 完成 [01](01-vps-selection.md)，知道 VPS 公网 IP。
- 服务商 Web Console 可以打开。
- 本地有终端：macOS“终端”、Windows Terminal/PowerShell，或 Linux 终端。
- 有 root 密码或服务商提供的初始凭据。
- 大约 15–30 分钟，期间不要关闭唯一一个 SSH 会话。

## 0. 先分清命令在哪执行

| 标记 | 含义 | 怎么进入 |
|---|---|---|
| **服务器** | 你的 VPS | SSH 登录，或服务商 Web Console |
| **本地** | 你自己的电脑 | 系统终端 |

代码块中整行以 `#` 开头的都是注释，不要输入。实际终端里的 root 提示符通常以 `#` 结尾，别和注释混淆。

## 1. 更新系统与安装工具

**服务器：**

```bash
apt update && apt upgrade -y
apt install -y curl ca-certificates openssl ufw nano dnsutils tcpdump
```

这两条命令在做什么：

- `apt update`：刷新软件包索引，不安装更新。
- `apt upgrade -y`：安装已有更新，`-y` 表示自动确认。遇到配置文件冲突的提示时，先读文字再选，不确定就保持默认；不要盲目覆盖现有配置。
- 第二条安装后面会用到的工具：

| 软件包 | 用途 |
|---|---|
| `curl` | 下载安装脚本、测试 HTTP 请求 |
| `ca-certificates` | 验证 HTTPS 证书 |
| `openssl` | 生成随机密码、查看证书 |
| `ufw` | 配置本机防火墙 |
| `nano` | 简单的命令行文本编辑器 |
| `dnsutils` | 提供 `dig`，检查 DNS 解析 |
| `tcpdump` | 抓包，排查网络问题 |

如果更新了内核，系统可能提示需要重启。先完成本章的 SSH 验证，再执行 `reboot`，不要在只有一条 SSH 会话时重启。

## 2. 防火墙：只开需要的门

UFW 是 Debian/Ubuntu 上的防火墙配置工具。它只管理**服务器本机**的规则；服务商控制台里可能还有一层“安全组/上游防火墙”，两层都要放行。

先确认 SSH 实际监听端口。下面假设是 TCP 22：

```bash
ss -tlnp | grep ':22 '
```

`ss` 的参数：`-t` 只看 TCP，`-l` 只看监听，`-n` 显示数字端口，`-p` 显示进程。输出里应该能看到 `sshd`。如果端口不是 22，把后面所有 `22` 换成实际端口。

设置默认策略并放行端口：

```bash
ufw default deny incoming
ufw default allow outgoing
ufw allow 22/tcp
ufw allow 443/tcp    # Xray Reality
ufw allow 443/udp    # Hysteria2
ufw allow 80/tcp     # ACME HTTP-01
ufw status verbose
```

| 端口 | 协议 | 用途 | 什么时候需要 |
|---|---|---|---|
| 22 | TCP | SSH 远程登录 | 现在，且必须保留 |
| 443 | TCP | Xray Reality | 第 03 章 |
| 443 | UDP | Hysteria2 | 第 04 章 |
| 80 | TCP | ACME HTTP-01 证书验证 | 第 04/05 章；用 DNS-01 时可不开放 |

放行端口不会自动启动服务。没有程序监听时，外部仍然连不上；防火墙只是决定“门卫放不放行”。

**确认 Web Console 可用、规则列表正确后**，再启用：

```bash
ufw enable
```

UFW 可能提示 “Command may disrupt existing ssh connections”。因为你已经放行 22/tcp，可以继续。启用后立刻在**新终端**测试 SSH，确认没被锁在外面。

如果新终端连不上，用 Web Console 登录服务器：

```bash
ufw status verbose
ufw allow 22/tcp
```

仍然不行，可以临时关闭再修：

```bash
ufw disable
```

修好规则后重新 `ufw enable`。不要长期关闭防火墙来“图省事”。

## 3. SSH 公钥登录

SSH 密钥是一对数学钥匙：

- **私钥**留在你的电脑上，相当于钥匙，不能发给任何人。
- **公钥**放到服务器上，相当于锁；服务器用它验证持有私钥的人。

使用公钥登录后，服务器不再需要传输密码。私钥始终不离开本地。

### 3.1 在本地生成密钥

**本地：**

```bash
ssh-keygen -t ed25519 -a 100 -f ~/.ssh/id_ed25519_myvps -C myvps
```

参数含义：

- `-t ed25519`：使用 Ed25519 算法，密钥短、速度快，现代系统普遍支持。
- `-a 100`：对私钥口令做 100 轮 KDF，增加暴力破解成本。
- `-f ~/.ssh/id_ed25519_myvps`：私钥保存路径；公钥会自动生成在同目录的 `.pub` 文件。
- `-C myvps`：给密钥加注释，方便以后辨认。

建议设置私钥口令。设置后每次使用要输入，或者由 `ssh-agent` 缓存。不要为了方便设空口令，除非你明确知道风险。

生成后会得到两个文件：

```text
~/.ssh/id_ed25519_myvps       # 私钥，留在本地
~/.ssh/id_ed25519_myvps.pub   # 公钥，上传到服务器
```

### 3.2 上传公钥

**本地：**

```bash
cat ~/.ssh/id_ed25519_myvps.pub | ssh root@203.0.113.10 'umask 077; mkdir -p ~/.ssh; cat >> ~/.ssh/authorized_keys'
```

这条命令把本地公钥通过 SSH 追加到服务器的 `authorized_keys`：

- `cat` 读取本地公钥。
- `|` 把内容交给后面的 `ssh`。
- `umask 077` 保证新建文件只有属主可读写。
- `mkdir -p ~/.ssh` 已存在也不会报错。
- `>>` 是追加，不会覆盖服务器上已有的其他公钥。

这次连接仍会要求输入 root 密码。第一次连接如果询问主机指纹，核对服务商控制台提供的信息（如果有）。

> 不要在服务器上为本地用户生成私钥。私钥应只存在你的电脑；服务器只需要公钥。

### 3.3 在服务器上检查权限

**服务器：**

```bash
chmod 700 /root/.ssh
chmod 600 /root/.ssh/authorized_keys
sshd -t
systemctl status ssh --no-pager
```

`sshd` 对权限很敏感：如果 `.ssh` 或 `authorized_keys` 允许组或其他用户写，公钥登录会被拒绝。`sshd -t` 只检查配置语法，不检查密钥是否正确；没有输出通常表示语法通过。

### 3.4 用新终端验证

**本地，打开一个新的终端窗口**（不要关闭当前会话）：

```bash
ssh -i ~/.ssh/id_ed25519_myvps root@203.0.113.10
```

如果设置了私钥口令，会先要求输入口令；之后应该直接进入服务器，而不再询问 root 密码。

**只有在新的终端确认登录成功后**，才关闭旧会话。暂时保留密码认证作为备用；确认公钥稳定可用后，再考虑关闭密码登录。本指南不强制关闭密码认证，避免把你锁在外面。

### 3.5 配置 SSH 别名（可选）

每次输入 IP 和密钥路径很麻烦，可以在**本地** `~/.ssh/config` 中加：

```sshconfig
Host myvps
    HostName 203.0.113.10
    User root
    IdentityFile ~/.ssh/id_ed25519_myvps
    IdentitiesOnly yes
```

之后直接 `ssh myvps`。`IdentitiesOnly yes` 表示只使用这里指定的密钥，避免客户端把一堆密钥都试一遍导致 “Too many authentication failures”。

`~/.ssh/config` 建议设为 `600`：

```bash
chmod 600 ~/.ssh/config
```

### 3.6 关于 root 与 sudo

本指南后续仍以 root 示例，便于和配置文件路径对应。生产环境可以另外创建最小权限管理用户，通过 `sudo` 执行管理命令，降低直接使用 root 的风险。如果你创建了普通用户，后面所有需要写系统配置的命令前要加 `sudo`。

## 4. 做完检查

| 检查项 | 命令 | 期望 |
|---|---|---|
| SSH 服务在运行 | `systemctl is-active ssh` | `active` |
| 22 端口在监听 | `ss -tlnp \| grep ':22 '` | 有 `sshd` 的 `LISTEN` |
| 防火墙规则 | `ufw status numbered` | 22/443/80 已放行，默认拒绝入站 |
| 公钥登录 | 新终端 `ssh -i ... root@IP` | 不输密码即可进入（或只输私钥口令） |
| 配置语法 | `sshd -t` | 无输出或明确通过 |

## 5. 卡住了

| 现象 | 处理 |
|---|---|
| `ufw enable` 后 SSH 超时 | 用 Web Console 登录，`ufw status verbose`，补 `ufw allow 22/tcp`；仍不行则 `ufw disable`，修好再启用。 |
| `Permission denied (publickey)` | 确认 `-i` 路径正确、`authorized_keys` 包含公钥、权限为 700/600；用 `ssh -vvv` 看服务器接受了哪把钥匙。 |
| `Connection timed out` | 通常没到认证层：查服务商上游防火墙、IP 是否正确、22 是否被限制。见 [07](07-troubleshooting.md)。 |
| `sshd -t` 报错 | 按行号修配置；修好前不要重启 SSH，保留当前会话。 |
| `apt upgrade` 弹出配置选择 | 先读说明；不确定保持默认，不要盲目覆盖。 |

[← 上一章：VPS 选型](01-vps-selection.md) · [下一章：VLESS + Reality →](03-vless-reality.md)
