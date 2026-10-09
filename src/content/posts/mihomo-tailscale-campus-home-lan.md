---
title: 基于 mihomo 内置 Tailscale 的家庭内网远程访问方案
published: 2026-05-23
updated: 2026-06-07
description: 记录一套通过 Tailscale subnet router 与 mihomo tailscale outbound 实现异地访问家庭内网的配置方案，并补充 Android 与 Windows 的接入细节。
tags: [mihomo, Tailscale, 网络]
category: 教程
draft: false
---

本文记录一套通过 mihomo 内置 Tailscale 出站能力访问家庭内网的方案。它的核心目标是：在不额外占用 Android 系统 VPNService 的前提下，让外部设备访问家中的软路由、光猫、NAS、虚拟机、LuCI、SSH 等内网服务。

最终链路可以概括为：

```text
外部设备 -> mihomo -> Tailscale 出站 -> 家庭侧 subnet router -> 家庭内网设备
```

这篇文章的场景来自校园网，但方案并不局限于校园网。只要外部设备所在网络无法直接访问家里内网，例如学校、公司、酒店、移动网络或其它 NAT 环境，都可以参考这个思路。

## 方案背景

我的原始需求很明确：人在外面时，希望直接访问家里的内网服务。

如果使用 frp 或其它端口映射工具，需要为每个服务分别暴露端口。服务数量一多，端口管理、访问地址、权限控制和后期维护都会变得麻烦。Tailscale 的 subnet router 更适合这种场景：家庭侧只需要一台长期在线的设备加入 tailnet，并发布家庭内网网段，外部设备就可以像在家里一样访问 `192.168.x.x` 地址。

问题在于 Android。官方 Tailscale App 会占用 Android 的 VPNService，而 Box for Root、Nikki、Clash Meta 等透明代理方案也依赖类似的网络接管能力。两者同时使用时，很容易出现路由冲突或代理分流失效。

mihomo 从 `v1.19.25` 开始加入 Tailscale outbound support，配置中可以直接写 `type: tailscale`。这意味着 mihomo 本身可以作为一个 Tailscale 节点加入 tailnet，然后只把指定网段的流量送进 Tailscale。

因此本文采用下面的分工：

```text
普通上网流量 -> 继续走 mihomo 原有代理分流
家庭内网流量 -> 走 mihomo 内置 Tailscale 出站
```

这样既保留了原来的代理规则，又避免 Android 官方 Tailscale App 占用系统 VPN。

## 网络拓扑

我的实际环境如下：

```text
外部网络
  |
  |-- Android 手机
  |     |-- root
  |     |-- Box for Root
  |     |-- mihomo v1.19.25+
  |
  |-- Windows 电脑
        |-- Clash Party
        |-- mihomo v1.19.25+

公网
  |
  |-- 自建 DERP 中继

家庭网络
  |
  |-- 光猫网段: 192.168.0.0/24
  |-- 软路由 LAN: 192.168.1.0/24
  |-- Debian 虚拟机
        |-- 直连出网
        |-- 运行 Tailscale
        |-- 发布 subnet routes:
              192.168.1.0/24
              192.168.0.0/24
```

家庭侧 Debian 是整套链路的入口。它不需要运行代理客户端，也不建议被家庭软路由上的透明代理二次接管。作为 subnet router，它需要稳定连接 Tailscale 控制面、DERP、STUN 以及其它 tailnet 节点；如果外层连接又被代理套一层，排错会复杂很多。

## 前提条件

本文假设你已经具备以下条件：

- 有一个可正常使用的 Tailscale 账号。
- 可以进入 Tailscale Admin Console。
- 家里有一台长期在线的 Debian 设备，虚拟机或物理机均可。
- Debian 能访问家庭内网，例如 `192.168.1.0/24` 和 `192.168.0.0/24`。
- Android 设备已 root，并运行 Box for Root。
- Windows 设备运行 Clash Party 或其它可使用 Mihomo 内核的客户端。
- Mihomo 内核版本为 `v1.19.25` 或更新版本。

如果启动后出现：

```text
proxy 0: unsupport proxy type: tailscale
```

说明当前内核不支持 `type: tailscale`，需要升级到 `v1.19.25` 或更新版本。

## 家庭侧部署 subnet router

首先在 Debian 上安装 Tailscale：

```bash
curl -fsSL https://tailscale.com/install.sh | sh
```

登录 tailnet：

```bash
sudo tailscale up --hostname=home-debian-router
```

命令会输出登录链接。完成授权后，检查设备状态：

```bash
tailscale status
tailscale ip -4
```

