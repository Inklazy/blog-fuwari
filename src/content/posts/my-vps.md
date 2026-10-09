---
title: 我的自建服务
published: 2025-12-31
description: '整理了部署在小鸡上的一些服务。'
image: '/images/posts/my-vps/image.png'
tags: [工具, VPS]
category: 工具
draft: false
---

## 🛠️ 服务器基础配置

:::note[硬件与环境]
* **CPU**: AMD Ryzen 9 9950X 16-Core Processor (x8) 
* **内存**: 4GB DDR5 6400mhz
* **OS**: **Debian 13**

目前这台机器使用了以下核心服务作为底座：
* `Caddy` — 负责反向代理和自动 SSL
* `Astro + Fuwari` — 极速博客驱动

---

## 📦 部署服务清单

### 🔧 网络与工具
- [x] **[Github加速](https://ghproxy.inklazy.com/)**
  > 用于各种 GitHub 地址的加速下载
  ::github{repo="WJQSERVER-STUDIO/ghproxy"}
- [x] **[Zashboard](https://github.com/Zephyruso/zashboard)**
  > 灵动的Mihomo API控制面板
  ::github{repo="Zephyruso/zashboard"}
- [x] **Substore**
  > 订阅管理、聚合，脚本编辑为一体
  ::github{repo="sub-store-org/Sub-Store"}

### 📊 监控与统计
- [x] **Wallos**
  > 用于统计每月服务器各项开支
  ::github{repo="ellite/Wallos"}
- [x] **[komari](https://komari.inklazy.com/)**
  > 简单好用的探针，实时掌握机器动态
  ::github{repo="komari-monitor/komari"}

### 📩 通讯与远程
- [x] **mail2telegram**
  > 邮件转发至 TG，添加过滤黑白名单、群内管理，奈飞合租必备
  ::github{repo="Heavrnl/Mail2Telegram"}
- [x] **rustdesk**
  > 开源远程控制，支持自建服务器
  ::github{repo="rustdesk/rustdesk"}
- [x] **nexus terminal**
  > 一款现代化、功能丰富的 Web SSH / RDP / VNC 客户端
  ::github{repo="Heavrnl/nexus-terminal"}

---

> [!IMPORTANT]
> 以上所有服务均通过 Caddy 进行反代，确保全站 HTTPS 安全。