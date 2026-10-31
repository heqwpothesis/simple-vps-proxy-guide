---
layout: default
title: 06 · 客户端
---

# 06 · 客户端：桌面与手机（Throne、Clash Verge、v2rayNG、Hiddify）

{% include page-toc.html %}

[← 上一章：Cloudflare、TLS 与 CDN](05-cloudflare-tls.md) · [下一章：故障排查 →](07-troubleshooting.md)

服务端显示 `active`、端口在监听，只说明进程活着。这一章把服务端参数填进客户端，并用一次真实 HTTPS 请求确认：**流量确实从 VPS 出去了**。

## 本章目标

- 在 Throne、Clash Verge / Mihomo 中导入 VLESS 或 Hysteria2 节点。
- 在 Android（v2rayNG / Hiddify）和 iOS（Hiddify）上导入节点并连接。
- 知道客户端每个字段对应服务端的哪个值。
- 用 `curl` 或手机浏览器验证实际出口 IP。
- 理解 System Proxy、TUN 和规则分流的区别。
- 避免多个代理/VPN 工具同时争用系统代理。

## 开始前

- 完成 [03](03-vless-reality.md) 或 [04](04-hysteria2.md) 中的至少一个。
- 手上有完整的 URI，或服务端参数（地址、端口、UUID/密码、SNI、公钥、Short ID）。
- 本地有终端，能运行 `curl`；如果只用手机，可以跳过 `curl`，用手机浏览器完成第 4.3 节的验证。
- 关闭其他正在使用系统代理、TUN 或 VPN 的软件。桌面上的 Throne 和 Clash Verge 同时开，很容易互相覆盖设置；手机上也不要同时连接多个 VPN 客户端。

## 1. 先理解客户端在做什么

```text
浏览器 / App
  └─ 系统代理、TUN 或规则
       └─ 客户端本地端口（例如 127.0.0.1:2080）
            └─ 加密隧道 ──> VPS ──> 目标网站
```

客户端会在本机开一个**本地端口**，通常是 HTTP/SOCKS 混合端口。应用把请求发给这个端口，客户端再通过隧道送到 VPS。

三种接管方式：

| 模式 | 怎么接管流量 | 适合 | 注意 |
|---|---|---|---|
| **System Proxy** | 修改操作系统的代理设置 | 浏览器、遵循系统代理的应用 | 有些应用不读系统代理，仍然直连。 |
| **TUN** | 创建虚拟网卡，接管路由 | 需要覆盖不遵守代理的应用 | 可能影响 DNS、局域网和设备路由，需单独测试。 |
| **Rule / 规则** | 按域名或 IP 决定直连还是代理 | 国内外分流、避免本地服务绕路 | 规则写错会把该直连的流量也送出国。 |

客户端日志里的 `sing-box started`、`outbound connection` 只能说明进程启动、请求开始进入出站，**不等于最终 HTTP 请求成功**。以 `curl` 和实际网页为准。

## 2. Throne（Windows / macOS / Linux）