如果可以看到 `100.x.y.z` 形式的 Tailscale IP，说明 Debian 已成功加入 tailnet。

### 开启 IP forwarding

Subnet router 需要帮助其它 tailnet 设备转发到家庭内网，因此 Debian 必须开启 IP 转发：

```bash
echo 'net.ipv4.ip_forward = 1' | sudo tee /etc/sysctl.d/99-tailscale.conf
echo 'net.ipv6.conf.all.forwarding = 1' | sudo tee -a /etc/sysctl.d/99-tailscale.conf
sudo sysctl -p /etc/sysctl.d/99-tailscale.conf
```

检查 IPv4 转发状态：

```bash
sysctl net.ipv4.ip_forward
```

应返回：

```text
net.ipv4.ip_forward = 1
```

### 发布家庭内网路由

我的家庭网络有两个网段：

```text
192.168.1.0/24  软路由 LAN
192.168.0.0/24  光猫网段
```

在 Debian 上发布这两个网段：

```bash
sudo tailscale set --advertise-routes=192.168.1.0/24,192.168.0.0/24
```

这一步只是声明路由，还需要到 Tailscale 后台批准。

打开：

```text
https://login.tailscale.com/admin/machines
```

找到 `home-debian-router`，进入设备设置，在 Subnet routes 中启用刚才发布的两个网段。

如果这台 Debian 会长期作为家庭入口，建议关闭 key expiry。否则设备密钥过期后，subnet route 仍然显示在后台，但实际访问可能失败。

### 关于 SNAT

Tailscale subnet router 默认会启用 SNAT。家庭内网设备看到的来源地址是 Debian subnet router，而不是远端设备的真实 Tailscale IP。

默认 SNAT 的优势是简单：光猫、软路由、NAS 等设备不需要额外配置回程路由。对于普通家庭网络，这是更稳妥的选择。

如果关闭 SNAT，家庭内网设备必须知道如何回到 Tailscale 网段。例如需要有类似下面的回程路由：

```text
100.64.0.0/10 via Debian 的家庭内网 IP
```

只有在明确需要让内网设备看到远端 Tailscale 设备真实 `100.x` 地址时，才建议进一步研究 `--snat-subnet-routes=false` 和回程路由。

## 为 mihomo 生成 auth key

mihomo 的 `type: tailscale` 会让 mihomo 自己作为一个独立 Tailscale 节点加入 tailnet。它不能复用手机或电脑上官方 Tailscale 客户端的身份。

打开：

```text
https://login.tailscale.com/admin/settings/keys
```

生成 auth key，建议设置为：

```text
Reusable: 关闭
Expiration: 1 day
Ephemeral: 关闭
Pre-approved: 如果可用则开启
```

生成后会得到类似：

```text
tskey-auth-xxxxxxxxxxxxxxxx
```

注意不要使用 `tskey-api-`。`tskey-auth-` 用于设备加入 tailnet，`tskey-api-` 是 API 访问令牌，两者用途不同。

auth key 只用于首次登录。只要 `state-dir` 保留，后续重启不需要重新生成 key。`auth-key` 和 `state-dir` 都属于敏感信息，不要提交到公开仓库。

## Android + Box for Root 配置

Android 上不要开启官方 Tailscale App，而是让 Box for Root 中的 mihomo 作为 Tailscale 节点入网。

在 `proxies` 中新增：

```yaml
proxies:
  - name: Tailscale-Home
    type: tailscale
    hostname: android-box
    auth-key: "tskey-auth-REPLACE_ME"
    control-url: https://controlplane.tailscale.com
    state-dir: /data/adb/box/mihomo/tailscale
    ephemeral: false
    udp: true
    accept-routes: true
    ip-version: ipv4-prefer
```

几个字段需要特别注意：

- `hostname`：这台 mihomo 节点在 Tailscale 后台显示的名称。
- `auth-key`：首次加入 tailnet 使用的登录密钥。
- `state-dir`：保存 tsnet 状态和节点身份的目录。
- `accept-routes`：接受家庭 subnet router 发布的内网路由。
- `udp`：启用 UDP，有助于 Tailscale 直连和部分应用通信。

`state-dir` 不能多个设备共用，也不要从手机复制到电脑。每台设备都应该有独立身份。

### 配置家庭内网策略组

新增一个专门处理家庭内网的策略组：

```yaml
proxy-groups:
  - name: Home LAN
    type: select
    proxies:
      - Tailscale-Home
      - DIRECT
      - Proxy
```

这样可以在不同环境下切换：

```text
Tailscale-Home  外部网络访问家庭内网
DIRECT          在家中 Wi-Fi 下直连访问
Proxy           临时排错
```

