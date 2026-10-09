---
title: 基于 SubStore 与 Mihomo 的链式代理配置方案
published: 2026-01-01
updated: 2026-06-07
description: 记录一套通过 SubStore 聚合订阅、注入 dialer-proxy 配置，并在 Mihomo 多平台客户端中统一使用的链式代理方案。
image: "/images/posts/mihomo-chain/mihomo-chain-cover.png"
tags: [mihomo, SubStore, 代理]
category: 教程
draft: false
---

本文记录一套基于 SubStore 与 Mihomo 的链式代理配置方案。它的目标不是单纯“叠一层代理”，而是在入口线路质量与出口 IP 质量之间取得平衡：使用线路较好的节点作为中转入口，再将流量转发到自建落地节点，由自建节点作为最终出口访问目标网站。

最终链路可以概括为：

```text
本地客户端 -> 中转节点 Relay -> 自建落地节点 Exit -> 目标网站
```

这类方案适合已经同时拥有机场/专线节点和自建 VPS 的用户。中转节点负责改善到落地节点的网络路径，自建 VPS 则提供相对独立、可控的出口 IP。它不适合追求“最简单配置”的场景，也不应该被理解为无条件提升速度的通用方案；链路变长后，延迟、MTU、UDP 兼容性和节点稳定性都需要额外关注。

## 为什么不再使用 relay

过去 Mihomo 中常见的链式代理做法是使用 `relay` 策略组。但在当前 Mihomo 文档中，`relay` 已经被标记为弃用，更推荐使用 `dialer-proxy`。`dialer-proxy` 的含义是：让某个出站代理在建立连接时，先通过另一个代理或策略组拨出。

以落地节点 `Exit` 为例：

```yaml
proxies:
  - name: Exit
    type: ss
    server: your-vps.example.com
    port: 8388
    cipher: 2022-blake3-aes-128-gcm
    password: "REPLACE_ME"
    dialer-proxy: Relay
```

当客户端选择 `Exit` 访问目标网站时，Mihomo 会先通过 `Relay` 策略组建立到 `Exit` 的连接，再由 `Exit` 访问最终目标。目标网站看到的是 `Exit` 的出口 IP；本地网络侧看到的则是客户端正在连接 `Relay`。

本文使用 SubStore 的原因，是把这类 `dialer-proxy` 注入逻辑放在订阅后端统一维护。客户端只导入最终生成的配置，不需要在每台设备上重复编辑落地节点。

## 方案结构

本文将节点拆成两个职责明确的订阅池：

- `Relay`：中转节点池，通常来自机场、专线入口或其他线路较好的节点。
- `Exit`：落地节点池，通常是自建 VPS 节点，作为最终出口。

配置的关键点是：只在 `Exit` 节点上注入 `dialer-proxy`，让落地节点通过 `Relay` 策略组拨出。`Relay` 自身不要再指向 `Exit`，两个订阅池也不要互相包含。

:::warning[避免循环引用]
`Relay` 与 `Exit` 必须保持单向关系：`Exit -> Relay`。不要让 `Relay` 订阅里包含落地节点，也不要让 `Exit` 再被其它规则反向选回 `Relay`。节点名称也应避免重复，否则容易出现选择混乱或连接循环。
:::

## 准备条件

开始前需要确认以下条件：

- 已部署并可以正常访问 SubStore。
- 至少有一组可作为中转入口的节点。
- 至少有一个自建落地节点，例如 Shadowsocks、VMess 或 VLESS。
- 使用的 Mihomo 内核支持 `dialer-proxy`。
- 客户端可以导入 Mihomo 配置或通用订阅。

落地节点协议建议尽量选择简单、稳定、对中转友好的类型。Mihomo 文档也提醒过，如果没有特殊需求，被中转的自建 VPS 节点不建议优先选择 Hysteria2、TUIC、WireGuard 这类 UDP 或伪装复杂的协议。实践中，简单的 Shadowsocks AEAD 或 VMess 往往更容易排查。

