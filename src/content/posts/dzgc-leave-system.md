---
title: 用反向代理和本地快照“克隆”一套请假系统：从静态备份到 VPS 完整闭环
published: 2026-09-10
description: 记录如何通过纯 Node.js 原生 HTTP 代理、前端资源重写和本地 JSON 记录持久化，打造一个与学校原版前端无缝对接、支持离线/免原站提交的请假系统镜像。
tags: [Node.js, Docker, Caddy, 逆向分析]
category: 开发记录
draft: false
---

最近抽空折腾了一个挺有意思的小项目——**DZGC-Leave-System**。

起因其实很现实：学校现有的请假系统基于一套通用的微服务/企业移动端 Webview 架构，偶尔会遇到服务器抽风、网络卡顿，或者在某些特殊测试/留档场景下，原站记录无法按需灵活查看和调取。最开始我只是想做一份离线的高保真 HTML 静态快照以备不时之需，但后来发现纯静态页面无法支持动态登录、表单提交与记录跳转。

于是索性顺藤摸瓜，用纯 Node.js 撸了一套带**动态反向代理 + 响应体 URL 重定向重写 + 本地审批流引擎**的完整中间件，并打包成了开箱即用的 Docker 容器部署在 VPS 上。

项目开源在 GitHub：[DZGC-Leave-System](https://github.com/Inklazy/DZGC-Leave-System)。

---

## 核心要解决的矛盾：原站依赖 vs 本地闭环

通常克隆一个 Webview 页面有两种极端路线：

1. **纯前端 Mock**：把 CSS/JS 和 HTML 扒下来，写死假数据。这种方式的致命缺点是原站的 JS 逻辑（如复杂的表单校验、日期联动选择器、UniApp 运行时）非常庞大，手动替换接口和动态逻辑极易报错，且无法使用真实的学号验证码登录。
2. **纯反向代理**：直接用 Nginx 反代原站。但这会导致数据完全依赖原站服务器，如果在本地或者镜像站提交了新的测试请假，依然会打入原站数据库，失去“本地独立留档”的意义。

为了平衡这两点，我设计的系统架构如下：

```text
               +-------------------------------------------------------------+
               |                       VPS (Docker)                          |
               |                                                             |
客户端 (手机) -->| Caddy (:80) -> Node.js (:8123)                              |
               |                   |                                         |
               |                   |-- [只读/认证请求] -------> 学校原站 API   |
               |                   |   (登录、验证码、基础字典)                  |
               |                   |                                         |
               |                   |-- [本地拦截与合成]                                |
               |                   |   POST /submitForm   -> 写入 applications.json
               |                   |   POST /getMyApply   -> 混合原站记录与本地提交
               |                   |   GET  /flowRecord   -> 自动计算审批流程节点
               |                   |   静态资源            -> 本地落盘静态资源池
               +-------------------------------------------------------------+
```

系统由 Node.js 运行时担任“智能中继”：
- **认证与底表走原站**：登录校验、学号字典和初始用户画像直接安全代理到原站，省去模拟复杂认证协议的成本；
- **写操作与详情拦截在本地**：一旦用户在前端点击“提交请假”，中间件拦截该请求，自动分配本地全局唯一流水号，并将数据结构化落盘到 VPS 挂载的 `/app/data/applications.json` 中；
- **记录无缝混入**：当页面请求“我的申请列表”时，服务端把本地新增的请假单与原站已有的审批历史智能合并、按时间戳倒序排列后返回给前端。

---

## 关键技术实现与踩坑

### 1. 零外部 npm 依赖的高性能流式代理

这个项目在 Docker 镜像构建上非常轻量（基于 `node:24-bookworm-slim`，无任何庞大第三方库）。服务核心使用 Node.js 原生的 `http` 与 `https` 模块实现反向代理。

在处理微信 Webview 网页时，一个棘手的问题是**静态资源后缀名与硬编码域名**。原站前端页面写死了 `http://esp.qmxy.com` 的基础绝对路径，且某些离线下载的打包脚本带了特殊的 `.js.下载` 后缀。

服务层对响应流做了透明的编码识别与文本替换：

```javascript
function rewriteTextForLocalOrigin(text, req) {
  const localOrigin = `http://${req.headers.host || `127.0.0.1:${port}`}`;
  return normalizeSavedScriptUrls(text)
    .replace(/(pages-tool-approvalDetailPage-approvalDetailPage\.[\w-]+\.js)(?!\?local-detail-v=)/gu, `$1?local-detail-v=${detailBundleVersion}`)
    .replaceAll('http://esp.qmxy.com', localOrigin)
    .replaceAll('https://esp.qmxy.com', localOrigin);
}
```

凡是原站吐出的 HTML、JS 或 JSON 文本，其中的绝对 API 域名都会被动态修正为客户端当前访问的 Host，从而确保所有的后续 XHR 请求、路由跳转全部紧锁在本镜像站内部。

### 2. 本地审批流自动合成

原站使用的是基于 Camunda / Activity 改良的审批工作流引擎，在查看请假详情页时，前端会连续向后端发起三个查询：
1. `/api-general/ScBusinessFormSubmit/querySubmitInfo`（表单详情）
2. `/api-general/workflow/flowRecord`（流转节点审批记录）
3. `/api-general/workflow/app/status`（当前流程状态）

对于本地提交的假单，原站数据库并不存在对应记录。如果请求直接穿透到原站，前端会直接崩溃并弹出“流程不存在”。

为此，我根据原站返回的合法 JSON 结构，在 `scripts/backend-records.mjs` 中实现了一个微型审批流模拟器：
- 提取用户提交的出发时间与返校时间；
- 结合原站接口已捕获的班主任、辅导员姓名上下文；
- 按照学校固有的审批链路，自动在本地推导生成“辅导员审批通过”、“学工处备案”等标准流程节点及合规的时间戳结构。

这样一来，前端原版 UniApp 页面在渲染请假详情时完全无感知，绿色审批章、时间轴节点全部能以原汁原味的效果正常展示。

### 3. 数据持久化与容器化权限

在编写 Dockerfile 和部署编排时，数据安全性是首要考虑点。

很多初学者容易把 JSON 数据直接放在容器根目录下，容器一更新或者 `docker compose down`，数据就全部丢失了。

在这个项目中，我通过目录分层将运行时与状态彻底剥离：
- 容器以非 root 用户 `node (uid 1000)` 运行；
- 宿主机创建独立目录 `/opt/leave-system-data`，并通过 Docker Volume 挂载至 `/app/data`；
- 构建脚本支持双模式：既支持 GitHub Actions 自动推送到 GHCR，也支持直接在 VPS 本地通过 `compose.vps-build.yaml` 执行无依赖的本地构建。

---

## 部署上线

生产环境我们搭配轻量现代的 Caddy 作为前端反向代理，只需两行配置：

```text
:80 {
    reverse_proxy 127.0.0.1:8123
}
```

拉起容器：

```bash
sudo docker compose -f compose.yaml -f compose.vps-build.yaml build --pull
sudo docker compose -f compose.yaml -f compose.vps-build.yaml up -d --no-build
```

容器内部还配置了基于 Node 原生 `fetch` 的健康检查：

```dockerfile
HEALTHCHECK --interval=30s --timeout=5s --start-period=10s --retries=3 \
  CMD node -e "fetch('http://127.0.0.1:8123/index.html').then((r) => process.exit(r.ok ? 0 : 1)).catch(() => process.exit(1))"
```

只要服务出现死锁或崩溃，Docker daemon 就会立即捕获并执行预设的重启策略。

---

## 总结

这个项目虽小，但在解决实际痛点的过程中，把前端 Webview 调试、流式 HTTP 代理重写、工作流协议逆向和 Docker 最小化生产部署都串联了一遍。使用 AI（Vibe Coding）协同开发的过程中，最爽快的就是让 AI 快速推演原站复杂的 JSON 结构差异，迅速补全 TypeScript/ESM 的字段映射。

有类似内网系统备份或离线代理需求的同学，也可以参考这种“只读走原站、写操作走本地分流”的设计模式。