如果你的主选择组使用了 `include-all: true`，建议排除 `Tailscale-Home`，避免它混入普通代理节点：

```yaml
exclude-filter: "^(Tailscale-Home)$"
```

### 规则顺序

规则顺序非常关键。如果配置中已有：

```yaml
- RULE-SET,private_ip,DIRECT,no-resolve
```

那么 `192.168.x.x` 会提前被送去直连，无法进入 Tailscale。家庭网段规则必须放在 `private_ip` 规则之前：

```yaml
rules:
  - IP-CIDR,192.168.1.0/24,Home LAN,no-resolve
  - IP-CIDR,192.168.0.0/24,Home LAN,no-resolve
  - IP-CIDR,100.64.0.0/10,Home LAN,no-resolve
  - RULE-SET,private_ip,DIRECT,no-resolve
```

`100.64.0.0/10` 是 Tailscale 使用的 CGNAT 网段。加入该规则后，可以访问 tailnet 中其它设备的 `100.x.y.z` 地址。

### 重启 Box for Root

保存配置后重启 Box for Root：

```bash
su
/data/adb/box/scripts/box.service restart
/data/adb/box/scripts/box.iptables renew
```

随后到 Tailscale 后台检查是否出现 `android-box`。访问家庭内网地址，例如：

```text
http://192.168.1.1
http://192.168.0.1
```

在 Mihomo 面板中，目标为 `192.168.x.x` 的连接应命中：

```text
Home LAN -> Tailscale-Home
```

## Windows + Clash Party 配置

Windows 上可以有两种方式：

```text
不开 TUN：仅浏览器或支持代理的软件走 127.0.0.1:7890/7891
开启 TUN：让不支持代理的软件也能访问家庭内网
```

如果只需要浏览器访问 LuCI、NAS Web 面板，不开 TUN 也能用。但 FinalShell、SSH、部分启动器或客户端不一定遵守系统 HTTP 代理，因此更推荐开启 TUN，并只接管家庭网段。

### 使用独立 Tailscale 身份

Windows 不要复用 Android 的 `hostname`、`auth-key` 或 `state-dir`。建议写成：

```yaml
proxies:
  - name: Tailscale-Home
    type: tailscale
    hostname: win-mihomo
    auth-key: "tskey-auth-REPLACE_ME"
    control-url: https://controlplane.tailscale.com
    state-dir: ./tailscale-state
    ephemeral: false
    udp: true
    accept-routes: true
    ip-version: ipv4-prefer
```

每台设备都应使用独立 auth key 首次登录，并保留自己的 `state-dir`。

### 系统代理绕过列表

Windows 系统代理绕过列表中经常包含：

```text
localhost;127.*;192.168.*;10.*;172.16.*;...
```

如果保留 `192.168.*`，浏览器访问 `192.168.1.1` 时不会走 Clash Party，也不会进入 `Tailscale-Home`。如果只是浏览器访问，可以删除这一项。

但这仍然解决不了所有软件，因为很多程序不会使用系统 HTTP 代理。此时需要 TUN。

### TUN 只接管家庭网段

不要为这个场景开启全局 TUN。全局接管容易把 DNS、普通代理流量，甚至 Tailscale 自身外层连接重新抓回 Mihomo，形成回环。常见现象是日志中出现类似：

```text
reject loopback connection to 223.5.5.5:53
reject loopback connection to 119.29.29.29:53
```

更合适的做法是只把家庭内网和 Tailscale CGNAT 网段交给 TUN：

```yaml
tun:
  enable: true
  stack: mixed
  device: Mihomo
  auto-route: true
  auto-detect-interface: true
  strict-route: false
  route-address:
    - 192.168.1.0/24
    - 192.168.0.0/24
    - 100.64.0.0/10
```

在这个场景下，不建议同时开启：

```yaml
dns-hijack:
auto-redirect:
strict-route: true
```

最终路径为：

```text
FinalShell / 浏览器 / 任意软件
  -> Windows 路由表命中 192.168.1.0/24
  -> Mihomo TUN
  -> Home LAN
  -> Tailscale-Home
  -> 家庭侧 Debian subnet router
  -> 家庭内网设备
```

普通网站、普通 DNS 和日常代理分流不会进入这条 TUN 路由，因此原来的代理逻辑不会被破坏。

### Windows 测试

不开 TUN时，可以强制走 HTTP 代理测试：

```powershell
curl.exe -v -x http://127.0.0.1:7890 http://192.168.1.1/ --connect-timeout 5
```

开启只接管家庭网段的 TUN 后，直接测试：