## 在 SubStore 创建中转订阅

首先在 SubStore 中创建一个只包含中转节点的订阅。

建议命名为：

```text
Mihomo-chain-relay
```

这个订阅可以来自单条机场订阅，也可以由多个订阅组合而成。它的目标是提供“从本地到落地节点之间”的连接入口，因此只需要保留适合中转的节点。可以按地区、倍率、剩余流量、节点名称等条件做过滤，但不要把自建落地节点放进这个订阅池。

如果有多个机场，可以先分别添加为单条订阅，再通过组合订阅聚合。这样后期更换中转节点时，不需要改客户端配置。

## 在 SubStore 创建落地订阅

然后创建第二个订阅，只包含自建落地节点。

建议命名为：

```text
Mihomo-chain-exit
```

这个订阅需要通过脚本给每个落地节点添加中转信息。脚本内容为：

```javascript
$server["dialer-proxy"] = "Relay";
```

其中 `Relay` 必须与最终 Mihomo 配置中的策略组名称完全一致。如果你在模板中把策略组命名为 `中转节点`，这里也要同步改成：

```javascript
$server["dialer-proxy"] = "中转节点";
```

如果旧配置中使用过 `underlying-proxy`，建议先确认最终生成的 YAML。为了和当前 Mihomo 文档保持一致，本文统一使用 `dialer-proxy`。

## 生成分享链接

建议不要直接把原始机场订阅或自建节点订阅暴露给客户端，而是通过 SubStore 的分享功能生成独立链接。

操作步骤：

1. 点击订阅右侧菜单。
2. 进入分享管理。
3. 设置有效期。
4. 使用高强度 Token。
5. 分别复制中转订阅和落地订阅的分享链接。

![创建订阅分享链接](/images/posts/mihomo-chain/mihomo-chain-create-share.png)

![设置分享链接有效期](/images/posts/mihomo-chain/mihomo-chain-Set%20validity%20period.png)

Token 应视为敏感信息。如果分享链接泄露，别人可以获取你的节点池。建议设置合理有效期，并定期轮换。

## 添加 Mihomo 配置模板

接下来需要准备一个 Mihomo 配置模板，让它同时引用上面两个 SubStore 分享链接。

示例结构如下：

```yaml
proxy-providers:
  Mihomo-chain-relay:
    type: http
    url: "填入中转订阅分享链接"
    interval: 86400
    health-check:
      enable: true
      url: https://www.gstatic.com/generate_204
      interval: 300
    proxy: DIRECT

  Mihomo-chain-exit:
    type: http
    url: "填入落地订阅分享链接"
    interval: 86400
    health-check:
      enable: true
      url: https://www.gstatic.com/generate_204
      interval: 300
    proxy: DIRECT

proxy-groups:
  - name: Relay
    type: select
    use:
      - Mihomo-chain-relay

  - name: Exit
    type: select
    use:
      - Mihomo-chain-exit

  - name: Proxy
    type: select
    proxies:
      - Exit
      - Relay
      - DIRECT

rules:
  - MATCH,Proxy
```

也可以使用我的模板作为起点：

