---
title: 告别弹窗点按与 403 拦截：给 GiWiFi 校园网写一个全平台自动登录助手
published: 2026-09-13
description: 记录逆向分析国创校园网 (GiWiFi) Portal 认证协议的全过程：拆解其动态 IV 与 AES-128-CBC 零填充算法，并针对 Android 5G 流量漂移与 Windows 多网卡冲突，打造极简的原生 Android App 与后台静默 Windows 工具。
tags: [Android, Java, Python, 逆向分析, 校园网, 自动化]
category: 开发记录
draft: false
---

很多在校大学生都对校园网的 Web 认证（Portal 登录）深恶痛绝：
- 每次连上 Wi-Fi 都要弹出一个卡顿的浏览器页面；
- 手机锁屏一会儿或者在校园里走动切换了 AP，网络就悄然断开；
- 如果同时开着 5G 蜂窝数据，手机经常因为 Wi-Fi “未连接互联网”而直接自动走移动流量，导致连网关登录页面都打不开；
- 电脑开机第一件事总是要打开浏览器输入账号密码，而且偶尔还会报“并发设备冲突被挤下线”。

为了彻底终结这套反人类的操作，我最近抽时间把学校通用的**国创校园网（GiWiFi / GPortal，网关常见为 `10.101.0.1`）**的认证接口与前端加密逻辑完整逆向了一遍，并做成了一套覆盖 Windows 与 Android 的自动化全平台工具：**GiWiFi-AutoLogin**。