```powershell
curl.exe -v http://192.168.1.1/ --connect-timeout 5
```

如果成功，通常会看到：

```text
HTTP/1.1 200 OK
```

这时 FinalShell 可以直接填写家庭内网地址：

```text
Host: 192.168.1.x
Port: 22
```

不需要为每个 SSH 连接单独指定 SOCKS5。

## 验证 Tailscale 直连状态

在家庭侧 Debian 上可以测试到 Android 或 Windows mihomo 节点的连接：

```bash
tailscale ping android-box
tailscale ping win-mihomo
```

如果看到公网 IP 和端口，通常表示已经 UDP 直连。如果显示 DERP 或 relay 相关字样，则说明当前走中继。

可以持续观察：

```bash
tailscale ping --c=0 --until-direct=false win-mihomo
```

如果你的 Tailscale 版本不支持持续 ping，可以用循环：

```bash
while true; do
  tailscale ping --c=1 --timeout=3s --until-direct=false win-mihomo
  sleep 1
done
```

Android 侧延迟比 Windows 更抖是正常现象。Wi-Fi 省电、后台调度、root 透明代理以及 tsnet 用户态栈都会带来额外波动。只要不是频繁掉 DERP 或超时，访问 LuCI、NAS、SSH 一般都能接受。

## 常见问题

### 后台看到新设备，但访问内网不通

按顺序检查：

```text
1. Debian 是否发布了家庭内网网段。
2. Tailscale 后台是否批准了 subnet routes。
3. mihomo 是否启用了 accept-routes。
4. 家庭网段规则是否位于 private_ip,DIRECT 前面。
5. Home LAN 策略组是否选中 Tailscale-Home。
6. Windows 是否被系统代理绕过列表拦截。
7. TUN 是否接管范围过大，导致 DNS 或外层连接回环。
```

### `curl -x 127.0.0.1:7893` 连接拒绝

说明 Clash Party 实际没有监听 `7893`。检查端口：

```powershell
netstat -ano | findstr "7890 7891 7892 7893 7894 9090"
```

很多配置中 HTTP 或 mixed 端口是 `7890`，SOCKS 端口是 `7891`。

### 可以复用同一个 auth key 和 state-dir 吗

不建议。每台设备都应有独立的：

```text
hostname
auth-key
state-dir
```

`state-dir` 是设备身份，不要多个设备共用。

### Windows 官方 Tailscale 客户端是否需要开启

二选一即可。

如果使用 Windows 官方 Tailscale 客户端，家庭网段可以直接走系统 Tailscale 路由，mihomo 中不一定需要 `type: tailscale`。

如果希望所有分流规则都在 Clash Party/mihomo 中统一管理，就关闭官方客户端，使用 mihomo 内置 `Tailscale-Home`。

Android 更适合使用 mihomo 内置 Tailscale，因为官方 App 会占用 VPNService。

## 总结

这套方案的关键不是全局代理，而是按网段接管。

家庭侧 Debian 负责把传统内网发布到 tailnet：

```text
192.168.1.0/24
192.168.0.0/24
```

外部设备上的 mihomo 负责把访问这些网段的流量送进内置 Tailscale：

```text
192.168.1.x -> Home LAN -> Tailscale-Home
192.168.0.x -> Home LAN -> Tailscale-Home
100.x.y.z   -> Home LAN -> Tailscale-Home
```

Windows TUN 只负责接管不支持代理的软件访问家庭网段时产生的流量：

```text
FinalShell -> 192.168.1.x -> Mihomo TUN -> Tailscale-Home
```

普通上网、DNS 和日常代理分流仍然按原有 Mihomo 配置工作。

一句话概括：

```text
家庭侧 Debian 发布内网路由；
mihomo 内置 Tailscale 接入 tailnet；
Windows TUN 只接管家庭网段；
Android 不开启官方 Tailscale App，避免 VPNService 冲突。
```

最终效果是，在外部网络中也可以直接访问 `192.168.1.1`、`192.168.0.1` 或其它家庭内网地址。

## 参考链接

- [mihomo v1.19.25 Release](https://github.com/MetaCubeX/mihomo/releases/tag/v1.19.25)
- [mihomo Tailscale 出站文档](https://wiki.metacubex.one/config/proxies/tailscale/)
- [mihomo TUN 文档](https://wiki.metacubex.one/config/inbound/tun/)
- [Tailscale Linux 安装文档](https://tailscale.com/docs/install/linux)
- [Tailscale subnet router 文档](https://tailscale.com/docs/features/subnet-routers)
- [Tailscale auth keys 文档](https://tailscale.com/docs/features/access-control/auth-keys)
