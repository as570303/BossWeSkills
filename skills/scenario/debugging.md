---
name: debugging
description: 调试与日志工具链知识（Auto：用户提到调试、打日志、抓包时自动触发）
activation: auto
triggers:
  - "调试"
  - "日志"
  - "抓包"
  - "debug"
  - "log"
  - "Charles"
---

# Purpose

本项目使用的日志、调试、抓包工具组合使用方式与约定。

# MUST

1. **日志分级**：
   - Release：仅 Error / Warn（Logan / xlog 加密写文件，上传 Bugly/Firebase）
   - Debug：Verbose/Debug/Info 全开（logger + xlog + Logcat）
2. **日志加密解密**：
   - 加密：release 走 `xm_log` SDK / Logan / xlog（看模块初始化）
   - 解密：`log_decrypt/` 目录工具
     - `python log_decrypt/decrypt_log_file.py <log_file>`
     - `java -jar log_decrypt/decrypt_new.jar <log_file>`
3. **网络抓包**（仅 beta/debug 包，`usesCleartextTraffic=true`）：
   - Charles 配置：Wifi 代理 → 安装 Charles 证书 → user trusted credentials（Android 7+ 不默认信任用户证书，需 network_security_config 放 debug 版本）
   - 若抓不到 HTTPS：OkHttp `CertificatePinner` 是否开启了证书锁定（release 才开，debug 应关闭）
4. **断点调试**：
   - Kotlin 协程断点：Use "Kotlin Coroutine Debugger" Agent（Android Studio 设置中启用）
   - 多进程：SocketService / PushService 运行在独立进程，Attach Debugger 要选对应进程

# MUST NOT

1. 不要在 release 构建保留 `Log.d/v`、`System.out.println`（R8 会移除但仍建议统一用 Timber/Logger 带 tag 封装）
2. 不要把 Access Token / 支付密码 / 用户手机号明文打到日志（脱敏 `***` 或只打 hash 前缀）
3. 不要在 Jenkins / CI 机器上抓包产生 Charles SSL 配置（会污染本地 keystore）

# SHOULD

1. **Stetho 可选**：本地调试可临时引入 Stetho 看 DB / 网络 / SP
2. **Flipper 可选**：Facebook Flipper 看 Layout / 网络 / SharedPreferences（需引入 Flipper SDK，当前未接入）
3. **日志 Tag 约定**：
   - 推荐：`XM_<模块>`，如 `XM_Chat`、`XM_RTC`、`XM_HTTP`
   - 过滤：Logcat Filter `package:mine` + `tag:XM_`
4. **性能埋点**：`buildTrace.gradle` 已配置，构建过程看耗时；运行时用 `AppInitManager` 每步 Init 打耗时

# Decision

| 需求 | 工具 |
|---|---|
| 看 release 崩溃堆栈 | Firebase Crashlytics → Bugly → 本地 traces.txt（ANR） |
| 看加密聊天日志解密 | log_decrypt/decrypt_log_file.py |
| HTTPS 抓包请求/响应 | Charles + debug network_security_config |
| 查看本地 Room/SQLite 表 | Stetho / Room Debug DB / DB Browser（拷出 db 文件） |
| 看主线程调用耗时 | Android Studio Profiler → CPU → Java/Kotlin method recording |
| 分析 native so 崩溃 | addr2line / ndk-stack 对应 NDK 版本 + mapping / Bugly native 符号表 |