项目代码已开源：[GitHub: GiWiFi-AutoLogin](https://github.com/Inklazy/GiWiFi-AutoLogin)。


---

## 逆向第一步：抓包与协议生命周期还原

GiWiFi 的网关防线比很多老旧校园网要稍微严格一点点。我们抓取认证链路时，发现了几个关键的机制：

1. **直接访问 403 机制**：
   如果浏览器直接打开 `http://10.101.0.1/gportal/web/login`，网关的 Nginx 会直接拒接并返回 `403 Forbidden`。它必须由未认证状态下的设备发起任意 HTTP 80 端口请求（例如请求小爱/小米的连网检测地址 `http://connect.rom.miui.com/generate_204`），被网关劫持后 302 重定向至带有 `wlanuserip=<STA_IP>&wlanacname=<AC_NAME>` 两个关键参数的合法链接才能拿到登录页。
2. **表单动态签名与下发**：
   重定向拿到的 HTML 中，`<form id="loginForm">` 包含了网关每次动态生成的几个防篡改字段：
   - `sign`：网关生成的防伪 Base64 数字签名；
   - `iv`：本次握手专用的 AES 加密初始向量（16 位 Hex 字符串，如 `074acd1000a406a1`）；
   - `nas_name` / `sta_ip`：当前设备内网 IP 与认证控制器 AC 名称。
3. **前端加密算法拆解**：
   审查网页的 `login.js` 代码，可以看到表单提交并不是明文传输：

   ```javascript
   function cryptoEncode(data, iv) {
       var key = CryptoJS.enc.Utf8.parse("1234567887654321");
       var ivv = CryptoJS.enc.Utf8.parse(iv);
       var encrypted = CryptoJS.AES.encrypt(data, key, { 
           iv: ivv, 
           mode: CryptoJS.mode.CBC, 
           padding: CryptoJS.pad.ZeroPadding 
       });
       return {'data': encrypted.toString(), 'iv': iv};
   }
   ```

   - **算法**：AES-128-CBC
   - **密钥 (Key)**：写死在 JS 里的固定密钥 `1234567887654321`
   - **向量 (IV)**：每次从 HTML 中动态提取的 16 字节 IV
   - **填充模式 (Padding)**：**ZeroPadding**（零填充，以 `\x00` 补齐至 16 字节整数倍，若刚好是 16 字节整数倍则不追加补齐块）
   - **入参内容**：整个登录表单的标准 URL 编码序列化字符串（等同于 jQuery `$(form).serialize()`）

4. **冲突自动抢占**：
   若账号已在其他终端登录，提交认证接口会返回 `status: 0` 且 `resultCode: "124"`，同时在返回的 `resultData` 中提供一条强制下线旧终端的链接。向该链接发一次 POST 即可秒级踢掉旧会话并完成本设备抢占。

---

## 痛点攻坚：双网卡环境与 5G 流量漂移

搞懂了加解密算法后，在电脑或手机上写个发包脚本其实并不难。真正拉开体验差距的，是对极端网络环境的处理。

### 1. Android 端的“网络漂移”与原生硬绑定
现代 Android 系统（特别是 Android 10+ 配合各类 5G 手机）都有非常激进的网络连通性检测（Captive Portal Detection）。

当手机连上 GiWiFi 但尚未登录时，系统发现 Wi-Fi 无法访问外网，便会默默开启“智能双通道加速”，**将后续所有新发起的 HTTP 请求强行分流走 4G/5G 蜂窝数据**。

这就造成了一个死循环：你的脚本在手机上拼命发请求想访问 `10.101.0.1`，但数据包全打到了移动/联通的公网基站上，结果只能收获无休止的“网络不可达”或超时超时超时。

在 Android 原生客户端中，我们利用系统底层 API 实现了进程级 Socket 硬绑定：

```java
ConnectivityManager cm = (ConnectivityManager) getSystemService(Context.CONNECTIVITY_SERVICE);
for (Network network : cm.getAllNetworks()) {
    NetworkCapabilities caps = cm.getNetworkCapabilities(network);
    if (caps != null && caps.hasTransport(NetworkCapabilities.TRANSPORT_WIFI)) {
        // 强行把当前进程所有流量锁在 Wi-Fi 网卡上
        cm.bindProcessToNetwork(network);
        break;
    }
}
```

这样无论系统当前如何判定外网连通性，App 发出的探测包和登录请求都 100% 走 Wi-Fi 通道，在 1 秒内瞬间完成探测并静默认证。

### 2. 拒绝臃肿：18 KB 的极简原生 Android App
很多同学开发类似工具喜欢套用 WebView 或混合框架（Flutter、React Native、UniApp 等），打包出来动辄三五十兆，冷启动要好几秒。

既然这个工具的核心诉求是“无感极速”，我们最终采用最纯粹的 **Java 17 + 原生 Android SDK + Material 3 基础组件** 构建。

没有引入任何冗余的第三方库（OkHttp、Retrofit 全都不要，直接用原生 `HttpURLConnection` 和标准库加密）。最终编译出的 Release APK **体积只有区区 18 KB**，冷启动耗时仅仅 140ms，打开即闪电认证完毕。

### 3. Windows 端：开机静默后台运行与多网卡适配
在 Windows 电脑上，很多宿舍同学既插了网线又连着 Wi-Fi，或者电脑上常驻了各种开发代理。
若直接发起 Socket，请求往往会走优先级的有线网卡，导致网关报 `CHALLENGE_ERR_DENY`。

我们在 Windows 脚本中加入了自动枚举系统 WLAN 适配器 IP 的逻辑，并将核心业务打包成了单文件免安装的绿色 EXE（使用 PyInstaller 打包 Tkinter 图形配置向导）。

初次配置保存密码后，程序会自动利用 Windows Task Scheduler（任务计划程序）注册一个开机自启任务：
```powershell
GiWiFi_AutoLogin.exe --silent
```
日常开机时，电脑后台静默完成认证，**没有任何黑色的 CMD 命令行窗口闪烁**，真正做到润物细无声。

---

## 总结与反思

这个小项目虽然不算庞大，但在协议分析和多端落地时踩遍了校园网环境的各种奇葩坑点。通过 AI 工具（Vibe Coding）协助推导 Python 和 Java 之间的 ZeroPadding 差异、生成干净的标准库加密实现，整个开发过程效率极高。

目前这套工具已经在我和身边很多同学的设备上日常服役，彻底告别了每天被校园网踢下线再手动点按的痛苦。如果你所在的学校也在用类似的 Portal 认证系统，不妨也动手写一个，技术就是用来改善日常生活的！