- [mihomo-chain.yaml](https://raw.githubusercontent.com/Inklazy/mihomo-chain/refs/heads/main/yaml/mihomo-chain.yaml)

需要注意三处名称必须对应：

- SubStore 落地订阅脚本中的 `dialer-proxy` 值。
- Mihomo 配置中的 `Relay` 策略组名称。
- `proxy-providers` 中的 provider 名称。

如果这三处不一致，最终表现通常是落地节点无法连接，或者面板中只看到直连落地而没有中转链路。

在 SubStore 中添加配置文件时，可以进入文件管理，新增本地文件，将模板内容粘贴进去。保存后点击该文件，选择通用订阅，即可得到最终导入客户端的订阅链接。

![添加 Mihomo 配置文件](/images/posts/mihomo-chain/mihomo-chain-flie.png)

## 客户端使用方式

在 Windows、Android、iOS 或路由器端客户端中导入上一步生成的通用订阅后，通常只需要关注三个策略组：

- `Relay`：选择中转节点。
- `Exit`：选择自建落地节点。
- `Proxy`：选择 `Exit`，让主流量进入链式代理。

如果只想临时测试中转节点，也可以在 `Proxy` 中直接选择 `Relay`。但正常链式代理场景下，主策略组应选择 `Exit`，因为 `Exit` 节点本身已经通过 `dialer-proxy` 指向 `Relay`。

我测试过的客户端包括 Windows Clash Party、Android FlClash、Android Box 模块、OpenClash、Nikki 和 iOS Clash Mi。不同客户端对面板展示、订阅刷新和 provider 健康检查的处理略有差异，但只要最终 YAML 结构一致，核心逻辑是相同的。

## 如何验证链式代理生效

验证时可以从三处观察：

1. 访问 IP 查询网站，出口 IP 应显示为自建落地节点的 IP。
2. 在 Mihomo 面板中查看连接，目标连接应命中 `Exit`。
3. 在连接详情中，通常可以看到用于拨出落地节点的内部连接，类型可能显示为 `Inner` 或类似名称。

![面板中的 Inner 连接](/images/posts/mihomo-chain/mihomo-chain-Inner.png)

如果出口 IP 显示为中转节点，说明主策略组选错了，或者落地节点没有正确注入 `dialer-proxy`。如果完全无法连接，优先检查中转节点是否能访问落地节点端口、落地节点协议是否适合被中转，以及 provider 中是否存在同名节点。

## 常见问题

### 选择什么节点做 Relay

Relay 的核心要求是到本地和到落地 VPS 都稳定。并不是延迟最低的节点一定最适合做中转。建议优先选择线路稳定、丢包低、倍率可接受、对落地 VPS 所在地区连通性较好的节点。

### 选择什么节点做 Exit

Exit 应该是你希望目标网站看到的出口 IP。它可以是自建 VPS，也可以是你可控的专用节点。落地节点更看重 IP 质量和服务端稳定性，而不是本地直连延迟。

### 为什么不要混用 Relay 和 Exit 订阅

因为 `dialer-proxy` 是节点级拨号关系。如果中转池里混入落地节点，或者落地池里混入中转节点，实际选择时很容易出现自己拨自己、重复套链或策略组循环。轻则连接失败，重则导致内核资源异常增长。

### 订阅下载是否也会走中转

本文示例中 provider 的 `proxy` 为 `DIRECT`，表示订阅下载本身直连获取。如果你的网络环境无法直连下载订阅，需要额外设置 provider 下载使用的代理。这个问题与链式代理本身不同，不建议和正文链路混在一起排查。

## 总结

这套方案的关键是把链路拆清楚：

```text
Relay 负责中转入口
Exit 负责最终出口
SubStore 负责把 dialer-proxy 注入到 Exit
Mihomo 客户端只负责选择策略组
```

相比旧的 `relay` 策略组，`dialer-proxy` 更符合当前 Mihomo 的配置方向；相比在每台客户端手动修改节点，SubStore 后端注入也更便于维护。只要订阅池职责清晰、节点名称不冲突、模板分组名称保持一致，这套配置可以比较稳定地复用到多个平台。

## 参考链接

- [Mihomo dialer-proxy 文档](https://wiki.metacubex.one/config/proxies/dialer-proxy/)
- [Mihomo relay 弃用说明](https://wiki.metacubex.one/config/proxy-groups/relay/)
- [SubStore 项目](https://github.com/sub-store-org/Sub-Store)
- [示例配置模板](https://raw.githubusercontent.com/Inklazy/mihomo-chain/refs/heads/main/yaml/mihomo-chain.yaml)
