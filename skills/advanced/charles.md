---
name: charles
description: Charles 抓 HTTPS 包与弱网模拟（Manual：用户要求"抓包"、"弱网"、"mock 接口"时引用）
activation: manual
---

# Purpose

如何用 Charles 在 beta/debug 包抓 HTTPS 请求，以及弱网模拟、请求重写 / map local 来 mock 接口。

# MUST

1. **前置条件**：必须用 **beta 包**（`usesCleartextTraffic=true` 且 `network_security_config` 对 debug 放开用户证书信任），**official release 包严禁信任用户证书**（会被商店扫描）。
2. **安装 Charles 根证书**：
   - 手机 WiFi 代理设为 Charles 机器 IP + 8888
   - 手机浏览器访问 `chls.pro/ssl` 下载安装证书，命名 "Charles"
   - Android 7+ 需要额外 `network_security_config` 信任用户证书（beta 包已配置；官方包 **不要** 配置）
3. **抓包过滤**：
   - Charles → Proxy → Recording Settings → Include 只加你要的域名：
     - `*.sharexm.cn`、`*.yofolive.com`（业务服务器）
     - `*.weixin.qq.com`（微信回调）
   - 屏蔽 `*.googleapis.com` / `*.firebase` 这些噪音
4. **SSL Proxying 开启**：
   - Charles → Proxy → SSL Proxying Settings → Enable SSL Proxying → Include `*:443`
   - 若出现 "Certificate Unknown" / "Client SSL handshake rejected"：说明 OkHttp 启用了 CertificatePinner，beta 包先临时关闭（见 app/build.gradle OkHttp 配置）

# MUST NOT

1. 不要给 official release 包添加 `network_security_config` 的 `<trust-anchors><certificates src="user"/></trust-anchors>`（严重安全红线）
2. 不要在 Charles 抓到 Token 后明文截图发群（脱敏后再发，或仅发 request/response body 不含 Authorization）
3. 不要用 Charles 的 "Breakpoints" 在 release 构建改请求包体（会触发签名校验失败 / 加密包体不可读）

# SHOULD

1. **弱网模拟**（验证 App 在 2G/3G/丢包场景行为）：
   - Charles → Proxy → Throttle Settings → Enable Throttling
   - 预设：Slow 3G（1.6 Mbps down / 768 Kbps up / 400 ms latency）
   - 可自定义 5% 丢包率测试 Socket 重连逻辑
2. **Map Local（mock 接口）**：
   - 例：要 mock `/chat/api/messages` 返回体
   - Charles → Tools → Map Local → Add:
     - Host: `api.test.sharexm.cn`
     - Path: `/chat/api/messages`
     - Local path: `./mock/messages_200.json`
   - 好处：后端联调前 UI 能先把 200/空/异常 三种路径都跑通
3. **Rewrite（改请求参数）**：
   - 例：分页 page=1 改 page=999，测试空列表 UI
   - Charles → Tools → Rewrite → Enable → Add Rule → Replace Query Parameter
4. **Repeat / Advanced Repeat（压测）**：右键请求 Repeat 100 次，看接口限流 & 并发下应用是否异常

# Decision

| 场景 | 功能 |
|---|---|
| 看接口参数/响应是否正确 | SSL Proxying + Include 目标域名 |
| 模拟加载慢/丢包/断网 | Throttle Settings + 自定义丢包 |
| 后端没写完先测 UI | Map Local → 返回本地 JSON |
| 测边界条件（空/超长/非法数据） | Rewrite 改响应码 / body / 字段值 |
| 并发 & 限流 | Advanced Repeat n 次 / 指定并发数 |