官方项目：[Throne Releases](https://github.com/throneproj/Throne/releases)。Throne 是跨平台桌面客户端，Release 里同时提供 Windows、macOS 和 Linux 构建，按系统和 CPU 架构选择：

- **Windows**：`windows64.zip` 或 `windows-arm64.zip`；也可以直接用 `windows-universal-installer.exe`。老系统有 `windowslegacy64`。
- **macOS**：Apple Silicon 选 `macos-arm64.zip`，Intel 选 `macos-amd64.zip`；老系统有 `macoslegacy-amd64`。
- **Linux**：按发行版选 `.deb` 或 `.rpm`，也可以下载 `linux-amd64.zip` / `linux-arm64.zip`。

文件名会随版本变化，以 Release 页面实际显示为准。Windows 和 macOS 首次打开可能触发系统安全提示，确认来源后按系统提示放行。

步骤：

1. 从官方 Release 安装，完成系统安全验证。
2. 复制 `vless://` 或 `hysteria2://` URI。
3. 使用 Throne 的“从剪贴板导入”。
4. 核对字段：
   - Hysteria 协议版本必须是 **2**。
   - Address、Port、Password/Auth、SNI 与服务端一致。
5. HY2 服务端没有配置混淆时，客户端不要启用 obfs；证书正常时不要勾选 `Allow insecure`。
6. 选中节点并启动。需要时再开启 **System Proxy**。

### 2.1 两种节点的字段对照

| 字段 | VLESS + Reality | Hysteria2 |
|---|---|---|
| Address | VPS 公网 IP（或解析到它的域名） | `hy2.example.com` |
| Port | `443` | `443` |
| 用户凭据 | UUID | `用户名:密码` |
| SNI | Reality 目标域名（`serverNames` 之一） | 证书域名 `hy2.example.com` |
| 额外参数 | `pbk`、`sid`、`flow=xtls-rprx-vision` | `sni`，通常没有 `pbk`/`sid` |
| 证书 | Reality 借用目标站点，不需要自己的证书 | 自己的域名证书，不能 `insecure` |

Throne 的界面会随版本变化，字段名称以实际界面为准。本地混合端口在某些版本中默认为 `127.0.0.1:2080`，以日志里的 `mixed-in` 监听地址为准。

## 3. 验证实际出口

**本地：**

```bash
curl --noproxy '' -x http://127.0.0.1:2080 --connect-timeout 5 --max-time 15 -v https://api.ipify.org
```

参数解释：

- `--noproxy ''`：清空 `NO_PROXY`，避免 curl 对 localhost 或某些地址自动直连。
- `-x http://127.0.0.1:2080`：使用本地混合代理；端口按你的客户端实际值改。Throne 常见 `2080`，Clash Verge 常见 `7897`，以设置页或日志为准。
- `-v`：显示 CONNECT、TLS 和 HTTP 过程，第一次排查很有用；确认正常后可以去掉。
- `--connect-timeout 5 --max-time 15`：避免网络不通时一直等待。

正确结果：

```text
< HTTP/2 200        # 也可能是 HTTP/1.1 200
...
你的 VPS 公网 IP
```

同时看服务端日志，应该有对应用户的成功认证记录。`curl` 返回 VPS IP + 服务端认证成功，才算基本通过。

| 现象 | 含义 | 下一步 |
|---|---|---|
| 返回 VPS IP | 代理链路基本正常 | 再用浏览器打开实际目标网站。 |
| 返回家宽 IP | 请求没走代理 | 检查客户端是否启动、System Proxy/TUN 是否开启、端口是否写对。 |
| `407 Proxy Authentication Required` | 本地代理要求认证，或连错了端口 | 确认端口和客户端类型；混合端口通常不需要认证。 |
| `Connection timed out` | 本地端口没监听，或隧道不通 | 看客户端日志、服务端状态和防火墙。 |
| TLS 错误 | SNI、证书、系统时间或目标站点问题 | 核对 SNI/证书；不要先关证书校验。 |
| 浏览器打不开但 curl 成功 | 应用没走代理，或规则/DNS 有问题 | 检查系统代理、TUN、规则和浏览器设置。 |

`api.ipify.org` 只证明出口 IP。它不能证明 DNS 解析、QUIC、视频或登录会话都正常，最后仍要打开实际目标应用。

## 4. 手机客户端：Android 与 iOS

手机端优先用 **Hiddify**：免费、开源，支持 VLESS + Reality 和 Hysteria2，Android 和 iOS 都有。Android 也可以用 v2rayNG（同样支持 Hysteria2）；iOS 也可以用 Shadowrocket、Stash 等付费客户端。

Hiddify 官网：[hiddify.com](https://hiddify.com/)。Android 可用 Google Play，或从[官方 GitHub Releases](https://github.com/hiddify/hiddify-app/releases)下载 APK；iOS 从 App Store 安装。不要从第三方下载站安装。

### 4.1 Android：v2rayNG

官方项目：[v2rayNG Releases](https://github.com/2dust/v2rayNG/releases)。

1. 安装并打开 v2rayNG。
2. 复制第 03/04 章生成的 `vless://` 或 `hysteria2://` 链接。
3. 点右上角 `+`，选择“从剪贴板导入”。
4. 核对字段（与第 2.1 节相同）：UUID、SNI、`pbk`、`sid`、`flow`、端口和传输方式。
5. 选中节点并连接；第一次会请求 Android VPN 权限，允许即可。
6. v2rayNG 支持 Hysteria2，直接把 `hysteria2://` 链接导入即可；如果看不到该节点，先更新到较新版本并确认链接完整。
7. 连接后用第 4.3 节的方法验证。

不同版本的 v2rayNG 按钮名称可能略有差异，以实际界面为准。如果改用 Hiddify，导入步骤与 4.2 相同：复制 URI → `+` → `Clipboard`。

### 4.2 iOS：Hiddify

iOS 的客户端要从 App Store 安装。推荐 Hiddify；如果已经在用 Shadowrocket 或 Stash，也可以导入同一条 URI。

1. 从 [hiddify.com](https://hiddify.com/) 进入 App Store，安装 Hiddify。中国区 App Store 通常不提供这类代理客户端；如果搜不到，需要非中国大陆区 Apple ID，或使用你已经安装的 Shadowrocket/Stash。
2. 复制 `vless://` 或 `hysteria2://` 链接。链接很长时，先发到自己的私密备忘录或文件传输助手，再在手机上复制。
3. 打开 Hiddify，在首页点 `+` → `Clipboard`；也可以选 `Manually`，填一个名字并粘贴 URL。
4. 点中间的连接按钮。iOS 第一次会要求添加 VPN 配置，允许并用 Face ID/密码确认。
5. 状态栏出现 VPN 图标只代表系统 VPN 已接管；是否真的能上网，继续看第 4.3 节。
6. 如果提示 `Invalid URL` 或导入失败，确认复制了完整链接，并把 Hiddify 更新到较新版本；Hysteria2 需要版本支持。

如果使用 **Shadowrocket / Stash**：它们是付费应用，导入 URI 后字段与第 2.1 节相同；Hysteria2 支持取决于版本。不同地区的 App Store 可购买/下载情况不同，以实际商店为准；不要从不明来源侧载应用。

### 4.3 手机端验证与注意事项

1. 先在 Wi-Fi 下连接，再切换到蜂窝网络各测一次。
2. 用手机浏览器打开 `https://api.ipify.org`，应显示你的 VPS 公网 IP，而不是家宽或运营商 IP。
3. 同时看服务端日志，应该出现该用户的认证记录。
4. 再打开一个实际目标网站或 App，不要只停在“已连接”。
5. 如果 Wi-Fi 能用、蜂窝不能用，可能是运营商限制 UDP；临时切到 Reality（TCP 443）。
6. 同一时间只让一个 VPN/代理客户端处于连接状态。多个客户端同时开，iOS/Android 的 VPN 会互相抢占。
7. Android 的电池优化可能杀掉后台 VPN；在系统设置里允许客户端后台运行。
8. 不要把完整 URI 或二维码发到公开群、issue 或截图里；它们等同于账号凭据。

## 5. Clash Verge / Mihomo（Windows、macOS、Linux）

Clash Verge（维护中的版本是 **Clash Verge Rev**）是跨平台图形界面，内部运行 Mihomo 内核；Mihomo 也可以单独运行在 Linux 服务器上。桌面用户优先用 Clash Verge，服务器或需要脚本化时用 Mihomo 内核。

官方项目：[Clash Verge Rev Releases](https://github.com/clash-verge-rev/clash-verge-rev/releases)。

### 5.1 安装 Clash Verge

Release 里提供各平台构建：

- **Windows**：`x64-setup.exe` 或 `arm64-setup.exe`；缺少 WebView2 的系统可用 `fixed_webview2` 版本。
- **macOS**：Intel 选 `x64.dmg`，Apple Silicon 选 `aarch64.dmg`。
- **Linux**：按架构选 `.deb` 或 `.rpm`。

安装后新建或导入一个 profile。Clash Verge 使用 Clash YAML，**不能把 `vless://` 或 `hysteria2://` URI 直接粘成 profile**；需要把节点参数写成下面的 `proxies` 条目，或使用包含该节点的 Clash 订阅。不同版本的菜单名称可能不同，以实际界面为准。

### 5.2 节点 YAML

Mihomo 支持规则分流、`proxy-providers` 和代理组。HY2 节点示例：

```yaml
proxies:
  - name: VPS-HY2
    type: hysteria2
    server: hy2.example.com
    port: 443
    password: "user1:YOUR_PASSWORD"
    sni: hy2.example.com
    skip-cert-verify: false

proxy-groups:
  - name: PROXY
    type: select
    proxies:
      - VPS-HY2
      - DIRECT

rules:
  - DOMAIN,api.z.ai,DIRECT
  - MATCH,PROXY
```

每一段的作用：

- `proxies`：定义一个节点。`password` 是 `用户名:密码`，不是单独密码。
- `proxy-groups`：把节点放进一个手动选择组；`DIRECT` 表示直连。
- `rules`：按顺序匹配。示例把 `api.z.ai` 直连，其余走 `PROXY`。

> 这只是一个结构示例，**不要直接覆盖**已经包含监听端口、DNS、Provider、日志或其他组的生产配置。

如果是 Mihomo 内核/命令行的配置，先备份：

```bash
cp ~/.config/mihomo/config.yaml ~/.config/mihomo/config.yaml.bak.$(date +%F)
```

Clash Verge 的 profile 由图形界面管理；备份时从界面打开 profile 文件，复制到安全位置，不要直接改它的内部目录。

### 5.3 校验与运行

Clash Verge 图形界面：保存或刷新 profile 后，在日志或连接页确认节点加载成功，再选中节点并开启 System Proxy/TUN。它的混合端口常见默认是 `7897`，以设置页为准。

Mihomo 内核/命令行校验配置：

```bash
mihomo -t -d ~/.config/mihomo -f ~/.config/mihomo/config.yaml
```

`-t` 只测试配置，不启动服务；`-d` 指定工作目录；`-f` 指定配置文件。命令不存在时，先确认 Mihomo 的安装路径和实际启动参数，不要假定所有发行版都一样。

使用 `proxy-providers` 和订阅更新脚本时，要防止自建节点被覆盖。最稳妥的做法是给自建节点单独的 provider 或单独的配置文件，并在更新后检查节点是否还在。

部分 Linux 服务器用 `nohup` 后台运行 Mihomo，而不是 systemd。重载前先核对：

```bash
pgrep -af '[m]ihomo'
```

确认进程、控制 API 和启动参数，再决定是发信号、调 API 还是重启。不要直接 `pkill` 一个不认识的进程。

## 6. System Proxy、TUN 与分流（桌面为主）

手机端（v2rayNG / Hiddify）通常直接创建一个系统 VPN，效果类似 TUN；不需要再设置系统代理。下面的选择主要针对桌面客户端。

| 选择 | 优点 | 代价 |
|---|---|---|
| System Proxy | 配置简单，浏览器通常生效 | 应用是否遵循系统代理取决于软件；命令行工具经常不读。 |
| TUN | 覆盖更广，能接管不遵守代理的应用 | 影响 DNS 和路由；可能连不上局域网设备；需要测试。 |
| Rule | 可以国内外分流，减少绕路 | 规则需要维护；写错会导致该直连的流量走代理。 |

建议的顺序：

1. 先用 **System Proxy + 单节点** 跑通 `curl` 和浏览器。
2. 需要覆盖更多应用时，再开 TUN，并单独测试 DNS 和局域网。
3. 最后再配置规则分流；不要把本地可直连的服务都绕到海外 VPS。

## 7. 做完检查

- [ ] 桌面或手机客户端显示节点已启动。
- [ ] 桌面 `curl` 返回 VPS 公网 IP，而不是家宽 IP；手机浏览器访问 `https://api.ipify.org` 也显示 VPS IP。
- [ ] 服务端日志有对应的成功认证记录。
- [ ] 浏览器或手机 App 能打开实际目标网站。
- [ ] 手机在 Wi-Fi 和蜂窝网络下至少各测一次。
- [ ] 需要直连的本地/国内服务仍然正常。
- [ ] 只有一个代理/VPN 工具在接管流量。
- [ ] 完整 URI 没有出现在截图、公开仓库或群聊里。

## 卡住了

| 现象 | 先查 |
|---|---|
| `curl` 返回家宽 IP | 客户端没启动、System Proxy/TUN 没开、端口写错。 |
| `curl` 超时 | 本地端口是否监听；服务端是否 active；TCP/UDP 路径是否通。 |
| `407` | 连到了需要认证的本地代理，或端口对应了别的服务。 |
| TLS 错误 | SNI 与证书域名是否一致；系统时间是否正确；是否误开了 `insecure`。 |
| v2rayNG 导入 HY2 失败 | 更新到较新版本；确认 `hysteria2://` 链接完整；仍失败再换 Hiddify。 |
| 手机显示“已连接”但网页打不开 | 看服务端是否有认证日志；确认导入的是自己的 URI；切换 Wi-Fi/蜂窝对比。 |
| iOS Hiddify 提示 `Invalid URL` | 确认复制了完整 URI；更新 Hiddify；仍不行可换 Shadowrocket/Stash。 |
| 手机 Wi-Fi 可用、蜂窝不可用 | 运营商可能封 UDP；临时切到 Reality（TCP 443）。 |
| 浏览器不走代理 | 应用不读系统代理；改用 TUN，或检查应用自身的代理设置。 |
| 开了 TUN 后局域网异常 | 检查路由和 DNS 设置；暂时关 TUN 对比。 |

[← 上一章：Cloudflare、TLS 与 CDN](05-cloudflare-tls.md) · [下一章：故障排查 →](07-troubleshooting.md)
