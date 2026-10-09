---
title: 在 Cloudflare Workers 上跑一个 OKX P2P 汇率监控机器人：Cron 定时与 Telegram 联动
published: 2026-09-14
description: 记录使用 Cloudflare Workers、KV 存储与 Telegram Bot 构建高可用、免服务器的 OKX C2C 实时汇率监测系统的全过程，支持动态多监控项管理与自定义报警。
tags: [Cloudflare, TypeScript, Telegram Bot, Webhook, 自动化]
category: 开发记录
draft: false
---

平时关注数字货币 C2C/P2P 交易汇率时，经常需要在交易所 App 里来回切换查看商户挂单。如果能有一个轻量级的机器人，在群组或私聊里每小时定时播报当前的买入/卖出真实牌价，还能随时通过指令增删自定义监控项，体验就会舒服很多。

但如果为了这么一个小功能单独在 VPS 上跑常驻 Python 脚本或 Node.js 服务，不仅费电费钱，还要操心进程守护（PM2）、证书配置和系统重启等琐事。

结合我最近全面转向 Cloudflare Serverless 的技术栈，我用纯 TypeScript 写了一个基于 **Cloudflare Workers + Workers KV + Telegram Webhook** 的自动化监控服务：**Okx-Monitoring**。

代码已完整开源在 GitHub：[Okx-Monitoring](https://github.com/Inklazy/Okx-Monitoring)。

---

## 整体架构设计

整个系统的设计哲学是：**完全无服务器（Serverless）、零服务器运维开销、高频轻量调度**。

```text
                        +-------------------------------------------------------------+
                        |                 Cloudflare Edge Network                     |
                        |                                                             |
触发源 1: Cron Triggers --+                                                             |
 (每小时执行 "0 * * * *")  |                                                             |
                        |                                                             |
触发源 2: Telegram Hook  --+--> Cloudflare Worker (TypeScript / nodejs_compat)         |
 (交互式指令与配置)        |         |                                                |
                        |         +---> fetch() OKX P2P 网页市场接口 (模拟买卖深度)         |
                        |         |                                                |
                        |         +---> Workers KV (MONITORS) (持久化存储监控项配置)    |
                        |         |                                                |
                        |         +---> Telegram Bot API (向群组或管理员异步下发排版播报)   |
                        +-------------------------------------------------------------+
```

系统核心包含三部分：
1. **Cron Triggers（定时触发器）**：在 `wrangler.toml` 中配置 `crons = ["0 * * * *"]`，利用 Cloudflare 全球分布的基础设施，按整点免唤醒执行；
2. **Telegram Webhook**：统一入口路由 `/telegram/webhook`，通过请求头中的 `x-telegram-bot-api-secret-token` 进行防伪签名校验，接收并解析管理员发来的实时指令；
3. **Workers KV**：作为轻量键值数据库，存储当前活跃的监控任务列表（支持币种、法币类型、拟交易金额、支付渠道限制等字段）。

---

## 核心实现与细节

### 1. OKX 市场接口的方向映射

OKX 官方并未提供开放免签的高频 P2P 公共 API。为了获取最贴近普通散户在网页端和 App 端看到的真实商家报价，我们调用的是其面向 Web 端的市场公开数据通道。

在逆向其接口时，有一个非常容易混淆的业务逻辑：
- **普通用户买入加密货币（Buy）**：在交易所前台选择“买入 USDT”，本质上是在筛选那些“正在向市场**出售（Sell）**加密货币的商户广告”；
- 因此在向 OKX 接口发起请求时，如果用户的监控意图是 `side=buy`，底层传给后端的查询参数必须映射为 `side=sell`，反之亦然。

同时，我们针对不同交易规模设置了 `quoteMinAmountPerOrder`（单笔最低成交流水）过滤。例如默认以消费 1000 元 CNY 为基准，自动剔除那些“挂着虚低价格但单笔起售金额要求 5 万元以上”的不可用大额商户广告，提取出散户真正能够立刻成交的前排最优买单。

### 2. 多维度指令管理系统

除了被动的定时播报，机器人还支持丰富的即时管理命令：

```text
/help                                    # 查看指令帮助
/quote CNY USDT amount=3000 pay=aliPay   # 实时查询当前指定金额与支付方式的最优报价
/add CNY USDT amount=1000 pay=aliPay,wxPay chat=here label=日常监测  # 新增一个定时任务
/list                                    # 列出当前所有在跑的监控项
/remove <id>                             # 删除指定的监控配置
/test <id>                               # 立即触发一次指定任务的试运行测试
/payments CNY USDT                       # 查询该交易对目前支持的支付网关代码
```

在权限控制上，我们在 Worker 环境变量中注入了 `ADMIN_USER_IDS`。凡是涉及 `/add`、`/remove` 等修改持久化配置的操作，均要求调用者的 Telegram ID 严格命中管理员白名单，普通群成员或外部路人仅可查看基础报价或被动接收推送。

### 3. 优雅降级与异常保护

在加密货币市场中，某些冷门小币种或特定的冷门法币偶尔会出现流动性枯竭（或者 OKX 临时维护返回特定的业务错误码如 `17007`）。

如果此时没有完善的容错机制，Worker 执行到一半直接抛出未捕获异常，会导致整批定时播报任务全部中断。

为此，我们在 `okx.ts` 和 `reporter.ts` 中做了细粒度的容错包裹：
当探测到“无有效商家报价”或“市场代码不支持”时，机器人不会崩溃，而是会将这一状态格式化为温和的提示信息附在报告尾部，确保其他正常币种的聚合汇率播报能够毫发无损地准时送达。

---

## 部署与上手

借助 Wrangler 工具链，整个部署流程只需不到两分钟：

```bash
# 1. 克隆代码并安装依赖
git clone https://github.com/Inklazy/Okx-Monitoring.git
cd Okx-Monitoring
npm install

# 2. 创建用于存储监控任务的 Cloudflare KV
npx wrangler kv namespace create MONITORS
# 将生成的 ID 填入 wrangler.toml

# 3. 注入 Telegram 与调试 Secrets
npx wrangler secret put TELEGRAM_BOT_TOKEN
npx wrangler secret put TELEGRAM_WEBHOOK_SECRET
npx wrangler secret put ADMIN_USER_IDS

# 4. 发布到 Cloudflare 全球边缘节点
npm run deploy

# 5. 注册 Webhook 路由
curl "https://api.telegram.org/bot<TOKEN>/setWebhook" \
  -d "url=https://<your-worker-subdomain>.workers.dev/telegram/webhook" \
  -d "secret_token=<YOUR_SECRET>"
```

部署完成后，在 Telegram 中向你的机器人发送一条 `/quote CNY USDT`，就能立即收到毫秒级返回的最优商户汇率排行。

---

## 总结

利用 Cloudflare Workers 的免费额度（每天 10 万次请求 + 免费 Cron Triggers + 免费 KV 读写），搭建这样一个私人量化监控机器人完全不需要任何硬件成本，更不需要担心服务器宕机、断网或磁盘被日志写爆的问题。

这也是现代 Serverless 开发非常迷人的地方：**写完核心逻辑，直接推向边缘，然后彻底忘掉运维**。
